# Review: Flux hub-and-spoke GitOps demo

**Reviewer context:** Evaluating this demo as a foundation for deploying **20 apps** across **~30 managed clusters** from a **CAPI + Sveltos management cluster**.

---

## TL;DR

Your teammate is right: this follows Flux best practices. But "Flux best practice" and "simplest solution for our problem" are not the same thing. The complexity you're feeling is real, structural, and will multiply at scale. The demo is technically sound—the question is whether the abstraction cost is worth it for Gladia's fleet.

---

## 1. What the demo actually does (the mental model)

Before judging complexity, here's the chain of indirection for deploying one app to one cluster:

```
deploy/flux.yaml (ResourceSet "demo")
  └─ creates: GitRepository + Kustomization per app
       └─ reconciles: apps/app/deploy/
            ├─ input-providers.yaml (ResourceSetInputProvider × N variants)
            ├─ flux.yaml (ResourceSet "demo-app")
            │    └─ per input: HelmRepository + HelmChart + HelmRelease
            │         └─ HelmRelease uses kubeConfig to reach remote cluster
            └─ clusters/staging/gladia-eu-200/
                 └─ Kustomize generates ConfigMaps for valuesFrom
```

That's **5 layers of indirection** from "I pushed code" to "it runs on a cluster." Each layer is justified in isolation; together they stack into a system that requires significant Flux Operator expertise to debug.

---

## 2. What works well

### 2.1 Correct hub-and-spoke pattern

`HelmRelease.spec.kubeConfig.secretRef` is the standard way to deploy from a central Flux to remote clusters without installing Flux on every spoke. This is the right call for 30 clusters.

### 2.2 OCI-based Helm distribution

CI pushes charts to `oci://ghcr.io/...` — no Chartmuseum, no HTTP chart repos. Clean and modern.

### 2.3 Git-backed values via ConfigMaps

The `configMapGenerator` + `reconcile.fluxcd.io/watch: Enabled` label pattern is elegant: cluster-specific values live in Git, get rendered to ConfigMaps, and HelmReleases pick them up via `valuesFrom`. Value changes trigger reconciliation without touching the HelmRelease manifest.

### 2.4 Reusable CI workflow

`ci.yaml` as a `workflow_call` with `inputs.app` + path-filtered triggers per app. Scales to 20 apps by adding one small workflow file each.

### 2.5 Drift detection

`driftDetection.mode: enabled` with ignore rules for `/spec/replicas` on Deployments — practical for HPA coexistence.

### 2.6 Mise for local tooling

Pinned tool versions (`helm: 4`, `kubectl`, `yq`, `buildx`) and task scripts replace a Makefile with something more portable. Good DX choice.

---

## 3. Where the complexity lives (and why you're right to be concerned)

### 3.1 Two-layer ResourceSet is the core cost

| Layer | File | Creates |
|-------|------|---------|
| Platform | `deploy/flux.yaml` | `GitRepository` + one `Kustomization` per app |
| App fleet | `apps/app/deploy/flux.yaml` | `HelmRepo` + `HelmChart` + `HelmRelease` per cluster×variant |

The platform layer exists to avoid manually creating a `Kustomization` per app. The app layer exists to avoid manually creating a `HelmRelease` per cluster. Both are DRY — but **two nested template engines** (ResourceSet templating `<< inputs.* >>` inside Kustomize inside Flux) means three different YAML-processing systems touching your manifests before anything hits a cluster.

**Debugging path:** When a deployment fails, you need to check: Did the ResourceSet render correctly? Did Kustomize generate the right ConfigMaps? Did the HelmChart pull the right version? Did the HelmRelease reconcile? Did the kubeconfig work? That's a lot of layers to inspect.

### 3.2 Namespace-as-cluster-name on the hub

HelmRelease/HelmChart/HelmRepository all live in namespace `gladia-eu-200` on the **management cluster**, while workloads land in `targetNamespace: default` on the **remote cluster**. This is a Flux convention, but it's a source of confusion: `kubectl get helmrelease -n gladia-eu-200` runs on the hub, not on `gladia-eu-200`.

At 30 clusters, the hub gets 30 namespaces just for routing. That's manageable but adds cognitive load when troubleshooting.

### 3.3 Object count projection

For the current demo (1 app, 1 cluster, 2 variants), the hub creates:

| Object | Count |
|--------|-------|
| GitRepository | 1 |
| Kustomization | 1 |
| ResourceSet | 2 |
| ResourceSetInputProvider | 2 |
| HelmRepository | 2 |
| HelmChart | 2 |
| HelmRelease | 2 |
| ConfigMap | 3 |
| **Total** | **15** |

Projected at **20 apps × 30 clusters × 2 variants**:

| Object | Formula | Count |
|--------|---------|-------|
| GitRepository | 1 (shared) | 1 |
| Kustomization | 20 | 20 |
| ResourceSet (platform) | 1 | 1 |
| ResourceSet (app) | 20 | 20 |
| ResourceSetInputProvider | 20 × 30 × 2 | **1,200** |
| HelmRepository | 20 × 30 | **600** |
| HelmChart | 20 × 30 × 2 | **1,200** |
| HelmRelease | 20 × 30 × 2 | **1,200** |
| ConfigMap | 20 × 30 × 3 | **1,800** |
| **Total** | | **~6,000+** |

That's ~6,000 Flux-managed objects on the management cluster, all reconciling on intervals. It may work — Flux is designed for scale — but it's a significant control plane load and a lot of YAML to maintain/generate.

### 3.4 InputProvider file explosion

With `ResourceSetInputProvider` being static YAML, each app needs `apps × clusters × variants` provider files. For 20 apps × 30 clusters × 2 variants, that's **1,200 InputProvider resources** committed to Git. Even if you consolidate them into fewer files, someone has to author or generate them.

The current demo has 2 providers in one file. At scale, this is the biggest YAML-maintenance burden in the repo.

### 3.5 Blue/green doubles everything

The demo bakes in blue/green with two variants per cluster. Unless every app genuinely needs blue/green on every cluster, this doubles object count for no benefit. Most apps probably just need a single release per cluster, with blue/green reserved for the few that need zero-downtime switching.

---

## 4. Gaps that matter for production

### 4.1 No bootstrap story

`deploy/flux.yaml` (the root ResourceSet) needs to be installed *somehow* on the management cluster. The demo doesn't show this. Options:
- Sveltos `ClusterProfile` targeting the management cluster
- A `flux bootstrap`-equivalent that applies this manifest
- Manual `kubectl apply` (fragile)

This is the "who watches the watchmen" problem and it needs a documented answer.

### 4.2 No kubeconfig lifecycle

HelmReleases reference `<< inputs.cluster >>-kubeconfig` secrets, but there's no automation for creating or rotating them. In a CAPI environment, these typically come from CAPI `Cluster` resources or a secret syncing mechanism. Without this, adding a new cluster is a manual secret-creation step.

### 4.3 No promotion workflow

CI runs on PRs only (`pull_request` trigger). There's no `push` trigger for `main`, no tag-based release, and no mechanism to promote a pre-release chart (`0.0.0-pr-42-abc1234`) to a stable version. The `version: ">=0.0.0-0"` in InputProviders would auto-upgrade to any version — including broken PRs if they merged.

### 4.4 No Sveltos boundary definition

The demo doesn't address which resources Sveltos manages vs. Flux. Both are GitOps controllers that can deploy Helm charts. Running both on the same resources creates reconciliation fights. You need a clear contract:
- **Sveltos:** cluster addons, CNI, CSI, cert-manager, monitoring agents, etc.
- **Flux:** application workloads (this repo)

### 4.5 No monitoring or alerting

No `Alert` or `Provider` resources for Flux notifications. At 30 clusters, you need to know when a HelmRelease fails to reconcile — Slack/PagerDuty integration should be part of the base pattern.

---

## 5. Specific issues in the manifests

### 5.1 ~~`GitRepository` created per input~~ — not an issue

In `deploy/flux.yaml`, the `GitRepository` is inside `resourcesTemplate` alongside templated resources. This looks like it would render N copies, but the `GitRepository` has a **static name** (no `<< inputs.* >>`), so ResourceSet [deduplicates it](https://fluxoperator.dev/docs/crd/resourceset/#resource-deduplication) based on `apiVersion`, `kind`, `metadata.name` and `metadata.namespace` — shared resources are applied only once regardless of the number of inputs. This matches the [official monorepo example](https://fluxoperator.dev/docs/resourcesets/app-definition/#monorepo-example).

### 5.2 Version constraint is too loose

```yaml
version: ">=0.0.0-0"
```

This matches *any* semver including pre-release. In production, you want pinned versions per environment (e.g., `1.2.3` for staging, `1.2.2` for production) to prevent uncontrolled upgrades.

### 5.3 `targetNamespace: default` is a smell

All workloads land in `default` on the remote cluster. In a real deployment, each app should have its own namespace for isolation, RBAC, and resource quotas.

### 5.4 Chart is a `helm create` scaffold

`apps/app/charts/app/` is an unmodified `helm create` output with verbose comments. This is fine for a demo but should be stripped/customized before becoming a template for 20 apps. The boilerplate comments add noise.

### 5.5 `secretRef` commented out on GitRepository

```yaml
# secretRef:
#   name: github-auth
```

This works for a public repo. For Gladia's private monorepo, this will need to be enabled, and the secret needs to exist before the ResourceSet creates the GitRepository.

### 5.6 Missing `push` event in CI

`ci-app.yaml` only triggers on `pull_request`. When a PR merges to `main`, no artifact is built. Either:
- Add a `push` trigger to build release artifacts from `main`/tags
- Or accept that the PR artifact *is* the release (document this)

---

## 6. Licensing: Flux Operator is AGPL-3.0

`ResourceSet` and `ResourceSetInputProvider` (`fluxcd.controlplane.io/v1`) are **Flux Operator** APIs, licensed under **AGPL-3.0** — not the Apache 2.0 of the CNCF Flux toolkit.

| What | License | Cost |
|------|---------|------|
| Flux controllers (`*.toolkit.fluxcd.io`) | Apache 2.0 | Free |
| Flux Operator + ResourceSet | AGPL-3.0 | Free (OSS) |
| ControlPlane Enterprise for Flux | Commercial | Paid (hardened images, support, FIPS) |

AGPL-3.0 is fine for internal use (running on your own clusters). It becomes relevant if you distribute the software or expose it as a service. **Get legal sign-off before production.**

The alternative (if AGPL is blocked) is generating plain `HelmRelease` manifests with an external tool instead of using ResourceSet templating.

---

## 7. Alternatives to consider

Before committing to this pattern, the team should briefly evaluate:

### 7.1 Flux on every spoke (decentralized)

Install Flux on each of the 30 clusters. Each cluster reconciles its own `HelmRelease` from the same Git repo. No `kubeConfig` secrets, no hub bottleneck, no ResourceSet needed.

**Trade-off:** 30× Flux installations to manage (Sveltos can deploy Flux as an addon), but simpler per-cluster debugging and no hub blast radius.

### 7.2 Sveltos-only for app delivery

Sveltos already has `ClusterProfile` and `Profile` for fleet-wide Helm chart deployment with cluster selectors. If Sveltos is already managing the fleet, adding a *second* GitOps controller for apps introduces operational overhead.

**Trade-off:** Less mature Helm lifecycle than Flux (no drift detection, different reconciliation model), but eliminates the Flux layer entirely.

### 7.3 Hybrid: Sveltos for fleet targeting, Flux per-spoke

Sveltos deploys Flux + a base `GitRepository` on each spoke. Each cluster's Flux reconciles only its own slice of the monorepo. No ResourceSet, no hub kubeconfigs.

**Trade-off:** Best of both worlds for isolation, but more moving parts in the bootstrap chain.

### 7.4 ResourceSet without blue/green

Keep this exact pattern but drop the blue/green variants. One HelmRelease per app per cluster. Cuts object count in half and removes a layer of ConfigMap indirection. Add blue/green later, only for apps that need it.

---

## 8. Recommendations

### Adopt if:
- You want a **single pane of glass** for all app deployments on the hub
- Your team is comfortable with Flux Operator and `ResourceSet` templating
- You invest in **automation for InputProvider generation** (not hand-maintaining 1,200 YAML blocks)
- You're OK with AGPL-3.0 for the Flux Operator

### Simplify first:
1. **Drop blue/green from the base pattern** — add it per-app when needed
2. **Pin chart versions per environment** — replace `>=0.0.0-0` with explicit versions
3. **Extract the GitRepository** from the ResourceSet template — it should be a standalone resource
4. **Add a promotion workflow** — CI on `main`/tags, not just PRs
5. **Document the bootstrap** — how does the root ResourceSet get installed? How do kubeconfig secrets get created?
6. **Add Flux alerts** — `Provider` + `Alert` resources for Slack/PagerDuty
7. **Define the Sveltos boundary** — write it down: "Sveltos manages X, Flux manages Y"

### Evaluate before scaling:
- Run a **load test** with 20 apps × 5 clusters to see how the hub handles ~1,500 reconciling objects
- Compare hub-spoke Flux vs. per-spoke Flux with Sveltos bootstrap (section 7.3) — the operational simplicity difference may justify the extra Flux installations
- Get **legal sign-off on AGPL-3.0** before building production infrastructure on Flux Operator

---

## 9. Verdict

| Aspect | Rating | Comment |
|--------|--------|---------|
| Flux correctness | Good | Follows documented patterns for hub-and-spoke, OCI, valuesFrom |
| Production readiness | Incomplete | Missing bootstrap, kubeconfig lifecycle, promotion, monitoring |
| Complexity | High | 5 layers of indirection; two nested template engines; object count scales multiplicatively |
| Monorepo fit | Partial | Good CI pattern, but InputProviders don't scale without generation tooling |
| Sveltos integration | Missing | No boundary defined; potential for controller conflict |

**Bottom line:** This is a correct Flux demo, not a production-ready deployment system. Your feeling that it's complicated is valid — the abstraction tower is tall. Whether it's *too* complicated depends on whether the team builds automation around it (InputProvider generation, monitoring, bootstrap) or tries to maintain 1,200 YAML blocks by hand.

The hardest question isn't "is this good Flux?" (it is) — it's "do we need this much Flux, or would a simpler pattern serve 20 apps × 30 clusters with less operational overhead?"
