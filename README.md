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

## Not yet here

- **Webhook receiver ingress** (`values/flux.yaml.gotmpl`, `notificationController.webhookReceiver`):
  needs the domain and TLS issuer of the cluster.
- **sops decryption**: an age key as a Secret in `flux-system`, so Flux can decrypt the deployments'
  `*.secrets.yaml`. Its public key then joins the recipients in the deployments' `.sops.yaml`. Secrets
  for this repo would go into `environments/*.secrets.yaml` (`.sops.yaml` is in place).
