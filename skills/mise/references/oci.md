## OCI Container Images

**Contents**

- [OCI Container Images](#oci-container-images)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

Build and publish container images containing mise-managed tools. **Experimental** — requires `experimental = true`. `oci` is a top-level config key.

```bash
mise oci build -o ./img                  # build an image (default output ./mise-oci)
mise oci build --copy ./dist:/app/dist   # reproducible host-path copy layer
mise oci build --from ubuntu:24.04 --tag myorg/dev:latest
mise oci build --include-global          # include ~/.config/mise tools (default is project-only)
mise oci build --no-mise                 # don't embed the mise binary
mise oci build --owner 1000:1000 --no-cache   # file ownership in layers; skip the layer cache
mise oci push myregistry.io/myimg:tag    # built-in registry client (no skopeo/crane)
mise oci push --cache-from myimg:prev    # reuse layers
mise oci push --update-index             # upsert into a multi-arch image index
mise oci run --engine docker
```

```toml
[oci]
from = "debian:bookworm-slim"
workdir = "/app"
entrypoint = ["/app/dist/server"]
env = { API_TOKEN = "build-placeholder" }   # also satisfies [env] `required` vars during mise oci commands (2026.9.18)
# also: tag, cmd, user, user_id, group_id, mount_point, labels

[[oci.copy]]
host = "./dist"        # relative to the config file
image = "/app/dist"    # absolute; no . or .. — `source`/`dest` are NOT valid keys (schema rejects them)

[settings.oci]
default_from = "debian:bookworm-slim"     # base image
default_mount_point = "/mise"
insecure_registries = ["registry.lan:5000", "10.0.0.8:5000"]   # plain HTTP
```

Each tool version becomes its own content-addressable OCI layer, so bumping one tool invalidates only that layer. Output conforms to the OCI image-layout spec (consumable by `skopeo`, `crane`, `podman load`). Since v2026.7.12 the registry client is built in — `docker login` / `podman login` is the only setup needed. `mise oci build` also bakes `[dotfiles]` and `apt:` `[bootstrap.packages]` into images as dedicated, annotated layers.

**Limits:** only **asdf** plugins are rejected. Since **2026.9.15** vfox plugin tools (including custom backend plugins) are packaged — one layer per tool plus one per plugin at `/mise/plugins/<name>/`; the plugin's env hook runs on the build host. Build on a **Linux host** with the target arch (binaries are host-native) or pass `--no-mise`. Image `Env` order: base → `[env]` → tool exec env → `[oci].env` → PATH → `MISE_DATA_DIR=/mise`/`MISE_CONFIG_DIR=/etc/mise`. `[env]` secrets **are baked into the image** (mise warns). Dotfile groups are not included.

> `mise oci push --tool` was **removed** in 2026.7.12. Use `mise oci build -o ./img` + `skopeo copy` instead.
