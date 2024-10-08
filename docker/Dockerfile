# If you want to include a file in the Docker image, add it to .dockerignore.
#
# We are using five stages:
# - chef: installs cargo-chef
# - planner: computes the recipe file
# - deps: caches our dependencies and sets the needed variables
# - tests: builds tests
# - release: builds release binary
# - runtime: is our runtime environment
#
# We first set default values for build arguments used across the stages.
# Each stage must define the build arguments (ARGs) it uses.
#
# Build zebrad with these features
#
# Keep these argument defaults in sync with GitHub vars.RUST_PROD_FEATURES and vars.RUST_TEST_FEATURES
# https://github.com/ZcashFoundation/zebra/settings/variables/actions
ARG FEATURES="default-release-binaries"
ARG TEST_FEATURES="lightwalletd-grpc-tests zebra-checkpoints"
ARG EXPERIMENTAL_FEATURES=""

ARG APP_HOME="/opt/zebrad"
# This stage implements cargo-chef for docker layer caching
FROM rust:bookworm as chef
RUN cargo install cargo-chef --locked

ARG APP_HOME
ENV APP_HOME=${APP_HOME}
WORKDIR ${APP_HOME}

# Analyze the current project to determine the minimum subset of files
# (Cargo.lock and Cargo.toml manifests) required to build it and cache dependencies
#
# The recipe.json is the equivalent of the Python requirements.txt file
FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

# In this stage we download all system requirements to build the project
#
# It also captures all the build arguments to be used as environment variables.
# We set defaults for the arguments, in case the build does not include this information.
FROM chef AS deps
SHELL ["/bin/bash", "-xo", "pipefail", "-c"]
COPY --from=planner ${APP_HOME}/recipe.json recipe.json

# Install zebra build deps and Dockerfile deps
RUN apt-get -qq update && \
    apt-get -qq install -y --no-install-recommends \
    llvm \
    libclang-dev \
    clang \
    ca-certificates \
    protobuf-compiler \
    rsync \
    rocksdb-tools \
    ; \
    rm -rf /var/lib/apt/lists/* /tmp/*

# Install google OS Config agent to be able to get information from the VMs being deployed
# into GCP for integration testing purposes, and as Mainnet nodes
# TODO: this shouldn't be a hardcoded requirement for everyone
RUN if [ "$(uname -m)" != "aarch64" ]; then \
      apt-get -qq update && \
      apt-get -qq install -y --no-install-recommends \
      curl \
      lsb-release \
      && \
      echo "deb http://packages.cloud.google.com/apt google-compute-engine-$(lsb_release -cs)-stable main" > /etc/apt/sources.list.d/google-compute-engine.list && \
      curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add - && \
      apt-get -qq update  && \
      apt-get -qq install -y --no-install-recommends google-osconfig-agent; \
    fi \
    && \
    rm -rf /var/lib/apt/lists/* /tmp/*

# Build arguments and variables set for tracelog levels and debug information
#
# We set defaults to all variables.
ARG RUST_LOG
ENV RUST_LOG=${RUST_LOG:-info}

ARG RUST_BACKTRACE
ENV RUST_BACKTRACE=${RUST_BACKTRACE:-1}

ARG RUST_LIB_BACKTRACE
ENV RUST_LIB_BACKTRACE=${RUST_LIB_BACKTRACE:-1}

ARG COLORBT_SHOW_HIDDEN
ENV COLORBT_SHOW_HIDDEN=${COLORBT_SHOW_HIDDEN:-1}

ARG SHORT_SHA
# If this is not set, it must be the empty string, so Zebra can try an alternative git commit source:
# https://github.com/ZcashFoundation/zebra/blob/9ebd56092bcdfc1a09062e15a0574c94af37f389/zebrad/src/application.rs#L179-L182
ENV SHORT_SHA=${SHORT_SHA:-}

ENV CARGO_HOME="${APP_HOME}/.cargo/"

# In this stage we build tests (without running then)
#
# We also download needed dependencies for tests to work, from other images.
# An entrypoint.sh is only available in this step for easier test handling with variables.
FROM deps AS tests

COPY --from=electriccoinco/lightwalletd:latest /usr/local/bin/lightwalletd /usr/local/bin/

# cargo uses timestamps for its cache, so they need to be in this order:
# unmodified source files < previous build cache < modified source files
COPY . .

# Skip IPv6 tests by default, as some CI environment don't have IPv6 available
ARG ZEBRA_SKIP_IPV6_TESTS
ENV ZEBRA_SKIP_IPV6_TESTS=${ZEBRA_SKIP_IPV6_TESTS:-1}

# Use ENTRYPOINT_FEATURES to override the specific features used to run tests in entrypoint.sh,
# separately from the test and production image builds.
ARG FEATURES
ARG TEST_FEATURES
ARG EXPERIMENTAL_FEATURES
# TODO: add empty $EXPERIMENTAL_FEATURES when we can avoid adding an extra space to the end of the string
ARG ENTRYPOINT_FEATURES="${FEATURES} ${TEST_FEATURES}"

# Re-hydrate the minimum project skeleton identified by `cargo chef prepare` in the planner stage,
# over the top of the original source files,
# and build it to cache all possible sentry and test dependencies.
#
# This is the caching Docker layer for Rust tests!
# It creates fake empty test binaries so dependencies are built, but Zebra is not fully built.
#
# TODO: add --locked when cargo-chef supports it
RUN cargo chef cook --tests --release --features "${ENTRYPOINT_FEATURES}" --workspace --recipe-path recipe.json
# Undo the source file changes made by cargo-chef.
# rsync invalidates the cargo cache for the changed files only, by updating their timestamps.
# This makes sure the fake empty binaries created by cargo-chef are rebuilt.
COPY --from=planner ${APP_HOME} zebra-original
RUN rsync --recursive --checksum --itemize-changes --verbose zebra-original/ .
RUN rm -r zebra-original

# Build Zebra test binaries, but don't run them
RUN cargo test --locked --release --features "${ENTRYPOINT_FEATURES}" --workspace --no-run
RUN cp ${APP_HOME}/target/release/zebrad /usr/local/bin
RUN cp ${APP_HOME}/target/release/zebra-checkpoints /usr/local/bin

COPY ./docker/entrypoint.sh /etc/zebrad/entrypoint.sh

# Entrypoint environment variables
ENV ENTRYPOINT_FEATURES=${ENTRYPOINT_FEATURES}
# We repeat the ARGs here, so they are available in the entrypoint.sh script for $RUN_ALL_EXPERIMENTAL_TESTS
ARG EXPERIMENTAL_FEATURES="shielded-scan journald prometheus filter-reload"
ENV ENTRYPOINT_FEATURES_EXPERIMENTAL="${ENTRYPOINT_FEATURES} ${EXPERIMENTAL_FEATURES}"

# By default, runs the entrypoint tests specified by the environmental variables (if any are set)
ENTRYPOINT [ "/etc/zebrad/entrypoint.sh" ]

# In this stage we build a release (generate the zebrad binary)
#
# This step also adds `cargo chef` as this stage is completely independent from the
# `test` stage. This step is a dependency for the `runtime` stage, which uses the resulting
# zebrad binary from this step.
FROM deps AS release

COPY . .

ARG FEATURES

# This is the caching layer for Rust zebrad builds.
# It creates a fake empty zebrad binary, see above for details.
#
# TODO: add --locked when cargo-chef supports it
RUN cargo chef cook --release --features "${FEATURES}" --package zebrad --bin zebrad --recipe-path recipe.json

# Undo the source file changes made by cargo-chef, so the fake empty zebrad binary is rebuilt.
COPY --from=planner ${APP_HOME} zebra-original
RUN rsync --recursive --checksum --itemize-changes --verbose zebra-original/ .
RUN rm -r zebra-original

# Build zebrad
RUN cargo build --locked --release --features "${FEATURES}" --package zebrad --bin zebrad

COPY ./docker/entrypoint.sh ./

# This stage is only used when deploying nodes or when only the resulting zebrad binary is needed
#
# To save space, this step starts from scratch using debian, and only adds the resulting
# binary from the `release` stage
FROM debian:bookworm-slim AS runtime

# Set the default path for the zebrad binary
ARG APP_HOME
ENV APP_HOME=${APP_HOME}
WORKDIR ${APP_HOME}

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    rocksdb-tools \
    gosu \
    && \
    rm -rf /var/lib/apt/lists/* /tmp/*

# Create a non-privileged user that the app will run under.
# Running as root inside the container is running as root in the Docker host
# If an attacker manages to break out of the container, they will have root access to the host
# See https://docs.docker.com/go/dockerfile-user-best-practices/
ARG USER=zebra
ENV USER=${USER}
ARG UID=10001
ENV UID=${UID}
ARG GID=10001
ENV GID=${GID}

RUN addgroup --system --gid ${GID} ${USER} \
    && adduser \
    --system \
    --disabled-login \
    --shell /bin/bash \
    --uid "${UID}" \
    --gid "${GID}" \
    ${USER}

# Config settings for zebrad
ARG FEATURES
ENV FEATURES=${FEATURES}

# Path and name of the config file
ENV ZEBRA_CONF_DIR=${ZEBRA_CONF_DIR:-/etc/zebrad}
ENV ZEBRA_CONF_FILE=${ZEBRA_CONF_FILE:-zebrad.toml}

RUN mkdir -p ${ZEBRA_CONF_DIR} && chown ${UID}:${UID} ${ZEBRA_CONF_DIR} \
    && chown ${UID}:${UID} ${APP_HOME}

COPY --from=release ${APP_HOME}/target/release/zebrad /usr/local/bin
COPY --from=release ${APP_HOME}/entrypoint.sh /etc/zebrad

# Expose configured ports
EXPOSE 8233 18233

# Update the config file based on the Docker run variables,
# and launch zebrad with it
ENTRYPOINT [ "/etc/zebrad/entrypoint.sh" ]
CMD ["zebrad"]
