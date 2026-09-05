# Learn ArgoCD — A Hands-On GitOps Journey

> A module-by-module, **hands-on** path to learning GitOps with **Argo CD**, built and tested on a real **EKS + Karpenter** cluster. Every concept is paired with working manifests in this repo and grounded in the [official Argo CD docs](https://argo-cd.readthedocs.io/en/stable/).

---

## 📖 What is GitOps?

GitOps **flips the deployment model**. Instead of *pushing* changes into the cluster (`kubectl apply`, `helm install` from your laptop or a CI runner), you **declare desired state in Git**, and an in-cluster agent **pulls** that state and **continuously reconciles** the live cluster to match it.

### The four GitOps principles

From the [OpenGitOps](https://opengitops.dev/) project that Argo CD follows:

| # | Principle | Meaning |
|---|-----------|---------|
| 1 | **Declarative** | The whole system is described declaratively (YAML). |
| 2 | **Versioned & immutable** | Git is the single source of truth, with full history. |
| 3 | **Pulled automatically** | An in-cluster agent pulls the desired state. |
| 4 | **Continuously reconciled** | The agent detects and corrects drift. |

### The two central concepts you'll live in

- **Application** — a CRD that says *"take manifests from this repo/path/revision and sync them into this cluster/namespace."* → [docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#applications)
- **Project (AppProject)** — a governance boundary that restricts which repos, clusters, and resource kinds a group of Applications may use. Everything starts in the `default` project. → [docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#projects)

---

## 🏗️ Environment

This journey runs on a real cluster built with Terraform ([EKS_Cluster_Terraform / Karpenter](https://github.com/himanshutak8/EKS_Cluster_Terraform/tree/main/Karpenter)):

- **EKS** with **Karpenter** for node autoscaling (NodePool / EC2NodeClass)
- **Argo CD** installed via Helm — namespace `argocd`, `server` exposed via **LoadBalancer**, running `--insecure` (TLS terminates at the LB)
- **Monitoring**: `kube-prometheus-stack` in the `monitoring` namespace, Grafana via LoadBalancer

> ⚠️ Public LB + `--insecure` + default admin is a **learning shortcut**. It gets hardened in **Module 6**.

---

## 🗺️ Learning Roadmap

Each module is hands-on and builds on the last.

| Module | Topic | Status |
|:------:|-------|:------:|
| **1** | Access Argo CD (UI + CLI), deploy your first Application, **Sync vs Health**, reconciliation & self-heal | ✅ Done |
| **2** | Sync policies deep-dive: **manual vs automated, prune, selfHeal**, sync waves & hooks | ✅ Done |
| **3** | Structure a **GitOps repo**, **Kustomize** overlays for dev/prod | ✅ Done |
| **4** | **Helm-based Applications** (and managing Karpenter NodePools / kube-prometheus via Argo) | ▶️ In progress |
| **5** | **App-of-Apps** pattern & **ApplicationSets** for scaling to many apps/envs | ⬜ Upcoming |
| **6** | **Projects, RBAC, SSO, secrets** (SOPS / Sealed Secrets / External Secrets), production hardening (drop `--insecure` + public LB) | ⬜ Upcoming |

---

## 📂 Repository Structure

```
Learn_ArgoCD/
├── module1/
│   └── guestbook_app.yaml            # First Application (manual sync)
├── module3/
│   ├── myapp-dev-app.yaml            # dev Application  (auto-sync)
│   ├── myapp-prod-app.yaml           # prod Application (manual sync — human gate)
│   └── myapp-gitops/
│       ├── base/                     # the ~80% that's identical
│       └── overlays/
│           ├── dev/                  # 1 replica, namespace myapp-dev
│           └── prod/                 # 3 replicas, namespace myapp-prod
└── module4/                          # Helm charts + Applications (in progress)
```

---

## Module 1 — Your First Application

### What an Application actually *is*

An **Application** is a **Custom Resource (CRD)** — a Kubernetes object of `kind: Application`, `apiVersion: argoproj.io/v1alpha1`. Installing Argo CD registered this CRD, so `kubectl get applications -n argocd` works just like `kubectl get pods`.

Think of it as a **contract with three answers** — *where's the source, where's the destination, how should it sync?* The `application-controller` reads this contract, asks the `repo-server` to render the source into real Kubernetes manifests, and continuously drives the destination to match.

### 🎯 Sync status vs Health status — the two independent axes

The concept students most often conflate. An Application **always reports two separate statuses**:

| Axis | Question it answers | Example values |
|------|--------------------|----------------|
| **Sync** | Does the cluster match Git? | `Synced` / `OutOfSync` |
| **Health** | Is the running workload actually OK? | `Healthy` / `Progressing` / `Degraded` |

They are **independent**:
- **Synced but Degraded** — Git was applied correctly, but the Deployment's pods are crash-looping.
- **OutOfSync but Healthy** — the running app is fine, but someone pushed a new commit not yet synced.

> **How health is computed:** Argo has built-in health assessments for common kinds — a Deployment is `Healthy` only when ready replicas meet spec; a `LoadBalancer` Service stays `Progressing` until the LB is provisioned. Custom checks (Lua) can be added for CRDs. → [Health docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/)

> 💡 **Key insight:** `OutOfSync` is a **diff report, not an error.**

### `syncPolicy` — manual vs automated

The single most important toggle. **Omit `automated:`** and the app only syncs when you click **Sync** / run `argocd app sync`. **Add it** and Argo syncs whenever Git changes.

### 🛠️ Hands-on commands

```bash
# a. Retrieve the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d ; echo

# b. Log in with the CLI (server runs --insecure, TLS at the LB)
argocd login <LB_HOSTNAME> --username admin --password <PASSWORD> --insecure

# c. Create and sync the application
kubectl apply -f module1/guestbook_app.yaml
kubectl get applications -n argocd
argocd app get guestbook          # → shows OutOfSync (manual policy)
argocd app sync guestbook         # → apply Git state to the cluster
kubectl get all -n guestbook
argocd app history guestbook
```

> **Gotchas solved here:**
> 1. Login failed because the zsh `%` no-newline marker got copied into the password → fix with `... | base64 -d ; echo` and an interactive password prompt.
> 2. `InvalidSpecError cluster not found` because `destination.server` was the EKS **public API URL** → use `https://kubernetes.default.svc` (the in-cluster name).

---

## Module 2 — Sync Policies, Waves & Hooks

This is where Argo CD goes from *"reports drift"* to *"enforces state,"* and where you learn to control **ordering** during a sync.

### The three layers of "how to sync"

```yaml
syncPolicy:
  automated:              # LAYER 1: WHEN does Argo sync?  (manual click vs auto on Git change)
    prune: true           # LAYER 2: WHAT is Argo allowed to do?  (delete resources removed from Git)
    selfHeal: true        # LAYER 2: (revert manual cluster changes back to Git)
  syncOptions:            # LAYER 3: HOW does each apply behave?  (create ns, replace, etc.)
    - CreateNamespace=true
# + sync waves & hooks    # ORDERING: in what ORDER do resources apply within one sync?
```

### `selfHeal` — the self-correcting behavior

| Value | Behavior |
|-------|----------|
| `false` *(default)* | Argo detects drift → marks `OutOfSync` → **waits for you**. |
| `true` | Argo detects drift → **automatically re-applies Git state**, usually within seconds. The cluster becomes impossible to drift from Git by hand. |

### `prune` — the deletion safety switch

| Value | Behavior |
|-------|----------|
| `false` *(default)* | Delete a manifest from Git → Argo marks the leftover `OutOfSync` but **leaves it running** (guards against accidental mass-deletion). |
| `true` | Delete a manifest from Git → Argo **deletes the resource** from the cluster. This is what makes Git *truly* the source of truth (nothing lingers). |

> **Why they're separate:** `selfHeal` is about *modified/missing* resources; `prune` is about resources that *should no longer exist*. Argo makes destructive actions **opt-in on purpose.**

**Also covered:** `syncOptions` (per-apply behavior like `CreateNamespace=true`), **sync waves** (`argocd.argoproj.io/sync-wave`), and **resource hooks** (`PreSync` / `Sync` / `PostSync`). → [Sync docs](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)

---

## Module 3 — GitOps Repo Structure & Kustomize Overlays

### Part 1 — The two-repo pattern *(interview gold)* 🏆

The single most important architectural idea in GitOps: **separate the application source repo from the configuration (deployment) repo.**

**Why separate them? (Interviewers love this question)**

1. **Separation of concerns** — CI (build/test) and CD (deploy) are different lifecycles. A README typo in the app repo shouldn't trigger a redeploy.
2. **Clean audit trail** — the config repo's git history *is* your deployment history. `git log` = "what got deployed, when, by whom."
3. **Blast radius / RBAC** — developers write app code, but deploys go through reviewed PRs on the config repo.
4. **No credential leakage** — Argo CD only needs **read** access to the config repo, never to your app source.
5. **Avoids the infinite loop** — if CI committed image tags back into the same repo Argo watches, that commit would re-trigger CI… forever. Two repos break it cleanly.

**The flow:**
```
CI builds myapp:v1.2.3 → pushes to registry → commits "image tag: v1.2.3" to the config repo
      → ArgoCD detects the change → syncs.   (Git = single source of truth for desired state)
```

### Part 2 — Why Kustomize here

Kustomize's core idea is **template-free customization** — **no `{{ }}` placeholders**. You keep valid, real YAML in a `base/`, then each **overlay patches it**. It's built into `kubectl` (`kubectl kustomize`) and natively supported by Argo CD.

- The **base** is the *"80% that's identical."*
- The **overlays** are the *"20% that differs per environment"* — replica counts, resource limits, image tags, namespace, config values.

### 🔍 See what Kustomize actually produces — *before* pushing

The step most people skip and then get confused by Argo CD. **Render the overlays locally and read the output:**

```bash
# from the repo root
kubectl kustomize module3/myapp-gitops/overlays/dev
echo "===================="
kubectl kustomize module3/myapp-gitops/overlays/prod
```

**What to verify you understand:**
- Dev Deployment is named `dev-myapp`, prod is `prod-myapp` (the `namePrefix`)
- Dev has `namespace: myapp-dev`, prod has `namespace: myapp-prod`
- Dev = **1 replica**, prod = **3 replicas**
- Both get the `app.kubernetes.io/name: myapp` label injected everywhere — including into the selector
- The image tag is identical for now (`nginx:1.27.0`) — you diverge them in the promotion exercise

> 💡 **The lesson:** what Argo CD deploys is *exactly* this rendered output. **Argo CD literally runs `kustomize build` on the overlay path and applies the result.** If it looks right here, it'll look right in the cluster.

### The dev/prod split

| Overlay | Replicas | Namespace | Sync policy |
|---------|:--------:|-----------|-------------|
| **dev** | 1 | `myapp-dev` | `automated:` → **auto-sync** |
| **prod** | 3 | `myapp-prod` | **no** `automated:` block → **manual sync** (human gate) |

> **Gotcha solved:** prod appeared "stuck" at `OutOfSync` / `Missing`. Cause: there was **no `automated:` syncPolicy** — that's **manual sync by design, not a bug**. dev auto-deploys; prod waits for `argocd app sync myapp-prod`. Reinforced: **`OutOfSync` is a diff report, not an error.**

---

## Module 4 — Helm-based Applications ▶️

**The one fact that anchors everything:**

> **Argo CD does not run `helm install`. It runs `helm template` and applies the rendered manifests.**

→ [Helm docs](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/): *"Argo CD… renders the chart using `helm template` and then deploys the resulting manifests."*

**Consequences (all favorite interview probes):**

| Question | Answer (because it's `helm template`) |
|----------|---------------------------------------|
| Where's the Helm release stored? | **Nowhere** — `helm list` shows nothing. No release object, no Tiller. Git is the source of truth. |
| How do you roll back? | **Not `helm rollback`** — re-sync a previous Git commit (or Argo's History/Rollback). |
| What happens to Helm hooks? | Argo **translates** `helm.sh/hook` into its own PreSync/Sync/PostSync hooks. |
| Can Argo and the Helm CLI manage the same app? | **No — they'll fight.** One owner: Argo. |

**Three source patterns:** (a) chart in Git, (b) chart from a Helm repo, (c) **multi-source** — chart from a Helm repo + *your values from Git* (the production pattern). **Values precedence:** `valueFiles` → `values` / `valuesObject` → `parameters` (highest).

---

## 📚 Reference

- [Argo CD Documentation](https://argo-cd.readthedocs.io/en/stable/)
- [OpenGitOps Principles](https://opengitops.dev/)
- [Declarative Setup — Applications & Projects](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)
- [Resource Health](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/)
- [Sync Options, Waves & Hooks](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)
- [Kustomize with Argo CD](https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/)
- [Helm with Argo CD](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)

---

*Built and maintained as a personal, interview-focused GitOps learning log. Every module is validated on a live EKS + Karpenter cluster.*
