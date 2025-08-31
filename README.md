# Homelab Apps on K3s with Helmfile (Vaultwarden • Open WebUI • Glance)

This repo documents how I deploy and operate my apps on a 3‑node K3s cluster using Longhorn for persistent storage.

- Cluster: K3s (3 nodes), Longhorn manages all PersistentVolumes (one SATA disk per node)
- Apps: Vaultwarden, Open WebUI, Glance
- Chart sources:

  - `vaultwarden` – `https://guerzon.github.io/vaultwarden`
  - `open-webui` – `https://helm.openwebui.com` (or `https://open-webui.github.io/helm-charts`)
  - `rphllc-glance` – `https://rphllc.github.io/helm/glance`

- This repo: stores **values/** per app and **helmfile.yaml** to deploy/upgrade everything

---

## Goals

1. **Repeatable deploys** via Helmfile
2. **Safe upgrades** with diffs & rollbacks
3. **Automated version bumps** via Renovate (charts + container images)
4. **Notifications** (Slack/Webhook) on pending updates & results
5. **Data safety** with Longhorn recurring snapshots/backups

---

## Directory layout

**Your repo today** (`RPHllc/k3s-apps`) has the following top-level layout:

```
.
├─ values/
├─ helmfile.yaml
└─ README.md
```

The files inside `values/` are referenced directly in `helmfile.yaml`. You can name them however you like (e.g., `values/vaultwarden.yaml`, `values/openwebui.yaml`, `values/glance.yaml`), as long as they match the paths in `helmfile.yaml`. In this README I used example filenames to illustrate the pattern; replace them with your actual filenames.

**Optional (recommended) additions**

```
.github/
└─ workflows/
   └─ helmfile-diff.yml      # CI that renders/diffs on PRs
.github/renovate.json        # Renovate to raise version-bump PRs
```

---

## One‑time setup

```bash
# Add app chart repos (they’re also declared in helmfile.yaml for idempotency)
helm repo add vaultwarden https://guerzon.github.io/vaultwarden
helm repo add open-webui https://helm.openwebui.com
helm repo add rphllc-glance https://rphllc.github.io/helm/glance
helm repo update

# Verify Longhorn is the default StorageClass (or set it per‑chart in values)
kubectl get storageclass

# Namespace(s)
kubectl create namespace apps || true
```

---

## Helmfile basics

Common commands I use with `helmfile.yaml` in repo root:

```bash
# Refresh repo indexes referenced by Helmfile
helmfile repos

# See what will change (non‑destructive)
helmfile diff

# Apply all releases (create/upgrade as needed)
helmfile apply

# Apply a single app
helmfile -l name=vaultwarden apply

# Sync to declared state (installs missing, upgrades drifted, deletes removed)
helmfile sync

# List releases Helmfile manages
helm list -A | grep -E 'vaultwarden|open-webui|glance'

# Roll back the last failed/undesired revision (per release)
helm rollback <release-name> <revision>

# Template locally to inspect rendered manifests
helmfile template
```

> Tip: always run `helmfile diff` before `apply` on upgrades.

---

## Release definitions (suggested pattern)

In `helmfile.yaml`, prefer **pinned versions** and pass storageClass name + PVC sizes explicitly. Example snippet (illustrative):

```yaml
releases:
  - name: vaultwarden
    namespace: apps
    chart: vaultwarden/vaultwarden
    version: 0.x.y # pin chart version
    values:
      - values/vaultwarden.yaml

  - name: open-webui
    namespace: apps
    chart: open-webui/open-webui
    version: 7.x.y # pin chart version
    values:
      - values/open-webui.yaml

  - name: glance
    namespace: apps
    chart: rphllc-glance/glance
    version: 0.x.y
    values:
      - values/glance.yaml
```

> Keep image tags pinned in each values file (e.g., `image.tag: "1.32.3"`) for reproducible rollouts.

---

## Values guidelines per app

**Global (applies to all where supported)**

```yaml
# values/EXAMPLE.yaml
fullnameOverride: <release-name>
image:
  repository: <repo>
  tag: '<pinned>'
  pullPolicy: IfNotPresent

resources:
  requests: { cpu: '100m', memory: '256Mi' }
  limits: { cpu: '2000m', memory: '2Gi' }

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        podAffinityTerm:
          topologyKey: kubernetes.io/hostname
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: <chart>

nodeSelector: {}

persistence:
  enabled: true
  storageClass: longhorn
  size: 5Gi
  accessModes: ['ReadWriteOnce']
  # Use RWX only when the app supports it. Vaultwarden should remain RWO.

# Longhorn volume hints (optional)
volumeClaimTemplate:
  dataLocality: best-effort # or "strict-local" if you prefer
```

**Vaultwarden**

- Single replica (app is not truly HA). Add a `PodDisruptionBudget` with `minAvailable: 1`.
- Expose via Ingress, set `ADMIN_TOKEN` via Secret, configure SMTP env.
- Backups handled by Longhorn recurring jobs (see below).

**Open WebUI**

- Mount a data PVC for uploads/pipelines when needed.
- If connecting to Ollama in‑cluster, reference the `Service` DNS name.

**Glance**

- Use your custom chart; keep `ingress.hosts`, theme, and `persistence` pinned.

---

## Upgrades workflow (manual, safe)

```bash
# 1) Update repo indices
helmfile repos

# 2) Preview differences
helmfile diff

# 3) Apply upgrades
helmfile apply

# 4) Verify
kubectl -n apps get pods
kubectl -n apps rollout status deploy/<release-name>

# 5) If issues, roll back a single release
helm rollback <release-name> <revision>
```

### Canary / staggered upgrades

- Label releases in Helmfile (`labels: {tier: core|noncore}`) and apply non‑core first:

  ```bash
  helmfile -l tier=noncore diff && helmfile -l tier=noncore apply
  helmfile -l tier=core   diff && helmfile -l tier=core   apply
  ```

---

## Automated version bumps

Use **Renovate** to open PRs when:

- New chart versions are published
- Container image tags have updates

Example `.github/renovate.json` (minimal):

```json
{
  "extends": ["config:recommended"],
  "labels": ["deps"],
  "enabledManagers": ["helm-values", "helmfile", "regex"],
  "regexManagers": [
    {
      "fileMatch": ["^helmfile\\.yaml$"],
      "matchStrings": ["version:\\s+(?<currentValue>[^#\\n]+)"],
      "depNameTemplate": "<chart>",
      "datasourceTemplate": "helm"
    }
  ],
  "helmfile": {
    "fileMatch": ["^helmfile\\.yaml$"]
  },
  "helm-values": {
    "fileMatch": ["^values/.*\\.ya?ml$"]
  },
  "packageRules": [
    { "matchManagers": ["helm-values"], "rangeStrategy": "pin" },
    { "matchManagers": ["helmfile"], "rangeStrategy": "pin" }
  ]
}
```

> Renovate will raise PRs. Merge when CI (see below) and manual `helmfile diff` look good.

---

## CI check (dry‑run & diff)

Create a GitHub Actions workflow (e.g., `.github/workflows/helmfile-diff.yml`) that runs on PRs:

```yaml
name: helmfile-diff
on:
  pull_request:
    paths:
      - 'helmfile.yaml'
      - 'values/**'
jobs:
  diff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v4
        with: { version: 'v3.15.4' }
      - uses: helmfile/helmfile-action@v1
        with:
          helmfile-args: 'repos'
      - uses: helmfile/helmfile-action@v1
        with:
          helmfile-args: 'template' # or 'diff' if you supply a kube context
```

> Optional: add a matrix to render each app. If you can expose a read‑only kubeconfig for a staging cluster, use `helmfile diff` to catch breaking changes earlier.

---

## Notifications

- **Slack/Webhook**: Add a workflow step that posts summary of Renovate PRs & helmfile diff to Slack.
- **kubewatch** (or Argo Events) to notify on Pod restarts / rollouts in `apps` namespace.
- **kube‑prometheus‑stack** + **Alertmanager** for cluster alerts; add the Longhorn dashboards.

---

## Longhorn data safety

- Set **recurring snapshots** (e.g., hourly keep=24) and **backups** to an external target (S3/MinIO/NFS) for Vaultwarden and other critical PVCs.
- Consider `dataLocality: strict-local` for latency‑sensitive workloads.
- Add a `PodDisruptionBudget` and `priorityClassName` for Vaultwarden to avoid eviction during upgrades.

Example Longhorn recurring job annotation on a PVC:

```yaml
metadata:
  annotations:
    recurring-job.longhorn.io/source: 'enabled'
    recurring-job-group.longhorn.io/critical: 'default'
```

(Or configure in the Longhorn UI.)

---

## Backup & restore (app level)

- **Vaultwarden**: enable internal backups to filesystem; Longhorn will back that up off‑cluster.
- **Open WebUI**: back up configuration/data PVC if you store uploads, RAG indexes, etc.
- **Glance**: back up its config PVC.

---

## Useful Helm & K8s commands

```bash
# List releases in a namespace
helm -n apps list

# Show a release values (merged)
helm -n apps get values <release> --all

# Check history of a release
helm -n apps history <release>

# Rollback
helm -n apps rollback <release> <revision>

# Show rendered manifests for a release
helm -n apps get manifest <release>

# K8s helpers
kubectl -n apps get pods -owide
kubectl -n apps describe deploy <release>
kubectl -n apps logs deploy/<release> --tail=200
kubectl -n apps get ingress
```

---

## Operational notes

- Always run `helmfile diff` before `apply`.
- Keep chart versions and image tags **pinned**; upgrade one app at a time if it’s a core service.
- For Vaultwarden, prefer a single replica (RWO PVC) and strong SMTP/ADMIN secrets.
- Validate Ingress/Certificates before upgrade windows.
- If you change `storageClass`, plan a **PVC migration** (backup/restore or `velero`).

---

## Roadmap (nice‑to‑haves)

- Pre‑prod (staging) namespace with same charts, smaller resources
- Periodic `helmfile apply` in CI to a staging cluster
- Policy checks (OPA/Gatekeeper) on rendered manifests
- Velero for cluster‑wide backup/restore

---

## Appendix: Example per‑app values skeletons

**values/vaultwarden.yaml**

```yaml
replicaCount: 1
image:
  repository: vaultwarden/server
  tag: '<pin>'

env:
  ADMIN_TOKEN: '<from-secret>'
  SMTP_ENABLED: 'true'
  SMTP_HOST: smtp.example.com
  SMTP_FROM: vault@example.com
  SMTP_PORT: '587'
  SMTP_SECURITY: starttls
  SMTP_USERNAME: vault@example.com
  SMTP_PASSWORD: '<from-secret>'

persistence:
  enabled: true
  storageClass: longhorn
  size: 10Gi
  accessModes: ['ReadWriteOnce']

ingress:
  enabled: true
  hosts: ['vaultwarden.example.com']
  tls:
    - hosts: ['vaultwarden.example.com']
      secretName: vaultwarden-tls
```

**values/open-webui.yaml**

```yaml
image:
  repository: ghcr.io/open-webui/open-webui
  tag: '<pin>'

ollama:
  host: http://ollama.svc.cluster.local:11434 # if used

persistence:
  enabled: true
  storageClass: longhorn
  size: 20Gi

ingress:
  enabled: true
  hosts: ['openwebui.example.com']
  tls:
    - hosts: ['openwebui.example.com']
      secretName: openwebui-tls
```

**values/glance.yaml**

```yaml
image:
  repository: ghcr.io/glanceapp/glance
  tag: '<pin>'

persistence:
  enabled: true
  storageClass: longhorn
  size: 1Gi

ingress:
  enabled: true
  hosts: ['glance.example.com']
  tls:
    - hosts: ['glance.example.com']
      secretName: glance-tls
```

---

## License

MIT
