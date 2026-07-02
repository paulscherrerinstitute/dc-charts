# cron-chart

A small chart that deploys a single scheduled **CronJob**, with optional config
maps, secrets and volumes. Use it for periodic tasks that run on their own
schedule rather than as a long-running service (for a one-shot job attached to a
service deployment, use the job support in `generic-service-chart` instead).

## Compatibility

- **Chart API:** `apiVersion: v2`. Works on Helm **3.x** and **4.x**.
- Renders a `batch/v1` CronJob (Kubernetes ≥1.21).

## How it is used at PSI

Like `generic-service-chart`, each task ships a small values file and the CI
deploy step layers it on top of this chart, injecting deploy-time parameters and
secrets. Current consumers:

- **`scicat-to-pss`** (scicat-ci) — periodically pushes SciCat metadata into PSS.
- **`materialised-view`** (scicat-ci) — refreshes MongoDB materialised views from
  a mounted `mongosh` script (uses `configMaps` + `run` + `secrets`).
- **`proposal-sync`** (scilog-ci) — syncs proposals into SciLog, using an
  `initContainer` to pre-fetch assets before the run.

## Installing / uninstalling

```bash
helm install my-release path/to/cron -f my-values.yaml
helm delete my-release
```

> **Note:** Today the CI repos install this chart from a local path. Publishing it
> to a registry is separate future work.

## Parameters

| Parameter                  | Description                                                                   | Default     |
| -------------------------- | ----------------------------------------------------------------------------- | ----------- |
| `nameOverride`             | Partially override the generated name                                         | `""`        |
| `fullnameOverride`         | Fully override the generated name                                             | `""`        |
| `image.repository`         | Image name (templated)                                                        | `busybox`   |
| `image.tag`                | Image tag (templated)                                                         | `latest`    |
| `image.pullPolicy`         | Image pull policy                                                             | `Always`    |
| `cronjob.schedule`         | Cron schedule expression                                                      | `0 7 * * 1` |
| `cronjob.restartPolicy`    | Pod restart policy                                                            | `OnFailure` |
| `run.command` / `run.args` | Container command / arguments (templated)                                     | `nil`       |
| `env`                      | Environment variables (templated, k8s `env` syntax)                           | `nil`       |
| `initContainers`           | Init containers run before the job container (k8s syntax)                     | `nil`       |
| `volumes`                  | Pod volumes (templated, k8s syntax)                                           | `nil`       |
| `volumeMounts`             | Container volume mounts (templated, k8s syntax)                               | `nil`       |
| `configMaps`               | Map of `configMapName -> {key: value, ...}` (templated)                       | `{}`        |
| `secrets`                  | Map of `secretName -> {type, data: {...}}`. Values must be **base64-encoded** | `{}`        |

## Key behaviors

This chart shares the value-injection and secret conventions of
`generic-service-chart` — see its README for the full explanation:

- **Templated values:** `env`, `run`, `volumes`, `configMaps`, `secrets` and
  `image.*` are passed through Helm's `tpl`, so they may contain Go template
  expressions evaluated against the release (`{{ .Release.Name }}`, injected
  `{{ .Values.* }}`).
- **base64 secrets:** values under `secrets.<name>.data` must be base64-encoded.
  The chart validates this and fails the render otherwise.
- **Config-change rollout:** a checksum of `configMaps` and `secrets` is stored on
  the job-pod template, so changing config rolls the next scheduled run onto the
  new values.

## Example

```yaml
image:
  repository: "{{ .Values.ciRepository }}"
  tag: "{{ .Values.ciTag }}"
cronjob:
  schedule: "0 2 * * *"
run:
  command: ["python"]
  args: ["sync.py"]
env:
  - name: SCICAT_ENDPOINT
    value: "{{ .Values.scicatEndpoint }}"
secrets:
  "{{ .Release.Name }}-s":
    type: Opaque
    data:
      .env: "{{ .Values.secretsJson.SYNC_ENV }}"   # already base64-encoded
```

## Maintaining the chart

- Bump `version` in `Chart.yaml` (SemVer) on every change to the chart or its
  templates.
- Targets the Helm v2 chart API. No changes needed for Helm 4. Validate with
  `helm lint` and a `helm template` diff before/after any change.
