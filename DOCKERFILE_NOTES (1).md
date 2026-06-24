# Dockerfile Notes

## Baseline build

- **Image size:** 268 MB
- **Output of `docker run --rm rrrevan/m7-03-cat-detection:v1`:**

```
ONNX model loaded OK: /home/app/model.onnx
  inputs:  1
  outputs: 1
```

---

## Stage 1 (builder) — why it exists

Stage 1 exists because compiling a C program against ONNX Runtime requires a
full build toolchain (`build-essential`, `gcc`, `curl`, `file`) plus the ~80 MB
ORT release tarball. None of that belongs in a production image. The builder
stage is a disposable workshop: it installs compilers, downloads the ORT tarball
from GitHub, extracts it, compiles `check_model.c` into a static-linked binary,
and runs a validation gate to reject a missing or corrupted model *at build time*
rather than at runtime. Every artifact the builder produces that the runtime
actually needs is copied out via `COPY --from=builder` in stage 2; the rest —
gcc, curl, 200+ MB of build deps, the tarball — is discarded automatically
because Docker never writes the builder layer into the final image.

---

## Stage 2 (runtime) — why it exists

Stage 2 exists to produce the smallest possible image that can still run the
verifier binary. It starts from a fresh `debian:12-slim` (same base, zero
inherited layers from stage 1) and installs only the two packages the *running*
binary needs: `libstdc++6` (because ONNX Runtime's `.so` has C++ symbols, and
the system linker must resolve them at load time) and `ca-certificates` (for
network hygiene in case the container ever makes TLS calls). It then copies
exactly three things from stage 1: the compiled binary, the `.so` library, and
the model file. A non-root user (`app`, uid 1001) is created so the process
never runs as root — a hard requirement for most container security policies.
The result is an image in the ~268 MB range instead of the ~600 MB you'd get
from a single-stage build that retained the compiler and tarball.

---

## Three architectural decisions in this Dockerfile

### 1. Multi-stage split (`AS builder` / `AS runtime`)

Without the two-stage split we would have to either (a) install the build
toolchain and tarball in the runtime image, inflating it to ~600 MB and shipping
gcc and curl to production, or (b) pre-compile the binary outside Docker and
`COPY` it in, breaking the reproducibility guarantee (the binary would be linked
against whatever libs exist on the developer's machine, not inside the container
OS). The split gives us a fully reproducible, hermetic build *and* a lean
production image in a single `docker build` invocation.

### 2. `apt-get … && rm -rf /var/lib/apt/lists/*` cache purge

Docker commits each `RUN` instruction as a new layer. If the `apt-get update`
and `apt-get install` commands succeed but we leave `/var/lib/apt/lists/`
intact, those package index files (~30–50 MB) are frozen into the layer forever
— they cannot be removed by a later `RUN rm` because the earlier layer already
captured them. Chaining `&& rm -rf /var/lib/apt/lists/*` inside the *same* `RUN`
command ensures the lists never appear in any committed layer, so the image stays
as small as possible. Remove the purge and both the builder and runtime images
grow by 30–50 MB each with no runtime benefit.

### 3. Copying the `.so` with a glob (`libonnxruntime.so*`)

The ONNX Runtime release ships the shared library as a versioned file plus
several symlinks (e.g. `libonnxruntime.so.1.20.1`, `libonnxruntime.so.1`,
`libonnxruntime.so`). The dynamic linker resolves the `NEEDED` entry in
`check_model` by following the unversioned symlink → versioned symlink →
actual `.so`. If we copied only `libonnxruntime.so.1.20.1` by exact name, the
symlinks would be absent and the linker would report *"cannot open shared object
file"* at runtime, crashing the container. The glob `libonnxruntime.so*`
preserves all variants and symlinks in a single `COPY` instruction.

---

## Final build (v2)

- **Image size:** 268 MB *(the three additions add only metadata, not new packages)*
- **Labels** (`docker inspect rrrevan/m7-03-cat-detection:v2 --format '{{json .Config.Labels}}'`):

```json
{"maintainer":"apoplavsky","model.framework":"ultralytics-yolo26","model.source":"m6-09-assessment","ort.version":"1.20.1"}
```

- **Healthcheck** (`docker inspect rrrevan/m7-03-cat-detection:v2 --format '{{.Config.Healthcheck}}'`):

```
{[CMD-SHELL check_model /home/app/model.onnx || exit 1] 30s 10s 5s 0s 3}
```

  Interval: 30 s · Timeout: 10 s · Start period: 5 s · Retries: 3

- **Non-root user** (`docker run --rm --entrypoint /bin/sh rrrevan/m7-03-cat-detection:v2 -c id`):

```
uid=1001(app) gid=1001(app) groups=1001(app)
```

- **v2 run output:**

```
ONNX model loaded OK: /home/app/model.onnx
  inputs:  1
  outputs: 1
```
