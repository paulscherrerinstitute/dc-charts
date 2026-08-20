# dc-charts

Shared Helm charts for the PSI Data Curation group. The charts deploy the
SciCat and SciLog services, and they are generic enough for other applications.

Each chart is published as an OCI package to the GitHub container registry under
`ghcr.io/paulscherrerinstitute/dc-charts`.

## The charts

| Directory | Published name | Version | Purpose |
| --- | --- | --- | --- |
| `charts/generic_service` | `generic-service` | 2.0.0 | One configurable chart for a simple service. Image, env, config maps, secrets, ingress, probes, jobs. |
| `charts/cron` | `cron` | 1.0.0 | A scheduled job. Same values style, no service and no ingress. |
| `charts/duo_facility_proposals` | `proposals-cronjob` | 0.1.1 | One CronJob per DUO facility, for the proposal sync. |

**The directory name is not the chart name.** Helm reads `name:` from
`Chart.yaml`, and the OCI package and the release tag both follow that name. So
`charts/generic_service` publishes as `generic-service`. Search `Chart.yaml`, not
the directory tree.

## Consume a chart

Pull the chart from the registry and pin the version.

```bash
helm show chart oci://ghcr.io/paulscherrerinstitute/dc-charts/generic-service --version 2.0.0
helm upgrade --install my-release \
  oci://ghcr.io/paulscherrerinstitute/dc-charts/generic-service --version 2.0.0 \
  -f my-values.yaml
```

The packages are public, so no registry login is necessary to pull them. A push
needs a token with `packages: write`.

Always pin `--version`. `generic-service` 2.0.0 is a breaking change against
1.0.0, and the two majors are both in use today.

## Who consumes what

| Chart | Repository | Components |
| --- | --- | --- |
| `generic-service` 2.0.0 | scicat-ci | backend, backend-next, frontend, frontend-next, globus-proxy, ingestor, landing-page-server, oaipmh, pan-ontologies-api, pss, s3-broker, scicat-exporter, search-api |
| `cron` 1.0.0 | scicat-ci | materialised-view, scicat-to-pss |
| `proposals-cronjob` 0.1.1 | scicat-ci | proposals |
| `generic-service` 1.0.0 | scilog-ci | backend, frontend |
| `cron` 1.0.0 | scilog-ci | proposal-sync |

scicat-ci declares each component in `helm/configs/<component>/helmfile.yaml.gotmpl`.
scilog-ci declares each component in its own workflow under `.github/workflows/`.

A consumer that moved from a vendored copy of a chart carries `nameOverride` in
its values. Keep it. Kubernetes makes the Deployment selector immutable, and the
chart feeds the chart name into that selector.

## Release a chart

1. Change the chart.
2. Bump `version:` in its `Chart.yaml`. Follow SemVer.
3. Open a pull request. `validate-charts.yml` lints and renders each changed chart.
4. Merge to `main`. `release-charts.yml` publishes the new version.

**The bump is mandatory.** `release-charts.yml` skips a chart whose
`<name>-<version>` GitHub release already exists. Without a bump nothing
publishes, and the workflow still reports success.

Consumers pin a version, so a fix reaches a component only when that component
raises its pin.

## The workflows

| Workflow | Runs on | What it does |
| --- | --- | --- |
| `.github/workflows/validate-charts.yml` | pull request, `charts/**` | `helm lint --strict` and `helm template` for each changed chart, with default values only |
| `.github/workflows/release-charts.yml` | push to `main`, `charts/**` | Packages each unreleased chart, pushes it to ghcr, and cuts a GitHub release named `<chart>-<version>` |

Note two limits. `validate-charts.yml` runs on pull requests only, so nothing
renders on the push that publishes. And `helm package` does not render templates,
so a chart with a broken template packages and publishes without an error.

Real consumer values live in the consumer repositories, so the validation here
covers the default values alone.

## Documentation

Each chart carries its own README with the full parameter table and the behaviour
notes.

- [`charts/generic_service/README.md`](charts/generic_service/README.md)
- [`charts/cron/README.md`](charts/cron/README.md)
- [`charts/duo_facility_proposals/README.md`](charts/duo_facility_proposals/README.md)

## Compatibility

All charts use `apiVersion: v2`. They run unchanged on Helm 3 and Helm 4. CI
pins Helm 4.2.2.
