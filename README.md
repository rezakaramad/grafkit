# grafkit

Generic Grafana dashboards I maintain, published as OCI artifacts and consumed by the Grafana operator.

## Layout

```
dashboards/<name>/<name>.json   # one directory per dashboard
```

## Writing a generic dashboard

- Datasources are variables (`ds_metrics`, `ds_logs`, `ds_traces`), so no fixed UIDs.
- Environment-specific values such as namespace, cluster and service are variables too.
- CI rejects a dashboard that contains a hardcoded datasource UID.

## Why ORAS and OCI

**The problem.** A dashboard is just a JSON file. To use the same dashboard in several clusters or repos you would normally copy the file around, and the copies drift apart.

**OCI registries.** Container registries such as GHCR already solve "store a versioned thing and let anyone pull it by name". The OCI standard they follow is not limited to container images. A registry can hold any file as an *OCI artifact*.

**ORAS.** [ORAS](https://oras.land) (OCI Registry As Storage) is a small command-line tool that pushes and pulls ordinary files to and from an OCI registry. It is to files what `docker push` is to images.

**How it works here.**

1. CI runs `oras push` and uploads a dashboard JSON to `ghcr.io/rezakaramad/grafkit/<name>`.
2. The registry stores it and gives it a *digest* (`sha256:...`), a fingerprint of the exact content. A tag such as `latest` or `sha-<short>` is a movable label, but a digest never changes.
3. A consumer's `GrafanaDashboard` points at that reference, and the Grafana operator pulls the file from the registry.
4. Each consumer sets its own datasource names through `variables`, so one published dashboard works in every environment.

**What you get.**

- One published copy, many consumers, no copy-pasting JSON.
- Versioning and rollback: pin a digest to freeze a version, change the digest to upgrade or roll back.
- The same access control and hosting as your container images.

**Trying it by hand.**

```
oras login ghcr.io
oras push ghcr.io/rezakaramad/grafkit/otel-collector:test otel-collector.json:application/json
oras pull ghcr.io/rezakaramad/grafkit/otel-collector:test
```

ORAS is only the tool CI uses to publish. Consumers never need it, because the operator does the pulling.

## Build and publish

GitHub Actions (`.github/workflows/dashboards.yaml`) validates every dashboard and publishes them with ORAS.

- Pull requests: validate only.
- Push to `main`: publish to `ghcr.io/rezakaramad/grafkit/<name>`.
- Tags: `sha-<short>` and `latest`. Consumers should pin the digest.
- Manual run: use `workflow_dispatch`.

## Consuming a dashboard

```yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDashboard
metadata:
  name: otel-collector
spec:
  instanceSelector:
    matchLabels:
      dashboards: grafana
  oci:
    reference: ghcr.io/rezakaramad/grafkit/otel-collector@sha256:<digest>
    path: otel-collector.json
  variables:
    - name: ds_metrics
      value: mimir
```

Set `ds_logs` and `ds_traces` the same way for dashboards that use them.

## Adding a dashboard

Create `dashboards/<name>/<name>.json` and push. The workflow picks it up automatically.
