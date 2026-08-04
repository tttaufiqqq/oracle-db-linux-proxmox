<!-- Not yet sequenced into a numbered docs/ folder, lives here in
     docs/19-devops-practice/ until the full devops-practice-plan.md is complete.
     Date below is provisional (the date this actually happened); revisit
     once final sequencing/dating is decided. -->

# GitOps (ArgoCD)

**Date:** 2026-07-26/27
**Repo the actual manifests live in:** `Animal-Shelter-Workshop` (`k8s/`,
already existed from Stage 5), this write-up lives in the homelab meta-repo
instead, alongside the devops practice plan it's a stage of
(`devops-practice-plan.md`, Stage 7's checklist, all 4 items).

## Why I built this

- Every stage up to this point still relied on a human (or a CI job) to
  push changes into the cluster directly, `kubectl apply`, `helm install`,
  a Terraform run.
- Nothing was watching to make sure the cluster actually stayed in the
  state git said it should be in.
- Stage 7 closes that gap: git becomes the one source of truth, ArgoCD
  becomes the thing that enforces it.
- This works both on the way in (a git change lands in the cluster on its
  own) and on the way out (a manual change straight to the cluster gets
  silently undone).
- This also depends on Stage 5's k3s cluster existing already, since
  ArgoCD installs *inside* it, not alongside it.

![Illustrated GitOps flow: developer git-pushes to the GitHub repo (source of truth); ArgoCD continuously watches and compares it against the cluster, then applies/syncs. A manual change made by SSHing directly into the k3s cluster is caught as drift and ArgoCD reverts it back to match Git. Caption: the repo stays the single source of truth, every change is tracked.](images/stage7-gitops-flow-diagram.png)

## Flow

```
┌────────────────────────────────────┐
│  STAGE 7, GITOPS (ARGOCD)          │▏
└────────────────────────────────────┘▔▔

┌────────────────────────────────┐     done, CT 100 was too tight
│ 0. Capacity re-check + headroom  │▏    (781Mi avail) for ArgoCD's ask;
└────────────────────────────────┘▔▔    bumped to 3GB/768MB swap, live
              │
              ▼
┌────────────────────────────────┐     done, official stable manifests,
│ 1. Install ArgoCD in k3s         │▏    argocd namespace; the oversized
└────────────────────────────────┘▔▔    ApplicationSet CRD needed --server-side
              │
              ▼
┌────────────────────────────────┐     done, pushed 3 pending local
│ 2. Point at ASW repo's k8s/ dir  │▏    commits first (k8s/ didn't exist
└────────────────────────────────┘▔▔    on origin yet); adopted cleanly
              │
              ▼
┌────────────────────────────────┐     done, bumped replicas 2→3 in
│ 3. Git change → auto-sync test   │▏    git, pushed, ArgoCD synced it,
└────────────────────────────────┘▔▔    new Pod up within seconds
              │
              ▼
┌────────────────────────────────┐     done, kubectl patch to 5 replicas,
│ 4. Manual drift → self-heal test │▏    selfHeal reverted to 3 in ~10s,
└────────────────────────────────┘▔▔    confirmed via ArgoCD's event log
```

## What I built

### 0. Capacity re-check + headroom for ArgoCD on `linux-k3s`

- `free -h` on `proxmox` at the start of this session: 269Mi free, 4.4Gi
  available, swap already at 5.4Gi/7.6Gi (up from 2.2Gi at the start of
  Stage 6, the trend the Stage 6 handoff flagged as worth watching).
- Inside the `linux-k3s` CT itself: only 1.5GB total, 781Mi available, not
  enough headroom for ArgoCD's ~1-2GB ask on its own.
- Bumped the CT's own allocation live, no restart needed (LXC memory is a
  cgroup limit, not a boot-time setting):

```
pct set 100 --memory 3072 --swap 768
```

- Confirmed inside the CT immediately (`free -h` showed 3.0Gi total before
  ArgoCD was even installed).
- Host-side `proxmox` stayed steady afterward (541Mi free / 3.8Gi
  available, swap ticked up slightly to 5.6Gi).
- The whole install fit without pushing the host past where it already was.

### 1. ArgoCD installed into the k3s cluster

- Official stable manifests, not Helm (kept it to plain `kubectl`,
  consistent with how the rest of the cluster was built by hand in
  Stage 5):

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

- Hit one real snag: `applicationsets.argoproj.io`'s CRD is large enough
  that a plain `kubectl apply` fails,
  `metadata.annotations: Too long: may not be more than 262144 bytes`.
- This happens because `kubectl apply` stores the whole previous manifest
  in a `last-applied-configuration` annotation and this CRD alone blows
  past the 256KB cap.
- Every other object in the install applied fine; only this one CRD
  failed, repeatedly, on a second `apply` too.
- Fixed by switching to server-side apply, which tracks field ownership
  instead of stuffing the whole config into an annotation:

```
kubectl apply -n argocd -f <same URL> --server-side --force-conflicts
```

- All 7 ArgoCD components (`application-controller`,
  `applicationset-controller`, `dex-server`, `notifications-controller`,
  `redis`, `repo-server`, `server`) came up `1/1 Running` within about a
  minute.
- One transient `CreateContainerConfigError` on `argocd-server` (`secret
  "argocd-redis" not found`) self-resolved on kubelet's own retry once the
  secret finished being created, no manual fix needed.

**Also needed:**

- The k3s CT's own kubeconfig (`/etc/rancher/k3s/k3s.yaml`) is root-only
  by default.
- k3s's bundled `kubectl` (a symlink to the `k3s` binary) ignores the
  usual `~/.kube/config` convention unless `KUBECONFIG` is explicitly set,
  it falls back to `/etc/rancher/k3s/k3s.yaml` regardless.
- Copied the file to the SSH user's own `~/.kube/config` and exported
  `KUBECONFIG` in `~/.bashrc`/`~/.profile` for future sessions, so
  `kubectl` works passwordless going forward without `sudo` every time.

**Web GUI:**

- ArgoCD ships its own, `argocd-server` was `ClusterIP`-only by default,
  so nothing outside the cluster could reach it.
- Patched it to `NodePort` (same pattern as the app's own `Service` from
  Stage 5, which already uses NodePort `30080`):

```
kubectl -n argocd patch svc argocd-server --patch-file argocd-server-patch.yaml
# port 443 -> nodePort 30943
```

- Now reachable at `https://linux-k3s.taufiq.lab:30943` (self-signed
  cert, browser will warn once), `admin` / the auto-generated password
  from `argocd-initial-admin-secret` in the `argocd` namespace.

### 2. Pointed at `Animal-Shelter-Workshop`'s own repo

- Stage 5's manifests already live in that repo's `k8s/` directory, no new
  repo needed, matching the plan's own instruction.
- One real blocker first: `Animal-Shelter-Workshop` was **3 commits ahead
  of `origin/main`** (Stage 4 Docker, Stage 5 k8s manifests, Stage 6
  observability, all local-only per this project's own convention of not
  pushing until asked).
- Origin had no `k8s/` directory at all yet.
- Pushed all 3 (`48c918e`), confirmed with the user first, since it also
  triggers the real `Tests` → `Deploy` GitHub Actions pipeline against
  production, not just a local git operation.
- Confirmed the repo is public (`gh repo view` / GitHub API,
  `"private": false`).
- ArgoCD pulls over plain HTTPS, no deploy key or PAT needed.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: animal-shelter-workshop
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/tttaufiqqq/Animal-Shelter-Workshop.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

- `kubectl apply -f` this and ArgoCD adopted Stage 5's existing
  `Deployment` cleanly, same Pods, same age, no restart.
- Immediately reported `Synced` / `Healthy`.

### 3. Git-driven change → auto-sync

- Bumped `asw-app` from 2 to 3 replicas in `k8s/deployment.yaml`,
  committed and pushed (`cd53331`).
- Forced an immediate refresh rather than waiting for ArgoCD's default
  ~3-minute poll:

```
kubectl -n argocd annotate application animal-shelter-workshop \
  argocd.argoproj.io/refresh=hard --overwrite
```

- Confirmed synced within seconds, `Deployment` back to `replicas: 3`.
- A third Pod (`asw-app-...-4kmtw`) came up `2/2 Running`, the original
  two Pods untouched.

### 4. Manual cluster change → drift detected and reverted

```
kubectl patch deployment asw-app -p '{"spec":{"replicas":5}}'
```

- `selfHeal: true` caught this before it was even worth re-checking by
  hand, `kubectl get deploy asw-app` already read back `3` moments later.
- Confirmed it was a real detect-and-revert, not just no-op luck, via
  `kubectl -n argocd get events --sort-by='.lastTimestamp'`:

```
Updated sync status: Synced -> OutOfSync
Initiated automated sync to 'cd533317fdf8b18a9171fe2846d9f4e5e6f28589'
Partial sync operation to cd533317fdf8b18a9171fe2846d9f4e5e6f28589 succeeded
Updated sync status: OutOfSync -> Synced
```

- The event log names the exact commit it reverted back to, the same one
  git already had, not a stale or different state.

## How to independently verify each item

| # | Command | Expected |
|---|---------|----------|
| 1 | `kubectl -n argocd get pods` | all 7 components `1/1 Running` |
| 2 | `kubectl -n argocd get application animal-shelter-workshop` | `SYNC STATUS: Synced`, `HEALTH STATUS: Healthy` |
| 3 | edit `k8s/deployment.yaml`, push, `kubectl -n argocd annotate application animal-shelter-workshop argocd.argoproj.io/refresh=hard --overwrite` | cluster state matches the new git value within seconds |
| 4 | `kubectl patch deployment asw-app -p '{"spec":{"replicas":<anything else>}}'`, then re-check | reverts back to git's value on its own, no manual fix |

## What carries forward to Stage 8, and what doesn't

- The `Application` resource, the `argocd` namespace, and the NodePort GUI
  all stay live and running, Stage 8 (Public Cloud) has no dependency on
  any of it, since that stage runs on Azure, not this k3s node.
- This is genuinely the last stage that touches `linux-k3s` directly in
  this plan, worth another `free -h`/`swapon --show` pass before adding
  anything else to that CT in the future, since it's now carrying k3s +
  the app + Vault injector + all of ArgoCD on 3GB.

## Screenshots

![ArgoCD Applications view, animal-shelter-workshop, Project default, Status Healthy/Synced, Repository https://github.com/tttaufiqqq/Animal-Shelter-Workshop, Target Revision main, Path k8s, Destination in-cluster, Namespace default, Created 07/26/2026 23:59:22, Last Sync 07/27/2026 00:01:34](images/stage7-argocd-application-synced-healthy.png)

- The ArgoCD UI itself (`https://linux-k3s.taufiq.lab:30943`), confirming
  everything verified via `kubectl` above holds up in the GUI too.
- The `animal-shelter-workshop` `Application` card shows `Healthy`/
  `Synced`, pointed at the right repo/branch/path.
- `Last Sync` timestamp matches the drift-revert test from section 4.

![ArgoCD Application Details Tree for animal-shelter-workshop, APP HEALTH Healthy, SYNC STATUS Synced to main (cd53331), LAST SYNC Sync OK to cd53331 succeeded 29 minutes ago, author tttaufiqqq, comment "feat(k8s): Stage 7 GitOps sync test - bump asw-app to 3 replicas". Sync status counts in the left sidebar: 5 Synced, 0 OutOfSync, 7 Healthy. Full resource tree, expanded to Pod level: animal-shelter-workshop fans out to asw-app-config (cm), asw-nginx-config (cm), asw-app-secret (secret), asw-app (svc), and asw-app (deploy, rev:2), the Deployment fans out to two ReplicaSets, asw-app-c7857ccf8 (rev:2, current) and asw-app-794dcdd558 (rev:1, old, 0 pods); the current ReplicaSet fans out to all 3 live Pods (asw-app-c7857ccf8-4kmtw, -8h7t5, -569rj), each 2/2 running, the exact 3 Pods from the replica-bump test in section 3, all green](images/stage7-argocd-application-resource-tree.png)

- The `Application` detail resource-tree view is expanded all the way
  down to individual Pods.
- Every managed object (`ConfigMap`s, `Secret`, `Service`, `Deployment`,
  both `ReplicaSet` revisions, and all 3 live `Pod`s from the
  replica-bump in section 3) is shown individually, each green.
- Confirms ArgoCD is tracking the full object graph down to the Pod
  level, not just reporting an aggregate status.
- The sidebar's own counts (5 Synced / 0 OutOfSync, 7 Healthy) corroborate
  the same picture.
- `LAST SYNC` here matches the same `cd53331` commit and timestamp as the
  card view above.

Still worth adding next session:
- The sync history / timeline view showing the auto-sync and self-heal
  events from sections 3 and 4 above.

## Where things live

- **ArgoCD itself:** `argocd` namespace inside the `linux-k3s` CT (100),
  same cluster as Stage 5.
- **The `Application` resource + Service patch YAML:** applied directly via
  `kubectl`, not committed anywhere, they describe ArgoCD's own
  configuration, not app code. Worth committing to
  `Animal-Shelter-Workshop`'s `k8s/` directory in a future session
  (`k8s/argocd-application.yaml`) so ArgoCD's own config is itself
  git-tracked, the same principle this whole stage is about.
- **The actual manifests ArgoCD watches:** `Animal-Shelter-Workshop`'s own
  `k8s/` directory (`deployment.yaml`, `service.yaml`,
  `nginx-configmap.yaml`, `app-configmap.yaml`, `app-secret.yaml`), same
  files Stage 5 created, `main` branch, public repo.

## Credentials created this session (not in git, keep track here)

- **ArgoCD admin password:** `XLf9bvMnYcDLmqjx`, ArgoCD's own
  auto-generated `argocd-initial-admin-secret` (namespace `argocd`), not
  chosen. Worth rotating via the UI once logged in once, same general
  practice as any other freshly-installed admin credential in this
  homelab.
