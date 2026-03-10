# MLServer Docker Image Analysis

Review date: 2026-03-09 | Version: 1.7.0.dev0

---

## 1. Build Architecture

The project uses a **single multi-stage Dockerfile** with a build arg `RUNTIMES` to produce all image variants from the same build definition.

```
Dockerfile
├── Stage 1: wheel-builder  (python:3.11.14-slim)
│   ├── Installs Poetry 2.1.1 + poetry-plugin-export
│   ├── Runs hack/build-wheels.sh → builds ALL wheels (core + 9 runtimes)
│   └── Exports constraints.txt from poetry lockfile
│
└── Stage 2: final image    (registry.access.redhat.com/ubi9/ubi-minimal)
    ├── Installs Miniforge3 (Conda 25.9.1) + Python 3.11.14 + FFmpeg
    ├── Creates non-root user (UID 1000)
    ├── Installs wheels filtered by RUNTIMES arg
    └── Entrypoint: activate conda → activate hot-loaded env → mlserver start
```

---

## 2. Image Variants (11 images per release)

All images use the tag pattern `seldonio/mlserver:<version>[-variant]` and are pushed to both DockerHub and Quay.io (Red Hat ISV certification).

### Core Images

| Image Tag | `RUNTIMES` Arg | Contents |
|---|---|---|
| `seldonio/mlserver:<ver>` | `all` | Core + all 8 runtimes (mllib excluded) |
| `seldonio/mlserver:<ver>-slim` | `""` (empty) | Core only, zero runtimes |

### Per-Runtime Images

| Image Tag | `RUNTIMES` Arg | ML Framework |
|---|---|---|
| `seldonio/mlserver:<ver>-sklearn` | `mlserver-sklearn` | scikit-learn + joblib |
| `seldonio/mlserver:<ver>-xgboost` | `mlserver-xgboost` | XGBoost |
| `seldonio/mlserver:<ver>-lightgbm` | `mlserver-lightgbm` | LightGBM |
| `seldonio/mlserver:<ver>-catboost` | `mlserver-catboost` | CatBoost |
| `seldonio/mlserver:<ver>-mlflow` | `mlserver-mlflow` | MLflow PyFunc |
| `seldonio/mlserver:<ver>-huggingface` | `mlserver-huggingface` | HuggingFace Transformers |
| `seldonio/mlserver:<ver>-alibi-detect` | `mlserver-alibi-detect` | Alibi Detect (drift/anomaly) |
| `seldonio/mlserver:<ver>-alibi-explain` | `mlserver-alibi-explain` | Alibi Explain (explainability) |
| `seldonio/mlserver:<ver>-mllib` | `mlserver-mllib` | Spark MLlib |

**Note:** mllib is deliberately excluded from the `all` image due to CVE-2022-25168 and CVE-2022-42889 in PySpark's bundled JARs. It is only available as a standalone image.

### Quay.io Red Hat Certification Path

All images are also pushed as:
```
quay.io/redhat-isv-containers/63566bb9822ce8cef9ba27fc:<version>[-variant]
```

---

## 3. User-Generated Custom Images (`mlserver build`)

When users run `mlserver build -t <tag> <folder>`, a **second Dockerfile** is generated from the template in `mlserver/cli/constants.py:4-70`. This produces custom images with user model code baked in.

```
Generated Dockerfile
├── Stage 1: env-builder  (continuumio/miniconda3:24.4.0-0)
│   ├── Installs conda-pack + conda-libmamba-solver
│   ├── Looks for environment.yml / conda.yml in user's folder
│   └── Packs conda env into ./envs/base.tar.gz
│
└── Stage 2: final        (seldonio/mlserver:<version>-slim)
    ├── Copies packed conda env from stage 1
    ├── Copies user's settings.json, model-settings.json, requirements.txt
    ├── Runs hack/build-env.sh (pip install -r requirements.txt)
    ├── Runs hack/generate_dotenv.py (bakes settings as env vars in .env)
    ├── COPY . . (copies all user model code)
    └── CMD: activate base env → activate hot-loaded env → mlserver start
```

Base image for custom builds: `seldonio/mlserver:<version>-slim`

---

## 4. Base Image Stack

### Production Images
```
registry.access.redhat.com/ubi9/ubi-minimal          <- RHEL 9 minimal (security-focused)
  + tar, gzip, libgomp, mesa-libGL, glib2-devel       <- System libs
  + git                                                <- For alibi git branch deps (temporary)
  + Miniforge3 (conda-forge distribution)              <- Package manager
    + conda 25.9.1                                     <- Conda
    + python 3.11.14                                   <- Python runtime
    + ffmpeg                                           <- Media processing (HuggingFace)
  + pip packages:
    + mlserver (core wheel)                            <- Server
    + mlserver-<runtime> (runtime wheel(s))            <- ML framework integration
    + uvloop 0.21.0                                    <- High-performance event loop
```

### Custom Images (user-generated)
```
continuumio/miniconda3:24.4.0-0                       <- Stage 1 only (env builder)

seldonio/mlserver:<version>-slim                      <- Stage 2 base
  + user's packed conda environment (./envs/base.tar.gz)
  + user's pip requirements (requirements.txt)
  + user's model code, settings, artifacts
```

---

## 5. Runtime Selection Logic

From `Dockerfile:99-119`:

```bash
if RUNTIMES == "all":
    # Install every mlserver_*.whl found in dist/ EXCEPT mllib
    for _wheel in ./dist/mlserver_*.whl; do
        if [[ ! $_wheel == *"mllib"* ]]; then
            pip install $_wheel --constraint ./dist/constraints.txt
        fi
    done
elif RUNTIMES != "":
    # Install only specified runtime(s), space-separated
    for _runtime in $RUNTIMES; do
        _wheelName=$(echo $_runtime | tr '-' '_')
        pip install ./dist/${_wheelName}-*.whl --constraint ./dist/constraints.txt
    done
fi

# Always install core mlserver wheel last
pip install ./dist/mlserver-*.whl --constraint ./dist/constraints.txt

# Always install uvloop for performance
pip install uvloop==0.21.0
```

Constraints file is generated via `poetry export --with all-runtimes --format constraints.txt` ensuring reproducible dependency resolution.

---

## 6. Container Runtime Behavior

### Environment Variables

| Variable | Default | Purpose |
|---|---|---|
| `MLSERVER_MODELS_DIR` | `/mnt/models` | K8s model volume mount point |
| `MLSERVER_ENV_TARBALL` | `/mnt/models/environment.tar.gz` | Hot-loaded conda environment |
| `MLSERVER_PATH` | `/opt/mlserver` | Server installation directory |
| `CONDA_PATH` | `/opt/conda` | Conda installation path |
| `HF_HOME` | `/opt/mlserver/.cache` | HuggingFace model cache |
| `NUMBA_CACHE_DIR` | `/opt/mlserver/.cache` | Numba JIT compilation cache |
| `PATH` | `/opt/mlserver/.local/bin:/opt/conda/bin:$PATH` | Extended PATH |

### Startup Sequence

**Production images** (`Dockerfile:133-135`):
```bash
CMD . /opt/conda/etc/profile.d/conda.sh && \
    source ./hack/activate-env.sh $MLSERVER_ENV_TARBALL && \
    mlserver start $MLSERVER_MODELS_DIR
```

1. Activate conda base environment
2. If `$MLSERVER_ENV_TARBALL` exists (`/mnt/models/environment.tar.gz`):
   - Extract tarball to `./envs/<name>/`
   - Source the packed env's `bin/activate`
   - Run `conda-unpack` to fix hardcoded paths
   - Set `PYTHONNOUSERSITE=True` to isolate from user packages
3. Run `mlserver start /mnt/models`

**Custom images** (generated by `mlserver build`):
```bash
CMD source ./hack/activate-env.sh ./envs/base.tar.gz && \
    mlserver start $MLSERVER_MODELS_DIR
```

1. Activate the build-time embedded conda env from `./envs/base.tar.gz`
2. Run `mlserver start /mnt/models`

### Exposed Ports

| Port | Protocol | Service |
|---|---|---|
| 8080 | HTTP | REST API (FastAPI/Uvicorn) |
| 8081 | gRPC | gRPC inference service |
| 8082 | HTTP | Prometheus metrics |

---

## 7. Build & Publish Pipeline

### Local Build (`hack/build-images.sh`)

```bash
./hack/build-images.sh <version>
```

Produces:
1. `seldonio/mlserver:<version>` (full, all runtimes)
2. `seldonio/mlserver:<version>-slim` (core only)
3. `seldonio/mlserver:<version>-sklearn` (one per runtime)
4. ... (9 runtime-specific images)

### CI/CD Release Pipeline (`.github/workflows/release.yml`)

Triggered by: `workflow_dispatch` with version input.

**Per image, the pipeline executes:**

| Step | Tool | Purpose |
|---|---|---|
| 1. Build | `docker build` with BuildKit | Produce image |
| 2. Scan | Snyk (`snyk/actions/docker`) | `--fail-on=upgradable --severity-threshold=high --app-vulns` |
| 3. Push DockerHub | `docker push` | `seldonio/mlserver:<tag>` |
| 4. Push Quay.io | `docker tag` + `docker push` | `quay.io/redhat-isv-containers/...:<tag>` |
| 5. Preflight | `preflight check container` | Red Hat OpenShift container certification |
| 6. Submit | Preflight + Pyxis API | Submit certification results to Red Hat |

**Parallel execution:** The `runtimes` job uses a strategy matrix to build all 9 runtime images in parallel. Core (`all`) and slim images are built in separate parallel jobs.

### PyPI Publishing (same workflow)

Core `mlserver` wheel and all 9 runtime wheels are published to PyPI via `poetry publish`.

---

## 8. Security Posture

### Strengths

| Practice | Details |
|---|---|
| Non-root user | `useradd -u 1000`, `USER 1000` at runtime |
| Security-focused base | Red Hat UBI 9 Minimal (RHEL patching, minimal attack surface) |
| Snyk scanning | Every image scanned before publish (HIGH threshold, fail on upgradable) |
| Red Hat preflight | OpenShift certification validates image structure and security |
| Pinned versions | Python 3.11.14, Conda 25.9.1, Poetry 2.1.1, uvloop 0.21.0 |
| Reproducible builds | constraints.txt from poetry lockfile pins all transitive deps |
| pip cache cleared | `rm -rf /root/.cache/pip` reduces image size and prevents cache poisoning |
| Random UID support | `chmod -R 776` enables OpenShift's arbitrary UID assignment |
| CVE exclusion | mllib excluded from `all` image due to known PySpark CVEs |

### Weaknesses & Gaps

| Issue | Severity | Details |
|---|---|---|
| No image signing | Medium | No cosign/Notary/Sigstore signatures. Users cannot verify image authenticity. |
| No SBOM generation | Medium | No syft/trivy SBOM. No software bill of materials alongside images. |
| git in production image | Low-Med | Installed for alibi git branch dependencies. Increases attack surface. Dockerfile comments indicate this is temporary. |
| `chmod 776` overpermissive | Low | World-writable `/opt/mlserver` for random UID support. `775` would suffice for group-writable. |
| No `HEALTHCHECK` | Low | No Docker-native health check. Fine in K8s (uses probes), but standalone Docker deployments have no health monitoring. |
| No multi-arch builds | Low | Only `linux-x86_64` (Miniforge download). No ARM64/aarch64 support. |
| wget in build layer | Low | `wget` is installed for Miniforge download but removed after (`microdnf remove -y wget`). Clean. |
| Hot-loaded env risk | Medium | `$MLSERVER_ENV_TARBALL` extracts and sources arbitrary conda environments at runtime from the model volume. If the volume is compromised, the extracted env runs arbitrary code. |

### Snyk CVE Exceptions (`.snyk`)

8 Java CVEs are explicitly ignored - all from PySpark's bundled JARs (only affects mllib image):

| CVE | Component | Reason |
|---|---|---|
| SNYK-JAVA-COMGOOGLEPROTOBUF-3167772 | protobuf-java | PySpark bundle, no upgrade path |
| SNYK-JAVA-COMGOOGLEPROTOBUF-2331703 | protobuf-java | PySpark bundle, no upgrade path |
| SNYK-JAVA-COMGOOGLEPROTOBUF-173761 | protobuf-java | PySpark bundle, no upgrade path |
| SNYK-JAVA-IONETTY-5725787 | netty-handler | PySpark bundle, no upgrade path |
| SNYK-JAVA-ORGAPACHEHADOOP-3034197 | hadoop-client | PySpark bundle, no upgrade path |
| SNYK-JAVA-ORGAPACHEHIVE-2952701 | hive-exec | PySpark bundle, no upgrade path |
| SNYK-JAVA-ORGAPACHETHRIFT-1074898 | libthrift | PySpark bundle, no upgrade path |
| SNYK-JAVA-ORGAPACHETHRIFT-474610 | libthrift | PySpark bundle, no upgrade path |

---

## 9. Image Relationship Diagram

```
                        Dockerfile (multi-stage)
                              |
                    +---------+---------+
                    |                   |
              wheel-builder        ubi9/ubi-minimal
           (python:3.11-slim)     + miniforge + python
                    |                   |
            builds all wheels     RUNTIMES arg selects
                    |                   |
              dist/*.whl          +-----+---------------------+
                    |             |     |                     |
                    +-------->  "all"  ""(slim)        "mlserver-X"
                                  |     |                     |
                                  v     v                     v
                        :version    :version-slim    :version-sklearn
                                                     :version-xgboost
                                                     :version-lightgbm
                                                     :version-catboost
                                                     :version-mlflow
                                                     :version-huggingface
                                                     :version-alibi-detect
                                                     :version-alibi-explain
                                                     :version-mllib

    +---------------------------------------------+
    |  mlserver build (CLI-generated Dockerfile)  |
    |                                             |
    |  miniconda3:24.4.0 --> env-builder          |
    |  :version-slim     --> final image          |
    |    + user code, settings, requirements      |
    |    + packed conda env                       |
    +---------------------------------------------+
```

---

## 10. Key File Locations

| File | Purpose |
|---|---|
| `Dockerfile` | Main multi-stage build for all image variants |
| `mlserver/cli/constants.py` | Template Dockerfile for `mlserver build` custom images |
| `mlserver/cli/build.py` | `build_image()` function that invokes `docker build` |
| `hack/build-wheels.sh` | Builds Poetry wheels for core + all runtimes |
| `hack/build-images.sh` | Shell script to build all 11 image variants locally |
| `hack/build-env.sh` | Installs requirements.txt inside container |
| `hack/activate-env.sh` | Extracts and activates packed conda environments |
| `hack/generate_dotenv.py` | Converts settings.json to .env file for custom images |
| `.github/workflows/release.yml` | CI/CD pipeline: build, scan, push, certify |
| `.snyk` | Snyk policy with PySpark CVE exceptions |
