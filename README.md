# pulseline-gitops

The desired state of PulseLine on Kubernetes. Argo CD watches this repository and keeps the cluster equal to it, so **a deployment is a commit here**, not a `kubectl apply`.

The application code, the images and the script that creates the local cluster live in the `pulseline` repository.

## Layout

```text
apps/
  base/        Everything PulseLine runs: PostgreSQL, Kafka (Strimzi), flow-api,
               flow-processor, dashboard, the autoscaler and the migration job
  demo/        The environment Argo CD deploys. Pins the image versions
  dev/         For trying manifests by hand, without Argo CD
argocd/
  pulseline-application.yaml   The Argo CD Application for apps/demo
```

`base` is never applied on its own. Each overlay adds the image tags with Kustomize.

## How a change reaches the cluster

```mermaid
flowchart LR
    Commit["Commit on main"] --> ArgoCD["Argo CD compares Git with the cluster"]
    ArgoCD -->|different| Sync["Sync: apply, then run migrations"]
    Sync --> Healthy["Synced and Healthy"]
```

The Application syncs automatically, with two options turned on:

- **prune**: what is deleted from Git is deleted from the cluster.
- **selfHeal**: a manual change in the cluster is reverted to what Git says.

The database migrations are an Argo CD sync hook, so they run on every sync, before it is reported as finished.

## Release a new version

1. Build the images with the new tag and load them into the cluster. From the `pulseline` repository:

   ```bash
   ./infra/kind/build-images.sh 0.5.1
   ```

2. Change the tags in [`apps/demo/kustomization.yaml`](apps/demo/kustomization.yaml), commit and push.

3. Argo CD notices within three minutes. To skip the wait:

   ```bash
   kubectl -n argocd annotate application pulseline argocd.argoproj.io/refresh=normal --overwrite
   ```

4. Watch it:

   ```bash
   kubectl -n argocd get application pulseline -w
   kubectl -n pulseline get pods -w
   ```

Rolling back is `git revert` of that commit.

## What is not in this repository

- **Credentials.** The Secret `pulseline-secrets` (`POSTGRES_USER`, `POSTGRES_PASSWORD`) is created in the cluster by the setup script from a git-ignored file. In a real environment this would be Sealed Secrets or External Secrets.
- **Operators and platform tools.** Strimzi, Argo CD, Argo Workflows and metrics-server are installed with the cluster.
- **Batch workflows.** The Argo Workflows definitions are in `pulseline/infra/argo-workflows`, next to the code they run.

## Check the manifests locally

```bash
kubectl kustomize apps/demo
```
