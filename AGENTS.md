# Application Management Guide

This repository manages a single k3s cluster with Flux. Treat Git as the source
of truth for cluster state: make application changes under `kubernetes/`, commit
them, and let Flux reconcile them into the cluster. Do not use direct `kubectl
apply` commands for normal application changes.

## Mandatory Approval Boundary

Agents must **NEVER commit changes, deploy changes, or reconcile Flux or
Kubernetes resources without explicit user approval for that specific action**.
Preparing and validating local file changes does not grant permission to publish
or apply them.

Without explicit user approval, agents must not:

- Run `git commit`, `git push`, or create/merge a pull request.
- Run deployment commands such as `kubectl apply`, `helm install`, or
  `helm upgrade`.
- Run Flux reconciliation commands, including `flux reconcile` or
  `task cluster:reconcile`.
- Trigger any equivalent operation that changes the live cluster or publishes
  repository changes.

Approval for one action does not imply approval for later actions. For example,
approval to commit does not authorize pushing, deploying, or reconciling.

## Reconciliation Flow

Flux applies the repository in this order:

1. `kubernetes/bootstrap/` installs the Flux controllers and CRDs. This directory
   is used only during initial bootstrap and is not reconciled by Flux.
2. `kubernetes/flux/config/` creates the `home-kubernetes` GitRepository and the
   top-level `cluster` Kustomization.
3. The `cluster` Kustomization reconciles `kubernetes/flux/`, which defines Helm
   and Git sources, shared cluster variables, and the `cluster-apps`
   Kustomization.
4. `cluster-apps` reconciles `kubernetes/apps/`. Flux recursively discovers the
   namespace-level Kustomizations there.
5. Each namespace-level `kustomization.yaml` includes its `namespace.yaml` and
   one or more application `ks.yaml` files.
6. Each `ks.yaml` creates a Flux Kustomization pointing at the application's
   manifests, usually in an `app/` directory.
7. The application Kustomization installs a HelmRelease and any related
   resources such as secrets, config maps, monitoring, certificates, RBAC, or
   custom resources.

Most Flux resources use a 30-minute reconciliation interval. A GitHub webhook
may trigger reconciliation sooner, or an operator can run:

```sh
task cluster:reconcile
```

## Application Layout

Applications are grouped by Kubernetes namespace:

```text
kubernetes/apps/<namespace>/
|-- namespace.yaml
|-- kustomization.yaml
`-- <app>/
    |-- ks.yaml
    `-- app/
        |-- kustomization.yaml
        |-- helmrelease.yaml
        `-- secret.sops.yaml       # when needed
```

Follow the existing nearby application as the primary template. Common
variations include:

- Additional sibling directories such as `cluster/`, `config/`, `rules/`,
  `certificates/`, or `storage-class/`.
- Multiple Flux Kustomizations in one `ks.yaml` when an operator must be ready
  before its custom resources are applied.
- Raw Kubernetes resources instead of a HelmRelease.
- Shared Gatus monitoring from `kubernetes/templates/gatus/internal`.

## Adding Or Changing An App

For a new application:

1. Choose the namespace that owns the workload. Add a new namespace directory
   only when the app does not fit an existing grouping.
2. Create `kubernetes/apps/<namespace>/<app>/ks.yaml`.
3. Point `spec.path` at the directory containing the app's Kustomize resources.
4. Add the `ks.yaml` to the namespace's `kustomization.yaml`.
5. Add an OCIRepository alongside the application when the chart source does
   not already exist, then include it in the application's `kustomization.yaml`.
6. Put the HelmRelease and supporting resources in `app/`, and list every
   resource in `app/kustomization.yaml`.
7. Add explicit `dependsOn` entries for infrastructure that must be healthy
   first, such as `ingress-nginx`, `cloudnative-pg-cluster`, an operator, issuer,
   or storage configuration.

Keep Flux Kustomization names unique across the cluster. Existing `ks.yaml`
files generally set:

- `metadata.namespace: flux-system`
- `spec.targetNamespace` to the workload namespace
- `sourceRef.name: home-kubernetes`
- `prune: false`
- `wait: true`
- `interval: 30m`, `retryInterval: 1m`, and `timeout: 5m`
- `commonMetadata.labels.app.kubernetes.io/name`
- `postBuild.substitute.APP` when shared templates use the app name

Preserve these defaults unless the application has a concrete reason to differ.
Because pruning is disabled throughout this repository, removing a manifest
from Git may not delete the live Kubernetes resource. Plan and perform removals
deliberately.

## Helm Releases

Before starting development on any application, always check the upstream
sources for the latest available Helm chart version and container image
version. Do not rely on versions already present in the repository or on cached
knowledge.

Define Helm chart sources as OCI registries, and keep each chart's OCIRepository
alongside the application that consumes it.

Most workloads use Flux `HelmRelease` resources. Pin chart versions and image
tags; image digests are preferred where the existing app pattern uses them.
Renovate scans Kubernetes manifests and proposes dependency updates.

Helm sources are colocated with their applications and are normally referenced
from HelmReleases in the workload namespace. Many standalone applications use
the `bjw-s` `app-template` chart, while infrastructure components use their
upstream charts.

Match the neighboring HelmRelease conventions for:

- install and upgrade remediation
- rollback behavior
- probes and resource requests/limits
- security contexts
- services, ingress, persistence, and monitoring
- Reloader annotations when configuration or secrets should trigger restarts

## Variables And Secrets

Non-sensitive shared values are stored in
`kubernetes/flux/vars/cluster-settings.yaml`. Sensitive shared values are stored
in `kubernetes/flux/vars/cluster-secrets.sops.yaml`.

The top-level Flux Kustomizations inject both resources through
`postBuild.substituteFrom`, and `cluster-apps` patches the same substitution and
SOPS decryption settings into child Flux Kustomizations. Manifests can therefore
use variables such as `${SECRET_DOMAIN}`, `${TIMEZONE}`, and load balancer
addresses. App-specific `postBuild.substitute` values such as `${APP}` and
Gatus overrides are defined in the app's `ks.yaml`.

Store application secrets in files named `*.sops.yaml`. The root `.sops.yaml`
encrypts only `data` and `stringData` fields in Kubernetes secret manifests.
Encrypt secrets before committing:

```sh
sops --encrypt --in-place kubernetes/apps/<namespace>/<app>/app/secret.sops.yaml
```

Never commit plaintext credentials, Age private keys, decrypted temporary
files, kubeconfigs, or command output containing secret values. Do not edit the
encrypted payload manually; decrypt with SOPS, make the change, and re-encrypt.

Some generated ConfigMaps disable Flux substitution with the
`kustomize.toolkit.fluxcd.io/substitute: disabled` annotation. Preserve that
annotation when the embedded application configuration contains dollar syntax
that Flux must not interpret.

## Networking, DNS, And Monitoring

Internal web applications generally use an nginx ingress at:

```text
<app>.${SECRET_DOMAIN}
```

TLS certificates are managed by cert-manager. Public DNS exposure is opt-in:
the ingress must include the `external-dns.alpha.kubernetes.io/target`
annotation used by existing public applications. Do not add this annotation to
an internal-only app.

To add the standard internal Gatus check:

1. Include `../../../../templates/gatus/internal` in the app's
   `kustomization.yaml`.
2. Set `postBuild.substitute.APP` in `ks.yaml`.
3. Set optional `GATUS_SUBDOMAIN`, `GATUS_PATH`, or `GATUS_STATUS` substitutions
   when the defaults are not correct.

Use ServiceMonitor, PodMonitor, PrometheusRule, and Gatus resources consistently
with neighboring monitored applications.

## Validation And Operations

Before finishing an application change:

1. Confirm every new file is reachable from a Kustomization.
2. Confirm every HelmRelease source exists and chart/image versions are pinned.
3. Confirm `dependsOn` names exactly match the referenced Flux Kustomizations.
4. Confirm every `*.sops.yaml` file is encrypted and no decrypted artifact is
   staged.
5. Build changed Kustomize directories when `kustomize` or `kubectl` is
   available.
6. Run YAML linting or the configured pre-commit hooks when available.
7. Review `git diff` for accidental secret disclosure or unrelated changes.

Useful cluster checks:

```sh
task cluster:kustomizations
task cluster:helmreleases
task cluster:pods
task cluster:resources
flux get ks -A
flux get hr -A
```

When debugging, start with the Flux Kustomization, then the HelmRelease, then
the workload pods and events. Prefer fixing the declarative manifests in Git
over making persistent live-cluster edits.

## Git Commits

- Use the Conventional Commits standard for commit messages.
- Format commit subjects as `<type>(<scope>): <description>` when a scope is
  useful, or `<type>: <description>` when no clear scope applies.
- Prefer common types such as `feat`, `fix`, `docs`, `chore`, `refactor`,
  `test`, `ci`, and `build`.
- Include a reasonably detailed commit body for non-trivial changes. The body
  should explain what changed, why it changed, and any important operational or
  validation notes.
- Keep the subject concise and imperative. Put detail in the body instead of
  making the subject long.
