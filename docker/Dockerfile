FROM ghcr.io/astral-sh/uv:0.13.10@sha256:a3ea1357342c1a4bf9a0c53a3deb1b55379aefa037338d8113d5771efa01774f AS uv

FROM python:3.14-slim@sha256:9aaa474097d094040649f2162a627c0049b408f52138d980916788d80f04802f AS builder

COPY --from=uv /uv /usr/local/bin/uv

ENV RUSTUP_HOME=/opt/rustup \
    CARGO_HOME=/opt/cargo \
    PATH=/opt/cargo/bin:${PATH}

# rustup 1.29.3, each rustup-init checked against the SHA-256 that static.rust-lang.org
# publishes beside it; gcc links the extension.
RUN apt-get update \
    && apt-get install --yes --no-install-recommends ca-certificates curl gcc libc6-dev \
    && rm -rf /var/lib/apt/lists/* \
    && arch="$(uname -m)" \
    && case "${arch}" in \
        x86_64) sha=b1c8cb14e30889e7e2999e6b69e62d9e9b9ffc0f4df8180564d306d9680e6c55 ;; \
        aarch64) sha=9f608c142e0e38a78957d0661ba1dfaf0b6fb37184cfaa4b9a0aa57af6184282 ;; \
        *) echo "E: no pinned rustup-init for ${arch}" >&2; exit 1 ;; \
    esac \
    && curl --proto "=https" --tlsv1.2 --fail --silent --show-error --output /tmp/rustup-init \
        "https://static.rust-lang.org/rustup/archive/1.29.3/${arch}-unknown-linux-gnu/rustup-init" \
    && echo "${sha}  /tmp/rustup-init" | sha256sum --check --strict - \
    && chmod +x /tmp/rustup-init \
    && /tmp/rustup-init -y --no-modify-path --profile minimal --default-toolchain none \
    && rm /tmp/rustup-init

WORKDIR /src
COPY rust-toolchain.toml ./
RUN rustup toolchain install
COPY pyproject.toml README.md LICENSE Cargo.toml Cargo.lock ./
COPY crates ./crates
COPY e8s ./e8s
# cargo fails on a stale Cargo.lock instead of rewriting it.
ENV MATURIN_PEP517_ARGS=--locked
RUN uv build --wheel --out-dir /dist

FROM python:3.14-slim@sha256:9aaa474097d094040649f2162a627c0049b408f52138d980916788d80f04802f AS runtime

COPY --from=uv /uv /usr/local/bin/uv

ENV UV_LINK_MODE=copy \
    UV_PROJECT_ENVIRONMENT=/opt/venv \
    UV_NO_CACHE=1 \
    NOX_UV_NO_SYNC=1 \
    PYTHONDONTWRITEBYTECODE=1

# jq applies the --jq filters of the fake gh that the nightly report tests run.
RUN apt-get update \
    && apt-get install --yes --no-install-recommends jq \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY pyproject.toml uv.lock .python-version README.md LICENSE ./
# The dev group holds e8s[all], so the locked sync installs every extra's dependencies.
RUN uv sync --locked --no-install-project --group dev

COPY --from=builder /dist /dist
RUN uv pip install --python /opt/venv/bin/python --no-deps /dist/*.whl

COPY noxfile.py Cargo.toml Cargo.lock rust-toolchain.toml ./
COPY crates ./crates
COPY tests ./tests
COPY scripts ./scripts
COPY fuzz ./fuzz
COPY .github/workflows/ci.yml .github/workflows/wheels.yml .github/workflows/nightly.yml \
    .github/workflows/nightly-core.yml .github/workflows/nightly-storage.yml \
    .github/workflows/report-nightly.yml ./.github/workflows/
RUN chown -R 10001:10001 /app
USER 10001

CMD ["uv", "run", "--no-sync", "nox", "-s", "tests", "--", "tests", "--pyargs", "e8s"]
