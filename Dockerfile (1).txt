# Modified Dockerfile — m7-03 lab (Containerization with Docker for ML).
# v2: adds digest-pinned base, provenance LABELs, and HEALTHCHECK.
#
# Multi-stage build that:
#   * Stage 1 (builder)  — fetches the ONNX Runtime release, compiles the
#                          provided C verifier against it, and validates
#                          the bundled model.onnx artifact.
#   * Stage 2 (runtime)  — ships only the compiled binary, the ONNX
#                          Runtime shared library, the model file, and
#                          a non-root user. No compilers, no tarballs.
#
# Before building: copy your cat-detection model.onnx from the m6-09
# assessment into the repo root. The file is gitignored by default — do
# NOT commit it.
#
# Build:    docker build -t <ns>/m7-03-cat-detection:v2 .
# Run:      docker run --rm <ns>/m7-03-cat-detection:v2
# Verify:   uid should be 1001; image size should be < ~250 MB

ARG ORT_VERSION=1.20.1

# ──────────────────────────────────────────────────────────────
# Stage 1 — builder
# ──────────────────────────────────────────────────────────────
# 3b: Base image pinned by digest — tag-based pulls can silently drift to a
# new OS patch, changing the build environment without warning. A digest is
# immutable: this exact byte sequence is what every CI run and teammate gets.
FROM debian:12-slim@sha256:0104b334637a5f19aa9c983a91b54c89887c0984081f2068983107a6f6c21eeb AS builder
ARG ORT_VERSION

RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential ca-certificates curl file \
    && rm -rf /var/lib/apt/lists/*

# Fetch and extract the official ONNX Runtime release tarball
WORKDIR /opt
RUN curl -sSL -o ort.tgz \
        "https://github.com/microsoft/onnxruntime/releases/download/v${ORT_VERSION}/onnxruntime-linux-x64-${ORT_VERSION}.tgz" \
    && tar -xzf ort.tgz \
    && mv "onnxruntime-linux-x64-${ORT_VERSION}" onnxruntime \
    && rm ort.tgz

# Compile the model verifier against the extracted library
WORKDIR /build
COPY src/check_model.c .
RUN mkdir -p /out \
    && gcc -O2 -o /out/check_model check_model.c \
        -I/opt/onnxruntime/include \
        -L/opt/onnxruntime/lib \
        -lonnxruntime

# Validation gate — fail the build now if the model is missing/empty/bogus
COPY model.onnx /tmp/model.onnx
RUN test -s /tmp/model.onnx \
    && file /tmp/model.onnx | grep -qi onnx \
    && echo "Model SHA-256: $(sha256sum /tmp/model.onnx | awk '{print $1}')" \
    && echo "Model size:    $(stat -c%s /tmp/model.onnx) bytes"

# ──────────────────────────────────────────────────────────────
# Stage 2 — runtime
# ──────────────────────────────────────────────────────────────
# 3b: Same digest as builder — both stages must pin to the same immutable base.
FROM debian:12-slim@sha256:0104b334637a5f19aa9c983a91b54c89887c0984081f2068983107a6f6c21eeb AS runtime
ARG ORT_VERSION

# Runtime-only deps: ca-certs for general hygiene; libstdc++6 because the
# ONNX Runtime shared library uses C++ symbols internally
RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates libstdc++6 \
    && rm -rf /var/lib/apt/lists/*

# Non-root user
RUN useradd --create-home --uid 1001 app

# Copy the shared library (with symlinks preserved), the binary, and the model
COPY --from=builder /opt/onnxruntime/lib/libonnxruntime.so* /usr/local/lib/
COPY --from=builder /out/check_model /usr/local/bin/check_model
COPY --from=builder --chown=app:app /tmp/model.onnx /home/app/model.onnx

# Tell the dynamic linker where to find libonnxruntime at runtime
ENV LD_LIBRARY_PATH=/usr/local/lib

# 3a: Provenance LABELs — lets any engineer who pulls the image immediately
# know which model artifact is baked in, which framework exported it, which
# ORT version it links against, and who to contact. Visible via docker inspect.
LABEL model.source="m6-09-assessment"
LABEL model.framework="ultralytics-yolo26"
LABEL ort.version="${ORT_VERSION}"
LABEL maintainer="apoplavsky"

# 3c: HEALTHCHECK — Docker (and orchestrators like Compose/Swarm) use this to
# determine whether the container is healthy. For a verifier container the same
# binary that runs as CMD is the correct liveness probe: if the model file
# becomes unreadable or ORT fails to parse it, the container is unhealthy.
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD check_model /home/app/model.onnx || exit 1

USER app
WORKDIR /home/app

CMD ["check_model", "/home/app/model.onnx"]
