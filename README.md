# fleetctl

A self-managed Kubernetes platform on AWS, owned from the machine image up: the
control plane, the identity chain, and the pipeline that delivers workloads onto
it.

It runs services behind Keycloak SSO, delivered by Argo CD, on workers that
scale with load and join themselves, one EC2 Auto Scaling group per
availability zone. Every mechanism a managed control plane normally hides —
certificate rotation, the OIDC trust chain, the cloud-controller-manager, node
join — lives as configuration in these repositories. All of it can be read,
changed, and destroyed and rebuilt on purpose.

**Stack:** `AWS` · `Terraform` · `Packer` · `Ansible/Kubespray` · `Argo CD` ·
`Helm` · `Keycloak` · `oauth2-proxy` · `Cloudflare` · `ingress-nginx` ·
`Prometheus` · `External Secrets` · `OIDC/IRSA`

---

## Key results

|                                              | Result                                                                                                                            |
| -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| **Bootstrap from an empty account**          | Image bake → Terraform → running cluster in ~30 min, fully config-driven. No console steps.                                       |
| **Worker scale-out and replacement**         | 12 min → 65 s. A new worker node joins and is Ready in 65 seconds instead of 12 mins, because it joins itself from a baked image. |
| **Static AWS credentials**                   | Zero, in CI, in the cluster, or on nodes. Everything is OIDC-federated.                                                           |
| **Inbound network paths**                    | Zero. Ingress is outbound-only through a tunnel.                                                                                  |
| **Manual `kubectl` in the delivery path**    | Zero. Platform components and services alike arrive as Argo CD Applications, in ordered sync waves.                               |
| **Credential-rotation failure detection**    | Surfaced ~12 h before any worker could be affected.                                                                               |
| **Stateful layers after deliberate destroy** | VPC · RDS · both AMIs · registry · Keycloak realm all survived.                                                                   |

![kubectl get nodes -o wide](docs/images/nodes-wide.png)

![kubectl get nodes -L topology.kubernetes.io/zone](docs/images/nodes-by-zone.png)

One worker per availability zone, and it stays that way. Each zone has its own
Auto Scaling group, pinned to that zone's subnet with a floor(min_size) of one,
so a replacement always comes back in the zone it left. It cannot land in a
different one and leave the volumes behind it with nowhere to attach.
[Decision 2](docs/deep-dives.md#2--workers-join-themselves).

---

## Architecture

Packer bakes an image, Terraform provisions on it and renders an inventory,
Ansible turns that into a cluster, Argo CD fills the cluster. Each layer
produces exactly one artifact that the next layer consumes, and that is the
entire contract between them. Any layer can be rebuilt, replaced or tested on
its own, because nothing downstream knows how its input was produced.

The boundaries sit where the rate of change does:

- Node images change when Kubernetes changes
- Infrastructure changes when the topology changes
- The cluster changes when either of those does
- Workloads change many times a day

One pipeline across all of them means the slowest thing sets the pace for the
fastest, and a workload deploy carries the risk of a control-plane change.

Each handoff is checked rather than assumed. The inventory syncs with
`--exact-timestamps`, after a plain sync compared size and mtime, found both
unchanged on a regenerated file, and skipped it. Ansible then ran against a
cluster that no longer existed in that shape.

| Layer | Repository                            | Owns                                                          | Produces                                            |
| ----- | ------------------------------------- | ------------------------------------------------------------- | --------------------------------------------------- |
| 1     | `fleetctl-packer`                     | node images, baked from Kubespray's own download manifest     | an AMI, version-pinned in SSM                       |
| 2     | `fleetctl-terraform`                  | AWS: 12 layers, isolated state, two IAM permission boundaries | an inventory artifact in S3                         |
| 3     | `fleetctl-ansible`                    | Kubespray, plus 7 roles for what it does not ship             | a running control plane, and Argo CD installed once |
| 4     | `fleetctl-gitops` + `fleetctl-charts` | everything inside the cluster                                 | running workloads                                   |

![What each layer hands to the next: AMI in SSM, inventory in S3, root Argo CD Application, then charts from ECR](docs/images/what_each_layer_hands_to_next.png)

Twenty Applications, all Synced and Healthy, every chart version pinned
exactly.

![Argo CD, twenty Applications Synced and Healthy](docs/images/argocd-applications.png)

The same view without the browser:

![kubectl get applications -n argocd -o wide](docs/images/argocd-applications-cli.png)

### How it runs

Once it is built, every connection that crosses the VPC boundary was opened from
inside it. `cloudflared` holds the tunnel open outward, the SSM agent dials the
same broker the operator dials, and Argo CD pulls from git. None of the three is
a path in.

![How it runs: prod-use1-01, every connection crossing the VPC boundary opened from inside it](docs/images/how_it_runs.png)

---

## Design invariants

Properties enforced by the system, not left to discipline. Every decision below
follows from one of them.

- **The apiserver has no public endpoint.** Operator access is an SSM tunnel.
- **No SSH, no bastion, no durable key material** anywhere in the account.
- **No static AWS credentials.** Every identity is federated through OIDC.
- **No inbound network path.** Ingress works outbound-only.
- **Nodes join unattended**, with nothing outside the cluster holding
  credentials that could add a node.
- **The control plane is self-managed.** Certificate rotation, the
  cloud-controller-manager and the OIDC trust chain are configuration here.

---

## Design decisions

Key decisions, what each one bought, and what it cost.

### 1 · IRSA for pods, instance profile for the node, OIDC for CI

The simple way to give a pod AWS permissions is to attach them to the node it
runs on. Every pod on that node then inherits them, so a compromised container
in any namespace gets whatever the CSI driver needed.

IRSA scopes credentials to a ServiceAccount instead. The pod-identity-webhook
injects a projected token at pod creation, and each trust policy pins one
`namespace:serviceaccount`, never a wildcard. The apiserver has no public
endpoint and STS verifies tokens over the public internet, so the OIDC discovery
document and JWKS are published to S3 and the IAM provider registered against
that URL. Instance profiles remain only where the caller is not a pod: kubelet
image pulls, and the calls a machine makes before it is a node. CI federates the
same way into four scoped roles, all capped by permission boundaries.

**Result: no long-lived AWS keys anywhere, and a compromised pod reaches only
what its own ServiceAccount is allowed, not everything its node needed.**
_Cost: the issuer string must match byte-for-byte across three repositories, and
a narrowly-public S3 bucket exists to serve it._

### 2 · Workers join themselves

A worker needs no help to become a node. The AMI already carries every binary
and container image, so userdata reads join credentials from SSM with the
instance's own profile and runs `kubeadm join`. The launch hook defaults to
ABANDON, and CONTINUE is signalled only once kubelet is active and a kubeconfig
exists: _proof, not optimism_. An instance that did not join is not a node, so
it is replaced rather than left InService behind a healthy-looking ASG. The
kubelet journal ships off the machine first, because the ASG deletes the
instance the moment that call returns.

That makes scaling a question of when rather than how. Workers are three Auto
Scaling groups, one per availability zone, each pinned to that zone's subnet
with a floor(min_size) of one. Persistent volumes are zonal, and a group free to place
instances anywhere can empty a zone and strand every volume in it. Cluster
Autoscaler owns desired capacity across all three, raising it for pods the
scheduler cannot place and cordoning and draining before it terminates.

**Result: a new worker is Ready in 65 seconds instead of 12 minutes, which is
the difference between capacity arriving before the load and after it.**
_Cost: join logic lives outside Ansible and has to track Kubespray's config._

Three replacements, timed from the moment EC2 reports the instance launched:

|            | boot → join starts | `kubeadm join` | → Ready | **total** |
| ---------- | ------------------ | -------------- | ------- | --------- |
| us-east-1a | 25 s               | 23 s           | 13 s    | **61 s**  |
| us-east-1b | 24 s               | 19 s           | 4 s     | **47 s**  |
| us-east-1c | 22 s               | 17 s           | 5 s     | **44 s**  |

Quoted as 65 s rather than the 51 s mean, to stay above the slowest of the
three. The join is quick because it downloads nothing: the image already
carries every binary and container image the node needs.

[Full timing breakdown](docs/deep-dives.md#2--workers-join-themselves).

```console
$ cat /var/log/fleetctl-join.log          # on a replacement
[fleetctl] join starting 2026-09-19T20:35:04+00:00
[fleetctl] joining api.<cluster>.fleetctl.internal:6443
[fleetctl] node name ip-10-0-24-38.ec2.internal
...
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.
[fleetctl] join complete 2026-09-19T20:35:23+00:00
```

The image is what buys this: every binary and container image the node needs is
already on disk, so the join does no downloading.

### 3 · The join token is rotated on a timer and monitored by its absence

A node joining the cluster needs a credential, and bootstrap tokens expire by
design. Something has to keep a fresh one where a booting instance can read it,
with nobody watching.

A systemd timer on the control plane mints a 24-hour token every six hours, so
four consecutive failures must pass before any worker is affected, and it
refuses to publish an expiry that is not in the future. Refusing to publish a
bad value trades a wrong token for no token, which is safer but just as silent.
So the rotator emits its heartbeat only on the success path, as its last
statement, and the alarm treats missing data as breaching.

**Result: one alarm covers a stopped timer, a failed unit, broken IAM and a dead
host, and none of them can fake a healthy signal.**
_Cost: a rotator broken for more than a day blocks every scale-out, which is what
makes the heartbeat load-bearing rather than decorative._

### 4 · Terraform state split by lifecycle, with the cluster as the unit of blast radius

How state is split decides what a single mistake can reach. A directory per
cluster, no workspaces and no `-var-file` promotion, so no root module can
address another cluster's state. **The layer that runs most often has the least
power:** the inventory layer owns no cloud resources at all, so the thing re-run
after every node change can only rewrite text files.

Upper layers read lower ones for things that stay put, like CIDRs and subnet
IDs. Anything that changes when an instance is replaced, such as a private IP,
is looked up live by tag instead.

**Result: a worker-count typo cannot produce a plan that replaces a subnet, and
the stateful layers outlived a deliberate full-cluster destroy.**
_Cost: twelve state files, and cross-layer reads must stay facts-only, so lower
layers must stay secret-free._

### 5 · Argo CD is installed once, then manages itself

A chicken-and-egg problem: everything in the cluster is delivered by Argo CD
from git, but Argo CD itself has to arrive somehow, and it cannot install
itself. Something outside git has to break the cycle exactly once.

Ansible does that. A minimal Helm install, then a single root Application
pointing at a directory in the gitops repository. That directory is an
app-of-apps: one Application per platform component, and one of those components
is Argo CD. From the first sync onward it reconciles its own manifests from git
alongside everything else, and the Ansible install has no further role. Fifteen
of them sync in ordered waves: cert-manager before anything that requests a
certificate, the pod-identity webhook before anything that needs IRSA, Keycloak
before oauth2-proxy.

The handover works because Argo CD renders charts with `helm template` and
applies the output. It creates no Helm release of its own, so it adopts the
objects the Ansible install already made rather than standing a second copy
beside them. The bootstrap play is skipped when the root Application already
exists, so re-running it changes nothing.

**Result: one imperative step at the very beginning, and everything after it is a
pull request. No manual `kubectl` in the delivery path.**
_Cost: Argo CD's own Application must stay in a late sync wave, or it can restart
the application-controller mid-sync and stall its own reconciliation._

### 6 · Ingress through a Cloudflare Tunnel, so there is no inbound path

The usual arrangement is a public load balancer with a proxy in front of it and
an allowlist keeping everything else out. This inverts it. `cloudflared` runs
in-cluster and dials _out_; requests return over that established connection.
Nothing listens: no public origin to resolve, find or port-scan, and no load
balancer whose AWS hostname is reachable by anyone who looks it up. The IP
allowlist an internet-facing origin would have needed, and the work of keeping
it current as Cloudflare's ranges change, does not exist. Terminating at the
edge also puts rate limiting and DDoS absorption in front of the cluster without
a second component to operate.

It is the right answer for HTTP traffic into a private cluster, and the wrong
one for non-HTTP protocols, for throughput-sensitive paths where a third-party
hop costs real latency, and anywhere traffic must not cross an external edge for
regulatory reasons. The route back to an NLB is written down in order, starting
with restoring TLS on every Ingress: change the network path first and nginx
serves its own default certificate to live traffic. A design with a written exit
is a decision; one without is a trap.

**Result: the attack surface is an outbound connection rather than a listening
endpoint.**
_Cost: every request into the cluster depends on a third party._

### 7 · etcd as a systemd unit, not a static pod

kubeadm's default is to run etcd as a static pod, which puts kubelet in charge
of it. That creates a loop: etcd depends on kubelet, kubelet depends on the
apiserver, and the apiserver depends on etcd. It is invisible while everything
works and unpleasant when it does not, because a kubelet that will not start
takes etcd down with it and you are debugging two things at once.

As a systemd unit the dependency runs one way. `journalctl -u etcd` is an
ordinary workflow rather than `crictl logs` against a container whose kubelet
may be unhealthy, and a restore is stop the unit, swap the data directory, start
it, rather than moving a manifest in and out while the apiserver flaps.

It also makes one guard possible. The volume uses fstab `nofail` so a slow
attach cannot hang boot, which leaves a window where `/var/lib/etcd` is an
ordinary root-disk directory that etcd will start on happily.
`RequiresMountsFor` makes systemd refuse to start until the mount is real. The
kubelet has no equivalent: it will not refuse a static pod over an empty
`hostPath`.

**Result: etcd recovery does not depend on kubelet being healthy, and a missing
data volume stops the service rather than silently writing to the root disk.**
_Cost: etcd is not visible as a pod, and the host owns the binary's lifecycle._

### 8 · The apiserver is addressed by name, never by IP

A Route 53 private-zone record, held in the certificate SANs. The decisive
reason is ordering: a joining worker must resolve the apiserver _before_ it is
in the cluster, when it has no kubelet config, no CoreDNS and no membership.
Cluster DNS cannot be the answer, and `/etc/hosts` only moves the update problem
into the AMI and onto every node. Pinning the name rather than the address also
means a control-plane replacement does not regenerate the apiserver certificate.

**Result: the HA promotion is one record change, repointing it at an NLB alias,
with no certificate change, no worker reconfiguration and no rolling restart. If
every client held an IP, that promotion would be a fleet-wide reconfigure, which
in practice means it never happens.**
_Cost: DNS is load-bearing in the join path. Existing nodes are unaffected, since
kubelet holds its connection, but new ones cannot join while VPC DNS is broken._

### 9 · One secret backend: SSM Parameter Store, delivered by External Secrets

Secrets have to reach pods without anything long-lived sitting in git or baked
into a manifest. External Secrets does the delivery: a custom resource names a
path in the backend, the operator fetches it over IRSA and materialises a native
Kubernetes Secret, and nothing in the repository holds a value.

The backend is Parameter Store rather than Secrets Manager. Everything — the
join token, the Keycloak admin password, database credentials — is a
SecureString under a per-cluster path prefix. Secrets Manager is the usual
choice and adds managed rotation; the KMS envelope is the same, the standard
tier is free, and Parameter Store was already load-bearing because a joining
worker reads its token from there before it is in the cluster.

**Result: one backend, one IAM shape, one place to look during an incident.
Moving to Secrets Manager later is one field in the ESO provider.**
_Cost: no managed rotation. Secrets Manager rotates RDS credentials with a Lambda
AWS provides; here anything that rotates needs a mechanism written for it._

### 10 · Two ECR mechanisms, because there are two different consumers

The usual way to pull from a private registry is an `imagePullSecret` holding a
token that expires in twelve hours, so something has to refresh it in every
namespace that pulls. Neither consumer here does that, and they avoid it
differently.

kubelet gets its credentials from the ECR credential provider: a binary that
calls `ecr:GetAuthorizationToken` with the node's instance profile and returns
the result on stdout, storing nothing. Kubernetes moved it out-of-tree, so it is
a separate binary rather than part of kubelet, and it has to be there before the
first pull. It is baked into the AMI, so every node has it.

Argo CD pulls _charts_, through its own repo-server rather than the CRI, so the
kubelet mechanism does not reach it. External Secrets mints an
`ECRAuthorizationToken` into a Secret it reads, refreshed at ten hours against a
twelve-hour token. Not 11h59m, because a refresh that fails once must still have
room to retry.

**Result: no `imagePullSecret` stored anywhere, and no CronJob writing
credentials into namespaces to keep them fresh.**
_Cost: two mechanisms rather than one, which the two consumers require._

### 11 · Human access is SSM Session Manager only

Reaching a node normally means a bastion in a public subnet, an inbound rule on
22, and a key that lives somewhere. None of that exists here. The SSM agent
dials out and holds the connection; sessions are brokered by IAM on both ends
independently, and every one lands in CloudTrail.

**Result: nothing to patch, nothing to rotate, and an audit trail that survives
the host. A bastion's `sshd` logs live on the machine you may have just lost.**
_Cost: latency on every connection, and a dead tunnel looks exactly like a dead
cluster until you check which one it is._

_Full reasoning, and the failure behind each guard →
[`docs/deep-dives.md`](docs/deep-dives.md)_

---

## Incidents that changed the design

Three failures that ended in a mechanism rather than a one-off fix. None of them
announced themselves where they happened: the symptom surfaced in another
component, or there was no symptom at all.

### Every new worker failed to join, and nothing said why

Every joining node downloads one kubelet config from the cluster, and that
config names the directory holding the client CA. Kubespray sets it to
`/etc/kubernetes/ssl`; `kubeadm join` writes the CA to its own default of
`/etc/kubernetes/pki` and never reads the value in the config it just fetched.
The kubelet could not find its CA, exited, and systemd restarted it until
`kubeadm` timed out four minutes later.

Nothing named the path. The join log ended on "Waiting for a healthy kubelet",
the lifecycle hook abandoned, and the autoscaler replaced the instance holding
the only journal that did name it.

Two fixes came out of it. The path is now pinned in the `JoinConfiguration` that
Terraform renders into userdata, and `check-cert-dir` in CI compares the two
repositories that have to agree on it. Userdata also ships the last fifty lines
of the kubelet journal to CloudWatch before it abandons the lifecycle hook, so
the evidence now outlives the instance that produced it. Failure diagnosis must
survive the thing that failed.

_Lesson: two components can each be correct and still disagree, and neither one
can see the disagreement._

### Self-joined workers served certificates the apiserver refused

`kubectl logs` against any worker failed with `x509: certificate signed by
unknown authority`, and metrics-server would not pass its readiness probe.
Neither symptom mentions kubelet configuration, which is where the fault was.

A kubelet either signs its own serving certificate or asks the cluster CA to
sign one, and `serverTLSBootstrap` decides which. Kubespray writes that setting
into each node's local config as it configures the node, which is a delivery
path workers deliberately do not use: they come up from an AMI and join
themselves, and Kubespray never touches them. So they signed their own
certificates, and the apiserver refused them, because Kubespray starts it with
`--kubelet-certificate-authority` and that makes it strict about what it
accepts.

The fix moves the setting off the per-node path and onto one every node does
read: the cluster-wide `kubelet-config` ConfigMap that each `kubeadm join`
downloads, patched as JSON rather than applied wholesale, which would drop
sibling keys. Finding which file actually counts took reading the running
kubelet's command line, since two configs exist on these nodes and kubeadm's
drop-in points at its own. The obvious alternative,
`--rotate-server-certificates` in the worker's extra args, was rejected. That
flag is deprecated in favour of this exact config field, and would have become a
silent regression the day it was removed.

_Lesson: a setting delivered per node does not reach nodes that bypass the
delivery mechanism._

### Fixing the token rotator removed the only sign it had broken

A systemd timer mints a bootstrap token every six hours and writes it to SSM
with its expiry. That expiry computation ended in a fallback that substituted
the current time whenever the primary expression was rejected, so the published
value was already in the past while rotation reported success on every run.
Every guard downstream read a value that was present, well-formed and wrong. The
fallback is gone, the expiry is asserted to be in the future before publishing,
and the node reads the parameter back and asserts on it again at join time.

That fix created the next failure. A rotator that refuses to publish bad values
leaves the last good token in place, so joins keep working and a stopped rotator
looks exactly like a healthy one. Nothing would surface until that token
expired, up to twenty-four hours later, and then every join would fail at once.

The mitigation is a CloudWatch alarm on a heartbeat the rotator publishes. The
heartbeat carries seconds-until-expiry and is the last statement on the success
path, so an abort anywhere upstream publishes nothing at all. The alarm treats
missing data as breaching, which makes silence the trigger rather than a bad
value having to arrive first. One alarm covers a stopped timer, a failed unit,
broken IAM and a dead host.

_Lesson: alarm on the absence of a success signal, not on the presence of a
failure one. A component that has stopped cannot report that it stopped._

_Root cause, rejected alternatives and verification for each of these →
[`docs/deep-dives.md`](docs/deep-dives.md)_

---

## Future improvements

1. **Replace ingress-nginx.** It reached
   [end of life in March 2026](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/):
   no further releases, bug fixes or security patches. Traffic reaches it
   through the Cloudflare Tunnel rather than a public load balancer, which
   narrows the exposure without removing it. The next migration is from Ingress
   to Gateway API, with Traefik or another Gateway API implementation behind it.
2. **Draining nodes before they are terminated.** Cluster Autoscaler drains
   the nodes it removes, but a health check, a manual termination or a spot
   reclaim takes one without warning and its pods go with it. Closing that means
   holding the instance on an ASG lifecycle hook while the node drains, and
   running a handler to release the hook once it is empty. Both halves have to
   land together: a hook with nothing to release it holds every terminating
   instance for its full timeout, an hour by default.
3. **Automated etcd snapshots.**
4. **Policy-as-code with Kyverno.** Security, compliance and governance rules
   enforced at admission rather than just reviewed at merge.
5. **Cluster API instead of Kubespray.** Adding a control-plane node changes
   etcd quorum, so it stays a manual step here. A controller holding cluster
   state is the answer, and it replaces Kubespray rather than augmenting it.
6. **Distributed tracing.** Diagnosis currently stops at which component is
   unhealthy. There is no way to follow one request through the services behind
   it.

---

## Running it

### First build, about 30 minutes

1. **Bake the node images with Packer.** `make build` publishes an AMI per role
   and an SSM pointer naming it.
2. **Apply the bootstrap Terraform layers once:** state backend, artifact store,
   OIDC store, AMI pointers, chart registry, CI roles.
3. **Apply the cluster Terraform layers in order:** network, identity, data,
   control plane, workers, inventory. The last renders the inventory into S3.
4. **Build the cluster with Ansible and Kubespray.** `make sync` pulls that
   inventory and `make cluster` runs Kubespray and the platform roles against
   the control plane (~13 min). Workers take no Ansible run at all: they come up
   from their Auto Scaling group and join themselves.
5. **Hand it to Argo CD.** `make argocd` has Ansible install it once, and
   everything after that is a pull request.

### Day 2

Operations on a running cluster. Each one runs a pre-flight check before it
touches anything and re-asserts the etcd mount guard afterwards, so no run can
leave a node without it. `make reset` will not start until the cluster name is
typed back.

|                                   |                                                                       |
| --------------------------------- | --------------------------------------------------------------------- |
| `make upgrade`                    | move every node to the cluster's pinned `kube_version`, one at a time |
| `make scale`                      | join workers that appeared after the first install                    |
| `make remove-node`                | drain a node and unjoin it, while it is still in the inventory        |
| `make delete-stale-node`          | delete a Node object whose instance is already gone                   |
| `make recover-control-plane`      | rebuild a lost control-plane node from a surviving one                |
| `make reset`                      | wipe Kubernetes off the nodes and leave the machines                  |
| `make kubeconfig` + `make tunnel` | open apiserver access over SSM, with no bastion                       |

### What runs before any of it

`make check-static` gates every change, with no AWS credentials and no cluster.

- **`check-vars`** rejects a cluster-vars file that redefines a key Terraform
  owns. Two sources of truth for one value is how a cluster ends up configured
  differently from the way it was provisioned.
- **`check-pins`** rejects a chart version range or an unpinned source, so what
  runs depends on what is in git rather than on when Argo CD last synced.
- **`check-inventory-fresh`** re-asserts the rendered host list against EC2
  after every sync. Terraform verified it when it rendered, which may have been
  weeks ago.
- **`check-cert-dir`** compares the certificate directory across the two
  repositories that have to agree on it.
- Every playbook, including Kubespray's own `cluster.yml`, is syntax-checked
  against a stub inventory.

Terraform preconditions carry the rest, the invariants that only exist at plan
time: a control-plane count that would silently become round-robin DNS, an OIDC
issuer URL whose trailing slash IAM would normalise into a provider no token
ever matches.

_Prerequisites, every target, and the ordering constraints →
[`docs/how_to_run.md`](docs/how_to_run.md)_

---

## Repositories

|                                                                        |                                                           |
| ---------------------------------------------------------------------- | --------------------------------------------------------- |
| [`fleetctl-packer`](https://github.com/seekzeek/fleetctl-packer)       | node images, baked from Kubespray's own download manifest |
| [`fleetctl-terraform`](https://github.com/seekzeek/fleetctl-terraform) | 12 layers, isolated state, two IAM permission boundaries  |
| [`fleetctl-ansible`](https://github.com/seekzeek/fleetctl-ansible)     | Kubespray plus 7 roles for what it does not ship          |
| [`fleetctl-gitops`](https://github.com/seekzeek/fleetctl-gitops)       | Argo CD app-of-apps, 15 platform Applications             |
| [`fleetctl-charts`](https://github.com/seekzeek/fleetctl-charts)       | service charts, published to ECR as OCI                   |

The reasoning behind every decision above is in
[`docs/deep-dives.md`](docs/deep-dives.md); the end-to-end build sequence is in
[`docs/how_to_run.md`](docs/how_to_run.md).
