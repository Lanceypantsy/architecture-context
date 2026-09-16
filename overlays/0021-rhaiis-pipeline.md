---
id: "0021"
title: RHAIIS Pipeline — vLLM Wheel Build Specification
status: active
created: 2026-07-15
affects:
  - platform
release:
  - "3.6"
provenance:
  - https://gitlab.com/redhat/rhel-ai/rhaiis/pipeline
author: Lance Barto
superseded_by: null
---

## Fact

`rhaiis/pipeline` is the **wheel build specification** for the Red Hat AI
Inference Server (RHAIIS, vLLM-based) and the Model Optimization (`model-opt`,
llmcompressor-based) toolchain. It declares per-hardware-variant which Python
packages to build and at what versions, then triggers the `builder` CI API to
execute those builds.

This repo does **not** publish packages to Pulp — that is handled separately by
`rhai/pipeline`. It contains no application code.

- **Product version:** `3.6-fast1` (unchanged)
- **Builder version pinned:** `v44.0.2` (was `v43.3.0`). As of the last refresh
  of the sibling [`0020-rhai-pipeline`](0020-rhai-pipeline.md) overlay
  (2026-07-30), `rhai/pipeline` was pinned to `v39.1.0` — this repo is now
  significantly ahead of that stale snapshot; the sibling overlay is due for
  its own refresh. The sibling [`0019-wheels-builder`](0019-wheels-builder.md)
  overlay (last refreshed 2026-07-30) documented builder's latest release as
  `v42.0.2`; this repo's `v44.0.2` is past that, so the wheels-builder overlay
  is also stale and comparisons should not be treated as authoritative.
- **Repeatable build mode:** `ENABLE_REPEATABLE_BUILD_MODE` is commented out
  for all variants on the main branch (not enabled); it is intended for release
  branches and must be uncommented when cutting a release branch.

### Variant Matrix

| Collection | Variant | Arch | vLLM Version | Torch | Notes |
|---|---|---|---|---|---|
| rhaiis | cuda13.0-ubi9 | x86_64, aarch64 | 0.26.0+rhaiv.8 | 2.11.0 | NeuralMagic fork; `cgraph-cuda13` extra |
| rhaiis | rubin-ubi9 | x86_64, aarch64 | 0.26.0+rhaiv.5 | 2.11.0 | NeuralMagic fork; `cgraph-cuda13` extra; geospatial and plugins commented out for Vera Rubin tech preview (INFERENG-9846); llm-d packages include `nixl-cu13` and `deep_gemm` not in cuda13.0 |
| rhaiis | rocm7.14-ubi9 | x86_64 | 0.26.0+rhaiv.7 | 2.11.0 | NeuralMagic fork |
| rhaiis | cpu-ubi9 | x86_64, s390x, ppc64le | 0.26.0+rhaiv.5 | 2.11.0 | NeuralMagic fork; `.cpu` local-version suffix dropped (plain `+rhaiv.N`); multi-arch; `zen` extra x86_64-only |
| rhaiis | gaudi-ubi9 | x86_64 | 0.26.0+rhaiv.7 | 2.11.0 | NeuralMagic-tagged fork; constraints-rules uses `[!c]*` filter |
| rhaiis | spyre-ubi9 | x86_64, ppc64le, s390x | 0.25.1+rhaiv.1.spyre | 2.11.0 | IBM fork; sendnn/IBM packages; spyremetrics + ibm-aiu-smi landed |
| rhaiis | neuron-ubi9 | x86_64 | 0.16.0+rhaiv.13 | 2.9.1 | Still furthest behind; neuronx stack pinned with git hashes |
| rhaiis | tpu-ubi9 | x86_64 | 0.26.0+rhaiv.2.tpu | 2.10.0 | Still uses torch-2.10.0 constraints-rules |
| model-opt | cuda13.0-ubi9 | x86_64, aarch64 | N/A | 2.11.0 | llmcompressor 0.12.0.1 + speculators 0.6.0.1 |
| vllm-omni | cuda13.0-ubi9 | x86_64, aarch64 | 0.26.0+rhaiv.5 (+vllm-omni 0.26.0+rhaiv.7) | 2.11.0 | AIPCC-12521: AIPCC vllm-omni productization |

vLLM rrev bumps this cycle: CUDA rhaiv.1→rhaiv.8, Rubin placeholder→rhaiv.5 (active),
ROCm rhaiv.1→rhaiv.7, CPU rhaiv.3.cpu→rhaiv.5 (suffix also dropped), Gaudi
rhaiv.4→rhaiv.7, Neuron rhaiv.12→rhaiv.13, vllm-omni base rhaiv.1→rhaiv.5 and
omni itself rhaiv.2→rhaiv.7. Spyre and TPU are unchanged.

Note: Rubin is no longer a placeholder — requirements and constraints are
populated as of this refresh, though geospatial and plugin packages are
commented out pending Vera Rubin CUDA stack availability (INFERENG-9846).

### Package Content by Variant

**cuda13.0-ubi9** (most package-rich):
- `vllm[audio,tensorizer,cgraph-cuda13]==0.26.0+rhaiv.8`
- `vllm-bart-plugin==0.6.0` (INFERENG-6019: BART plugin for custom BART model support)
- `vllm-beam-search-plugin==0.1.3` (INFERENG-9921: beam search decoding support; was `vllm-beamsearch-plugin==0.1.0` at last refresh — package name changed)
- FlashInfer: `flashinfer-cubin`, `flashinfer-jit-cache`, `flashinfer-python`
- llm-d / disaggregated inference: `nixl==1.3.1` (AIPCC-28994), `deep_ep==2.0.0+rhaiv.0`,
  `pplx-kernels==0.0.1` (INFERENG-1925)
- `bitsandbytes`, `triton`, `timm>=1.0.17`, `numba` (AIPCC-5334);
  `xformers` (AIPCC-2152: aarch64 missing dependency)
- `opentelemetry-exporter-prometheus` (INFERENG-2949)
- Geospatial: `algorithm-nexus[product]==0.2.3` (INFERENG-8558; was 0.2.2),
  `torchgeo` (AIPCC-8393). `geobenchv2` is no longer listed directly (pulled
  transitively via `algorithm-nexus`).
- CVE/security: `setuptools>=80.10.2` (AIPCC-9947)
- Constraints: `aiohttp>=3.13.3`, `urllib3>=2.6.3` (INFERENG-4285 CVE),
  `boto3==1.43.46` (INFERENG-4923), `numba==0.65.0` (INFERENG-6026),
  `numpy<2.5` (AIPCC-28010), `torchgeo<0.10` (INFERENG-10155),
  `transformers<5.15.0` (INFERENG-9834: vllm 0.26.0 range cap),
  `xgrammar<0.2.5` (INFERENG-10473: xgrammar/transformers version conflict),
  `grpcio-reflection<1.82.0` + `protobuf==6.33.6` (INFERENG-9832: nvidia-cutlass-dsl
  requires protobuf<7, newer grpcio-reflection requires protobuf>=7),
  `uv-build<0.12.6` (INFERENG-10262: builder image has rustc 1.95.0, uv-build 0.12.6+
  requires 1.96.0), `quack-kernels==0.6.4` (INFERENG-10588: cap before
  nvidia-cutlass-dsl>=4.7 requirement), `nvidia-cudnn-frontend<1.29.0`
  (INFERENG-10716: avoids apache-tvm-ffi unconditional dep promotion),
  `nvidia-cutlass-dsl==4.6.2` (INFERENG-10727: pin at version vllm+quack-kernels agree on)

**rubin-ubi9** (now populated; Vera Rubin CUDA variant):
- `vllm[audio,tensorizer,cgraph-cuda13]==0.26.0+rhaiv.5`
- FlashInfer: `flashinfer-cubin`, `flashinfer-jit-cache`, `flashinfer-python`
- `xformers` (AIPCC-2152), `timm>=1.0.17`, `bitsandbytes`, `triton`, `numba`
- llm-d stack: `nixl==1.3.1`, `nixl-cu13==1.3.1` (AIPCC-28994; both present,
  distinct from cuda13.0 which has only `nixl` without `-cu13`),
  `deep_gemm==2.5.0+rhaiv.0`, `deep_ep==2.0.0+rhaiv.0`, `pplx-kernels==0.0.1`
- `opentelemetry-exporter-prometheus` (INFERENG-2949)
- Geospatial and plugin packages (`vllm-bart-plugin`, `vllm-beam-search-plugin`,
  `geobenchv2`, `algorithm-nexus`, `torchgeo`) are commented out for Vera Rubin
  tech preview (INFERENG-9846)
- Constraints: same CVE/compat pins as cuda13.0 except no `grpcio-reflection<1.82.0`,
  `protobuf`, `uv-build<0.12.6`, `quack-kernels==0.6.4`, or `nvidia-cudnn-frontend`
  caps; shares `aiohttp`, `urllib3`, `boto3`, `numba`, `numpy<2.5`,
  `torchgeo<0.10`, `transformers<5.15.0`, `uv-build<0.12.6`, `quack-kernels==0.6.4`,
  `nvidia-cudnn-frontend<1.29.0`

**rocm7.14-ubi9:**
- `vllm[audio,tensorizer]==0.26.0+rhaiv.7`
- `amd-aiter==0.1.16.post3` — AMD attention/iteration kernel (required; unchanged)
- `flash-attn`, `triton`, `torch`, `torchaudio`, `torchvision` — all unpinned
  in requirements.txt; versions resolved by builder from the `torch-2.11.0`
  constraints-rules collection
- `timm>=1.0.17`, `bitsandbytes`, `numba` (AIPCC-5334)
- `opentelemetry-exporter-prometheus` (AIPCC-6630)
- Constraints: `grpcio==1.78.0` / `grpcio-reflection==1.78.0` (INFERENG-5970),
  `yarl<1.24` (INFERENG-7300), `numba==0.65.0` (INFERENG-7932),
  `onnx>=1.21.0` (INFERENG-8010 CVE), `transformers<5.15.0` (INFERENG-9834),
  `xgrammar<0.2.5` (INFERENG-10473)

**cpu-ubi9** (x86_64, s390x, ppc64le):
- `vllm[audio,tensorizer,zen]==0.26.0+rhaiv.5; platform_machine == 'x86_64'`
  and `vllm[audio,tensorizer]==0.26.0+rhaiv.5; platform_machine != 'x86_64'`
  — `.cpu` local-version suffix **dropped** this cycle (was `+rhaiv.3.cpu`); the
  `zen` extra (zentorch for AMD EPYC CPUs) remains x86_64-only
- `timm>=1.0.17`; geospatial stack (`geobenchv2`, `terrakit`, `terratorch`,
  `torchgeo`) all gated `; platform_machine == 'x86_64'`
- `setuptools>=80.10.2`
- Constraints: `boto3==1.43.46` (INFERENG-4923),
  `terrakit<0.2.0; platform_machine == 'x86_64'` (AIPCC-19339),
  `aiohttp>=3.13.3`, `urllib3>=2.6.3` (INFERENG-4285 CVE),
  `llguidance>=1.7.0,<1.8.0` (AIPCC-18235/AIPCC-29676),
  `matplotlib<3.11.0` (TEMP: build job fix),
  `transformers<5.15.0` (INFERENG-9834), `uv-build<0.12.6` (INFERENG-10262)

**gaudi-ubi9:**
- `vllm==0.26.0+rhaiv.7` + `vllm-gaudi==0.26.0` — carries NeuralMagic-style
  `+rhaiv.N` tag
- Habana ecosystem: `habana-torch-plugin`, `habana-gpu-migration`, `habana-pyhlml`,
  `habana-torch-dataloader`, `intel-transformer-engine`, `neural-compressor-pt`,
  `torch-tb-profiler` (listed twice; once standalone, once in Habana block)
- `symengine` (undeclared habana-torch-plugin dependency)
- Constraints: `yarl<1.24`, `propcache<0.5` (AIPCC-21773), `transformers<5.15.0`
  (INFERENG-9834), `cryptography>=48.0.1` (GHSA-537c-gmf6-5ccf: CVE fix, new),
  `cloudpickle<3.1` (new)
- `constraints-rules.txt` changed to `torch-2.11.0 [!c]*` — excludes packages
  whose name starts with `c` from builder delegation (was `torch-2.11.0 *` at
  last refresh)

**neuron-ubi9** (most constrained; build revision bump only):
- `vllm[tensorizer]==0.16.0+rhaiv.13` + `vllm-neuron==0.5.3` (rrev 12→13;
  INFERENG-10060: no-op trigger for 3.6-fast1 rebuild)
- `timm>=1.0.17`, `setuptools>=80.10.2`
- Full neuronx stack pinned with embedded git commit hashes (unchanged from last
  refresh): `torch==2.9.1`, `torch-neuronx==2.9.0.2.15.32035+de43f57c`,
  `libneuronxla==2.2.17544.0+fb9962bf`, `neuronx-cc==2.26.6360.0+6f180f47`,
  `neuronx-distributed==0.19.28492+435aae2b`,
  `neuronx-distributed-inference==0.10.18399+ed62453e`,
  `nki==0.5.0+28631259367.ga768afa6`, `torch-xla==2.9.0`, `torchaudio==2.9.1`,
  `torchcodec==0.9.1` (AIPCC-12191), `torchvision==0.24.1`, `triton==3.5.1`
- Torch pinned at 2.9.1 (not 2.11.0 used by most other variants)
- Constraints-rules API is **disabled** (rule commented out) — all pins are
  explicit in `constraints.txt` (INFERENG-5249)
- Additional constraints: `transformers<5` (AIPCC-9443), `urllib3>=2.6.3`
  (INFERENG-4285), `yarl<1.24.2`,
  `prometheus-fastapi-instrumentator>=8.0.1` (INFERENG-8555)

**spyre-ubi9** (multi-arch: ppc64le, s390x, x86_64; unchanged from last refresh):
- `vllm[tensorizer]==0.25.1+rhaiv.1.spyre`
- `sendnn-inference==2.5.2` for both platform-marker groups (unified version)
- `torch-sendnn==1.3.0`, `depyf`, `torchao`, `flash-linear-attention`,
  `pytest-asyncio`, `hf-xet`
- `torch-nnpa==1.5.0; platform_machine == 's390x'` (AIPCC-18989)
- `triton; platform_machine == 'x86_64'`, `intel-openmp; platform_machine == 'x86_64'`,
  `intel_cmplr_lib_ur; platform_machine == 'x86_64'`
- `llguidance` (AIPCC-27980)
- `ibm-aiu-smi==1.3.0`, `spyremetrics==0.5.0` (INFERENG-9813, confirmed landed)
- `timm>=1.0.17`, `opentelemetry-exporter-prometheus; platform_machine != 's390x'`
  (AIPCC-13600)
- Constraints: `torchao==0.11.0` (must match fms-model-optimizer[fp8-infer]),
  `llguidance>=1.7.0,<1.8.0`, `prometheus-fastapi-instrumentator>=8.0.1`,
  `ibm-fms==1.13.0` (AIPCC-27740), `intel-openmp==2024.2.1`,
  `intel-cmplr-lib-ur==2024.2.1`, `tokenizers==0.22.2` (AIPCC-11994)

**tpu-ubi9** (unchanged from last refresh):
- `vllm[tensorizer]==0.26.0+rhaiv.2.tpu`
- `llmcompressor` remains commented out (AIPCC-4341)
- `timm>=1.0.17`, `setuptools>=80.10.2`
- Constraints: `urllib3>=2.6.3` (INFERENG-4285), `transformers<5.15.0` (INFERENG-9834)
- Still uses `torch-2.10.0 *` constraints-rules (one torch minor behind most variants)

**model-opt/cuda13.0-ubi9** (unchanged from last refresh):
- `llmcompressor==0.12.0.1`
- `speculators==0.6.0.1`
- `setuptools>=80.10.2`, `pillow>=12.1.1`
- Explicit constraint pins: `loguru==0.7.3`, `PyYAML==6.0.3`, `numpy==2.4.6`,
  `requests==2.34.2`, `tqdm==4.68.2`, `transformers>=5.9.0,<=5.10.1`,
  `compressed-tensors==0.17.1`, `datasets==5.0.0`, `auto-round==0.13.0`,
  `accelerate==1.13.0`, `nvidia-ml-py==13.610.43` (INFERENG-8847)

**vllm-omni/cuda13.0-ubi9:**
- `vllm-omni==0.26.0+rhaiv.7` (AIPCC-12521; was rhaiv.2)
- `vllm[audio,tensorizer,cgraph-cuda13]==0.26.0+rhaiv.5` (was rhaiv.1)
- FlashInfer, `triton`, `xformers`, `bitsandbytes`, `timm>=1.0.17`, `numba`
- `setuptools>=80.10.2`
- Constraints: `grpcio==1.78.0` / `grpcio-reflection==1.78.0` (INFERENG-5970),
  `aiohttp>=3.13.3`, `urllib3>=2.6.3` (INFERENG-4285), `boto3==1.43.46`
  (INFERENG-4923), `numba==0.65.0` (INFERENG-6026), `numpy<2.5` (AIPCC-28010),
  `transformers<5.15.0` (INFERENG-9834), `quack-kernels==0.6.4` (INFERENG-10588),
  `nvidia-cudnn-frontend<1.29.0` (INFERENG-10716)

### Constraints-Rules Delegation

Eight of the ten collection × variant combinations opt in to `torch-2.11.0`
via `constraints-rules.txt`: cuda13.0-ubi9 (rhaiis), rubin-ubi9, rocm7.14-ubi9,
cpu-ubi9, spyre-ubi9, model-opt/cuda13.0-ubi9, and vllm-omni/cuda13.0-ubi9 all
use `torch-2.11.0 *`. gaudi-ubi9 now uses `torch-2.11.0 [!c]*` — the `[!c]*`
filter excludes packages whose name starts with 'c' from builder delegation
(changed from `torch-2.11.0 *` at the last refresh). Two exceptions:
- `neuron-ubi9`: delegation is disabled (rule commented out); all pins are
  explicit in `constraints.txt` (INFERENG-5249)
- `tpu-ubi9`: opts in to `torch-2.10.0 *` instead (one torch minor behind the rest)

### Pipeline Flow

**Stages:** checks → bootstrap → build → release → lint → notify (unchanged;
release stage is for internal CI release/lint bookkeeping, not Pulp publish)

All (collection × variant × arch) combinations include
`pipeline-api/ci-wheelhouse.yml` from `fondue@v44.0.2`.

**No publish jobs exist** in this repo. Artifact lifecycle:
1. `rhaiis/pipeline` builds wheels → stored in GitLab CI artifact storage
2. `rhai/pipeline` (separately triggered) publishes to Pulp indexes

`seccomp.json` in this repo provides the Linux seccomp BPF syscall filter
applied to wheel build containers.

### Update Automation

Renovate manages vLLM version updates per variant via per-variant custom regex
managers. The pip_requirements manager is globally disabled for `vllm`/`vllm-omni`
across all `collections/{rhaiis,vllm-omni}/*/requirements.txt` via an explicit
packageRule; the custom regex managers are the real vLLM tracking mechanism.

Gaudi's custom regex requires a `+rhai(?:v\.)?\\d+(?:\.gaudi)?` local-version
tag. The current `vllm==0.26.0+rhaiv.7` matches this pattern. On `main`, no
explicit disable rule exists for `gaudi-ubi9` (the `enabled: false` rules in
`renovate.json` for gaudi target only `matchBaseBranches: ["3.5"]`), so Gaudi
vLLM updates are Renovate-trackable on `main`. Neuron and TPU requirements.txt
files are excluded (`enabled: false` on the `main`-targeting `nm-vllm-ent`
packageRule that lists them), so those variants remain manually updated.

Builder version updates are gated by per-branch `allowedVersions` patterns
(`renovate.json` packageRules for `redhat/rhel-ai/wheels/builder`, one entry
per release branch from 3.0 through 3.5; no main-branch version restriction).

## Impact on Strategies

- **vLLM version fragmentation persists, but has narrowed further**: CUDA
  (rhaiv.8), ROCm (rhaiv.7), CPU (rhaiv.5), Gaudi (rhaiv.7), Rubin (rhaiv.5),
  and vllm-omni (rhaiv.5/omni rhaiv.7) are all at the `0.26.0` minor. Spyre
  (`0.25.1+rhaiv.1.spyre`) is one minor behind. TPU remains at `0.26.0+rhaiv.2.tpu`
  (no change). Neuron is the outlier at `0.16.0+rhaiv.13` — 10 minor releases
  behind the 0.26.0 mainstream. Cross-variant features must still be ported to
  each active fork version/build string.
- **Rubin is no longer a placeholder**: `rubin-ubi9` now has actual package
  definitions — vLLM, FlashInfer, llm-d stack (including `nixl-cu13` and
  `deep_gemm` that are absent in `cuda13.0`), xformers, timm, bitsandbytes.
  Geospatial and plugin packages are commented out pending the Vera Rubin CUDA
  stack (INFERENG-9846). Strategies may now treat Rubin as an active build
  variant, though it is still a tech preview with reduced package set.
- **cpu-ubi9 dropped the `.cpu` local-version suffix**: The vLLM version went
  from `0.26.0+rhaiv.3.cpu` to `0.26.0+rhaiv.5` (plain rrev). This changes the
  build string expected by the publish pipeline (`rhai/pipeline`) — strategies
  referencing the `.cpu` suffix are now stale.
- **Gaudi constraints-rules filtering changed**: `gaudi-ubi9` now uses
  `torch-2.11.0 [!c]*` (excludes packages starting with 'c' from builder
  delegation) instead of `torch-2.11.0 *`. The rationale is not documented in
  the file; strategies relying on complete torch-2.11.0 delegation for Gaudi
  should verify which packages are now excluded.
- **Neuron is still the most constrained variant**: Torch remains at 2.9.1
  while most other variants are at 2.11.0. Constraints-rules delegation remains
  disabled. The full neuronx stack carries embedded git commit hashes in version
  strings. The rrev bump (rhaiv.12→rhaiv.13) was a no-op rebuild trigger with
  no package changes. Any RFE touching Neuron requires explicit pin management
  for ~20 packages.
- **Spyre carries IBM-proprietary packages**: the IBM vLLM fork (`.spyre`
  suffix), `sendnn-inference==2.5.2` (IBM inference runtime, arch-conditional),
  `torch-sendnn==1.3.0` (pre-built from private index, x86_64 and ppc64le),
  `torch-nnpa==1.5.0` (pre-built, s390x only), `ibm-fms==1.13.0` (Foundation
  Model Stack), `spyremetrics==0.5.0`, and `ibm-aiu-smi==1.3.0`. `aiu-monitor`
  is NOT a wheel collection package — it lives in the base image. The Spyre RPM
  runtime stack (`SPYRE_VERSION`) is owned by the base image and is not visible
  here.
- **No Pulp publish path**: Wheels built by this pipeline are only accessible
  via GitLab CI artifact storage, not a customer-facing index. RFEs requiring
  customer-accessible inference server wheels must also involve `rhai/pipeline`.
- **Builder version lag/lead is uncertain and needs re-checking**: This repo
  uses `v44.0.2`. The sibling `rhai-pipeline` and `wheels-builder` overlays are
  stale (last refreshed 2026-07-30) and should not be used for version
  comparisons.
- **cpu-ubi9 `zen` extra remains platform-conditional**: `vllm[...,zen,...]`
  is gated `; platform_machine == 'x86_64'`; the non-x86_64 line omits `zen`.
  RFEs that assume `zen` applies unconditionally are incorrect.
- **vllm-omni remains a CUDA-only collection** (AIPCC-12521): productizes
  `vllm-omni==0.26.0+rhaiv.7` alongside the standard vLLM CUDA build on
  cuda13.0-ubi9 (x86_64, aarch64). No publish path to `rhai/pipeline` exists yet.
- **Python, gcc, and TensorFlow versions are not visible in this repo** — those
  are owned by the `builder`/base image. Any RFE citing specific versions for
  these must be validated against the builder or base-image repos (see
  `0019-wheels-builder`), not this overlay.

## Context

This overlay was created to capture the state of the RHAIIS wheel pipeline at the
3.5 release boundary. The repo defines the vLLM ecosystem for each supported
hardware variant. The breadth of vLLM version divergence and the complexity of
the Neuron, Spyre, and Gaudi variants are the primary feasibility constraints for
RFEs proposing changes to the inference server. This overlay allows the
feasibility reviewer to know the current vLLM version per variant, the
IBM/AWS/Intel-specific package dependencies, and the boundary between build
specification (this repo) and distribution (`rhai/pipeline`). Updated
2026-09-16 by running the `update-rhaiis-pipeline-overlay` skill against the
`3.6-fast1` product version.
