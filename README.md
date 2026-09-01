# dropp-heal-gitops

Desired state for the `heal` application, one directory per cluster.

```
fluxcd/heal-k3d-dev/
├── sources/           HelmRepositories, so charts resolve before anything needs them
├── secrets/           SOPS-encrypted credentials
├── mysql/             configuration and latest status
├── influxdb/          the historical latency dataset
├── phpmyadmin/        a database console
├── heal/              the application itself
├── image-automation/  Flux writes new image tags back into this repo
└── monitoring/        placeholder
```

This repository owns only application state. The cluster that follows it, the
`GitRepository` pointing here, and the Flux `Kustomization` that reconciles it
live in [dropp-infra](https://github.com/dropporg/dropp-infra) — a repository
cannot bootstrap itself as a source of truth, and pointing a cluster at a repo
is a provisioning decision.

## Why sources are their own directory

Every `HelmRepository` lives in `sources/`, not beside the release that uses it.
That is not tidiness: a `HelmRelease` whose `HelmRepository` was pruned reports
`failed to get source`, and if the two share a directory, whatever removed one
removed the other. Keeping sources separate means the thing a release depends on
outlives the release.

## What the chart does not bring

The `heal` chart declares no dependencies and defaults `mysql.host` to `mysql`
and `influxdb.url` to an in-cluster address, so both datastores are supplied
here. They are environment state, not application state: a production cluster
would point at managed instances and drop those directories.

The chart also ships a classic `networking.k8s.io/v1` Ingress. These clusters
run Envoy Gateway and have no Ingress controller, so `ingress.enabled` is false
and every route is an `HTTPRoute`.

## Hostnames

The cluster Gateway listens on `*.heal.local`. A Gateway API wildcard matches
exactly one label and never the apex, so `heal-dev.heal.local` attaches and
`heal.local` would not.

| | dev | prod |
| --- | --- | --- |
| application | `heal-dev.heal.local` | `heal.heal.local` |
| phpMyAdmin | `pma-dev.heal.local` | `pma-prod.heal.local` |
| InfluxDB UI | `influx-dev.heal.local` | `influx-prod.heal.local` |

## Versions

Each component pins its own image tag. The chart's fallback is its `appVersion`,
which tracks the product-wide `release/*` version and deliberately follows no
single service — inheriting from it resolves every image to an unrelated
version. The trailing `# {"$imagepolicy": ...}` marker on each tag is what Flux
rewrites; deleting it stops updates silently.

dev and prod diverge on purpose:

| | dev | prod |
| --- | --- | --- |
| image policy | accepts prereleases (`>=0.0.1-0`) | clean releases only (`>=0.0.1`) |
| automation pushes to | `main` | `promote/prod`, for a human to merge |

## Bitnami images

The MySQL and phpMyAdmin charts come from Bitnami, whose public images moved in
2025: `docker.io/bitnami/mysql` returns 404 for these tags and the same content
is served from `bitnamilegacy`. Both releases override the image registry
accordingly and set `global.security.allowInsecureImages`, which is the chart's
own switch for a deliberate substitution. Without the override the pods sit in
`ImagePullBackOff` with a manifest-unknown error.

`charts.pockost.com` and `helm.wso2.com` are not used: the first no longer
resolves in DNS, the second answers 403.

## Secrets

See [SOPS-GUIDE.md](SOPS-GUIDE.md).
