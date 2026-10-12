# pz-helm

Helm chart for a single Project Zomboid dedicated server using the
[`mwenkdev/project-zomboid-server-docker`](https://github.com/mwenkdev/project-zomboid-server-docker)
image. The chart deploys a Deployment, ConfigMap, and UDP Service. It does not
create a namespace, Secret, PV, or PVC: provide an existing Secret and a
claim for world/config data.

```sh
helm lint .
helm template pz-server . --namespace pz -f ci/values.yaml
```

Provide a data claim containing the server's world and configuration. Preserve
the claim name and take a backup before changing the server version or starting
the replica.

The fork contains the Steam game installation in the image. `FORCEUPDATE` can
update it at startup, but is not enabled by this chart. Keep `replicaCount: 0`
until the game files and world are ready for the first start.
The image is built without publishing on PRs and pushes to `main`; publishing a
GitHub Release tagged `vX.Y.Z` pushes `ghcr.io/mwenkdev/project-zomboid-server-docker:vX.Y.Z`
and moves `:latest`. The chart defaults to the bootstrap-enabled `:v0.1.1` release;
pin `image.tag` to a released version when upgrading the server image.

## Mod ownership

Set `mods.enabled: true` and list Workshop items under `mods.items`. With this
enabled, the chart owns exactly `WorkshopItems=`, `Mods=` and `Map=` in the live
`/home/steam/Zomboid/Server/${SERVERNAME}.ini`. **Do not hand-edit those three
keys:** they are replaced on each start. Other INI settings and SandboxVars
remain on the data PVC. `Password=` and `RCONPassword=` are set from the existing
Secret's `PASSWORD` and `RCON_PASSWORD` keys. An empty `mods.items` clears the
Workshop/mod lists and sets the map to `Muldraugh, KY`.

```yaml
mods:
  enabled: true
  items:
    - workshopId: "123456789" # use the real ID from a Workshop URL
      name: Example label
      modIds: [ExactModId] # omit only if exactly one is found
      maps: [ExampleMap]   # omit if none or exactly one is found
```

Items are processed in listed order; map names are added in that order, with
`Muldraugh, KY` last. Before the server starts, SteamCMD downloads each item
and the image reads `mod.info` and `media/maps` in the downloaded content. If an
item contains multiple mod IDs or maps, choose the appropriate IDs/names
explicitly; use `maps: []` for a multi-map item that should add no map. Missing
or ambiguous content fails startup rather than silently dropping a mod. This
does not generate the spawn-regions Lua file; maintain custom spawn regions on
the data PVC when a map needs them. Back up the world before changing map mods.

For name-only discovery, use `name: Exact Workshop Title` instead of
`workshopId` and add `STEAM_WEB_API_KEY` to the existing Secret. The resolver
requires exactly one case-insensitive exact-title match among the first 100
Steam Workshop search results; if that fails, pin the Workshop ID. Titles can
change, so IDs are preferable for ongoing deployments. Changes to the chart's
mod list roll the pod; ConfigMap updates are applied on the next start.

## Migration from the previous image

This chart used to mount the install PVC at `/project-zomboid` and the data PVC
at `/project-zomboid-config`. The fork expects the data PVC at
`/home/steam/Zomboid`; it no longer mounts the install PVC (mounting it would
hide the game binaries baked into the image). **Inspect the root of the existing
data claim before increasing `replicaCount`**: it must contain `Server/`,
`Saves/`, and the other expected Zomboid data directories at its root. If they
are nested, migrate the directory contents while the server is stopped and
after taking a backup. Confirm `${SERVERNAME}.ini` and the existing save name
match `config.SERVERNAME`. When upgrading from an older chart release, update
your values to select the forked image and its environment variable names.

The existing Secret's `ADMIN_PASSWORD` is supplied as
`ADMIN_BOOTSTRAP_PASSWORD`, never as the logged `-adminpassword` argument. On
the first start the image feeds the password and confirmation to the game's
stdin prompt. It checks the admin account in the server database before each
start and does not send the password to the game console after that account
exists. Changing the Secret later does not rotate the existing admin account's
password. `ADMIN_USERNAME` is optional; the game defaults to `admin`.

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
visibility to public after its first publication for unauthenticated pulls.
Pin a published chart version in your deployment configuration.
