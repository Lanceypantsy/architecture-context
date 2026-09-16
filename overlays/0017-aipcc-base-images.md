---
id: "0017"
title: AIPCC Base Images
status: active
created: 2026-05-26
affects:
  - platform
release:
  - "3.6"
  - "next"
provenance:
  - https://gitlab.com/redhat/rhel-ai/wheels/fondue/-/tree/main/images/base
author: Doug Hellmann
superseded_by: null
---

## Fact

The AIPCC base images provide RHEL-based application containers with
runtime dependencies for hardware accelerators used in AI
workloads. The images also provide access to versions of python
packages built in Red Hat's secure build pipelines.  Downstream teams
extend these images by installing Python wheels and additional
packages to create product containers (e.g., vLLM, InstructLab).

Images follow a layout similar to
[s2i-base-containers](https://github.com/sclorg/s2i-base-container)
but are not `s2i` images. Each image runs as an unprivileged user
(UID 1001) and ships `pip` and `uv` pre-configured with the RHEL AI
Python Package Index.

Two families of base images are produced:

- **RHAI base images** (`rhaibi-*`) — the primary images consumed by downstream
  product teams (vLLM, InstructLab, etc.). Use the standard RHAI package index.
- **Torch Day 0 base images** (`torch-base-*`) — bootstrap images for building
  PyTorch wheels. Use a separate Torch-specific package index
  (`torch-2.14.0`) and carry `DISTRIBUTION_SCOPE=authoritative-source-only`.

### Common Foundation

All images share:

- **Base OS:** RHEL 9.8 (`registry.redhat.io/rhel9-8-els/rhel:9.8-1789348956`)
- **Python:** 3.12
- **RHEL AI repo version:** 3.6
- **Package index version:** 3.6
- **Repositories:** BaseOS, AppStream, CodeReady Builder, RHELAI (EUS
  repos on even y-stream releases like 9.8)
- **Container layout:** `/opt/app-root/` with `pip.conf` and `uv.toml`
  pre-configured with a single `index-url` (variant-specific; see per-variant
  sections below — not all variants use the public RHEL AI index)
- **Environment metadata:** `/etc/rhaipcc/env` provides shell variables
  for variant, versions, and repository info
- **Helper script:** `/usr/libexec/rhaipcc/dnf` enables vendor repos
  for additional package installs

### Dependency Management Model

Package dependencies use a declarative lockfile approach (as of 3.6):

- **`context/<variant>/rpms.in.yaml`** — the authoritative declaration of
  packages per variant. Specifies `contentOrigin.repofiles` (which DNF repos to
  resolve against), `arches` (target architectures), and `packages` (flat list
  of package names, with optional `arches: {only: [...]}` per-package scoping).
  For accelerator-specific RPMs (Spyre IBM SDK, Neuron SDK, CUDA libraries,
  Gaudi/Habanalabs RPMs), the version is expressed directly in the package name
  (e.g. `ibm-aiu-toolbox-e2e-1.3.0`, `aws-neuronx-runtime-lib-2.33.10.0_3dcef56f0-1`,
  `habanalabs-graph-1.24.1-482.el9`). These versioned names are the single
  source of truth for SDK versions (AIPCC-29839).
- **`context/<variant>/rpms.lock.yaml`** — the hermetic lockfile produced by
  `rpm-lockfile-prototype`. Contains per-arch closures with full CDN URL,
  sha256/sha512 checksum, size, name, and EVR for every resolved package.
  Consumed directly by Konflux (via Cachi2) during hermetic builds; packages
  are fetched from the locked CDN URLs, not re-resolved at build time.
- **MintMaker** — handles routine version drift automatically via the
  `refresh-rpm-lockfiles` Renovate preset. Manual lockfile regeneration
  (`bin/hermetic-generate-lockfiles.sh`) is only needed when adding or removing
  packages from `rpms.in.yaml`. CI enforces coupling: if `rpms.in.yaml` changes,
  `rpms.lock.yaml` must also change in the same commit (AIPCC-29840).

### Accelerator Summary

| Accelerator                   | Version | Status         | Python | RHEL | aarch64 | ppc64le | s390x | x86\_64 |
|-------------------------------|---------|----------------|--------|------|---------|---------|-------|---------|
| CPU                           | --      | Active         | 3.12   | 9.8  | Yes     | Yes     | Yes   | Yes     |
| Torch Day 0 CPU               | --      | Active         | 3.12   | 9.8  | Yes     | --      | --    | Yes     |
| NVIDIA CUDA                   | 12.9.1  | Active         | 3.12   | 9.8  | Yes     | --      | --    | Yes     |
| NVIDIA CUDA                   | 13.0.2  | Active         | 3.12   | 9.8  | Yes     | --      | --    | Yes     |
| Torch Day 0 NVIDIA CUDA       | 13.0.2  | Active         | 3.12   | 9.8  | Yes     | --      | --    | Yes     |
| NVIDIA CUDA                   | 13.2.1  | Active         | 3.12   | 9.8  | Yes     | --      | --    | Yes     |
| NVIDIA Rubin                  | 13.4.0  | In development | 3.12   | 9.8  | Yes     | --      | --    | Yes     |
| AMD ROCm                      | 7.14    | Active         | 3.12   | 9.8  | --      | --      | --    | Yes     |
| Intel Gaudi                   | 1.24.1  | Active         | 3.12   | 9.8  | --      | --      | --    | Yes     |
| IBM Spyre                     | 1.3.0   | Active         | 3.12   | 9.8  | --      | Yes     | Yes   | Yes     |
| AWS Neuron                    | (lock)  | In development | 3.12   | 9.8  | --      | --      | --    | Yes     |
| Google TPU                    | --      | In development | 3.12   | 9.8  | --      | --      | --    | Yes     |
| AMD ROCm 6.4                  | 6.4     | Retired        | --     | --   | --      | --      | --    | --      |

### Status Legend

- **Active** -- supported and built in CI
- **In development** -- under active development, not yet GA
- **Disabled** -- configuration exists but builds are skipped
- **Retired** -- removed from the repository

### CPU

- **Status:** Active
- **Config:** `build-args/cpu-el9.8-app.conf`
- **Lockfile input:** `context/cpu/rpms.in.yaml`
- **Architectures:** aarch64, ppc64le, s390x, x86\_64
- **Container:** `rhaibi-cpu`
- **Python package index:** Public RHEL AI index (`https://packages.redhat.com/api/pypi/public-rhai/rhoai`)
- **Distribution scope:** `authoritative-source-only`
- **Extra dependencies:** None. The CPU image is the simplest variant
  with no accelerator-specific packages. `ppc64le`/`s390x` include
  `gcc-toolset-14` and `gcc-toolset-14-gcc-c++` for `torch.compile` support
  (AIPCC-29662).

### Torch Day 0 CPU

- **Status:** Active
- **Config:** `build-args/torch-cpu-el9.8-app.conf`
- **Architectures:** aarch64, x86\_64
- **Container:** `torch-base-cpu`
- **Python package index:** Public RHEL AI index, Torch Day 0 path (`https://packages.redhat.com/api/pypi/public-rhai/torch-2.14.0-cpu-ubi9-test/simple/`; uses `-` separator instead of `/`)
- **Distribution scope:** `authoritative-source-only`
- **Notes:** Bootstrap image for building CPU-based PyTorch wheels. Not
  intended for direct use by product teams consuming RHAI base images.

### NVIDIA CUDA

Three active CUDA versions are maintained, sharing a single
`Containerfile.cuda-app` with version-specific behavior controlled by
build args. A fourth version (Rubin / CUDA 13.4) is in development.
A Torch Day 0 variant is also produced for CUDA 13.0.

All CUDA library version pins (cuDNN, cuDSS, cuSPARSELt, cuTENSOR,
NVSHMEM, UCX) are declared in `context/cuda-<ver>/rpms.in.yaml` as
versioned package names; the lockfile resolves the full CDN closure.

#### CUDA 12.9.1

- **Status:** Active
- **Config:** `build-args/cuda12.9-el9.8-app.conf`
- **Lockfile input:** `context/cuda-12.9/rpms.in.yaml`
- **Architectures:** aarch64, x86\_64
- **Container:** `rhaibi-cuda12.9-el9.8`
- **Python package index:** Public RHEL AI index (`https://packages.redhat.com/api/pypi/public-rhai/rhoai`)
- **Distribution scope:** `private`
- **Driver requirement:** `>=525.60.13`
- **Key pinned libraries:**
  - cuDNN 9.22.0.52-1 (`libcudnn9-cuda-12-9.22.0.52-1`)
  - cuDSS 0.7.1.4-1
  - NVSHMEM 3.5.19-1
  - cuSPARSELt 0.8.1.1-1
  - cuTENSOR 2.7.0.5-1
  - UCX 1.21.0

#### CUDA 13.0.2

- **Status:** Active
- **Config:** `build-args/cuda13.0-el9.8-app.conf`
- **Lockfile input:** `context/cuda-13.0/rpms.in.yaml`
- **Architectures:** aarch64, x86\_64
- **Container:** `rhaibi-cuda13.0-el9.8`
- **Python package index:** Public RHEL AI index (`https://packages.redhat.com/api/pypi/public-rhai/rhoai`)
- **Distribution scope:** `private`
- **Driver requirement:** `>=580.95.05`
- **Key pinned libraries:**
  - cuDNN 9.19.0.56-1 (`libcudnn9-cuda-13-9.19.0.56-1`)
  - cuDSS 0.7.1.4-1
  - NVSHMEM 3.5.19-1
  - cuSPARSELt 0.9.1.1-1
  - cuTENSOR 2.7.0.5-1
  - UCX 1.21.0

#### Torch Day 0 CUDA 13.0.2

- **Status:** Active
- **Config:** `build-args/torch-cuda13.0-el9.8-app.conf`
- **Lockfile input:** `context/cuda-13.0/rpms.in.yaml` (shares lockfile variant with CUDA 13.0)
- **Architectures:** aarch64, x86\_64
- **Container:** `torch-base-cuda13.0-el9.8`
- **Python package index:** Public RHEL AI index, Torch Day 0 path (`https://packages.redhat.com/api/pypi/public-rhai/torch-2.14.0-cuda13.0-ubi9-test/simple/`; uses `-` separator)
- **Distribution scope:** `private`
- **Driver requirement:** `>=580.95.05`
- **Key pinned libraries:**
  - NCCL 2.30.4 (`LIBNCCL_VERSION=2.30.4-1`)
  - cuDNN 9.19.0.56 (`CUDNN_VERSION=9.19.0.56`)
  - UCX 1.20.1 (`CUDA_UCX_VERSION=1.20.1-1`)
  - cuBLASMp 0.x (`CUBLASMP_MAJOR_VERSION=0`)
  - cuDSS 0.7.1.4 (`CUDSS_VERSION=0.7.1.4`)
  - cuSPARSELt 0.9.1.1 (`CUSPARSELT_VERSION=0.9.1.1`)
  - cuTENSOR 2.7.0.5 (`CUTENSOR_VERSION=2.7.0.5`)
  - NVSHMEM 3.5.19 (`NVSHMEM_CUDA_VERSION=3.5.19-1`)

#### CUDA 13.2.1

- **Status:** Active
- **Config:** `build-args/cuda13.2-el9.8-app.conf`
- **Lockfile input:** `context/cuda-13.0/rpms.in.yaml` (shares lockfile variant with CUDA 13.0)
- **Architectures:** aarch64, x86\_64
- **Container:** `rhaibi-cuda13.2-el9.8`
- **Python package index:** Public RHEL AI index (`https://packages.redhat.com/api/pypi/public-rhai/rhoai`)
- **Distribution scope:** `private`
- **Driver requirement:** `>=595.58.03`

### NVIDIA Rubin

- **Status:** In development (CUDA 13.4 Developer Preview)
- **Config:** `build-args/rubin-el9.8-app.conf`
- **Lockfile input:** `context/rubin/rpms.in.yaml`
- **Architectures:** aarch64, x86\_64
- **Container:** `rhaibi-rubin-el9.8`
- **Python package index:** Public RHEL AI index (`https://packages.redhat.com/api/pypi/public-rhai/rhoai`)
- **Distribution scope:** `private`
- **Driver requirement:** `>=616`
- **Key pinned libraries (from rpms.in.yaml):**
  - cuDNN 9.19.0.56-1 (`libcudnn9-cuda-13-9.19.0.56-1`)
  - cuDSS 0.7.1.4-1
  - NVSHMEM 3.5.19-1
  - cuSPARSELt 0.9.1.1-1
  - cuTENSOR 2.7.0.5-1
  - UCX 1.21.0
- **Notes:** Rubin targets the NVIDIA Rubin GPU architecture with CUDA 13.4.0
  (Developer Preview). Despite active CI coverage on both aarch64 and x86_64,
  strategies must not assume GA availability — this variant is still in the
  CUDA 13.4 Developer Preview phase.

### AMD ROCm

- **Status:** Active
- **Config:** `build-args/rocm7.14-el9.8-app.conf`
- **Lockfile input:** `context/rocm/rpms.in.yaml`
- **ROCm version:** 7.14
- **Architectures:** x86\_64
- **Container:** `rhaibi-rocm7.14-el9.8`
- **Python package index:** Public RHEL AI index (`https://packages.redhat.com/api/pypi/public-rhai/rhoai`)
- **Distribution scope:** `authoritative-source-only`
- **Key RPM packages (declared in rpms.in.yaml):** `amdrocm-runtime`,
  `amdrocm-blas`, `amdrocm-dnn`, `amdrocm-fft`, `amdrocm-rccl`, `amdrocm-solver`,
  `amdrocm-sparse`, `amdrocm-rand`, `amdrocm-llvm`, `amdrocm-amdsmi`,
  `migraphx`, `gcc-toolset-14`. Package versions are not pinned by name in
  rpms.in.yaml; the lockfile resolves current RPM builds from the ROCm 7.14
  repo. ROCm 7.14 installs under `/opt/rocm/core-7.x/`; `/opt/rocm/core` is a
  symlink via alternatives to the versioned directory.

### Intel Gaudi

- **Status:** Active
- **Config:** `build-args/gaudi-el9.8-app.conf`
- **Lockfile input:** `context/gaudi/rpms.in.yaml`
- **Gaudi version:** 1.24.1 (revision 482)
- **Python:** 3.12
- **Architectures:** x86\_64
- **Container:** `rhaibi-gaudi`
- **Python package index:** Private RHAIIS index (`https://private.console.redhat.com/api/pypi/rhai/rhaiis`) — **replaces** the public RHEL AI index entirely. `pip.conf` and `uv.toml` are configured with a single `index-url` pointing to this private index; there is no fallback to the public index. The private index serves non-redistributable Habanalabs wheels. Downstream product container builds consume this index directly.
- **Distribution scope:** `private`
- **Key pinned RPMs (from rpms.in.yaml):**
  - `habanalabs-rdma-core-1.24.1-482.el9`
  - `habanalabs-thunk-1.24.1-482.el9`
  - `habanalabs-firmware-tools-1.24.1-482.el9`
  - `habanalabs-graph-1.24.1-482.el9`

Gaudi builds are fully enabled in GitLab CI (`VARIANT_ENABLED: true`). The
Tekton (Konflux) pipeline is gated to release tags (`refs/tags/v*` or
`refs/tags/gaudi-v*`) rather than every push; MR and tag builds run through
GitLab CI.

### IBM Spyre

- **Status:** Active
- **Config:** `build-args/spyre-el9.8-app.conf`
- **Lockfile input:** `context/spyre/rpms.in.yaml`
- **Architectures:** ppc64le, s390x, x86\_64
- **Container:** `rhaibi-spyre`
- **Python package index:** None. The Spyre conf sets the public RHEL AI index URL but carries a `# NOTE: index does not exist` comment — there are no Python wheels served via this index for Spyre. All Spyre Python dependencies come from RPMs (IBM SDK) or pre-built IBM wheels served via the builder's private Spyre PyPI (a separate mechanism internal to the wheel-build pipeline).
- **Distribution scope:** `private`
- **IBM SDK version:** 1.3.0 (all arches currently on same version)
- **IBM RPM packages pinned in `rpms.in.yaml` (AIPCC-29839):**

  | Package | Arches |
  |---------|--------|
  | `ibm-aiu-toolbox-e2e-1.3.0` | x86\_64, ppc64le, s390x |
  | `ibm-deeptools-1.3.0` | x86\_64, ppc64le, s390x |
  | `ibm-flex-1.3.0` | x86\_64, ppc64le, s390x |
  | `ibm-senlib-core-1.3.0` | x86\_64, ppc64le, s390x |
  | `ibm-senlib-dd2-1.3.0` | x86\_64, ppc64le, s390x |
  | `ibm-libaiupti-1.3.0` | x86\_64, ppc64le only |
  | `ibm-aiu-monitor-1.3.0` | ppc64le only |
  | `ibm-spyre-model-cache-1.3.0` | ppc64le, s390x only |
  | `ibm-z-spyre-runtime-1.3.0` | s390x only |
  | `libzdnn` | s390x only |
  | `gcc-toolset-14`, `gcc-toolset-14-gcc-c++` | ppc64le, s390x only |

  Version pins are expressed directly as versioned package names in `rpms.in.yaml`
  (the single source of truth, AIPCC-29839). The lockfile resolves the full CDN
  closure per arch. RPMs are served from a per-arch IBM Spyre yum repository.

### AWS Neuron

- **Status:** In development
- **Config:** `build-args/neuron-el9.8-app.conf`
- **Lockfile input:** `context/neuron/rpms.in.yaml`
- **Architectures:** x86\_64
- **Container:** `rhaibi-neuron`
- **Python package index:** Private RHAIIS index (`https://private.console.redhat.com/api/pypi/rhai/rhaiis`) — **replaces** the public RHEL AI index entirely. `pip.conf` and `uv.toml` are configured with a single `index-url` pointing to this private index; there is no fallback to the public index. The private index serves non-redistributable AWS Neuron SDK wheels. Downstream product container builds consume this index directly.
- **Distribution scope:** `private`
- **Neuron SDK packages pinned in `rpms.in.yaml` (AIPCC-29839):**
  - `aws-neuronx-runtime-lib-2.33.10.0_3dcef56f0-1`
  - `aws-neuronx-tools-2.31.15.0_5c7949a6a-1`
  - `aws-neuronx-collectives-2.33.10.0_068180c7a-1`

  Version strings include git commit hashes (e.g. `_3dcef56f0`), as is standard
  for Neuron SDK releases. Neuron SDK components are installed from a manually
  mirrored copy of the AWS Neuron yum repository. Strategies depending on Neuron
  updates must account for manual mirror synchronization delay and cannot assume
  same-day upstream releases.

### Google TPU

- **Status:** In development
- **Config:** `build-args/tpu-el9.8-app.conf`
- **Lockfile input:** `context/tpu/rpms.in.yaml`
- **Architectures:** x86\_64
- **Container:** `rhaibi-tpu`
- **Python package index:** Private RHAIIS index (`https://private.console.redhat.com/api/pypi/rhai/rhaiis`) — **replaces** the public RHEL AI index entirely. `pip.conf` and `uv.toml` are configured with a single `index-url` pointing to this private index; there is no fallback to the public index. The private index serves non-redistributable Google TPU / Torch-XLA wheels. Downstream product container builds consume this index directly.
- **Distribution scope:** `private`
- **Notes:** The TPU image uses Torch/XLA and has no version-specific
  accelerator dependencies in the conf file.

### Retired Accelerators

#### AMD ROCm 6.4

ROCm 6.4 was retired in RHAI 3.5-EA1
([AIPCC-15426](https://issues.redhat.com/browse/AIPCC-15426)). The
base image and Tekton pipelines were removed. ROCm 7.14 is the
current supported version.

## Impact on Strategies

- All RHAI components that use accelerator-specific Python libraries MUST use
  these base images; components that have not yet migrated must be updated to
  stay current with the platform.
- Package content is fixed at lockfile commit time. RFEs that add or remove
  accelerator RPMs must account for the full `rpms.in.yaml` → lockfile
  regeneration → CI lint validation cycle (AIPCC-29840). Version drift of
  existing packages is handled automatically by MintMaker; only package list
  changes require manual lockfile regeneration.
- Active vs. in-development status matters: AWS Neuron, Google TPU, and NVIDIA
  Rubin are not GA — strategies must not assume their availability in production
  workloads.
- Intel Gaudi is Active (AIPCC-3471 resolved in 3.5-EA2). GitLab CI builds are
  fully enabled on x86_64. The Tekton/Konflux pipeline is gated to release tags.
- AMD ROCm 6.4 is retired; any references in strategies or RFEs must be updated
  to ROCm 7.14, the current supported version.
- Three active CUDA versions (12.9, 13.0, 13.2) are maintained simultaneously,
  with a fourth (Rubin / CUDA 13.4) in development. Strategies and RFEs proposing
  CUDA-dependent features should specify the minimum driver version requirement,
  since each version has a different minimum driver (525, 580, 595, 616). Note
  that cuDNN versions differ between CUDA 12.9 (9.22.0.52) and CUDA 13.0/13.2
  (9.19.0.56); CUDA-version-specific cuDNN changes require coordinated lockfile
  updates.
- IBM Spyre and AWS Neuron SDK version pins are declared as versioned package
  names in `context/<variant>/rpms.in.yaml` (AIPCC-29839) — not in build-args
  conf files and not as free-standing lockfile variables. Strategies that
  reference specific SDK versions should cite the package names from rpms.in.yaml.
  IBM Spyre uses arch-conditional packages; the ppc64le and s390x package sets
  differ from x86_64 (see table above).
- Torch Day 0 base images (`torch-base-*`) are bootstrap images for building
  PyTorch wheels with `DISTRIBUTION_SCOPE=authoritative-source-only`. Strategies
  must not use these as product base images; they are for the wheel-build
  pipeline only.
- Python package indexes differ by variant and must not be treated as uniform:
  - **CPU, CUDA, ROCm, Rubin, Torch Day 0** — use the public RHEL AI index
    (`https://packages.redhat.com/api/pypi/public-rhai/rhoai`). Strategies
    adding Python dependencies for these variants must ensure packages are
    available in the public index.
  - **Gaudi, Neuron, TPU** — use the private RHAIIS index
    (`https://private.console.redhat.com/api/pypi/rhai/rhaiis`), which
    **replaces** the public index entirely (single `index-url` in pip/uv
    config; no fallback). These variants require a private index because
    their wheels (Habanalabs, AWS Neuron SDK, Torch-XLA) are non-redistributable.
    Downstream product container builds consume the private index directly.
    Strategies targeting these variants must not assume the public RHEL AI
    index is reachable or sufficient — dependency resolution will fail if
    directed to the wrong index.
  - **Spyre** — has no Python wheel index at the base image layer. All Spyre
    Python dependencies enter via RPMs or the builder's private Spyre PyPI
    (a separate mechanism internal to the wheel-build pipeline).

## Context

This overlay was created to capture the accelerator base image landscape as a
reference for evaluating RFEs proposing accelerator updates or additions. The
generated architecture docs for individual components (vLLM, InstructLab, etc.)
do not describe the shared base image layer or the current status of each
accelerator variant. This overlay fills that gap so that strategy pipelines,
architecture reviews, and design validation tooling have an authoritative,
up-to-date view of which accelerators are active, in development, disabled, or
retired, and what the common foundation looks like across all variants. Updated 2026-09-16 (re-verified) by running the `update-aipcc-base-images-overlay`
skill against the local fondue checkout. All variant data confirmed accurate: base
OS pin (`9.8-1789348956`), Gaudi 1.24.1 (rev 482), Neuron SDK package versions,
Spyre IBM SDK 1.3.0 RPM names, CUDA library pins (cuDNN, cuSPARSELt, cuTENSOR,
NVSHMEM), and per-variant index routing unchanged.
