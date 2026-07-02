# generic-service-chart

A single, highly configurable chart used to deploy most of the simple services
that make up SciCat and SciLog. Instead of maintaining one chart per service,
each application ships a small `values.yaml` and reuses this chart: the image,
environment, config maps, secrets, ingress, probes and (optionally) a job or
cron job are all driven from values.

## Compatibility

- **Chart API:** `apiVersion: v2`.
- **Helm:** works unchanged on Helm **3.x** and **4.x** (v2 charts are supported
  by both; no migration is required for Helm 4).
- **Kubernetes:** the ingress `apiVersion` is selected automatically from the
  cluster version (`networking.k8s.io/v1` on ≥1.19, falling back to the older
  `v1beta1`/`extensions` groups), so the same chart renders correctly across
  clusters.

## How it is used at PSI

The chart is generic; each service supplies its own values file (see
`scicat-ci/helm/configs/<service>/values.yaml`). The CI deploy step points Helm
at this chart and layers the service values on top, injecting a few
deploy-time parameters and secrets:

```bash
helm upgrade --install <release> path/to/generic_service \
  -f helm/configs/<service>/values.yaml \
  --set db=<release>-<env> \
  --set-json secretsJson='{ ... }'
```

The values file can reference those injected parameters with Go template syntax
(`{{ .Values.db }}`, `{{ .Values.secretsJson.FOO }}`, `{{ .Release.Name }}`) —
see [Templated values](#templated-values) below. This is what makes one chart
serve many services.

## Installing / uninstalling

Install with a service values file and a release name:

```bash
helm install my-release path/to/generic_service -f my-values.yaml
```

Uninstall:

```bash
helm delete my-release
```

> **Note:** the chart is currently consumed by path from the CI repos. Publishing
> it as a versioned package to a registry is tracked separately (DATACGRP-792);
> once published, the install command becomes `helm install my-release <repo>/generic-service-chart`.

## Parameters

### Common

| Parameter          | Description                            | Default |
| ------------------ | -------------------------------------- | ------- |
| `nameOverride`     | Partially override the generated name  | `""`    |
| `fullnameOverride` | Fully override the generated name      | `""`    |

### Image

| Parameter                  | Description                                          | Default   |
| -------------------------- | ---------------------------------------------------- | --------- |
| `image.repository`         | Image name (templated)                               | `busybox` |
| `image.tag`                | Image tag (templated; defaults to `.Chart.AppVersion`) | `latest`  |
| `image.pullPolicy`         | Image pull policy                                    | `Always`  |
| `image.tty`                | Allocate a TTY for the container                     | `nil`     |
| `image.imagePullSecrets`   | List of image pull secrets                           | `nil`     |

### Workload

| Parameter          | Description                                                        | Default |
| ------------------ | ----------------------------------------------------------------- | ------- |
| `replicaCount`     | Number of Deployment replicas                                     | `1`     |
| `strategy`         | Deployment update strategy (k8s `spec.strategy` syntax)           | `nil`   |
| `affinity`         | Pod affinity/anti-affinity rules                                  | `nil`   |
| `securityContext`  | Pod security context                                              | `nil`   |
| `initContainers`   | Init containers (templated; k8s syntax)                           | `nil`   |
| `run.command`      | Container command (`command:`)                                    | `nil`   |
| `run.args`         | Container arguments (`args:`)                                     | `nil`   |
| `env`              | Environment variables (templated; k8s `env` syntax)              | `nil`   |
| `envFrom`          | Environment sources (templated; k8s `envFrom` syntax)           | `nil`   |
| `volumes`          | Pod volumes (templated; k8s syntax)                              | `nil`   |
| `volumeMounts`     | Container volume mounts (templated; k8s syntax)                  | `nil`   |

### Config maps and secrets

| Parameter    | Description                                                                                       | Default |
| ------------ | ------------------------------------------------------------------------------------------------ | ------- |
| `configMaps` | Map of `configMapName -> {key: value, ...}`. Values are templated. e.g. `{cm1: {k1: v1, k2: v2}}` | `{}`    |
| `secrets`    | Map of `secretName -> {type, data: {key: value, ...}}`. Values must be **base64-encoded** (see [below](#base64-secret-enforcement)) | `{}` |

Changing `configMaps` or `secrets` triggers a rolling restart of the Deployment
automatically — a checksum of both is stored as a pod annotation, so an
`upgrade` that only changes config still rolls the pods.

### Probes

| Parameter            | Description                                            | Default |
| -------------------- | ----------------------------------------------------- | ------- |
| `probeChecksEnabled` | Add liveness/readiness probes to the container        | `true`  |
| `probeChecks`        | Probe tuning merged into both probes (templated)      | `nil`   |
| `service.probePath`  | HTTP path used by the probes                          | `/`     |

### Storage

| Parameter             | Description                                    | Default            |
| --------------------- | ---------------------------------------------- | ------------------ |
| `storage.name`        | PVC name                                        | `<release>-pvc`    |
| `storage.size`        | Requested storage size (required to enable PVC) | `nil`              |
| `storage.accessModes` | PVC access mode                                 | `ReadWriteOnce`    |
| `storage.annotations` | PVC annotations                                 | `nil`              |

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
`ingresses` instead; when set, it takes precedence over `ingress`.

| Parameter                | Description                                                                                     | Default    |
| ------------------------ | ---------------------------------------------------------------------------------------------- | ---------- |
| `ingress.enabled`        | Create the ingress                                                                             | `false`    |
| `ingress.name`           | Ingress name                                                                                    | `fullname` |
| `ingress.className`      | Ingress class                                                                                   | `nil`      |
| `ingress.annotations`    | Ingress annotations. Keys prefixed with `b64/` are base64-decoded at render (see [below](#base64-annotations)) | `{}` |
| `ingress.hosts[].host`   | Host (templated)                                                                                | `nil`      |
| `ingress.hosts[].paths[].path` / `.pathType` | Path and path type                                                          | `/`        |
| `ingress.tls[].hosts[]`  | Hosts covered by a TLS cert (templated)                                                        | `nil`      |
| `ingress.tls[].secretName` | Secret holding the TLS cert (templated)                                                       | `nil`      |
| `ingresses`              | List of ingress objects (same shape as `ingress`); overrides `ingress` when present            | `nil`      |

### Jobs and cron jobs

| Parameter          | Description                                                                        | Default |
| ------------------ | --------------------------------------------------------------------------------- | ------- |
| `jobContainers`    | List of containers (templated, k8s syntax) to run as a Job. Enables the job stack | `nil`   |
| `jobFromCronJob`   | Run the job by triggering a paused CronJob via RBAC (see [below](#jobs))           | `true`  |

### Tests

| Parameter | Description                                                          | Default |
| --------- | ------------------------------------------------------------------- | ------- |
| `test`    | Pod spec (templated) run by `helm test`. k8s Pod `spec` syntax      | `nil`   |

## Key behaviors

### Templated values

Most value fields (`env`, `volumes`, `image.*`, `configMaps`, `secrets`,
`ingress.hosts`, `initContainers`, `probeChecks`, `test`, …) are passed through
Helm's `tpl` function, so their contents may themselves contain Go template
expressions evaluated against the release. This is the mechanism that lets a
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

Values under `secrets.<name>.data` must already be base64-encoded; the chart
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

### base64 annotations

Ingress annotations whose key starts with `b64/` have their value base64-decoded
at render time and the prefix stripped. Use this for annotations that carry
secret-derived data (e.g. an IP allow-list) so the plaintext is not committed:

```yaml
annotations:
  b64/nginx.ingress.kubernetes.io/whitelist-source-range: "{{ .Values.secretsJson.WHITELISTED_IPS }}"
```

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
      b64/nginx.ingress.kubernetes.io/whitelist-source-range: "{{ .Values.secretsJson.WHITELISTED_IPS }}"
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
  templates; consumers pin to a version once the chart is published.
- The chart targets Helm v2 chart API and needs no changes for Helm 4. When the
  CI runners move to Helm 4, this chart is expected to render and deploy
  identically — validate with `helm lint` and a `helm template` diff before/after.
