![logo_ironhack_blue 7](https://user-images.githubusercontent.com/23629340/40541063-a07a0a8a-601a-11e8-91b5-2f13e4e6b441.png)

# Lab | Containerization with Docker for ML

## Overview

You'll take your **cat-detection ONNX model** from the m6-09 Week-2 assessment and ship it inside a slim, multi-stage Docker image. **The repo already includes a working Dockerfile** — this lab is about reading it, understanding it, modifying it, and shipping the result. That's how junior platform engineers actually learn Docker on the job: read existing Dockerfiles, modify them, observe the build.

This is a 90-minute hands-on lab. **No Python**. No HTTP server (Day 4 covers contracts; Day 5 covers serving at scale). The container's job is to be a *model verifier* — `docker run --rm <image>` loads the model with ONNX Runtime's C API and exits 0 if it parsed, 1 if not.

## Learning Goals

By the end of this lab you should be able to:

- Read a multi-stage Dockerfile and explain what each stage exists for
- Modify a Dockerfile to add provenance metadata, pin a base image by digest, and wire up a HEALTHCHECK
- Build, tag, push, and verify a public container image that runs as a non-root user under ~250 MB

## Setup

You need:

- Docker Desktop or Docker Engine (`docker --version` should work)
- A free account on **Docker Hub** or **GitHub Container Registry** (your GitHub account works)
- Your **`model.onnx`** from the m6-09 assessment (YOLO26 export). If you didn't keep a copy, grab it from your m6-09 lab repo.

Fork and clone this repo. It contains:

```
Dockerfile               # working reference — you'll modify this
.dockerignore            # working — adjust if you change scope
.gitignore               # already excludes model.onnx
src/
└── check_model.c        # 27-line C verifier — do not modify
```

Drop your `model.onnx` into the repo root before building. It's gitignored, so it won't be committed.

## Tasks

### Task 1 — Run the reference build, baseline the image

1. Copy your `model.onnx` into the repo root.
2. Build the image as-is:
   ```bash
   docker build -t <your-namespace>/m7-03-cat-detection:v1 .
   ```
3. Run it and confirm the verifier loads your model:
   ```bash
   docker run --rm <your-namespace>/m7-03-cat-detection:v1
   ```
   Expected output (your input/output counts may differ):
   ```
   ONNX model loaded OK: /home/app/model.onnx
     inputs:  1
     outputs: 1
   ```
4. Note the **baseline image size** from `docker images` — write it down; you'll compare against your final version.

If the reference build fails on your machine, that's a Task-1 deliverable: read the error, fix it, and write a one-paragraph post-mortem in `DOCKERFILE_NOTES.md` under a `## Baseline build` heading. (Most failures will be a stale Debian mirror or a corrupted ONNX Runtime download — try a retry first.)

### Task 2 — Annotate the Dockerfile

Create `DOCKERFILE_NOTES.md` and write a short, opinionated read-through. Required sections:

```markdown
# Dockerfile Notes

## Baseline build
- Image size: <MB>
- Output of `docker run --rm <image>`:
  <pasted>

## Stage 1 (builder) — why it exists
<one paragraph>

## Stage 2 (runtime) — why it exists
<one paragraph>

## Three architectural decisions in this Dockerfile
1. <name + 1-sentence "what would break if you removed it">
2. <name + 1-sentence "what would break if you removed it">
3. <name + 1-sentence "what would break if you removed it">
```

Concrete prompts to choose from for the "three architectural decisions": multi-stage split, `--no-install-recommends`, the `apt-get … && rm -rf /var/lib/apt/lists/*` pattern, the validation gate (`file | grep -qi onnx`), copying the .so with a glob (`libonnxruntime.so*`) instead of one file, `LD_LIBRARY_PATH`, the non-root user, the position of `COPY model.onnx` in the build (cache impact).

Pick the three you find most consequential. Defend the choice.

### Task 3 — Make three improvements

Modify the `Dockerfile` to add **all three** of these. Each is a small, real improvement a platform team would actually ship.

**3a. Add provenance LABELs to stage 2.**

```dockerfile
LABEL model.source="m6-09-assessment"
LABEL model.framework="ultralytics-yolo26"
LABEL ort.version="${ORT_VERSION}"
LABEL maintainer="<your-github-handle>"
```

Verify with `docker image inspect <image> --format '{{json .Config.Labels}}'`.

**3b. Pin the Debian base image by digest in both stages.**

Replace `FROM debian:12-slim` with `FROM debian:12-slim@sha256:<digest>`. Get the current digest:

```bash
docker pull debian:12-slim
docker inspect --format '{{index .RepoDigests 0}}' debian:12-slim
```

Use the printed digest. Both stages must pin to the same digest. (This is supply-chain hygiene — tag-based pulls can drift; digests can't.)

**3c. Add a HEALTHCHECK to stage 2.**

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD check_model /home/app/model.onnx || exit 1
```

(For a verifier container the HEALTHCHECK is the same command as CMD — fine for this lab. In a real serving container it would hit a `/health` endpoint.)

Rebuild after each change. Verify each one works:

```bash
docker build -t <your-namespace>/m7-03-cat-detection:v2 .
docker run --rm <your-namespace>/m7-03-cat-detection:v2          # still loads
docker inspect <your-namespace>/m7-03-cat-detection:v2 --format '{{.Config.Healthcheck}}'
docker inspect <your-namespace>/m7-03-cat-detection:v2 --format '{{json .Config.Labels}}' | jq .
```

Record the v2 image size in `DOCKERFILE_NOTES.md` under `## Final build` along with a paste of the labels and the healthcheck spec.

### Task 4 — Push and document

```bash
docker login                       # or: docker login ghcr.io
docker push <your-namespace>/m7-03-cat-detection:v2
```

The image must be **public**.

Update this README in the section titled `## Image` (add it at the bottom) with exactly these four items:

1. **Pull command** — fenced shell block with the working `docker pull` command
2. **Run command** — fenced shell block with `docker run --rm <image>`
3. **Image size** — final image size in MB
4. **Sample output** — the actual stdout from running your image

## Submission

Open a Pull Request to the lab repository with:

```
Dockerfile                  # your modified version (v2)
.dockerignore               # adjust only if you changed scope
.gitignore                  # already in starter
DOCKERFILE_NOTES.md         # NEW — your annotations and observations
README.md                   # with the ## Image section completed
src/check_model.c           # unchanged
```

**Do not commit `model.onnx`.** Paste the PR link as your deliverable. The PR description must include the public `docker pull` command on its own line.

## Quality bar

You will be reviewed on:

- **Does the image actually run** when pulled fresh, and produce the expected stdout?
- **All three improvements applied:** LABELs visible, base pinned by digest, HEALTHCHECK present? Inspecting the image must prove each one.
- **Are the annotations specific?** "It's multi-stage so the image is smaller" fails the bar. "Without stage 1's build-essential we couldn't compile check_model; without stage 2 discarding it we'd ship a 600 MB image" passes.
- **Is the final image under ~250 MB?** A non-multi-stage build would be ~600 MB; if you're there, the change didn't land.
- **Is the user non-root?** `docker run --rm --entrypoint /bin/sh <image> -c id` must show uid 1001.
- **Is the image public?** Login-walled registries fail.

This is a real Day-3 packaging exercise. Read carefully, modify intentionally, ship cleanly.

---

## Image

### Pull command

```shell
docker pull rrrevan/m7-03-cat-detection:v2
```

### Run command

```shell
docker run --rm rrrevan/m7-03-cat-detection:v2
```

### Image size

**268 MB**

### Sample output

```
ONNX model loaded OK: /home/app/model.onnx
  inputs:  1
  outputs: 1
```
