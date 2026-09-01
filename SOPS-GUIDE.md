# Secrets

Every credential in this repository is encrypted with SOPS before it is
committed. Nothing here is readable without a private key, and no key is stored
in this repository.

## One key per cluster

| Key | Encrypts | Held by |
| --- | --- | --- |
| `heal-k3d-dev` | `fluxcd/heal-k3d-dev/**` | the dev cluster |
| `heal-k3d-prod` | `fluxcd/heal-k3d-prod/**` | the prod cluster |

The split is the point: a compromised dev cluster cannot read production
credentials. Neither key can decrypt anything in `dropp-infra`, which uses a
third, management key.

The private halves are delivered into each cluster by the management cluster as
the `sops-gpg-heal` Secret, and the Flux `Kustomization` in `dropp-infra`
references it as `decryption.secretRef`. Flux decrypts at apply time; the
plaintext never exists in Git and never leaves the cluster.

## Editing a secret

`sops` looks for its configuration by walking up from the **working directory**,
not from the file being edited. Running it from the repository root finds no
rules and fails with *"config file not found, or has no creation rules"*. Either
work from the cluster directory:

```bash
cd fluxcd/heal-k3d-dev
sops secrets/mysql-credentials.sops.yaml
```

or pass the config explicitly:

```bash
sops --config fluxcd/heal-k3d-dev/.sops.yaml \
     -e -i fluxcd/heal-k3d-dev/secrets/mysql-credentials.sops.yaml
```

## Creating one

Render the Secret, then encrypt in place. Never commit the intermediate file.

```bash
cd fluxcd/heal-k3d-dev
kubectl create secret generic example -n heal \
  --from-literal=KEY=value --dry-run=client -o yaml > secrets/example.sops.yaml
sops -e -i secrets/example.sops.yaml
```

Only `data` and `stringData` are encrypted. Names, namespaces and labels stay
readable, so a diff still shows which secret changed and when — just not to
what.

## Key names the charts require

These are fixed by the upstream charts, not chosen here. Renaming a key silently
produces a release that starts with an empty password.

| Secret | Keys | Read by |
| --- | --- | --- |
| `mysql-credentials` | `mysql-root-password`, `mysql-password`, `mysql-replication-password` | Bitnami MySQL, via `auth.existingSecret` |
| `influxdb-credentials` | `admin-password`, `admin-token` | influxdb2, via `adminUser.existingSecret` |
| `heal-credentials` | `HEAL_API_DATABASE_PASSWORD`, `HEAL_API_INFLUXDB_TOKEN` | the heal chart, via `secrets.existingSecret` |

`mysql-password` and `HEAL_API_DATABASE_PASSWORD` are the same value, as are
`admin-token` and `HEAL_API_INFLUXDB_TOKEN`: one credential, two consumers.
Rotating one without the other leaves the application unable to authenticate
against a datastore that is otherwise healthy.

## Rotating a key

Changing a fingerprint in `.sops.yaml` does not re-encrypt anything. Existing
files keep their old recipient until each is rewrapped:

```bash
cd fluxcd/heal-k3d-dev
sops updatekeys -y secrets/*.sops.yaml
```

Then update the `sops-gpg-heal` Secret that `dropp-infra` ships into the cluster,
or Flux will decrypt with a key that no longer matches.
