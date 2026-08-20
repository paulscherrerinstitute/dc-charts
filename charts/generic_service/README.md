# generic-service

A single, highly configurable chart used to deploy most of the simple services
that make up SciCat and SciLog. Instead of maintaining one chart per service,
each application ships a small `values.yaml` and reuses this chart: the image,
environment, config maps, secrets, ingress, probes and (optionally) a job or
cron job are all driven from values.

## Compatibility

- **Chart API:** `apiVersion: v2`.
- **Helm:** works unchanged on Helm **3.x** and **4.x** (v2 charts are supported
  by both. No migration is required for Helm 4).
- **Kubernetes:** the ingress `apiVersion` is selected automatically from the
  cluster version (`networking.k8s.io/v1` on ≥1.19, falling back to the older
  `v1beta1`/`extensions` groups), so the same chart renders correctly across
  clusters.

## How it is used at PSI

The chart is generic. Each service supplies its own values file (see
`scicat-ci/helm/configs/<service>/values.yaml`). The CI deploy step points Helm
at this chart and layers the service values on top, injecting a few
deploy-time parameters and secrets:

```bash
helm upgrade --install <release> \
  oci://ghcr.io/paulscherrerinstitute/dc-charts/generic-service --version 2.0.0 \
  -f helm/configs/<service>/values.yaml \
  --set db=<release>-<env> \
  --set-json secretsJson='{ ... }'
```

scicat-ci runs that upgrade through helmfile. Each component declares the chart,
the version and the values in `helm/configs/<service>/helmfile.yaml.gotmpl`.
scilog-ci still calls helm from its own workflow.

The values file can reference those injected parameters with Go template syntax
(`{{ .Values.db }}`, `{{ .Values.secretsJson.FOO }}`, `{{ .Release.Name }}`).
See [Templated values](#templated-values) below. This is what makes one chart
serve many services.

## Installing / uninstalling

Install with a service values file and a release name:

```bash
helm install my-release \
  oci://ghcr.io/paulscherrerinstitute/dc-charts/generic-service --version 2.0.0 \
  -f my-values.yaml
```

Uninstall:

```bash
helm delete my-release
```

Always pin `--version`. Version 2.0.0 is a breaking change against 1.0.0, which
templates each ingress annotation on its own. See [secret annotations](#secret-annotations).

## Parameters

> `(templated)` values are run through Helm's `tpl` function. They can hold
> template expressions such as `{{ .Values.ciTag }}` or `{{ .Release.Name }}`.
> Helm fills these in at render time. See [Templated values](#templated-values) below.

### Common

| Parameter          | Description                           | Default |
| ------------------ | ------------------------------------- | ------- |
| `nameOverride`     | Partially override the generated name | `""`    |
| `fullnameOverride` | Fully override the generated name     | `""`    |

### Image

| Parameter                | Description                                            | Default   |
| ------------------------ | ------------------------------------------------------ | --------- |
| `image.repository`       | Image name (templated)                                 | `busybox` |
| `image.tag`              | Image tag (templated, defaults to `.Chart.AppVersion`) | `latest`  |
| `image.pullPolicy`       | Image pull policy                                      | `Always`  |
| `image.tty`              | Allocate a TTY for the container                       | `nil`     |
| `image.imagePullSecrets` | List of image pull secrets                             | `nil`     |

### Workload

| Parameter         | Description                                             | Default |
| ----------------- | ------------------------------------------------------- | ------- |
| `replicaCount`    | Number of Deployment replicas                           | `1`     |
| `strategy`        | Deployment update strategy (k8s `spec.strategy` syntax) | `nil`   |
| `affinity`        | Pod affinity/anti-affinity rules                        | `nil`   |
| `securityContext` | Pod security context                                    | `nil`   |
| `initContainers`  | Init containers (templated, k8s syntax)                 | `nil`   |
| `run.command`     | Container command (`command:`, templated)               | `nil`   |
| `run.args`        | Container arguments (`args:`, templated)                | `nil`   |
| `env`             | Environment variables (templated, k8s `env` syntax)     | `nil`   |
| `envFrom`         | Environment sources (templated, k8s `envFrom` syntax)   | `nil`   |
| `volumes`         | Pod volumes (templated, k8s syntax)                     | `nil`   |
| `volumeMounts`    | Container volume mounts (templated, k8s syntax)         | `nil`   |

### Config maps and secrets

| Parameter    | Description                                                                                                                         | Default |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------- | ------- |
| `configMaps` | Map of `configMapName -> {key: value, ...}`. Values are templated. e.g. `{cm1: {k1: v1, k2: v2}}`                                   | `{}`    |
| `secrets`    | Map of `secretName -> {type, data: {key: value, ...}}` (templated). Values must be **base64-encoded** (see [below](#base64-secret-enforcement)) | `{}`    |

Changing `configMaps` or `secrets` triggers a rolling restart of the Deployment
automatically. A checksum of both is stored as a pod annotation, so an
`upgrade` that only changes config still rolls the pods.

### Probes

| Parameter            | Description                                      | Default |
| -------------------- | ------------------------------------------------ | ------- |
| `probeChecksEnabled` | Add liveness/readiness probes to the container   | `true`  |
| `probeChecks`        | Probe tuning merged into both probes (templated) | `nil`   |
| `service.probePath`  | HTTP path used by the probes                     | `/`     |

### Storage

| Parameter             | Description                                     | Default         |
| --------------------- | ----------------------------------------------- | --------------- |
| `storage.name`        | PVC name                                        | `<release>-pvc` |
| `storage.size`        | Requested storage size (required to enable PVC) | `nil`           |
| `storage.accessModes` | PVC access mode                                 | `ReadWriteOnce` |
| `storage.annotations` | PVC annotations                                 | `nil`           |

A PersistentVolumeClaim is only rendered when `storage` is set.

### Service

| Parameter              | Description             | Default     |
| ---------------------- | ----------------------- | ----------- |
| `service.type`         | Kubernetes Service type | `ClusterIP` |
| `service.externalPort` | Service port            | `80`        |
| `service.internalPort` | Container target port   | `3000`      |

### Ingress

A single ingress is configured under `ingress`. To create several ingresses for
the same service (e.g. different annotations per path), provide a list under
`ingresses` instead. When set, it takes precedence over `ingress`.

| Parameter                                    | Description                                                                                                    | Default    |
| -------------------------------------------- | -------------------------------------------------------------------------------------------------------------- | ---------- |
| `ingress.enabled`                            | Create the ingress                                                                                             | `false`    |
| `ingress.name`                               | Ingress name                                                                                                   | `fullname` |
| `ingress.className`                          | Ingress class                                                                                                  | `nil`      |
| `ingress.annotations`                        | Ingress annotations. Each value is templated on its own (see [below](#secret-annotations))                     | `{}`       |
| `ingress.hosts[].host`                       | Host (templated)                                                                                               | `nil`      |
| `ingress.hosts[].paths[].path` / `.pathType` | Path and path type                                                                                             | `/`        |
| `ingress.tls[].hosts[]`                      | Hosts covered by a TLS cert (templated)                                                                        | `nil`      |
| `ingress.tls[].secretName`                   | Secret holding the TLS cert (templated)                                                                        | `nil`      |
| `ingresses`                                  | List of ingress objects (same shape as `ingress`), overrides `ingress` when present                            | `nil`      |

### Jobs and cron jobs

| Parameter        | Description                                                                       | Default |
| ---------------- | --------------------------------------------------------------------------------- | ------- |
| `jobContainers`  | List of containers (templated, k8s syntax) to run as a Job. Enables the job stack | `nil`   |
| `jobFromCronJob` | Run the job by triggering a paused CronJob via RBAC (see [below](#jobs))          | `true`  |

### Tests

| Parameter | Description                                                    | Default |
| --------- | -------------------------------------------------------------- | ------- |
| `test`    | Pod spec (templated) run by `helm test`. k8s Pod `spec` syntax | `nil`   |

## Key behaviors

### Templated values

Most value fields (`env`, `volumes`, `image.*`, `configMaps`, `secrets`,
`ingress.hosts`, `initContainers`, `probeChecks`, `test`, …) are passed through
Helm's `tpl` function, so their contents may themselves contain Go template
expressions that Helm fills in at render time. This is the mechanism that lets a
service file reference deploy-time parameters, e.g.:

```yaml
env:
  - name: PSS_DATABASE
    value: "{{ .Values.db }}"          # injected with --set db=...
  - name: PSS_MONGODB_URL
    valueFrom:
      secretKeyRef:
        name: "{{ .Release.Name }}-s"  # injected release name
        key: pss_mongodb_url
```

### base64 secret enforcement

Values under `secrets.<name>.data` must already be base64-encoded. The chart
validates this (`validateSecret`) and fails the render with
*"Please b64 encode your secrets!"* otherwise. This keeps plaintext out of the
rendered manifests and matches how secrets are injected from CI.

```yaml
secrets:
  "{{ .Release.Name }}-s":
    type: Opaque
    data:
      pss_mongodb_url: "{{ .Values.secretsJson.PSS_MONGODB_URL }}"  # already b64
```

### secret annotations

An annotation can carry secret-derived data, e.g. an IP allow-list. Reference the
secret like any other value:

```yaml
annotations:
  nginx.ingress.kubernetes.io/whitelist-source-range: "{{ .Values.secretsJson.WHITELISTED_CIDRS }}"
```

**Store the value in plain text.** Unlike `secrets.<name>.data`, an Ingress
annotation is a literal string, so base64 would reach the Ingress as base64.

Each annotation value is templated on its own, never as part of the serialised
map. So the reference above resolves, the result is never templated again, and a
broken annotation elsewhere on the same Ingress cannot print this value in its
error.

### Jobs

Setting `jobContainers` enables one-shot jobs (e.g. database migrations) that run
on `post-install`/`post-upgrade`.

- With `jobFromCronJob: true` (default), the chart renders a **paused CronJob**
  holding the real job spec, plus a small **Job + ServiceAccount + Role/RoleBinding**
  whose only task is to `kubectl create job --from=cronjob/...`. This lets the
  same job be re-run later from the CronJob without a redeploy.
- With `jobFromCronJob: false`, the containers run directly as a Helm hook Job.

## Examples

### Multiple ingresses

```yaml
ingresses:
  - enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
      cert-manager.io/cluster-issuer: letsencrypt-prod
    hosts:
      - host: "{{ .Values.host }}"
        paths:
          - path: "/"
            pathType: Prefix
    tls:
      - hosts: ["{{ .Values.host }}"]
        secretName: "scicat-be-certificate"
  - enabled: true
    name: backend-login
    annotations:
      kubernetes.io/ingress.class: nginx
      nginx.ingress.kubernetes.io/whitelist-source-range: "{{ .Values.secretsJson.WHITELISTED_CIDRS }}"
    hosts:
      - host: "{{ .Values.host }}"
        paths:
          - path: /api/v3/auth/login
            pathType: Exact
```

### A migration job

```yaml
jobContainers:
  - name: migrate
    image: "{{ .Values.ciRepository }}:{{ .Values.ciTag }}"
    command: ["/bin/sh", "-c"]
    args: ["npm run migrate:db:up"]
```

### Helm test

```bash
helm install my-release path/to/generic_service --set test="$(cat << 'EOF'
containers:
  - name: wget
    image: busybox
    command: ['wget']
    args: ['{{ include "helm_chart.fullname" . }}:{{ .Values.service.externalPort }}']
EOF
)"
helm test my-release
```

### Scale

Either `kubectl scale`, or upgrade with a new `replicaCount`.

## Maintaining the chart

- Bump `version` in `Chart.yaml` (SemVer) on every change to the chart or its
  templates. Consumers pin to a version once the chart is published.
- The chart targets Helm v2 chart API and needs no changes for Helm 4. When the
  CI runners move to Helm 4, this chart is expected to render and deploy
  identically. Validate with `helm lint` and a `helm template` diff before/after.
