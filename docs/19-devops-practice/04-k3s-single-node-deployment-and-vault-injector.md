<!-- Not yet sequenced into a numbered docs/ folder, lives here in
     docs/19-devops-practice/ until the full devops-practice-plan.md is complete.
     Date below is provisional (the date this actually happened); revisit
     once final sequencing/dating is decided. -->

# Kubernetes (k3s)

**Date:** 2026-07-26
**Repo the actual code/infra changes live in:** `Animal-Shelter-Workshop`
(k8s manifests), this write-up lives in the homelab meta-repo instead,
alongside the devops practice plan it's a stage of (`devops-practice-plan.md`,
Stage 5's checklist, all 10 items).

## Why I built this

- Stage 4 packaged the app into a portable image but proved it could only
  ever run one Pod's worth of it: no self-healing, no rolling deploys, no
  way to scale past "one container, one host."
- Stage 5's job is to hand that same image to something that actually
  manages it: kill a container and have it come back on its own, without
  a human running `docker compose up` again.
- k3s is the smallest real Kubernetes distribution that fits this host's
  RAM budget.
- It's a hard prerequisite for Stage 7 (ArgoCD runs *inside* a k3s
  cluster, not next to it).

## Concept: what changes vs. Stage 4

```
STAGE 4 (docker-compose, one host)         STAGE 5 (k3s, one CT so far)
┌─────────────────────────┐                ┌─────────────────────────┐
│  docker compose up       │                │   kubectl apply -f k8s/  │
│  you restart it by hand  │      ──▶       │   the Deployment          │
│  if a container dies     │                │   restarts it for you    │
└─────────────────────────┘                └─────────────────────────┘
   1 app container                            3 app Pod replicas,
   1 nginx container                           each still app+nginx,
   local .env dev credentials                  real 5-connection DB creds
                                                 injected live from Vault
```

- The real 5-database fleet is untouched either way, same as Stage 4,
  this stage doesn't containerize any database.
- What's new is *who* watches the app container and restarts it when it
  dies.

## Flow

```
┌────────────────────────────────────┐
│      STAGE 5, k3s (single node)   │▏
└────────────────────────────────────┘▔▔

┌────────────────────────────────┐     done, new LXC CT 100, 1 core /
│ 1. k3s in an LXC CT, start small │▏    1.5GB, checked free -h + swap
│    (capacity check first)        │▏    on proxmox before creating it
└────────────────────────────────┘▔▔
              │
              ▼
┌────────────────────────────────┐     done, v1.36.2+k3s1, single
│ 2. Single-node cluster           │▏    control-plane node, Ready
└────────────────────────────────┘▔▔
              │
              ▼
┌────────────────────────────────┐     done, kubectl run, exec'd in
│ 3. kubectl run: one ad-hoc Pod   │▏    to confirm php-fpm actually
└────────────────────────────────┘▔▔    running
              │
              ▼
┌────────────────────────────────┐     done, 2-replica Deployment +
│ 4. Deployment + Service YAML     │▏    NodePort Service, app+nginx
└────────────────────────────────┘▔▔    sidecar per Pod, HTTP 200 through it
              │
              ▼
┌────────────────────────────────┐     done, deleted one replica,
│ 5. Kill a Pod, confirm self-heal │▏    ReplicaSet replaced it inside
└────────────────────────────────┘▔▔    seconds, Service never dropped
              │
              ▼
┌────────────────────────────────┐     done, non-DB env vars split
│ 6. ConfigMap + Secret             │▏    into ConfigMap/Secret, rollout
└────────────────────────────────┘▔▔    clean, verified via kubectl exec
              │
              ▼
┌────────────────────────────────┐     done, injector installed
│ 7. Vault Agent Injector           │▏    pointed at the existing
└────────────────────────────────┘▔▔    linux-vault, demo secret injected
              │
              ▼
┌────────────────────────────────┐     deferred, documented, no
│ 8. Expand to 2+ nodes             │▏    second physical Proxmox node
└────────────────────────────────┘▔▔    exists; RAM already tight on this one
              │
              ▼
┌────────────────────────────────┐     done, taint/toleration and
│ 9. Scheduling, taints, draining  │▏    cordon/drain/uncordon all
└────────────────────────────────┘▔▔    proven on the single node
```

## What I built

### 1. New LXC CT for k3s, not a VM

- Capacity check first, per the plan's own instruction: `free -h` on
  `proxmox` showed only 744Mi free / 4.1Gi available, with 2.0Gi
  **already** sitting in swap (up from 1.3Gi at the start of the
  previous session), confirming the plan's Host Capacity Reality Check
  wasn't exaggerating.
- Created CT 100 (`linux-k3s`) anyway, deliberately small: **1 core,
  1.5GB RAM, 512MB swap**, unprivileged, VLAN tag 30 (matches
  `linux-vault`/`linux-gh-runner`, general infra, not app-server's 40 or
  the DB roles' 20).
- `features: nesting=1,keyctl=1` for containerd, plus the same
  `/dev/net/tun` passthrough lines every other Tailscale-joined CT in
  this homelab already carries.
- Had to append these to `/etc/pve/lxc/100.conf` directly after a
  stop/start cycle, `pct set` has no CLI flag for raw `lxc.*` config
  keys.

### 2. k3s, single node, one real bug fixed to get there

- `curl -sfL https://get.k3s.io | sh -` installed cleanly, but the
  service crash-looped:
```
Failed to create an oomWatcher (running in UserNS, Hint: enable
KubeletInUserNamespace feature flag to ignore the error)" err="open
/dev/kmsg: no such file or directory"
Error: failed to run Kubelet: failed to create kubelet: open /dev/kmsg:
no such file or directory
```
- An unprivileged LXC container has no `/dev/kmsg` and kubelet insists
  on one unless told otherwise, the log line names its own fix.
  Reinstalled with:
```
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="server --kubelet-arg=feature-gates=KubeletInUserNamespace=true" \
  sh -
```
- After that, `kubectl get nodes` showed `linux-k3s   Ready   control-plane`
  within 20 seconds, and every system Pod (CoreDNS, metrics-server,
  Traefik, local-path-provisioner) came up `Running` on its own.
- Two harmless warnings along the way, not fixed because they don't need
  fixing:
  - modprobe can't load `overlay`/`br_netfilter` inside the unprivileged
    container, but both were already loaded on the host kernel, so k3s
    picked them up anyway.
  - a sysctl write for `nf_conntrack_max` gets `permission denied`,
    which is expected and doesn't block anything.

### 3. `kubectl run`, one ad-hoc Pod, proven for real

```
kubectl run asw-app --image=docker.io/tttaufiqqq/animal-shelter-workshop:v1.0.0 --restart=Never
```
- `Running`, `1/1`, in under 30 seconds (image pull included).
- Since this image only runs `php-fpm` (no built-in HTTP listener, see
  Stage 4), the honest verification isn't a curl, it's:
```
kubectl exec asw-app -- ps aux
# PID 1: php-fpm master process, PID 7-8: pool www workers
```
- Confirmed and torn down, this was deliberately a throwaway step per
  the plan, not the real deliverable.

### 4. Deployment + Service, app+nginx sidecar, not app alone

- The real deliverable. Each Pod runs **two containers**, same shape as
  Stage 4's `docker-compose.yml` (`app` + `webserver`), minus the local
  `db` service:

```
k8s/
├── deployment.yaml       2 replicas, initContainer + app + nginx
├── service.yaml          NodePort 30080 → container port 80
├── nginx-configmap.yaml  same vhost as docker/nginx/default.conf,
│                         fastcgi_pass now 127.0.0.1:9000 (same Pod,
│                         not a Compose service name)
├── app-configmap.yaml    non-secret env (APP_NAME, DB_CONNECTION=sqlite, ...)
└── app-secret.yaml       APP_KEY only
```

- One real gap Docker's own behavior papered over: `docker-compose.yml`
  mounts an *empty* named volume over `/var/www/html/public`, and Docker
  silently copies the image's existing content into an empty named
  volume the first time it's used.
- **Kubernetes' `emptyDir` does not do this**: a bare emptyDir mounted
  over that path would shadow the Vite build baked into the image at
  build time, and nginx would 404 every asset.
- Fixed with an `initContainer` (same app image) that runs once per Pod
  startup:
```yaml
command: ["sh", "-c", "cp -r /var/www/html/public/. /shared/"]
```
- This populates the shared `emptyDir` before either the `app` or
  `nginx` container starts.

```
kubectl apply -f k8s/nginx-configmap.yaml -f k8s/deployment.yaml -f k8s/service.yaml
kubectl rollout status deployment/asw-app
# deployment "asw-app" successfully rolled out
curl http://localhost:30080/up   # -> 200
curl http://localhost:30080/     # -> 200 (homepage, not just the health route)
```

### 5. Killed a Pod on purpose

```
kubectl delete pod asw-app-794dcdd558-9flmq
```
- Within 1 second the ReplicaSet had a replacement `PodInitializing`;
  within 16 seconds it was `2/2 Running`.
- `curl http://localhost:30080/up` returned `200` the entire time, the
  surviving replica carried traffic while the replacement came up.
- This is the actual point of moving to Kubernetes at all: `docker
  compose` only restarts what you explicitly re-run; a `Deployment`
  does this by itself, unprompted.
- Reproduced live afterward with `kubectl get pods -l app=asw -w`
  running in one terminal while a delete ran in another, to actually
  watch the transition frame by frame instead of just the before/after
  snapshot:

![Terminal split showing `kubectl delete pod asw-app-c7857ccf8-lllwp` in the left pane, and the right pane's `kubectl get pods -l app=asw -w` stream catching the whole cycle live: the deleted pod going Terminating then Completed, followed by a new pod asw-app-c7857ccf8-569rj cycling Pending -> Init:0/1 -> PodInitializing -> 2/2 Running, the entire replacement taking about 5 seconds](images/stage5-self-heal-watch.png)

### 6. ConfigMap + Secret

- Moved the Deployment's inline `env:` list into `envFrom`: everything
  non-secret (`APP_NAME`, `APP_ENV`, `DB_CONNECTION=sqlite`,
  `SESSION_DRIVER`, etc.) into `asw-app-config`, just `APP_KEY` into
  `asw-app-secret`.
- The real production `DB1-5_HOST/PORT/DATABASE/USERNAME` values now live
  as plain literals in `asw-app-config` (not secret on their own, same
  values `env-app.j2` already renders for the bare-metal deploy) —
  `DB1-5_PASSWORD` and the real `APP_KEY` come from Vault instead (see
  step 7 below), never from this ConfigMap/Secret.
```
kubectl apply -f k8s/app-configmap.yaml -f k8s/app-secret.yaml -f k8s/deployment.yaml
kubectl rollout status deployment/asw-app   # successfully rolled out
kubectl exec deploy/asw-app -c app -- env | grep -E 'APP_NAME|APP_KEY|DB_CONNECTION'
# APP_KEY=base64:M0OU...   DB_CONNECTION=sqlite   APP_NAME=ASW
```

### 7. Vault Agent Injector, ties this cluster to the existing `linux-vault`

- `linux-vault` already runs, live, unsealed, serving real secrets to
  the rest of this homelab (Stage 2/3's AppRole-based reads).
- This stage doesn't stand up a *new* Vault, it points the injector at
  the one that already exists (`injector.externalVaultAddr`,
  `server.enabled=false`):
```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  --set server.enabled=false \
  --set injector.enabled=true \
  --set injector.externalVaultAddr=http://100.112.41.113:8200 \
  --set injector.resources.requests.memory=64Mi \
  --set injector.resources.limits.memory=128Mi
```
- Then, on `linux-vault` itself (an administrative, live-API operation,
  same category as Stage 2's one-time `vault kv patch`, **not** a
  config change that would reseal Vault):
```
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host='https://100.109.241.125:6443' \
  kubernetes_ca_cert=@k3s-ca.crt \
  token_reviewer_jwt=@reviewer.jwt
vault policy write asw-k8s-demo asw-k8s-demo-policy.hcl
vault write auth/kubernetes/role/asw-k8s-demo \
  bound_service_account_names=vault-demo \
  bound_service_account_namespaces=default \
  policies=asw-k8s-demo ttl=1h
```
- The reviewer JWT came from a long-lived
  `kubernetes.io/service-account-token` Secret bound to the `vault`
  ServiceAccount the Helm chart itself creates (already wired to
  `system:auth-delegator` via `vault-server-binding`).
- The CA cert came from `/var/lib/rancher/k3s/server/tls/server-ca.crt`.
- Proved it end to end with a demo Pod, annotated for injection:
```yaml
metadata:
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "asw-k8s-demo"
    vault.hashicorp.com/agent-inject-secret-hello.txt: "secret/data/k3s-demo"
```
```
kubectl exec vault-demo -c app -- cat /vault/secrets/hello.txt
vault-agent-injector-works
```

**Wired the app's own real credentials through the same injector,** past
the demo Pod above: a **new** Kubernetes-auth role, `asw-app`, bound to a
**new** `asw-app` ServiceAccount (`k8s/service-account.yaml`) — pointed at
the **same** `asw-deploy` policy the Ansible AppRole deploy already uses
(`docs/11-vault-approle-app-integration`), scoped read-only to
`secret/animal-shelter-workshop`. One secret, two independent auth paths
(AppRole for Ansible, Kubernetes auth for this cluster), same policy, same
blast radius:
```
vault write auth/kubernetes/role/asw-app \
  bound_service_account_names=asw-app \
  bound_service_account_namespaces=default \
  policies=asw-deploy ttl=1h
```
- The Deployment's pod template carries the same style of annotations as
  the demo above, but templating a small `export KEY="value"` file from
  `secret/data/animal-shelter-workshop`'s `db_password` and `app_key`
  fields to `/vault/secrets/db.env`.
- **The one real wrinkle:** the Injector only writes to a shared file, it
  can't inject directly into another container's process environment.
  Solved the same way the bare-metal Vault Agent setup already solved
  this exact problem (`docs/11-vault-approle-app-integration`'s "Vault
  Agent — secrets stopped touching disk at runtime too" section) —
  override the `app` container's `command`/`args` to
  `sh -c ". /vault/secrets/db.env && exec php-fpm"`, sourcing the
  rendered file into the shell's environment before `exec`'ing PHP-FPM,
  so the env vars are already present in `php-fpm`'s process image at
  startup. No changes needed to the Docker image itself.
- **This cluster is GitOps-managed (Stage 7, ArgoCD, `selfHeal: true`)** —
  a real gotcha hit applying this: `kubectl apply` against the live
  cluster appeared to work (pods briefly picked up the new config), but
  ArgoCD reverted it within seconds, since the change only existed in the
  local working copy, not in git. Manifests have to be committed and
  pushed to `main` for ArgoCD to adopt them; `kubectl apply` directly
  against a GitOps-managed cluster is a no-op against a healthy sync loop,
  not a shortcut around it. Forced an immediate re-sync instead of waiting
  for the default poll interval:
```
kubectl -n argocd patch application animal-shelter-workshop --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```
```
kubectl exec <asw-app-pod> -c app -- cat /vault/secrets/db.env
# export APP_KEY="base64:4CcU+..."
# export DB1_PASSWORD="workshop_2_prod"   (and DB2-5)

curl http://100.109.241.125:30080/api/database-status
# "allOnline": true, all 5 connections connected:true
```
- Confirmed through the real running app end-to-end, not just that the
  Injector wrote a file — the same standard this stage's original
  verification already held itself to.
- Local dev (`docker-compose.yml`) reaches the same 5 real databases too,
  but with a separate `workshop_2_dev` credential straight from a plain
  `.env` file, never Vault, never this Deployment's config. Full detail:
  `docs/19-devops-practice/03-docker-multi-stage-build-and-compose.md`.
  The two paths are deliberately parallel and non-overlapping: dev
  credentials never touch k3s, prod credentials never touch
  `docker-compose.yml`.

### 8. Expand to 2+ nodes, deferred, documented (not a gap)

- The plan's own capacity note anticipates this: expanding k3s to
  multiple nodes "pairs with adding a second physical Proxmox node
  rather than squeezing a second k3s node onto the same box."
- Checked: `pvecm status` confirms this homelab has exactly **one**
  standalone Proxmox host, no cluster, and its RAM is already the
  tightest resource in the whole plan (1.0Gi free, 2.2Gi in swap, even
  before this stage's own CT).
- Same shape as Stage 4's Harbor deferral, a real, documented decision
  given current hardware, not an oversight.
- Revisit once (if) a second physical node exists.

### 9. Scheduling, taints/tolerations, draining, practiced on the single node

- Multi-node expansion is deferred, but taints and draining are
  meaningful on one node too: they answer "can I keep a workload off
  this node" and "can I evacuate everything off this node for
  maintenance," neither of which needs a second node to demonstrate.

**Taint / toleration:**
```
kubectl taint nodes linux-k3s demo=busy:NoSchedule
kubectl run taint-test --image=busybox --restart=Never --command -- sleep 3600
# stays Pending: "1 node(s) had untolerated taint(s)"
```
- A second Pod with a matching `tolerations:` block scheduled
  immediately and reached `Running` while `taint-test` stayed
  `Pending`, side by side, same node, same taint.

**Cordon / drain / uncordon:**
```
kubectl taint nodes linux-k3s demo=busy-   # cleanup
kubectl cordon linux-k3s
kubectl drain linux-k3s --ignore-daemonsets --delete-emptydir-data --force
```
- Evicted every non-DaemonSet Pod on the node, both `asw-app` replicas,
  the Vault injector, CoreDNS, metrics-server, local-path-provisioner,
  Traefik, cleanly, `node/linux-k3s drained`.
- With no second node, everything sat `Pending` while cordoned
  (expected, this is exactly what a real drain during maintenance looks
  like on a cluster this size).
- `kubectl uncordon linux-k3s` brought every single one back to
  `Running` within 25 seconds, `curl http://localhost:30080/up`
  confirmed `200` again immediately after.

## What I found

- **Graceful degradation carries over into k3s, unchanged.** Proven while
  no DB connections were wired up yet (`DB_CONNECTION=sqlite` was the only
  DB config in `asw-app-config` at the time) — the homepage showed
  `0/5 databases online`, and the same `HandleDatabaseFailures`/
  `InjectDatabaseStatus` middleware Stage 4 already proved still rendered
  the page underneath instead of a 500, with a "Continue Anyway" escape
  hatch. Confirmed this behavior isn't specific to Docker Compose, it's
  the app itself, and it survives unchanged into a Kubernetes Deployment
  with zero extra work. (All 5 connections are wired for real now — see
  step 7 above — but the app's own resilience here is worth keeping on
  record independent of that.)

![Animal Shelter Workshop homepage through the k3s NodePort Service, with the app's own "Database Connection Notice" modal open showing 0/5 databases online (Users, Reporting, Animals, Shelter, Booking all OFFLINE), homepage still rendered blurred underneath, "Continue Anyway" button available](images/stage5-database-status-notice.png)

- **The k3s API server's default TLS certificate doesn't cover the
  Tailscale IP**, only the LAN IP, the cluster IP, and `localhost`.
- Vault's Kubernetes auth login kept returning a bare `403 permission
  denied` with zero detail (this is deliberate, Vault masks all
  kubernetes-auth login failures behind this exact generic message as a
  security hardening measure, so the client-facing error is genuinely
  uninformative by design).
- The real cause only showed up after bumping Vault's own log level
  live (`vault write sys/loggers level=debug`, a runtime API call, not
  a config file edit, so it needs no restart and doesn't reseal Vault)
  and reading `journalctl -u vault`:
```
tls: failed to verify certificate: x509: certificate is valid for
10.0.30.100, 10.43.0.1, 127.0.0.1, ::1, not 100.112.41.113
```
- Fixed by adding the Tailscale IP as an extra SAN and restarting k3s
  (affects only this CT, not Vault):
```yaml
# /etc/rancher/k3s/config.yaml
tls-san:
  - 100.109.241.125
```
- Reverted Vault's log level back to `info` afterward.
- Worth remembering: if a Vault Kubernetes-auth login ever returns a
  bare 403 with no further detail again, the log-level-bump-then-check-
  journalctl move is the way to actually see why.

## Bonus: hardening, proving the CT survives a real shutdown/reboot, not assuming it

- Every fix made earlier in this stage happened to land in a persistent
  place rather than a one-off runtime command, so in theory none of it
  should need redoing after a CT stop/start or a Proxmox host reboot.
- Rather than assume that, checked each piece explicitly and then
  actually shut the CT down and brought it back cold:

- **`onboot: 1`** already set on CT 100 at creation time, survives a
  `proxmox` host reboot without a manual `pct start`.
- **`k3s` and `tailscaled` are both `systemctl enable`d**, not just
  `--now`-started, `systemctl is-enabled k3s tailscaled` returns
  `enabled` for both, so they start on their own after the CT boots, no
  human needed.
- **The `KubeletInUserNamespace` fix is baked directly into
  `/etc/systemd/system/k3s.service`'s own `ExecStart=` line** (a static
  file on the CT's disk), not something set only for the process that was
  running at install time, confirmed by reading the file back.
- **The `tls-san` fix lives in `/etc/rancher/k3s/config.yaml`**, a real
  file k3s reads on every startup, not a flag that only applied to one
  invocation.
- **The `/dev/net/tun` passthrough and cgroup device allow lines live in
  Proxmox's own `/etc/pve/lxc/100.conf`** (host-side), reapplied by
  Proxmox itself every time the CT starts, regardless of why it stopped.
- **`tun` turned out to be compiled directly into this host's kernel**
  (`CONFIG_TUN=y`), not a loadable module, so there's no
  module-not-loaded-yet race to worry about on a fresh host boot either.

- Then did the actual test instead of trusting the above list on its
  own:
```
pct shutdown 100 --timeout 30    # clean stop
pct start 100                    # cold start, no manual fixes applied
```
- 25 seconds after `pct start`, with zero intervention:
```
kubectl get nodes
# linux-k3s   Ready   control-plane   ...

tailscale status --json | grep BackendState
# "BackendState": "Running"          <- reconnected, no re-auth needed

kubectl get pods -A
# every Pod back to Running (RESTARTS incremented by 1-2, the
# containers inside each Pod restarted in place; the Pods themselves
# were never recreated, since the k3s datastore lives on the CT's own
# disk and survived the stop/start cycle unchanged)

curl http://localhost:30080/up
# HTTP 200
```
- No fixes were needed to make this work, it already worked, because
  the earlier fixes were written to disk instead of applied only
  in-memory.
- This is the actual meaning of "hardened" here: not new configuration,
  but confirmation that nothing in this stage is secretly depending on
  a runtime state that a shutdown would wipe out.

## How to independently verify each item

**1-2. k3s CT + single-node cluster**
```bash
ssh proxmox "pct exec 100 -- kubectl get nodes -o wide"
# linux-k3s   Ready   control-plane   ...
```

**3. kubectl run**
```bash
kubectl run asw-app --image=docker.io/tttaufiqqq/animal-shelter-workshop:v1.0.0 --restart=Never
kubectl exec asw-app -- ps aux   # php-fpm master + pool processes
```

**4. Deployment + Service**
```bash
kubectl get deploy,pods,svc -l app=asw
curl -o /dev/null -w "%{http_code}\n" http://<node>:30080/up   # 200
```

**5. Self-heal**
```bash
kubectl delete pod <one-of-the-two-asw-app-pods>
kubectl get pods -l app=asw -w   # watch a replacement appear within seconds
```

**6. ConfigMap/Secret**
```bash
kubectl exec deploy/asw-app -c app -- env | grep APP_NAME
# comes from asw-app-config, not an inline Deployment value
```

**7. Vault Agent Injector**
```bash
kubectl exec vault-demo -c app -- cat /vault/secrets/hello.txt
# vault-agent-injector-works

# the real app's own credentials, injected the same way:
kubectl exec <asw-app-pod> -c app -- cat /vault/secrets/db.env
# export APP_KEY="base64:..."   export DB1_PASSWORD="workshop_2_prod"  (and DB2-5)

curl http://100.109.241.125:30080/api/database-status
# "allOnline": true, all 5 connections connected:true
```

**9. Taints / draining**
```bash
kubectl taint nodes linux-k3s demo=busy:NoSchedule
kubectl run t --image=busybox --restart=Never --command -- sleep 60
kubectl get pod t   # Pending
kubectl taint nodes linux-k3s demo=busy-
kubectl drain linux-k3s --ignore-daemonsets --delete-emptydir-data --force
kubectl uncordon linux-k3s
```

**Bonus: shutdown/reboot hardening**
```bash
ssh proxmox "pct shutdown 100 --timeout 30 && pct start 100"
# wait ~25s, then:
ssh proxmox "pct exec 100 -- kubectl get nodes"          # Ready
ssh proxmox "pct exec 100 -- tailscale status --json"    # BackendState: Running
ssh proxmox "pct exec 100 -- kubectl get pods -A"         # all Running
ssh proxmox "pct exec 100 -- curl -s -o /dev/null -w '%{http_code}\n' http://localhost:30080/up"  # 200
```

## What carries forward to Stage 6/7, and what doesn't

- The `k8s/` manifests (`Animal-Shelter-Workshop` repo) are the real,
  durable deliverable, Stage 7 (GitOps) points ArgoCD at exactly this
  directory, per the plan's own instruction, rather than a separate repo.
- The Vault Kubernetes auth method, the `asw-k8s-demo` policy/role, and the
  injector installation are all real and carry forward — proven with the
  demo secret path (`secret/k3s-demo`) and the `vault-demo` Pod/
  ServiceAccount first, then reused for real by the app's own
  `asw-app` role/ServiceAccount, both wired to the app's real config now
  (step 7 above), not just exploratory scaffolding.
- Multi-node expansion and its downstream scheduling practice
  (taints/tolerations/draining beyond what a single node can show) were
  assumed blocked on a second physical Proxmox host at the time this was
  written — turned out not to be the real constraint. See the
  continuation below: a second CT on the *same* physical host, sized the
  same way as this one, was all it took.

## Continued: 2-node expansion, full GitOps automation, and the cloudflared migration

`plans/06-k3s-multi-node-gitops-automation-plan.md`, executed
2026-08-02, picked up exactly where the section above left off. What
that plan built, in order:

**1. A real second node, not a second physical host.** The assumption
above (multi-node needs a second Proxmox host) didn't hold up once
actually checked — `linux-k3s-2` (CT 118, 1 core/2GB, same "start small"
sizing as CT 100 itself) joined as a plain `k3s agent` on the *same*
Proxmox host, `kubelet-arg=feature-gates=KubeletInUserNamespace=true`
carried over from CT 100's own fix (same unprivileged-LXC constraint,
same workaround). `kubectl get nodes` showed 2 `Ready` nodes within
minutes — the "needs a second physical box" note above was simply wrong,
not yet re-examined.

**2. `asw-app`'s app+nginx sidecar pair split into 2 real Deployments.**
Up to this point `app` (php-fpm) and `nginx` shared one Pod, one node,
talking over `localhost`. Splitting them onto separate nodes meant
breaking that same-Pod assumption for real: `nginx`'s static assets
(previously copied in by an initContainer into a shared `emptyDir`) got
baked into `nginx`'s own image at build time instead (a new `nginx`
Dockerfile stage, reusing the same Vite `frontend` build output the
`runtime` stage already depended on), and `nginx`'s `fastcgi_pass`
switched from `127.0.0.1:9000` to a real ClusterIP Service
(`asw-app-internal`) reaching `asw-app` on the other node. `nodeSelector`
pinned each Deployment explicitly, rather than trusting the scheduler to
keep them apart.

![Diagram of the 2-node k3s cluster: Node 1 (control-plane) running asw-app (php-fpm), Node 2 (agent) running the cloudflared k3s-asw-nginx tunnel and asw-nginx (static assets compiled), with the request flow arrows external requests → cloudflared → asw-nginx → asw-app, the only cross-node hop being asw-nginx to asw-app via the asw-app-internal ClusterIP Service. Part 1 of 2 — part 2 lands once a 3rd node is added for ArgoCD/observability.](images/plan06-k3s-cluster-node-split-diagram-part1.png)

**3. A real cross-node failover test, not simulated on one node.**
`kubectl delete pod` on an `asw-nginx` pod running on node 2 produced a
replacement, still correctly pinned to node 2, within single-digit
seconds — the real version of what the single-node section above could
only simulate with taints.

**4. The actual automation gap, closed.** `deploy.yml` used to only ever
touch `app-server`; nothing rebuilt the app image or bumped
`k8s/deployment.yaml` on a normal push. A new `build-and-push-k3s-images`
job now builds **both** images (`asw-app`, `asw-nginx`) tagged with the
commit SHA, pushes to Docker Hub, and commits the tag bump straight back
into `k8s/*.yaml` — `[skip ci]` on that bump commit is what keeps it from
re-triggering itself forever.

**5. `cloudflared` migrated fully into the cluster.** A brand-new
Cloudflare Tunnel (`k3s-asw-nginx`, deliberately separate from
`app-server`'s original `animal-shelter-workshop` tunnel) was created
first, its token stored in Vault (`cloudflare_tunnel_token`, same
Agent-Injector pattern as the app's own DB credentials — never a
committed k8s Secret), and proven end-to-end on a **temporary** hostname
before touching real traffic:

![Cloudflare "Route Traffic" wizard creating the k3s-asw-nginx tunnel's first published route: subdomain k3s-asw-nginx.tttaufiqqq.com, HTTP service pointed at the origin](images/plan06-cloudflare-tunnel-route-traffic-form.png)

![Cloudflare Tunnels list showing the original animal-shelter-workshop tunnel Healthy (still serving from app-server at this point) alongside the brand-new k3s-asw-nginx tunnel, still Inactive since no connector had been deployed yet](images/plan06-cloudflare-tunnels-list.png)

Once the `cloudflared` Deployment (its own ServiceAccount + Vault
Kubernetes-auth role, scheduled onto node 2 alongside `asw-nginx`) landed
via ArgoCD, the tunnel came up Healthy with 4 live QUIC connections:

![k3s-asw-nginx tunnel's own overview page: Status Healthy, one Connector shown Connected, running cloudflared 2026.7.2 on linux_amd64](images/plan06-cloudflare-k3s-tunnel-connected.png)

**What broke here:** the temporary-hostname proof worked, but cutting
the *real* production hostname over took an extra round — the first
attempt at adding `animal-shelter-workshop.tttaufiqqq.com`'s route
landed as `asw.tttaufiqqq.com` instead (a naming slip, not a technical
failure):

![Published application routes on the k3s-asw-nginx tunnel showing the wrong hostname — asw.tttaufiqqq.com — routed to asw-nginx, working but not the actual production domain](images/plan06-cloudflare-published-route-asw-subdomain.png)

`deploy.yml`'s own smoke test asserts against
`animal-shelter-workshop.tttaufiqqq.com` specifically, so this would
have silently broken the next CI run. Caught before it did — the correct
route replaced it:

![Published application routes on the k3s-asw-nginx tunnel, corrected: animal-shelter-workshop.tttaufiqqq.com routed to http://asw-nginx.default.svc.cluster.local:80, the actual production hostname](images/plan06-cloudflare-published-route-production-hostname.png)

`app-server`'s own `cloudflared` was then fully removed (service
stopped/disabled, package purged, config and binary deleted, systemd
unit files cleaned up) — not just stopped, since `app-server`'s eventual
shutdown made "still installed but idle" pointless to leave behind.

**6. The whole loop, proven once, timed, with zero manual steps.** A
one-line comment change, pushed to `main`: Tests (~15 min, mostly the
Playwright browser suite) → CI build/push/bump (~1.5 min) → ArgoCD's own
polled sync (no manual `kubectl`/`argocd` command run at all this time)
→ new pods `Running` on their pinned nodes → **23m07s** end to end,
confirmed by `grep`-ing the exact comment text back out of the live
pod's checked-out source.

**What else broke along the way, found and fixed, not part of the plan's
original scope:**
- `.env.testing.example` still pointed 3 of the 5 DB connections at the
  VMs retired by the earlier VM→CT migration (only 2 of 5 had been
  updated at the time) — silently undetected because recent pushes had
  all been docs-only commits, which `tests.yml`'s `paths-ignore` skips
  entirely. Repointed to the current CTs, and the missing
  `workshop_2_test` database/user (only `_dev`/`_prod` were carried over
  in that migration) provisioned fresh on all 3 affected hosts.
- `linux-gh-runner` had no UFW rule for the actual DB ports on any of the
  5 DB hosts, only SSH — invisible until the *previous* session's
  Tailscale-netfilter-bypass fix correctly closed the loophole that had
  been letting it through by accident. Added an explicit allow rule,
  same per-host pattern already used for `app-server`/`linux-k3s`.
- Two booking-procedure tests hardcoded `'2026-08-01'` as a "future"
  appointment date — real calendar time caught up to it, and the app's
  own past-date trigger started correctly rejecting it. Switched to
  `now()->addDay()`.
- `asw-app`'s default rolling-update strategy (`maxSurge: 25%`) deadlocked
  on node 1's single CPU core once a 4th (surge) pod tried to schedule
  alongside 3 running replicas + the Vault injector sidecar, already at
  ~95% of the core. Fixed with `maxSurge: 0, maxUnavailable: 1` —
  retire-then-create instead of create-then-retire.
- `linux-gh-runner`'s 10GB disk filled completely mid-session (Docker
  build cache/containerd layers from repeated image builds, plus
  accumulated `_work` checkouts and journal logs) and crashed a Tests
  run outright (`No space left on device`). Cleared with `docker system
  prune -af --volumes` (1.85GB), an `apt-get clean`, a
  `journalctl --vacuum-size=50M`, and removing a stale `_work` checkout —
  freed back to 2.7GB available. Worth a `df -h` check on this host
  occasionally; nothing here prunes itself automatically yet.
- `deploy.yml`'s `deploy-app`/`rollback`/`no-rollback-target` jobs still
  provisioned and smoke-tested `app-server` directly on every push, even
  after Stage 5's cloudflared cutover made k3s the real serving path —
  removed before powering `app-server` off, so CI wouldn't start failing
  on the very next push. The pre-deploy DB backup step that used to run
  from `app-server` before touching the 5 DB hosts went with it (it was
  tied to `app-server`'s own filesystem/Vault Agent config) — not yet
  re-homed anywhere, tracked as follow-up work rather than papered over
  with something untested.

## Where things live

| Piece | Path |
|---|---|
| k8s manifests | `Animal-Shelter-Workshop/k8s/*.yaml` |
| k3s CT | Proxmox CT 100, `linux-k3s`, Tailscale `100.109.241.125` |
| k3s extra TLS SAN | `/etc/rancher/k3s/config.yaml` (inside CT 100) |
| Vault kubernetes auth config | `linux-vault`, `auth/kubernetes/*` (live Vault API, not in git) |
| `asw-app` k8s auth role | `linux-vault`, `auth/kubernetes/role/asw-app` (live Vault API, not in git) |
| k3s-2 CT (agent node) | Proxmox CT 118, `linux-k3s-2`, Tailscale `100.96.7.52` |
| k3s-2 Terraform | `proxmox-homelab-taufiq/infrastructure/terraform/homelab-infra.tf` |
| `asw-app`/`asw-nginx` split manifests | `Animal-Shelter-Workshop/k8s/{deployment,nginx-deployment,service,service-account,nginx-configmap}.yaml` |
| `cloudflared` k8s auth role | `linux-vault`, `auth/kubernetes/role/cloudflared` (live Vault API, not in git) |
| `cloudflared` tunnel token | `linux-vault`, `secret/animal-shelter-workshop` field `cloudflare_tunnel_token` (live Vault API, not in git) |
| `cloudflared` manifest | `Animal-Shelter-Workshop/k8s/cloudflared-deployment.yaml` |
| CI image build/bump job | `Animal-Shelter-Workshop/.github/workflows/deploy.yml`, `build-and-push-k3s-images` |
| The plan this continuation executed | `proxmox-homelab-taufiq/plans/06-k3s-multi-node-gitops-automation-plan.md` |
| This write-up | `proxmox-homelab-taufiq/docs/19-devops-practice/04-k3s-single-node-deployment-and-vault-injector.md` |
| Continued: 3rd node, remaining secrets, soak test | [`13-k3s-production-cutover-and-soak-test.md`](https://github.com/tttaufiqqq/proxmox-homelab-taufiq/blob/main/docs/19-devops-practice/13-k3s-production-cutover-and-soak-test.md) |

### Screenshots

- `images/stage5-database-status-notice.png`, added above (browser hit
  against the k3s NodePort Service directly, taken by the user).
- `images/stage5-self-heal-watch.png`, added above (live `kubectl delete`
  + `kubectl get ... -w` catching the whole self-heal cycle, taken by the
  user).
- `images/plan06-cloudflare-tunnel-route-traffic-form.png`,
  `images/plan06-cloudflare-tunnels-list.png`,
  `images/plan06-cloudflare-k3s-tunnel-connected.png`,
  `images/plan06-cloudflare-published-route-asw-subdomain.png`,
  `images/plan06-cloudflare-published-route-production-hostname.png` —
  all added above, the full Cloudflare Tunnel dashboard sequence for the
  `cloudflared` migration, taken by the user.

Still worth adding next session:
- The Vault UI's Access → Kubernetes auth method page, showing the
  `asw-k8s-demo` role.
- Save to `docs/19-devops-practice/images/` if added.
