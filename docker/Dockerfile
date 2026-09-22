# syntax=docker/dockerfile:1.7
FROM rust:1.93-bookworm AS builder

RUN apt-get update && apt-get install -y pkg-config libssl-dev && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy the whole repo rather than selected directories.
#
# Selective COPY cannot work here: the tree uses symlinks that cross top-level
# directory boundaries, so each one that is copied exposes the next dangling
# link. Observed chain, all fatal at build time:
#
#   examples/custom_plugin              - declared workspace member in Cargo.toml
#   crates/bitview_website/website      - symlink -> ../../website
#   website/scripts/modules             - symlink -> ../../modules
#
# `include_dir!` walks the embedded directory eagerly, so any dangling link in
# it panics the proc macro. Copying everything is the only way to keep the links
# intact without tracking them by hand.
COPY . .

# Cache the cargo registry and the target dir across builds. Without these,
# every redeploy recompiles all ~470 crates from scratch, because `COPY . .`
# invalidates the layer cache on any repo change.
#
# The binary MUST be copied out inside this same RUN: a cache mount is not part
# of the image layer, so /app/target does not exist for later stages.
RUN --mount=type=cache,target=/usr/local/cargo/registry,sharing=locked \
    --mount=type=cache,target=/app/target,sharing=locked \
    cargo build --release --locked -p bitviewd \
    && cp /app/target/release/bitviewd /bitviewd

FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y ca-certificates procps && rm -rf /var/lib/apt/lists/*

RUN useradd -m -s /bin/bash bitview

COPY --from=builder /bitviewd /usr/local/bin/bitviewd

USER bitview

RUN mkdir -p /home/bitview/.bitview

EXPOSE 3110

WORKDIR /home/bitview

ENTRYPOINT ["bitviewd"]
