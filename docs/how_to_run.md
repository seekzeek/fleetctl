# How to run

Building the platform end to end: **bake the images, provision AWS, build the
cluster, then hand it to git.** Six steps, each in one repository.

The commands are the short part. The value is in the ordering constraints, which
are called out where they apply — most of them fail quietly and much later than
the mistake that caused them.

---

## Prerequisites

| | |
|---|---|
| AWS account | one, with permission to create IAM roles and an OIDC provider |
| Terraform | `~> 1.15` — the S3 backend uses native locking (`use_lockfile`), so no DynamoDB table |
| Packer | plus a Python venv built from Kubespray's own `requirements.txt` |
| Ansible | installed into that venv, not system-wide |
| Session Manager plugin | the only route to a node — there is no bastion |
| Cloudflare account | a zone, and an API token with `Zone:DNS:Edit` and `Zone:Zone:Read` |

**The SSM tunnel is transport, not authentication.** `AWS-StartSSHSession`
forwards TCP to port 22, and sshd still wants a credential — so `ssh_config`'s
ProxyCommand pushes an EC2 Instance Connect key with a **60-second TTL** before
opening the tunnel, and nothing persists in `authorized_keys`. `make setup`
generates that key at `~/.ssh/fleetctl-eic`. It is an operator credential, not
cluster state: it lives in `$HOME` and neither repository stores it. The path
must match `eic_key_path` in Terraform's `kubespray-inventory` module, or the
ProxyCommand fails with nothing pointing at the mismatch.

`packer`, `terraform` and `ansible` each ship `make check-static`, which runs
the checks that need no AWS credentials. Run it before anything else. The gitops
and charts repositories have no Makefile; their checks run in CI.

---

## 1 · Bake the node images — `fleetctl-packer`

```
make setup          # venv from kubespray's requirements.txt
make build          # both roles in parallel; depends on manifest + ecr-provider
```

`make manifest` asks Kubespray what each role's image must contain and writes it
out; `make ecr-provider` stages the out-of-tree ECR credential provider and
verifies its checksum locally, so a corrupt cache fails here rather than minutes
into a bake on a remote builder. `make build` runs both, so it is the only
target usually invoked.

Builds publish `/fleetctl/ami/<role>/versions/<k8s>-<timestamp>` and a `latest`
pointer. **Clusters pin a version.** A guard refuses `latest`, `stable`, or a
bare version number — a bare `1.35.4` is not unique, because rebuilding the same
Kubernetes release for a base-image CVE produces a second image with the same
version.

Two more worth knowing: `make prune ROLE=<role>` deregisters old AMIs and deletes
their snapshots, and `make check-tag` confirms the Kubespray submodule matches
the pin in `fleetctl-ansible`.

---

## 2 · Apply the bootstrap layers — `fleetctl-terraform`

Once per account, by hand. `state-backend` is first because everything else
stores state in it; the rest are independent of each other.

```
bootstrap/state-backend     # S3 + KMS state bucket. Everything else needs it.
bootstrap/node-ami          # seeds the AMI pointers until Packer takes over
bootstrap/ci-oidc           # GitHub OIDC provider, four scoped roles,
                            # and both permission boundary policies
bootstrap/artifact-store    # durable home for the inventory handoff
bootstrap/oidc-store        # public prefix for each cluster's OIDC documents
bootstrap/chart-registry    # ECR settings for the Helm charts
```

Each is `terraform init && terraform apply` in its own directory. After
`ci-oidc`, run `make verify-ci-oidc` — it proves the four roles are assumable
and correctly scoped **before** the first workflow run depends on them.

---

## 3 · Apply the cluster layers — `fleetctl-terraform`

```
make plan-all  CLUSTER=prod-use1-01
make apply-all CLUSTER=prod-use1-01
```

That walks `010-network` → `015-identity` → `020-control-plane` →
`030-workers` → `040-inventory`, stopping on the first failure.

> **`018-data` is not in that list, and is applied on its own:**
> `terraform -chdir=clusters/<cluster>/018-data apply`.
>
> It holds RDS and the credentials that outlive the cluster. No other layer
> reads its state — the low number encodes **lifecycle, not dependency**: like
> `010-network`, it survives a cluster teardown, which is why it sits below the
> layers that do not. It is excluded from `apply-all` so a routine apply can
> never touch the database.
>
> Its outputs are SSM parameter names, and their consumer is External Secrets
> inside the cluster, so the real deadline is **before `make argocd`** — Keycloak
> and oauth2-proxy sync last and read those parameters.

`make inventory CLUSTER=<c>` re-renders and republishes the handoff artifact on
its own; run it after any node replacement.

Also available: `make fmt-check`, `make validate`, `make conventions`,
`make policy-test` (rego unit tests, no AWS), `make lint`, `make policy`
(conftest against real rendered plans, needs AWS). `make check-static` runs
everything that works offline.

---

## 4 · Publish the service charts — `fleetctl-charts`

```
scripts/release.sh
```

Packages every non-library chart whose version is not already published and
pushes it to ECR as OCI. It is idempotent because **ECR tags are immutable**, so
re-pushing an existing version fails rather than silently replacing it — the
script asks the registry first and skips. `charts/common` is vendored into each
package at build time and is never published on its own, because a library chart
nothing pulls is a repository nothing reads.

Like `018-data`, the deadline is **before `make argocd`**: Argo CD would
otherwise sync an Application whose `targetRevision` does not resolve.

---

## 5 · Build the cluster — `fleetctl-ansible`

```
git submodule update --init
make setup                         # venv, galaxy collections, and the EIC key
make sync       CLUSTER=prod-use1-01
make ping       CLUSTER=prod-use1-01
make cluster    CLUSTER=prod-use1-01
make join-token CLUSTER=prod-use1-01
```

`make sync` pulls the inventory artifact from S3 with `--exact-timestamps`, then
verifies it against what EC2 actually reports. The flag is load-bearing: a plain
`aws s3 sync` skips a download when the local file is the same size and no
older, and an inventory whose hostnames changed while its byte count did not is
exactly that case.

**`make ping` fails for the first couple of minutes after nodes launch**, while
cloud-init finishes. The two failures look different and mean different things:
`Connection timed out during banner exchange` is cloud-init still running — wait
and retry. `Permission denied (publickey)` is a real problem with the EIC key.
`aws ssm send-command` works before SSH does, and is the better way to look at
early boot.

`make cluster` runs preflight → Kubespray → `etcd-guard` → `aws-ccm` →
`kubelet-certs` → `irsa`. Two things to know before the first run:

- **Preflight asserts `/var/lib/etcd` is a real mount**, not a directory on the
  root disk. If it is not, stop — do not run Kubespray.
- The run ends at `aws-ccm`, which cannot schedule on a tainted, workerless
  control plane. That is why the worker layer is applied back in step 3.

**`make join-token` publishes the first token and installs the rotator timer.**
It is a separate step, not part of `make cluster`. Until it runs there is no
token in SSM, so no *future* worker can self-join — which matters immediately,
for the reason below.

> **The first worker set is disposable, by design.** `030-workers` launches
> instances before `make cluster` has run, so their userdata looks for a token
> that does not exist yet, abandons, and cloud-init never retries. That is the
> userdata failing closed exactly as intended: an instance that did not join is
> not a node. The symptom is `cloud-init status: error` and
> `CompleteLifecycleAction … No active Lifecycle Action found`.
>
> After `make join-token`, terminate that first set with
> `--no-should-decrement-desired-capacity` — decrementing fails when the ASG is
> already at `MinSize`, and the error does not mention `MinSize`. Each
> replacement is Ready 65 seconds after its instance launches.

There is no separate join step for that first set: `cluster.yml` inits the
control plane **and joins every worker in the inventory**. `make scale` exists
for workers added later, and after the recycle above nothing needs it —
replacements self-join with no Ansible in the path at all.

Worth confirming before trusting anything downstream: `make kubeconfig` then
`make tunnel`, and check that every node has a `providerID` and a zone label. An
empty `providerID` means the cloud-controller-manager never initialised the
node, and EBS attach and load-balancer registration will fail later with errors
that mention neither.

---

## 6 · Hand it to git — `fleetctl-ansible`

Argo CD reads `fleetctl-gitops` over SSH, so the deploy key has to be in SSM
before it starts — otherwise the failure arrives as a sync error that names
nothing:

```
aws ssm put-parameter --type SecureString \
  --name /fleetctl/prod-use1-01/gitops/ssh-key --value file://deploy_key

make argocd CLUSTER=prod-use1-01
```

Installs Argo CD once and applies the root Application. **Argo CD needs a worker
to schedule on**, so this comes after the workers are actually joined — the
control plane is tainted and cannot take them. The install is guarded on the root
Application already existing, so running it twice is a no-op rather than a revert
to bootstrap values.

From here the cluster is driven by pull requests to `fleetctl-gitops`.

---

## Reaching the cluster

```
make kubeconfig CLUSTER=<c>    # fetch admin.conf, pointed at the local tunnel
make tunnel     CLUSTER=<c>    # forward the apiserver over SSM; blocks
make argocd-ui  CLUSTER=<c>    # needs the tunnel
make keycloak-ui CLUSTER=<c>
make argocd-password CLUSTER=<c>
make keycloak-password CLUSTER=<c>
```

There is no bastion: every one of these runs over SSM. `make tunnel` blocks, so
it wants its own terminal, and the UI targets assume it is already running.

---

## Day-2 targets worth knowing

Every one of these takes `CLUSTER=`.

| | |
|---|---|
| `make join-failures` | why recent workers failed to self-join — reads the shipped diagnostics, not the dead instance |
| `make join-token` | publish a fresh join token immediately rather than waiting for the timer |
| `make verify-ami-pin` | which AMI each layer pins, against what its nodes are actually running |
| `make verify-alerts` | proves this cluster's alarms can reach a human |
| `make delete-stale-node NODE=` | remove a Node object whose instance is already gone |
| `make upgrade` | rolling version bump from `cluster-vars` |
| `make reset` | wipe Kubernetes off the nodes and keep the machines; types the cluster name to confirm |

---

## Tearing down

`make destroy-all CLUSTER=<c>` walks the layers in reverse and requires typing
the cluster name. **It will fail at `020-control-plane`, deliberately** — the
etcd volume carries `prevent_destroy`. Removing it is a conscious edit, not an
accident of a destroy command.

`018-data` survives by design — it is not in the layer list, so a teardown never
reaches the database or any identity configured in Keycloak. `010-network` is in
the list, and survives for a different reason: the loop runs under `set -e` and
halts at the control plane, so it never gets that far. Continuing past that point
is a deliberate act, and `docs/teardown.md` in the Terraform repo covers it.

---

## Repository READMEs

Each repository documents its own layer in more depth:
[packer](https://github.com/seekzeek/fleetctl-packer) ·
[terraform](https://github.com/seekzeek/fleetctl-terraform) ·
[ansible](https://github.com/seekzeek/fleetctl-ansible) ·
[gitops](https://github.com/seekzeek/fleetctl-gitops) ·
[charts](https://github.com/seekzeek/fleetctl-charts)

The reasoning behind the design is in [`deep-dives.md`](deep-dives.md).
