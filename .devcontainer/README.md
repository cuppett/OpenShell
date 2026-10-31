<!-- SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# OpenShell dev container

This dev container is optional. It provides the Rust toolchain, Z3, and the system build dependencies that CI uses, so you can build OpenShell without installing packages on the host. It is most useful on image-based hosts such as Fedora Silverblue, Kinoite, Bluefin, and Bazzite, where contributors do not layer RPMs onto the base image and use rootless Podman instead of Docker, and it also works with Docker on any Linux or macOS host. For the no-container path, `mise install` plus a Z3 development library is sufficient; see the prerequisites in `CONTRIBUTING.md`.

## Use it with the devcontainer CLI and Podman

```shell
npm i -g @devcontainers/cli
devcontainer up --docker-path podman --workspace-folder .
devcontainer exec --docker-path podman --workspace-folder . bash
```

The first `up` builds the image and runs `postCreateCommand`, which trusts `mise.toml` and installs Rust, Python, Node, uv, and sccache through mise.

## Use it with VS Code

Install the Dev Containers extension. For Podman, set the engine in your VS Code user settings:

```json
{
  "dev.containers.dockerPath": "podman"
}
```

Then run **Dev Containers: Reopen in Container**.

## Design notes

- Base image: `docker.io/library/ubuntu:noble`. CI uses the NGC `nvcr.io/nvidia/base/ubuntu` image, which requires registry authentication. The Docker Hub image is the same Ubuntu release and works for outside contributors without credentials. The apt package list mirrors `deploy/docker/Dockerfile.ci`; keep the two in sync.
- Rust toolchain: the `Containerfile` installs Rust with rustup and hardcodes the version from `rust-toolchain.toml` in a `RUST_VERSION` build argument. Rustup still reads `rust-toolchain.toml` in the workspace, so if the two drift, the first build downloads the pinned toolchain instead of using the wrong compiler. Hardcoding avoids copying repository files into the image.
- `--security-opt label=disable`: stops SELinux from blocking access to the bind-mounted repository. The alternative, a `:Z` relabel, rewrites SELinux labels on your checkout.
- `--userns=keep-id` and `updateRemoteUserUID`: map your host UID to the `vscode` user so files created in the repository stay owned by you.
- Mise is pinned to the same version as CI. Interactive shells activate it from `~/.bashrc`.

## Persisted caches

These named volumes survive container rebuilds:

| Volume | Mount point | Contents |
| --- | --- | --- |
| `openshell-cargo-registry` | `/usr/local/cargo/registry` | Cargo registry cache, shared across checkouts |
| `openshell-cargo-git` | `/usr/local/cargo/git` | Cargo git dependency cache, shared across checkouts |
| `openshell-sccache` | `/home/vscode/.cache/sccache` | sccache compiler cache (`SCCACHE_DIR`) |
| `openshell-mise-data` | `/home/vscode/.local/share/mise` | Tools installed by mise |
| `openshell-target-<id>` | `/workspace/target` | Cargo build output, one volume per checkout |

`target/` uses a volume instead of the host bind mount because it holds many files that would otherwise be written with container UID mapping and SELinux labels onto the host checkout. That causes ownership and relabeling problems on immutable hosts, and volume I/O is faster under rootless Podman. Remove a volume with `podman volume rm <name>` to reset it.

## What is not installed

- The full CI tool list from `mise.toml`, such as Helm, kubectl, Go, Zig, and cargo-deny. Run `mise install --locked` inside the container if you need them. If GitHub API rate limits stop an install, export `MISE_GITHUB_TOKEN` in the container.
  Note that `mise install --locked` can rewrite `mise.lock` on Linux, because mise prunes platform entries for tools scoped to macOS such as `k3d`. Revert that change rather than committing it.
- A Docker or Podman CLI. Compiling the workspace does not need a container engine. `mise run gateway`, the driver tasks, and the E2E suites do, so run them on the host against your host engine rather than nesting an engine in the dev container.
- `protoc`. The Rust build uses `protoc-bin-vendored` unless `PROTOC` is set.

## Measuring build time

`mise run build:rust:workspace` runs `cargo build --workspace`, the debug build CI uses. `mise run build:rust:workspace:release` runs `cargo build --workspace --release`. Mise sets `RUSTC_WRAPPER=sccache`, and cache hits hide real compiler time, so unset it when you measure:

```shell
cargo clean
time env -u RUSTC_WRAPPER cargo build --workspace
```
