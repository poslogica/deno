# syntax=docker/dockerfile:1
#
# SonicSort's own deno, compiled from this fork (poslogica/deno, branch
# sonicsort/v2.9.7 = upstream tag v2.9.7) instead of downloading the release
# binary that denoland/deno_docker's images ship. Two targets, matching the
# upstream images they replace:
#
#   --target bin     -> sonicsort-deno-bin:local   (like denoland/deno:bin:    just /deno)
#   --target debian  -> sonicsort-deno:local       (like denoland/deno:debian: our Debian +
#                                                    /usr/bin/deno, a deno user, tini)
#
#   docker build -f sonicsort/Dockerfile --target bin \
#     --build-arg DEBIAN_BASE=sonicsort-debian:local -t sonicsort-deno-bin:local .
#
# Built natively on DEBIAN_BASE (SonicSort passes sonicsort-debian:local, its own
# trixie-slim from poslogica/debuerreotype) with Debian's clang/lld, in place of
# upstream CI's apt.llvm.org clang + old-glibc sysroot. That sysroot only exists
# so one binary runs on old distros; ours only has to run on our own Debian.
#
# Still upstream prebuilts: the Rust toolchain (rustup, pinned to
# rust-toolchain.toml's channel) and V8's static library (the rusty_v8 crate
# downloads it; V8_FROM_SOURCE=1 would build V8 too, at the cost of hours).
# Everything else -- deno, deno_core and the crate graph -- compiles here, with
# upstream's release profile (fat LTO, codegen-units=1), so the first build is
# long. Cargo's registry and target dir are BuildKit cache mounts, so rebuilds
# after a small change are much faster.

ARG DEBIAN_BASE=debian:trixie-slim

FROM ${DEBIAN_BASE} AS build
# Weekly cache-buster (SonicSort's build scripts pass the current ISO week).
ARG OS_PATCH_WEEK=unset
RUN set -eux; \
    apt-get update; \
    apt-get upgrade -y; \
    apt-get install -y --no-install-recommends \
        ca-certificates curl git build-essential pkg-config \
        clang lld cmake protobuf-compiler python3 libglib2.0-dev; \
    rm -rf /var/lib/apt/lists/*

ENV RUSTUP_HOME=/usr/local/rustup CARGO_HOME=/usr/local/cargo PATH=/usr/local/cargo/bin:$PATH
WORKDIR /src
COPY rust-toolchain.toml ./
# Only the compiler for rust-toolchain.toml's channel: the file also lists
# rustfmt/clippy/rust-analyzer for development, which a build doesn't need.
RUN set -eux; \
    channel="$(sed -n 's/^channel = "\(.*\)"/\1/p' rust-toolchain.toml)"; \
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \
        | sh -s -- -y --profile minimal --default-toolchain "$channel"; \
    rustc --version

COPY . .

# Linker via the per-target CARGO_TARGET_* variables, NOT RUSTFLAGS: RUSTFLAGS
# would replace .cargo/config.toml's rustflags, which include the required
# `--cfg tokio_unstable`; per-target settings are merged with them instead.
ENV CC=clang CXX=clang++ \
    CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_LINKER=clang \
    CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_RUSTFLAGS="-C link-arg=-fuse-ld=lld" \
    CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER=clang \
    CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_RUSTFLAGS="-C link-arg=-fuse-ld=lld"
# RUSTUP_TOOLCHAIN pins the channel explicitly so rustup doesn't auto-install
# the development components rust-toolchain.toml lists.
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/usr/local/cargo/git \
    --mount=type=cache,target=/src/target \
    set -eux; \
    export RUSTUP_TOOLCHAIN="$(sed -n 's/^channel = "\(.*\)"/\1/p' rust-toolchain.toml)"; \
    cargo build --release --locked --bin deno; \
    mkdir -p /out; \
    cp target/release/deno /out/deno; \
    /out/deno --version
# The release profile keeps line-table debug info (debug = "line-tables-only")
# and a full symbol table; upstream strips both before publishing, and so do
# we (214 -> ~110 MiB). --strip-all keeps .dynsym, which Node-API native
# addons need to resolve deno's napi_* exports. .eh_frame (unwind tables,
# ~8 MiB, which upstream's binaries also leave out) is kept for readable
# panic backtraces. Separate step so it never re-runs cargo.
RUN strip --strip-all /out/deno && /out/deno --version

FROM scratch AS bin
COPY --from=build /out/deno /deno

FROM ${DEBIAN_BASE} AS debian
ARG OS_PATCH_WEEK=unset
RUN set -eux; \
    apt-get update; \
    apt-get upgrade -y; \
    apt-get install -y --no-install-recommends ca-certificates tini; \
    rm -rf /var/lib/apt/lists/*; \
    ln -s /usr/bin/tini /tini; \
    useradd --uid 1993 --user-group deno; \
    mkdir /deno-dir/; \
    chown deno:deno /deno-dir/
ENV DENO_USE_CGROUPS=1 DENO_DIR=/deno-dir/ DENO_INSTALL_ROOT=/usr/local
COPY --from=build /out/deno /usr/bin/deno
# Same entrypoint behavior as denoland/deno_docker's _entry.sh: a deno
# subcommand or flag as the first argument runs deno; anything else runs as-is.
COPY --chmod=755 <<'EOF' /usr/local/bin/docker-entrypoint.sh
#!/bin/sh
set -e
if [ "$1" != "${1#-}" ]; then
    exec deno "$@"
fi
case "$1" in
    add | bench | bundle | cache | compile | completions | coverage | doc | eval | fmt | help | init | info | install | jupyter | lint | lsp | publish | remove | repl | run | serve | task | test | types | uninstall | upgrade | vendor )
    exec deno "$@";;
esac
exec "$@"
EOF
ENTRYPOINT ["/tini", "--", "docker-entrypoint.sh"]
CMD ["eval", "console.log('Welcome to Deno!')"]
