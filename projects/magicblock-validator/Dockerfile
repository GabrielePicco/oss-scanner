FROM rust:1.99-bookworm

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates clang cmake git libclang-dev libssl-dev \
      pkg-config python3 unzip curl \
    && rm -rf /var/lib/apt/lists/*

# The workspace pins protoc 26.1 (.github/actions/setup-protoc/protoc-version).
RUN arch="$(uname -m)"; \
    case "$arch" in x86_64) p=linux-x86_64;; aarch64) p=linux-aarch_64;; *) exit 1;; esac; \
    curl --fail --location --retry 3 -o /tmp/protoc.zip \
      "https://github.com/protocolbuffers/protobuf/releases/download/v26.1/protoc-26.1-$p.zip" \
    && unzip -q /tmp/protoc.zip -d /usr/local && rm /tmp/protoc.zip && protoc --version

# The workspace builds on-chain programs (build scripts call cargo-build-sbf), so install the pinned Agave CLI
# (.github/actions/setup-solana/agave-version). SBF platform-tools are downloaded on first use during the build below.
RUN curl --fail --silent --show-error --location --retry 3 https://release.anza.xyz/v4.2.2/install | sh
ENV PATH="/root/.local/share/solana/install/active_release/bin:${PATH}"

COPY . /src
WORKDIR /src

# Installs the toolchain pinned by rust-toolchain.toml, then fetches every crate (including the git dependencies on
# magicblock-engine, delegation-program and laserstream-sdk) so the audit can build offline.
RUN rustup toolchain install && cargo fetch --locked

ENV CARGO_INCREMENTAL=0 \
    RUSTFLAGS="-C force-frame-pointers=yes"

# Build the validator binary with debug info, then run the cheap unit tests of the crates that parse untrusted input.
# Both steps run while the network is still up, so the SBF program builds can fetch their own crates into the image.
# A failing test does not abort the build.
RUN cargo build --locked -p magicblock-validator
RUN cargo test --locked --lib -p magicblock-config -p magicblock-ledger-deprecated -p magicblock-aperture \
      || echo "WARNING: unit tests reported failures"

CMD ["bash"]
