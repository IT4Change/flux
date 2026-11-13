# flux

[Flux](https://fluxcd.io) for the IT4Change clusters, installed with helmfile from the
[community Helm chart](https://github.com/fluxcd-community/helm-charts) — the same layout as
[IT4Change/prometheus](https://github.com/IT4Change/prometheus).

This repo installs the Flux **controllers** only. What Flux deploys (GitRepository, HelmRelease,
ImagePolicy, Receiver, …) lives in the repos of the deployments, not here.

## Why

Deployments are pulled by the cluster instead of pushed from GitHub, so no kubeconfig has to live on
GitHub, not even sops-encrypted. A GitHub workflow only builds images and may send a webhook that
says "check now"; Flux decides by each deployment's rollout rule whether a new version goes out.

## Install / upgrade

```sh
helmfile -e default apply      # staging cluster
helmfile -e production apply   # production cluster
```

Check: `kubectl -n flux-system get pods`, or `flux check` with the [Flux CLI](https://fluxcd.io/flux/installation/#install-the-flux-cli).

The chart version is pinned in `helmfile.yaml`. Upgrading Flux means raising it there, reading the
chart's and Flux's release notes, and applying.

## sops

Each cluster has its own age key. Its private half is in `environments/<env>.secrets.yaml`
(sops-encrypted to the team's keys in `.sops.yaml`); the release puts it into the Secret
`sops-age` in `flux-system`, which a deployment's Kustomization names as its decryption key:

```yaml
decryption:
  provider: sops
  secretRef:
    name: sops-age
```

For Flux to decrypt a deployment's Secrets, add the cluster's public key to that deployment's
`.sops.yaml` and re-encrypt (`sops updatekeys <file>`):

| Environment | Cluster | Public key |
| --- | --- | --- |
| `default` | staging | `age1nu7ls2fj54f7zmlj329zfx3smv098tz7y6shl9r0yxz5fl83243qv44mra` |
| `production` | production | `age19crepcun9rw5yrskfzu2gwlcrdn7egwz9skl9qk9g6s0an8ehsesns59f6` |

A new key for a cluster:

```fish
set env default   # or production
set secret (age-keygen 2>/dev/null | string match "AGE-SECRET-KEY-*")
echo $secret | age-keygen -y   # the public key, for the table above
printf "sops_age:\n    AGE_KEY: %s\n" $secret \
    | sops encrypt --filename-override environments/$env.secrets.yaml /dev/stdin \
    > environments/$env.secrets.yaml
set -e secret
```

## Not yet here

- **Webhook receiver ingress** (`values/flux.yaml.gotmpl`, `notificationController.webhookReceiver`):
  needs the domain and TLS issuer of the cluster.
