# proposals-cronjob

Deploys the **DUO proposal sync** as one Kubernetes CronJob per PSI facility.

## What is DUO, and what does this sync?

The [Digital User Office (DUO)](https://duo.psi.ch/) is PSI's central web portal
where external users submit beamtime proposals, experimental reports and manage
their visits. It was originally developed at the Swiss Light Source to handle the
large user community of a synchrotron facility, and now serves all PSI facilities.

Proposal metadata (who is coming, for which experiment, on which instrument) must
also exist in SciCat so that collected data can be attached to the right proposal
and access can be granted to the right people. This chart runs a small job that
calls the DUO API (`https://duo.psi.ch/duo/api.php/v1/`) and imports/updates
proposals into SciCat. The running code lives in the application image, this
chart only schedules it.

Because each PSI facility opens proposals on its own cadence, the chart fans a
list of facilities out into **one CronJob each**, so every facility can have its
own schedule (and, for neutron proposals, its own reference year).

## Facilities

The default `duo_facilities` list covers the PSI large-scale facilities:

| `name`     | Facility                                         |
| ---------- | ------------------------------------------------ |
| `pgroups`  | Proposal groups (cross-facility)                 |
| `sls`      | Swiss Light Source (synchrotron)                 |
| `swissfel` | SwissFEL (X-ray free-electron laser)             |
| `smus`     | SμS, muon source (SmuS)                         |
| `sinq`     | SINQ, spallation neutron source (uses a `year`) |

Each entry becomes a CronJob named `<release>-<name>[-<year>]`, invoked with
`DUO_FACILITY` and `DUO_YEAR` environment variables so the same image knows which
facility/year to sync.

## Compatibility

- **Chart API:** `apiVersion: v2` (`kubeVersion: ">=1.9"`). Works on Helm **3.x** and **4.x**.

## How it is used at PSI

Like the other charts, the CI deploy step layers a small values file on top of
this chart. It injects deploy-time parameters and secrets at install. The
current consumer is `proposals` (scicat-ci), which schedules the per-facility
DUO to SciCat proposal sync.

## Installing / uninstalling

```bash
helm install my-release \
  oci://ghcr.io/paulscherrerinstitute/dc-charts/proposals-cronjob --version 0.1.1 \
  -f my-values.yaml
helm delete my-release
```

Always pin `--version`. Note the published name is `proposals-cronjob`, not the
directory name.

## Parameters

> `(templated)` values are run through Helm's `tpl` function. They can hold
> template expressions such as `{{ .Values.ciTag }}` or `{{ .Release.Name }}`.
> Helm fills these in at render time using the deploy values and the release name.

| Parameter                           | Description                                                                                            | Default        |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------ | -------------- |
| `nameOverride` / `fullnameOverride` | Override the generated name                                                                            | `""`           |
| `image.repository`                  | Image name (templated)                                                                                 | `busybox`      |
| `image.tag`                         | Image tag (templated)                                                                                  | `latest`       |
| `image.pullPolicy`                  | Image pull policy                                                                                      | `IfNotPresent` |
| `cronjob.schedule`                  | Default schedule for facilities that don't set their own                                               | `0 7 * * 1`    |
| `cronjob.restartPolicy`             | Pod restart policy                                                                                     | `OnFailure`    |
| `cronjob.ttlSecondsAfterFinished`   | Cleanup delay for finished jobs (optional)                                                             | `nil`          |
| `duo_facilities[]`                  | List of facilities to sync. Each has `name`, optional `schedule`, `year`, `ttlSecondsAfterFinished`    | see values     |
| `facilities_schedule`               | Optional override holding the `duo_facilities` list as a raw YAML **string**. See below                | `nil`          |
| `env`                               | Additional environment variables (templated, k8s `env` syntax). Merged after `DUO_FACILITY`/`DUO_YEAR` | `nil`          |
| `volumes` / `volumeMounts`          | Pod volumes and mounts (templated, k8s syntax)                                                         | `nil`          |
| `secrets`                           | Map of `secretName -> {type, data: {...}}` (templated). Values must be **base64-encoded**              | `{}`           |

> `env` is effectively required. The container always sets `DUO_FACILITY`/`DUO_YEAR`
> and then splices `env` after them, so a config is expected to supply it.

## Key behaviors

This chart shares the value-injection and secret conventions of
`generic-service`. See its README for the full explanation:

- **Templated values:** `env`, `volumes`, `secrets` and `image.*` are passed
  through Helm's `tpl`, so they may contain Go template expressions that Helm
  fills in at render time (`{{ .Release.Name }}`, injected `{{ .Values.* }}`).
- **base64 secrets:** values under `secrets.<name>.data` must be base64-encoded.
  The chart validates this and fails the render otherwise.

### `facilities_schedule` takes a string, not a map

`templates/_helpers.tpl` emits the value verbatim and the caller pipes it through
`fromYaml`. So the value must be raw YAML text that holds a `duo_facilities:` key.

**A map renders nothing and reports success.** Go prints a map as `map[...]`,
`fromYaml` cannot read that, and the range over the missing list produces no
CronJob. `helm template` exits 0 with empty output. Measured on 2026-08-20.

```yaml
facilities_schedule: |
  duo_facilities:
    - name: sls
      schedule: "0 7 * * 1"
```

Leave `facilities_schedule` unset to use the `duo_facilities` list in the values
file. That is what every consumer does today.

### A per-facility TTL needs the chart-level default

`templates/cronjob.yaml` reads `ttlSecondsAfterFinished` from the facility, but
only inside a `with` guard on `cronjob.ttlSecondsAfterFinished`. So a facility TTL
alone renders nothing. Set the chart-level value first, then override it per
facility.

## Examples

```yaml
image:
  repository: "{{ .Values.ciRepository }}"
  tag: "{{ .Values.ciTag }}"
env:
  - name: SCICAT_ENDPOINT
    value: "{{ .Values.scicatEndpoint }}"
  - name: DUO_ENDPOINT
    value: https://duo.psi.ch/duo/api.php/v1/
volumes:
  - name: secrets-volume
    secret:
      secretName: "{{ .Release.Name }}-s"
volumeMounts:
  - name: secrets-volume
    mountPath: /usr/src/proposals/.env
    subPath: .env
secrets:
  "{{ .Release.Name }}-s":
    type: Opaque
    data:
      .env: "{{ .Values.secretsJson.PROPOSAL_ENV }}"   # already base64-encoded
```

## Maintaining the chart

- Bump `version` in `Chart.yaml` (SemVer) on every change to the chart or its
  templates.
- Targets the Helm v2 chart API. No changes needed for Helm 4. Validate with a
  `helm template` diff (with a real config, since `env` is required) before/after
  any change.
