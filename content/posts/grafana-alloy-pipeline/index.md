---
title: "Building a Kubernetes Observability Pipeline with Grafana Alloy"
date: "2026-03-19"
tags: ["Kubernetes", "Observability", "Grafana", "Prometheus", "Alloy", "Helm"]
ShowToc: true
TocOpen: true
---

In my [previous post on Kubernetes metrics](https://florianschick.com/posts/k8smetrics/) I walked through what the various metric endpoints in a cluster actually expose — kubelet, cAdvisor, kube-state-metrics, and friends. That post answered the *what*. This one answers the *how*: how do you actually collect all of it reliably, ship it to Grafana Cloud, and not blow your observability budget in the process?

The answer we landed on is a self-contained Helm chart built around **Grafana Alloy** as the central scraping agent. Here's how it works.

---

## The Stack at a Glance

The chart composes four Helm dependencies:

| Component | Role |
|---|---|
| **Grafana Alloy** | Central observability agent — scrapes metrics, collects logs, receives traces |
| **Prometheus** | Local in-cluster storage for full-fidelity metrics |
| **kube-state-metrics** | Exposes Kubernetes object state as metrics |
| **prometheus-node-exporter** | Exposes node-level hardware and OS metrics |

Alloy is the brain. Everything else either feeds it targets or stores what it forwards.

---

## The Dual-Write Pattern

The central design decision is **dual-write**: every metric gets written to two places.

```
prometheus.scrape "kubelet" {
  forward_to = [
    prometheus.remote_write.local.receiver,   // always
    prometheus.relabel.kubelet.receiver,       // filtered → Grafana Cloud
  ]
}
```

- **Local Prometheus** receives everything, unfiltered. This is your full-fidelity data for in-cluster dashboards, alerting, and debugging.
- **Grafana Cloud** receives only a curated allow-list per component. Grafana Cloud is priced by series, so you want to be surgical about what crosses that boundary.

The cloud write path is toggled by a single values flag:

```yaml
enableCloudWrite: true
```

When it's `false`, the relabel stages for cloud filtering are never rendered into the Alloy config at all — the template just points `forward_to` at local only. This makes local development or air-gapped environments trivially easy.

---

## Shared Discovery Targets

One thing worth calling out explicitly: all scraping pipelines share a single Kubernetes discovery target, defined once in `targets.alloy`:

```alloy
discovery.kubernetes "pods" {
  role = "pod"
}

discovery.kubernetes "nodes" {
  role = "node"
  selectors {
    role = "node"
  }
}
```

This is deliberate. Each `discovery.kubernetes` component opens a watch against the Kubernetes API server. If you naively define one per pipeline you'd put unnecessary load on the API server as your pipeline count grows. Defining them once and referencing `discovery.kubernetes.pods.targets` everywhere is the [recommended approach](https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.kubernetes/#selectors).

---

## Pipeline Structure

Every scraping pipeline follows the same three-stage pattern:

```
discovery.relabel  →  prometheus.scrape  →  [prometheus.relabel]  →  remote_write
    (pre-scrape)                               (post-scrape filter)
```

### Stage 1 — Pre-scrape relabeling (`discovery.relabel`)

This is where targets are filtered and enriched *before* any HTTP request goes out. It's cheap — no network calls, just label manipulation on the discovered target list.

Here's what a typical pod pipeline looks like, using Traefik as an example:

```alloy
discovery.relabel "traefik" {
  targets = discovery.kubernetes.pods.targets

  // Only keep pods with this Kubernetes label
  rule {
    source_labels = ["__meta_kubernetes_pod_label_app_kubernetes_io_name"]
    regex         = "traefik"
    action        = "keep"
  }

  // Only keep the container port we care about
  rule {
    source_labels = ["__meta_kubernetes_pod_container_port_number"]
    regex         = "9100"
    action        = "keep"
  }

  // Attach cluster metadata to every target
  rule {
    action      = "replace"
    replacement = "my-cluster"
    target_label = "cluster"
  }
}
```

The `keep` rules do the heavy lifting: out of hundreds of pod targets discovered by the shared `discovery.kubernetes "pods"`, only the handful that match both filters survive. The scraper never even tries to reach the others.

### Stage 2 — Scraping (`prometheus.scrape`)

The scrape job is straightforward. For pod targets:

```alloy
prometheus.scrape "traefik" {
  job_name        = "integrations/traefik"
  targets         = discovery.relabel.traefik.output
  scrape_interval = "60s"
  scrape_timeout  = "10s"
  scheme          = "http"

  clustering {
    enabled = true
  }

  forward_to = [
    prometheus.remote_write.local.receiver,
    prometheus.relabel.traefik_remote.receiver,
  ]
}
```

The `clustering { enabled = true }` block is worth noting. Alloy runs as a clustered StatefulSet — when multiple replicas are running, they coordinate which replica scrapes which target. This avoids duplicate metrics and distributes the load automatically.

Node-level endpoints (kubelet, cAdvisor) look slightly different — they route through the Kubernetes API proxy so Alloy doesn't need direct node network access:

```alloy
discovery.relabel "kubelet" {
  targets = discovery.kubernetes.nodes.targets

  // Route through the API server proxy instead of hitting nodes directly
  rule {
    target_label = "__address__"
    replacement  = "kubernetes.default.svc.cluster.local:443"
  }
  rule {
    source_labels = ["__meta_kubernetes_node_name"]
    regex         = "(.+)"
    replacement   = "/api/v1/nodes/${1}/proxy/metrics"
    target_label  = "__metrics_path__"
  }
}

prometheus.scrape "kubelet" {
  scheme                 = "https"
  bearer_token_file      = "/var/run/secrets/kubernetes.io/serviceaccount/token"
  tls_config {
    ca_file              = "/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
    insecure_skip_verify = false
    server_name          = "kubernetes"
  }
  // ...
}
```

This is cleaner than poking holes in node firewall rules and reuses the service account token Alloy already has.

### Stage 3 — Post-scrape filtering (`prometheus.relabel`)

This stage only exists on the cloud write path. After scraping, a second relabel pass drops every metric that isn't on the allow-list for that component before forwarding to Grafana Cloud:

```alloy
prometheus.relabel "traefik_remote" {
  forward_to = [prometheus.remote_write.remote.receiver]

  // keep only allow-listed metrics
  rule {
    source_labels = ["__name__"]
    regex         = "traefik_entrypoint_requests_total|traefik_entrypoint_request_duration_seconds_bucket|traefik_service_requests_total|traefik_service_request_duration_seconds_bucket|traefik_router_requests_total"
    action        = "keep"
  }
}
```

The allow-lists themselves live in a `default-allow-lists/` directory — one YAML file per component listing the metric names to keep. A Helm helper template reads them at render time and generates the regex. This means tuning what goes to Grafana Cloud is a values change, not a template change:

```yaml
metricsTuning:
  traefik:
    useDefaultAllowList: true
    includeMetrics: []   # additional metrics to keep on top of the default list
    excludeMetrics: []   # metrics to explicitly drop even if allow-listed
```

---

## Log Collection

Alloy also handles log collection via Loki. The pattern mirrors the metrics pipelines:

1. `discovery.relabel "pod_logs"` — discovers all pods, extracts `namespace`, `pod`, `container`, and `app` labels, constructs the log file path from the pod UID.
2. A second relabel stage filters out pods annotated with `k8s.grafana.com/logs.autogather: "false"`.
3. `local.file_match` and `loki.source.file` tail the log files from the node filesystem (mounted at `/var/log/pods`).
4. `loki.process` parses CRI-O/containerd or Docker log format, drops noisy health probe lines, and moves high-cardinality labels like `pod` to Loki structured metadata (keeping index cardinality low).
5. A final `loki.process "logs_service"` attaches the cluster label before forwarding to `loki.write.remote`.

The health probe dropping is configurable:

```yaml
logsTuning:
  alloy:
    loki:
      stageDropProbeLogs:
        enabled: true
        expression: "(?i)(GET /actuator/health)"
```

---

## Packaging as a Helm Chart

The whole thing ships as a single Helm chart. There are two parts worth digging into: how the Alloy config files get into the cluster, and how the metric filtering rules get generated from plain YAML lists.

### Building the ConfigMap

Each `.alloy` pipeline is a real file on disk inside `config/`. The chart has a single ConfigMap template that picks them all up automatically:

```yaml
# templates/alloy-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alloy-config
data:
  {{- $files := .Files }}
  {{- $root := . }}
  {{- range $path, $_ := .Files.Glob "config/*" }}
  {{ base $path }}: |
    {{- tpl ($files.Get $path) $root | nindent 4 }}
  {{- end }}
```

There are two things happening here that are easy to miss.

First, `Files.Glob "config/*"` iterates over every file in the `config/` directory and drops it into the ConfigMap as a separate key — the filename becomes the key, the file content becomes the value. Adding a new pipeline is just adding a new `.alloy` file. No template changes, no registration step.

Second, and more importantly: `tpl ($files.Get $path) $root`. The `tpl` function treats each `.alloy` file as a **Go template** and renders it with the full Helm context before writing it into the ConfigMap. This is what makes constructs like this work inside `.alloy` files:

```alloy
// This is inside traefik.alloy — rendered by Helm before it reaches Alloy
rule {
  action      = "replace"
  replacement = "{{ $.Values.clusterName }}"
  target_label = "cluster"
}

{{- if not $.Values.enableCloudWrite }}
  forward_to = [prometheus.remote_write.local.receiver]
{{- else }}
  forward_to = [prometheus.remote_write.local.receiver, prometheus.relabel.traefik_remote.receiver]
{{- end }}
```

At `helm template` time, `{{ $.Values.clusterName }}` gets replaced with the actual cluster name, and the `if/else` block collapses to whichever branch applies. By the time the ConfigMap lands in the cluster, it contains fully resolved Alloy configuration — no templating left, just plain Alloy syntax that Alloy can read directly.

### Generating the Metric Filter Rules

The filtering rules in the post-scrape `prometheus.relabel` blocks aren't hand-written — they're generated by a shared Helm template helper called `metricFiltering`. Here's the full implementation:

```
{{- define "metricFiltering" -}}
{{- $allow := fromYamlArray (.root.Files.Get (printf "default-allow-lists/%s.yaml" .component)) }}
{{- $include := index .root.Values.metricsTuning .component "includeMetrics" | default (list) }}
{{- if index .root.Values.metricsTuning .component "useDefaultAllowList" }}
{{- $regexList := concat $allow $include }}
rule {
  action        = "keep"
  source_labels = ["__name__"]
  regex         = {{ $regexList | join "|" | quote }}
}
{{- else if gt (len $include) 0 }}
rule {
  action        = "keep"
  source_labels = ["__name__"]
  regex         = {{ $include | join "|" | quote }}
}
{{- end }}
{{- if index .root.Values.metricsTuning .component "excludeMetrics" }}
rule {
  action        = "drop"
  source_labels = ["__name__"]
  regex         = {{ index .root.Values.metricsTuning .component "excludeMetrics" | join "|" | quote }}
}
{{- end }}
{{- end }}
```

Walking through what it does:

1. **Load the allow-list** — `Files.Get` reads the matching file from `default-allow-lists/<component>.yaml` and `fromYamlArray` parses it into a Go slice. For Traefik that file looks like:
   ```yaml
   - traefik_entrypoint_requests_total
   - traefik_entrypoint_request_duration_seconds_bucket
   - traefik_service_requests_total
   - traefik_service_request_duration_seconds_bucket
   - traefik_router_requests_total
   ```

2. **Merge with `includeMetrics`** — any additional metric names from `values.yaml` are concatenated onto the allow-list with `concat $allow $include`.

3. **Emit a `keep` rule** — the merged list is joined with `|` into a single regex and written as an Alloy relabel rule. So a five-item allow-list becomes:
   ```alloy
   rule {
     action        = "keep"
     source_labels = ["__name__"]
     regex         = "traefik_entrypoint_requests_total|traefik_entrypoint_request_duration_seconds_bucket|..."
   }
   ```

4. **Emit a `drop` rule** — if `excludeMetrics` is set, a second rule is appended that explicitly drops those metric names even if they matched the keep rule. This is useful when you want to use the default allow-list but carve out a specific metric that's noisy for your environment.

The helper is called from within each `.alloy` pipeline file — which, as we just saw, are themselves rendered as Go templates:

```alloy
prometheus.relabel "traefik_remote" {
  forward_to = [prometheus.remote_write.remote.receiver]

  {{- include "metricFiltering" (dict "component" "traefik" "root" .) | nindent 2 }}
}
```

The end result is that the output of a `helm template` for this block looks like fully resolved Alloy config — the `include` call is gone, replaced by the generated relabel rules. The Alloy process in the cluster never knows any of this happened.

---

## What This Gets You

The pattern solves a few things that tend to bite teams as their cluster grows:

- **No scrape duplication** — one shared discovery target, clustering-aware scrape jobs.
- **Cost control without losing data** — full fidelity locally, curated subset to the cloud. Tunable per component without touching templates.
- **Easy onboarding for new services** — add a `.alloy` file following the same three-stage pattern, add an allow-list YAML, add a `metricsTuning` block to values. Done.
- **No Prometheus scrape config sprawl** — Prometheus itself is configured with `scrape_configs: []`. Alloy owns all scraping and pushes via remote write. Prometheus is just storage.

If you're already familiar with what the Kubernetes metric endpoints expose (if not, [start here](https://florianschick.com/posts/k8smetrics/)), this architecture gives you a clean way to collect all of it without the usual operational headaches.
