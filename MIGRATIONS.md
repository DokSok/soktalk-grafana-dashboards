# Git-backed config migration plan — org-wide docs-as-code

**Purpose:** Move cluster config that currently lives only inside Kubernetes ConfigMaps/Secrets into
Git repos as the canonical source of truth, using the same pattern as the Grafana dashboard repo
(git-sync sidecar → emptyDir → app reads from mounted path). This makes PRs the review path for
changes and gives the org a recoverable, auditable config history.

**Parent pattern:** `doksok/soktalk-grafana-dashboards` (already working): git-sync sidecar on the
Grafana Deployment pulls a repo into an emptyDir; Grafana's file provisioner reads from it.

---

## Repos to create

### 1. `DokSok/soktalk-monitoring`  (primary)
Owns the Prometheus alerting + scraping stack as files. Contents:

```
soktalk-monitoring/
├── README.md
├── monitoring/
│   ├── service-monitors/
│   │   ├── prometheus.yaml
│   │   ├── prometheus-operator.yaml
│   │   ├── node-exporter.yaml
│   │   ├── kube-state-metrics.yaml
│   │   ├── traefik.yaml
│   │   ├── loki.yaml
│   │   ├── gatekeeper.yaml
│   │   ├── sealed-secrets-controller.yaml
│   │   ├── sealed-secrets-metrics.yaml
│   │   ├── velero.yaml
│   │   ├── metrics-server.yaml
│   │   └── promtail.yaml
│   ├── scrape-configs/
│   │   └── proxmox-api.yaml        # → Secret proxmox-api-scrape-config
│   └── prometheus-rules/
│       └── platform-alerts.yaml    # → PrometheusRule prometheusrule-platform-alerts
└── alerting/
    └── alertmanager.yml            # → ConfigMap alertmanager-general
```

What migrates here:
- All 11 existing ServiceMonitors (extract live ones from cluster, compare with repo files)
- `prometheusrule-platform-alerts` (already a file in repo; promote into `monitoring/prometheus-rules/`)
- `proxmox-api-scrape-config` Secret (extract from cluster, store plaintext in repo, sync into Secret)
- `alertmanager-general` ConfigMap (extract from cluster, store as `alerting/alertmanager.yml`, sync into ConfigMap)

### 2. `DokSok/soktalk-logging`  (secondary, smaller)
Owns Promtail config as files.

```
soktalk-logging/
├── README.md
└── promtail/
    └── config.yaml                 # → ConfigMap promtail-config (loki ns)
```

What migrates here:
- `promtail-config` ConfigMap from `loki` namespace (extract from cluster)

---

## Migration steps (in order)

### Pre-flight
1. Create repos via GitHub API (classic PAT — already provided, not repeated here).
2. Build repo directory trees locally, populate from cluster extracts.
3. Push initial commits.

### Per-repo application
For each repo, decide the sync mechanism:
- **ConfigMap-backed items that an app reads from a file path** (grafana-datasources, grafana-alert-rules,
  alertmanager-general, promtail-config): add a git-sync sidecar (or a shared git-sync DaemonSet pod)
  that pulls the repo into an emptyDir, and point the app's provisioner/mount at the synced path.
- **Secret-backed items** (proxmox-api-scrape-config): git-sync cannot write Secrets directly. Options:
  (a) keep a small init/synctoken that renders the repo file into a Secret on change, or
  (b) use the same git-sync emptyDir + a tiny sidecar that `kubectl create secret --dry-run=client -o yaml`
  + apply on change. Pick (b) for now — simple, auditable, no extra controller.
- **PrometheusRule** (already managed by PrometheusOperator from a YAML file): the file can live in the
  repo and get applied by a periodic apply job, or just stay as-is since it's already a single source file.
  Prefer: document it in the repo and apply from there; no sidecar needed since it's not a long-running
  reader.

### Verification
After each migration:
- Confirm the app sees the config (e.g. grafana API `/api/datasources`, alertmanager `/api/v2/alerts`,
  promtail logs, prometheus `/api/v1/status/config`).
- Confirm git-sync pod is Running and not crashlooping.
- Confirm the live ConfigMap/Secret matches the repo file (naively, via `kubectl get ... -o yaml`).

---

## Files produced in this repo

```
deploy/grafana/                 # already exists (dashboards, datasource, alert-rules, provisioning)
deploy/monitoring/              # NEW — soktalk-monitoring repo content
deploy/logging/                 # NEW — soktalk-logging repo content
deploy/migration-plan.md        # this file
deploy/extract-configs.py       # NEW — extract ConfigMaps/Secrets from cluster for repo population
```

---

## Open questions (decide before executing)

1. Two repos (`soktalk-monitoring`, `soktalk-logging`) or one (`soktalk-cluster-config`)?
   - Recommendation: two — monitoring and logging have different release cadence and different owners
     (even if both are doksok today). But if you want a single repo for simplicity, do one.
2. For Secret syncing (proxmox-api-scrape-config): accept the "sidecar applies a secret" approach,
   or defer that one (it's a scrape config for the Proxmox API, lower urgency)?
   - Recommendation: do it now so the pattern is proven, but keep it minimal.
3. Do we also want the Prometheus `additionalScrapeConfigs` Secret to be git-backed, or leave it as
   a one-off cluster Secret? It's already extracted above, so including it is cheap.

---

## Execution order (my plan)

1. Write this plan file (done — `deploy/migration-plan.md`).
2. Write `deploy/extract-configs.py` — one script that pulls a named ConfigMap or Secret out of the
   cluster into a local file, decoding Secrets for plaintext storage.
3. Run the extraction for each item above.
4. Create `DokSok/soktalk-monitoring` and push monitoring/* + alerting/.
5. Create `DokSok/soktalk-logging` and push promtail/config.yaml.
6. For each app, add git-sync sidecar + remount (Grafana datasources, Grafana alert-rules, Alertmanager,
   Promtail). Update the manifests in `deploy/` so they reflect reality.
7. For proxmox-api-scrape-config Secret: add a git-sync + secret-sync sidecar on the Prometheus pod (or
   a standalone sync pod).
8. Verify each.

---

## Out of scope for this pass

- Dashboard panel description pass (separate task).
- Git-backed document surface for non-config org docs (runbooks etc.) — that's a later, larger decision.
- Any config that is genuinely cluster-identity-native and shouldn't live in Git (e.g. TLS secrets,
  cloud credentials — leave those alone; this plan only touches non-secret or low-sensitivity configs).
- Sealed Secrets conversion — not doing that here; we're using plaintext-in-Private-Repo + git-sync for
  now because the repo is private. If you want sealed secrets later, that's a separate step on top of this.
