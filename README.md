# Grafana Dashboards

[![Grafana](https://img.shields.io/badge/grafana-dashboard-F46800?logo=grafana)](https://grafana.com/)
[![Prometheus](https://img.shields.io/badge/prometheus-exporter-E6522C?logo=prometheus)](https://prometheus.io/)
[![License](https://img.shields.io/github/license/rfmoz/grafana-dashboards)](https://github.com/rfmoz/grafana-dashboards/blob/master/LICENSE)
[![Since](https://img.shields.io/badge/since-2017-blue)](https://github.com/rfmoz/grafana-dashboards)
[![GitHub stars](https://img.shields.io/github/stars/rfmoz/grafana-dashboards)](https://github.com/rfmoz/grafana-dashboards/stargazers)

## Node Exporter Full

- For node_exporter
- Monitor Linux system.

Only requires the default job_name: node, add as many targets as you need in `/etc/prometheus/prometheus.yml`.

```yaml
  - job_name: node
    static_configs:
      - targets: ['localhost:9100']
```

Recommended for prometheus-node-exporter the arguments `--collector.systemd` and `--collector.processes` because the graph uses some of their metrics.

> - `timeInterval` in the Grafana data source has to be set accordingly to the `scrape_interval` configured in Prometheus. You can do this by navigating to connections > Data sources > Prometheus and set Scrape Interval under Interval behaviour. When using provisioning, this is set with the attribute `jsonData.timeInterval`.
> - For prometheus-node-exporter v.0.16 or older, use `node-exporter-full-old.json`
> - Thanks to the [PCP project](http://pcp.io) for documenting the values reported by the kernel in `/proc` (in their `/pmdas/linux/help` src file mainly).



## Node Exporter BSD

- For node_exporter in BSD system
- Monitor BSD system.

Only requires a configured target under any `job_name`.



## HAProxy

- For HAProxy v.2 or above compiled with Prometheus support
- Monitor HAProxy service.

Only requires a configured target under any `job_name`.



## Apache Full

- Monitor Apache service.
- For [apache_exporter](https://github.com/Lusitaniae/apache_exporter)

Only requires a configured target under any `job_name`. For example:

```yaml
  - job_name: 'apache'
    static_configs:
        - targets:
           - server_hostname:9117
```

> Also available as [apache-http-mixin](https://github.com/grafana/jsonnet-libs/tree/master/apache-http-mixin) in Grafana jsonnet-libs.



## NFS Full

- For node_exporter
- Monitor all NFS and NFSd exported values.

Check that the process was started with the arguments `--collector.nfs` and `--collector.nfsd`.

The same as Node Exporter Full. Only requires the default `job_name: node`, add as many targets as you need in `/etc/prometheus/prometheus.yml`.



## BIND 9 Full

- For [prometheus-bind-exporter](https://github.com/prometheus-community/bind_exporter)
- Monitor BIND 9 service.

Required configuration in `/etc/bind/named.conf.options`:

```c++
statistics-channels {
  inet 127.0.0.1 port 8053 allow { 127.0.0.1; };
};
```

On Grafana, it only requires a configured target under any `job_name`. For example:

```yaml
  - job_name: 'bind'
    static_configs:
        - targets:
           - server_hostname:9000
```



## Unbound Full

- For [unbound_exporter](https://github.com/letsencrypt/unbound_exporter)
- Monitor Unbound DNS service.

Required configuration in `/etc/unbound/unbound.conf`:

```yaml
server:
        extended-statistics: yes

remote-control:
        control-enable: yes
        control-interface: /run/unbound.ctl
```

On Grafana, it only requires a configured target under any `job_name`. For example:

```yaml
  - job_name: 'unbound'
    static_configs:
        - targets:
           - server_hostname:9167
```



## Disk Usage

- For [node_exporter](https://github.com/prometheus/node_exporter)
- Monitor per-mount usage and inode pressure across a fleet, with thresholds that match the included alert rules.

Lives in [`cluster/disk-usage.json`](cluster/disk-usage.json).

Every panel — including the `Host` variable — is driven by a single recording rule, so the dashboard renders empty until it exists:

```yaml
groups:
  - name: disk-usage
    rules:
      - record: node_filesystem:usage_ratio
        expr: |
          1 - (
            node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs|ramfs|fuse.*|nsfs|vfat"}
            /
            node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs|ramfs|fuse.*|nsfs|vfat"}
          )
```

The filter matches on `fstype` only, never `mountpoint`, so a newly created mount is picked up without editing this repo. `vfat` is excluded because firmware and boot partitions regularly report near-full by design.

Scraping follows the default job, so nothing changes if `node` is already configured:

```yaml
  - job_name: node
    static_configs:
      - targets: ['localhost:9100']
```

> - The datasource is chosen by the dashboard's own `ds_prometheus` variable, and `Host` lists `label_values(node_filesystem:usage_ratio, instance)` — both empty until the rule above is loaded.
> - Warning and critical thresholds are raw ratios on a `percentunit` gauge (0.85 / 0.95), so the panel, the PromQL and the alert rule read identically.
> - The `Critical mounts` row pins one mount and one instance per gauge — edit it for your own fleet. The `Usage by mount` and `Trend` rows stay unpinned and cover everything the rule sees.



## Importing dashboards with a GrafanaDashboard CR

A dashboard that imports cleanly through the Grafana UI can fail as a `GrafanaDashboard` CR, and an edited dashboard can appear never to have changed. Three separate causes, all reproducible.

### 1. `__inputs` placeholders are not resolved for you

Dashboards published to grafana.com are exported for sharing: a `__inputs` array plus `${DS_PROMETHEUS}` placeholders in every panel. The import wizard prompts for the datasource and rewrites them in place. A CR posts the JSON verbatim, so the placeholders survive and every panel reports *the datasource is not valid*.

See what the file expects:

```bash
jq -c '.__inputs // "none"' dash.json
```

Resolve it declaratively:

```yaml
spec:
  datasources:
    - inputName: DS_PROMETHEUS     # exactly __inputs[].name, character for character
      datasourceName: prometheus   # the datasource's name as Grafana knows it, not its UID
```

A dashboard that carries its own `datasource` template variable instead of a `__inputs` array needs none of this.

### 2. The CRD is the contract, not the documentation

Upstream docs track master and can describe fields your installed version does not have. Ask the cluster instead:

```bash
kubectl explain grafanadashboard.spec.datasources --recursive

kubectl get crd grafanadashboards.grafana.integreatly.org -o json \
  | jq -r '.spec.versions[] | select(.name=="v1beta1")
           | .schema.openAPIV3Schema.properties.spec.properties | keys[]' | sort
```

Always apply with `--dry-run=server`. `--dry-run=client` accepts an unknown field and the apply becomes a silent no-op; only the server decodes against your schema.

### 3. `resyncPeriod` re-applies the cache, it does not re-fetch it

Content fetched from `spec.url` is cached — default **24h**, controlled by `contentCacheDuration`. Pushing a new revision can therefore stay invisible for a day while every reconcile reports success: a green status that says nothing about freshness.

```yaml
spec:
  url: https://example.com/dash.json
  resyncPeriod: 10m
  contentCacheDuration: 1m   # must be shorter than resyncPeriod
```

Verify what Grafana serves rather than what the CR reports:

```bash
curl -s -u admin:$GRAFANA_PASS "$GRAFANA/api/dashboards/uid/disk-usage" | jq '.dashboard.panels | length'
```

### 4. A private source repository needs `urlAuthorization`

Making the repository private returns **404** to an unauthenticated `spec.url` fetch. The URL that resolved yesterday stops working, and because the content cache is still warm the dashboard keeps rendering the last bytes it saw while every reconcile reports success — another stale success, invisible until the cache expires.

`raw.githubusercontent.com` does accept HTTP Basic auth: any username with a token as the password returns 200.

**Step 1 — confirm the field exists on your build.** The operator's documentation tracks upstream master, so a field may be documented but absent, or shaped differently:

```bash
kubectl explain grafanadashboard.spec.urlAuthorization.basicAuth --recursive
```

**Step 2 — create the Secret.** Scope the token to *Contents: Read* on that single repository; fetching a dashboard never needs write access.

```bash
kubectl create secret generic dashboard-git -n grafana \
  --from-literal=username=x-access-token \
  --from-literal=password=$PAT \
  --dry-run=client -o yaml | kubectl apply -f -
```

**Step 3 — add both fields to the CR.** `contentCacheDuration` belongs here too: it is what made this failure invisible in the first place.

```yaml
spec:
  instanceSelector:
    matchLabels:
      dashboards: grafana
  url: https://raw.githubusercontent.com/owner/repo/refs/heads/master/dash.json
  urlAuthorization:
    basicAuth:
      username:
        name: dashboard-git
        key: username
      password:
        name: dashboard-git
        key: password
  resyncPeriod: 10m
  contentCacheDuration: 1m    # must be shorter than resyncPeriod
```

**Step 4 — apply, then verify.** Server-side dry run first: an unknown field would otherwise apply cleanly and change nothing.

```bash
kubectl apply -f grafanadashboard-disk.yaml -n grafana --dry-run=server
kubectl apply -f grafanadashboard-disk.yaml -n grafana

kubectl get grafanadashboard disk-usage -n grafana -o jsonpath='{.status.conditions}'; echo
```

Then confirm the fetch actually succeeded instead of trusting the status:

```bash
kubectl logs -n grafana -l app.kubernetes.io/name=grafana-operator --since=5m \
  | grep -iE 'disk-usage|404|failed to (fetch|get)'
```

Any hit here is the failure. Silence means the fetch went through.
