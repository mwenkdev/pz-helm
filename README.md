# pz-helm

Helm chart for a single Project Zomboid dedicated server using the
[`indifferentbroccoli/projectzomboid-server-docker`](https://github.com/indifferentbroccoli/projectzomboid-server-docker)
image. The chart deploys a Deployment, ConfigMap, and UDP Service. It does not
create a namespace, Secret, PV, or PVC: provide an existing Secret and two
existing claims for server files and world/config data.

```sh
helm lint .
helm template pz-server . --namespace pz -f ci/values.yaml
```

For the Marauders cluster, the Argo CD Application and cluster-specific values
live in `k3s-marauders/apps/pz.yaml` and `k3s-marauders/pz/values.yaml`.
World data stays on a static local PV on `moony`. Preserve existing claim names
and take a backup before changing the server version or starting the replica.

`config.SERVER_BRANCH: ""` selects Steam's default public (Build 42 stable)
branch; `unstable` opts into the beta. With `UPDATE_ON_START: "true"`, Steam may
install a newer stable version at each start, independently of the chart version
or Docker image tag. Keep `replicaCount: 0` until the game files and world are
ready for the first start.

## Releases

PRs and pushes to `main` run Helm lint and render checks. To publish, update
`Chart.yaml` to a new `X.Y.Z`, merge the change to `main`, then create and
publish a GitHub Release named `vX.Y.Z` from that commit (for example,
`gh release create v0.1.0 --target main --generate-notes`). The `published`
event validates the tag and chart version, packages the chart, and publishes
`oci://ghcr.io/mwenkdev/pz-helm` at version `X.Y.Z`. Publishing the Release
through the GitHub UI or an authenticated `gh` CLI session triggers the workflow;
a Release created by another workflow with its default `GITHUB_TOKEN` does not.

If validation or publication fails, the workflow tries to remove the GitHub
Release but keeps its tag for a retry. After a failed OCI push, it also tries
to delete the new GHCR version; registry writes and cleanup cannot be atomic.
It leaves a Release intact if its OCI version already exists (including a
rerun of a successful release). GitHub Actions uses `GITHUB_TOKEN` with package
write permission and package admin access for cleanup. Set the GHCR package
visibility to public after its first publication for unauthenticated Argo CD
pulls. Pin a published chart version in the GitOps Application; do not point
it at a version that has not been published yet.
