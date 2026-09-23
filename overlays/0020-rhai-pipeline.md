---
id: "0020"
title: RHAI Pipeline — Wheel Package Index
status: active
created: 2026-07-15
affects:
  - platform
release:
  - "3.6"
provenance:
  - https://gitlab.com/redhat/rhel-ai/wheels/fondue/-/tree/main/rhai-pipeline
author: Lance Barto
superseded_by: null
---

## Fact

`rhai/pipeline` is the **package index management system** for all Red Hat AI
(RHAI) products. It does not compile wheels — compilation is fully delegated to
the `redhat/rhel-ai/wheels/builder` system, which provides global constraints,
security constraints, and variant-specific build requirements. Constraint files
in this repo must not conflict with the builder's constraints. This repo declares
which packages to build per collection × variant × architecture, and publishes
approved wheels to the customer-facing Pulp index.

This repo is part of the **fondue monorepo** (`redhat/rhel-ai/wheels/fondue`);
the pipeline lives at `rhai-pipeline/` within that monorepo. The authoritative CI
build matrix (collections, variants, arches, Pulp routing) lives in
`ci-job-definitions.yml` at the **monorepo root**, not inside `rhai-pipeline/`.

Some products (notably RHAIIS) also maintain their own dedicated pipeline repos
(`rhaiis/pipeline`), though certain variants and collections are duplicated
across both repos.

- **Product name:** `rhoai`
- **Product version:** `3.6` (from `rhai-pipeline/product-version.yml`)
- **Base OS version:** `0.0-el9.8` = RHEL/UBI **9.8** (from
  `builder/product-version.yml` — this `PRODUCT_VERSION` is the OS/platform
  version the builder targets, **not** a builder release tag).
- **Builder resolution (not a fixed pin):** rhai-pipeline does **not** pin a
  fixed builder release. It builds against the fondue-monorepo builder resolved
  **dynamically at tip-of-`main`**, via the in-tree CI images whose
  `BUILDER_IMAGE_VERSION` is `ci-${BUILDER_PRODUCT_VERSION}-${CI_MERGE_REQUEST_IID}`
  (the CI trigger template `builder-image-version.yml`, resolved only at pipeline
  runtime — not a tag). The tag `v46.0.0` in `releases/builder-release.yaml` is
  fondue's **own** release manifest (the builder declaring its current release;
  notable items include AIPCC-31285 migrating torch-2.11 CPU collection packages
  to torch-2.13, AIPCC-31069 enabling vLLM 0.28 across accelerator collections,
  and wiring the dual-repo promote plan/apply jobs into the parent pipeline) — it
  is **not** a pin held by rhai-pipeline. Contrast `rhaiis/pipeline`, which pins a
  fixed fondue release ref (see [0021](0021-rhaiis-pipeline.md)).
- **Pulp domain (default):** `public-rhai`
- **Public index base URL:** `https://packages.redhat.com/api/pypi/public-rhai/rhoai/`

### Published Variants (3.6)

One row per variant wired for production publication in `publish_config.yml`.

| Variant | Architectures | Torch | vLLM | Public Index Path |
|---|---|---|---|---|
| `cpu-ubi9` | aarch64, ppc64le, s390x, x86_64 | 2.13.0 | 0.28.0+rhaiv.1 (NeuralMagic) | `rhoai/3.6/cpu-ubi9/simple/` |
| `cuda13.0-ubi9` | aarch64, x86_64 | 2.13.0 | 0.28.0+rhaiv.1 (NeuralMagic) | `rhoai/3.6/cuda13.0-ubi9/simple/` |
| `cuda12.9-ubi9` | aarch64, x86_64 | 2.13.0 | n/a (not in `rhaiis`; vLLM dropped CUDA 12 support) | `rhoai/3.6/cuda12.9-ubi9/simple/` |
| `rocm7.14-ubi9` | x86_64 | 2.12.0 | 0.28.0+rhaiv.1 (NeuralMagic) | `rhoai/3.6/rocm7.14-ubi9/simple/` |
| `spyre-ubi9` | ppc64le, s390x, x86_64 | 2.11.0 | 0.27.1+rhaiv.1.spyre (IBM fork) | `rhoai/3.6/spyre-ubi9/simple/` |

Torch versions are pinned per variant via `constraints-rules.txt` (lines of the
form `torch-X.Y.Z *`). cpu, cuda13.0, and cuda12.9 share torch 2.13.0; rocm7.14
pins torch 2.12.0; spyre pins torch 2.11.0. vLLM `+rhaiv` tags indicate the
NeuralMagic enterprise fork; `.spyre` indicates the IBM fork
(`vllm[tensorizer]==0.27.1+rhaiv.1.spyre`, RHAI-688). `cuda12.9-ubi9` is not a
member of the `rhaiis` collection (vLLM dropped CUDA 12 support upstream). ROCm
7.1 has been fully retired; only ROCm 7.14 remains.

The `rhaiis` collection additionally builds (but `publish_config.yml` does not
publish to the public per-variant index) the accelerator variants `gaudi-ubi9`
(vLLM 0.26.0 + vllm-gaudi 0.26.0), `neuron-ubi9` (vLLM 0.16.0+rhaiv.12 +
vllm-neuron 0.5.3), and `tpu-ubi9` (vLLM 0.27.1 upstream) — these route to the
**private** `rhai` Pulp domain (see below). A `cpu-torch-day0-ubi9` variant
exists only in the `torch-day0` collection and publishes to a version-pinned
public base path.

Distribution repos follow the dual-repo pattern (ADR 0199). As of AIPCC-32489,
**production** wheel/sdist repositories and base paths are **unsuffixed**;
**test** distributions retain a `-test` suffix (`rhoai/3.6/<variant>-test/simple/`).
The obsolete `-prod` suffix is rejected by the promotion tooling. Legacy
single-repo releases already used unsuffixed names; the pre-AIPCC-32489 `-prod` /
`-sdists-prod` repos are still recognized by the deletion tooling for releases
created before this change.

### Collections and Variant Coverage

The CI matrix is generated by `make regen` (monorepo-root `bin/regen-ci.py`) from
`ci-job-definitions.yml` (monorepo root) into `.generated/rhai-<collection>.yml`
files. `ci-job-definitions.yml` is the authoritative source for collection names,
variant lists, base arch sets, `omit_jobs`, Pulp routing, and optional keys
(`enable_test_jobs`, `enable_multi_version_bootstrap`, `max_release_age`).

**Arch coverage is computed, not copied.** For each (collection, variant), the
effective arch set = `VARIANTS.<variant>.arches` minus every `omit_jobs` entry
matching `[<collection>, <variant>, <arch>]` **where the first element is a real
collection key**. Base arch sets: cpu-ubi9 and cpu-torch-day0-ubi9 =
[aarch64, ppc64le, s390x, x86_64]; cuda12.9-ubi9 and cuda13.0-ubi9 =
[aarch64, x86_64]; spyre-ubi9 = [ppc64le, s390x, x86_64]; gaudi/neuron/rocm7.14/
tpu = [x86_64].

| Collection | Purpose | Variants (effective arches) |
|---|---|---|
| `onboarding` | Intake staging area; test builds before production graduation (`enable_test_jobs`) | cpu (all 4 arches), cuda12.9, cuda13.0, rocm7.14, spyre |
| `rhai` | Primary production packages, organized by owning team | cpu (all 4 arches), cuda12.9, cuda13.0, rocm7.14, spyre |
| `rhai-innovation` | AI Innovation products: Docling, SDG Hub, and related | cpu (all 4 arches), cuda12.9, cuda13.0, rocm7.14 |
| `rhaiis` | Red Hat AI Inference Server (vLLM). `gaudi-ubi9`, `neuron-ubi9`, `tpu-ubi9` route to the **private** `rhai` Pulp domain via `variant_overrides` (AIPCC-28553) — vendor wheels that cannot be publicly redistributed. All other `rhaiis` variants use the default public `public-rhai` domain. | cpu (all 4 arches), cuda13.0, gaudi-ubi9, neuron-ubi9, rocm7.14, spyre, tpu-ubi9 |
| `model-opt` | Model Optimization (CUDA 13 only) | cuda13.0 (aarch64, x86_64) |
| `torch-deps` | PyTorch team exact-pin dependencies matching upstream PyTorch CI | cpu (all 4 arches), cuda12.9, cuda13.0, rocm7.14 |
| `torch-day0` | Rapid PyTorch version availability; `enable_multi_version_bootstrap`, `max_release_age` 60 days. Both variants carry a `PULP_BASE_PATH` override (public domain, version-pinned path). torch pinned to 2.14.0 (AIPCC-30387). | cpu-torch-day0-ubi9 (all 4 arches), cuda13.0-ubi9 (aarch64, x86_64) |
| `ogx` | OGX / Llama Stack inference framework; `enable_test_jobs`, `enable_multi_version_bootstrap`, `max_release_age` 30 days | cpu (aarch64, ppc64le, x86_64) — s390x removed by an effective `omit_jobs` entry |
| `vllm-deps/torch-2.11` | vLLM build dependencies; pinned to torch 2.11; published to **private** Pulp domain `rhai` (`PRODUCT_NAME: vllm-deps`, `PRODUCT_VERSION: torch2.11`) — not the public `public-rhai` index. Provides the vLLM team a private index for test builds during the transition to them supplying pre-built wheels (AIPCC-19939 / AIPCC-12506, ADR0211); `enable_multi_version_bootstrap`, `max_release_age` 10 days | cuda12.9-ubi9, cuda13.0-ubi9 |

Publishing to Pulp is controlled per **variant** in `publish_config.yml`, not per
collection. For default-routed collections/variants (no `PULP_DOMAIN` and no
`PULP_BASE_PATH` override), all wheels for a published variant end up in the same
public `public-rhai` Pulp index at the default per-product path. For 3.6, the
published variants are: cpu-ubi9, cuda12.9-ubi9, cuda13.0-ubi9, rocm7.14-ubi9,
and spyre-ubi9.

**`OMIT_JOBS` (matrix exclusions in `ci-job-definitions.yml`):** an entry is only
effective if its **first element is a collection key**. `regen-ci.py` skips a job
only when the tuple `(collection, variant, arch)` matches; an entry naming a
*package* instead of a collection matches nothing and is **dead**.

| `omit_jobs` entry | Status | Notes |
|---|---|---|
| `[ogx, cpu-ubi9, s390x]` | **Effective** | `ogx` is a collection; no s390x index for this collection. Narrows `ogx`/cpu to (aarch64, ppc64le, x86_64). |
| `[docling, cpu-ubi9, s390x]` | **Dead** | `docling` is a *package*, not a collection key — never matched. The real s390x skip is a PEP 508 marker `docling ; platform_machine != 's390x'` in `collections/rhai-innovation/cpu-ubi9/requirements.txt` (AIPCC-8297). |
| `[sdg-hub, cpu-ubi9, s390x]` | **Dead** | `sdg-hub` is a *package*, not a collection key — never matched. The real s390x skip is the marker `sdg-hub[examples] ; platform_machine != 's390x'` in the same file (AIPCC-10512). |

Because the docling/sdg-hub `omit_jobs` entries are dead, `rhai-innovation`/cpu
still builds on **all 4 cpu arches** at the collection level; per-package s390x
exclusion is enforced by PEP 508 markers (a separate mechanism from `omit_jobs`).
The same marker mechanism guards docling-family packages in `rhai`/cpu and
`rhai`/spyre (e.g. `docling-slim ... ; platform_machine != "s390x"`,
AIPCC-28691/AIPCC-30770), so those collections also carry no matching
`omit_jobs` entry.

**Variant linter** (`rhai-pipeline/bin/variant-linter.py`) reads
`ci-job-definitions.yml` on every MR and blocks any variant not matching an
approved pattern. The full `ALLOWED_VARIANT_PATTERNS` list (case-insensitive
substring match) is: `["cuda", "rocm", "cpu", "spyre", "tpu", "neuron",
"gaudi"]`. New accelerator types require approval before the pattern is added.

**Pulp routing — two independent dimensions.** Both driven by
`overrides`/`variant_overrides`, with precedence
`overrides._default → overrides.<collection> → variant_overrides.<collection>.<variant>`:

1. **`PULP_DOMAIN`** — which Pulp instance (public `public-rhai` vs private
   `rhai`). Default is `public-rhai` (`overrides._default`, which also sets
   `PUBLISH_WHEEL_RELEASES: "true"` and `GITLAB_PRIVATE_TOKEN`). Overrides:
   - Collection-level: `vllm-deps/torch-2.11` → private `rhai`
     (`PRODUCT_NAME: vllm-deps`, `PRODUCT_VERSION: torch2.11`).
   - Variant-level (`variant_overrides.rhaiis`): `gaudi-ubi9`, `neuron-ubi9`,
     `tpu-ubi9` → private `rhai` (`PRODUCT_NAME: rhaiis`; `PRODUCT_VERSION`
     intentionally omitted so it is inherited from `product-version.yml`;
     AIPCC-28553).
2. **`PULP_BASE_PATH`** — the path/repo name **within** a domain. Overrides
   (`variant_overrides.torch-day0`):
   - `cpu-torch-day0-ubi9` → `PULP_BASE_PATH: torch-2.14.0-cpu-ubi9`
   - `cuda13.0-ubi9` → `PULP_BASE_PATH: torch-2.14.0-cuda13.0-ubi9`
   These keep the **public** `public-rhai` domain but publish to a version-pinned
   path, **not** the default `rhoai/<PRODUCT_VERSION>/<variant>` path. Confirmed
   in `bin/dual-repo-promote.sh` and `src/rhai_pipeline/pulp_ops.py`
   (`repo_name_from_base_path`): when `PULP_BASE_PATH` is set, the repo name is
   derived from that path (`/` → `-`) and `PRODUCT_VERSION` is ignored. The
   production repo is the unsuffixed base path; the test repo appends `-test`.

All collections/variants without a `PULP_DOMAIN` and without a `PULP_BASE_PATH`
override use the default public domain at the default per-product path. The
available documentation does not establish different deletion or promotion
behavior for the private-routed or base-path-overridden variants beyond what the
source establishes.

### The `rhai` Collection

The production collection is organized by team ownership. The
`cpu-ubi9/requirements/` directory holds 39 files: `rhai.txt` (shared base),
`onboarded.txt` (packages graduated from onboarding), and per-team files
(`team-*.txt`) for each owning team. Notable team files:

- `team-notebooks-images.txt` / `team-notebooks-extensions.txt` — data science
  stack (JupyterLab, notebooks tooling)
- `team-mlserver.txt` — mlserver and serving plugins
- `team-vllm-runtime.txt` — vLLM runtime auxiliary packages
- `team-fine-tuning.txt` — fine-tuning / kernel packages
- `team-pytorch.txt` — PyTorch team packages
- `team-infereng-midstream.txt`, `team-speculators.txt`, `team-llmd.txt`,
  `team-llm-d.txt`, `team-serving-orchestration.txt` — inference / llm-d stack
- `team-sdg.txt`, `team-autorag.txt`, `team-rag-vector-db.txt`,
  `team-data-processing.txt`, `team-data-connect-hub.txt` — AI / data teams
  (docling-family markers guard s390x)
- `team-llama-stack-core.txt`, `team-ogx-core.txt` — Llama Stack and OGX
- `team-guardrails-detectors.txt`, `team-ai-safety.txt` — safety / guardrails
- `team-model-eval.txt`, `team-lm-evaluation-harness.txt` — evaluation
- `team-kserve.txt`, `team-model-serving.txt`, `team-model-runtimes.txt`,
  `team-mlserver.txt` — serving
- `team-training-kubeflow.txt`, `team-kubeflow-devx.txt` — training / Kubeflow
- `team-ai-hub.txt`, `team-ai-navigator.txt`, `team-ai-core-platform.txt`,
  `team-aipcc-ecosystems.txt`, `team-accelerator-enablement.txt`,
  `team-development-platform.txt`, `team-devops.txt`, `team-perfscale.txt`,
  `team-service-mesh.txt`, `team-notebooks-images.txt`,
  `team-wheel-package-index.txt` — platform / infra teams

Spyre-ubi9 carries a reduced team set (omitting teams whose packages are
incompatible with ppc64le/s390x; e.g. it has no `onboarded.txt`,
`team-ai-core-platform`, `team-ai-safety` copy where those packages don't build).

### Onboarding Pipeline

New dependency requests land in `collections/onboarding/{variant}/` first
(shared `requirements/0000-core-packages.txt` plus per-package files; the
collection root also carries `core-packages.txt`). Test builds are enabled
(`enable_test_jobs`), giving teams CI feedback before production. A weekly
scheduled GitLab job (`move-onboarded-packages`, triggered by
`SCHEDULE_TYPE=move-onboarded`) runs `./bin/move-onboarded-packages.sh`, which
moves graduated packages to `collections/rhai/*/requirements/onboarded.txt` and
creates a bot MR (branch `auto/move-onboarded-packages`). All new top-level
dependencies must go through onboarding before appearing in the `rhai` collection.

### Pipeline Flow

**Stages:** checks → lint → bootstrap → build → release → package-deletion →
publish → pre-notify → notify

**Trigger types:**
- Push to branch: standard CI (cancels concurrent interruptible jobs, except on
  main)
- Nightly schedule (`SCHEDULE_TYPE=nightly`): main branch builds only
- Weekly `move-onboarded` schedule: runs `move-onboarded-packages.sh`
- CVE scan schedule: runs `bin/cve_scanner.py`
- AutoQA trigger: fires when `publish_config.yml` changes

**Key checks-stage gates:**
- `variant-linter` — validates variant names in `ci-job-definitions.yml` on every MR
- `verify-publish-config` (`bin/verify_publish_config.py`) — on
  `publish_config.yml` changes, queries the Pulp API to compare old vs new
  versions; saves `publish-delta/index-delta.md` as an artifact
- `validate-package-deletion-manifests` — validates deletion YAML on MR; on
  merge, executes actual deletion

### Pulp Publishing Mechanics

**Dual-repo architecture (ADR 0199).** Each `{product}-{version}-{variant}`
maps to four Pulp repositories:

```
{product}-{version}-{variant}-test           wheels test
{product}-{version}-{variant}                wheels prod (unsuffixed)
{product}-{version}-{variant}-sdists-test    sdists test
{product}-{version}-{variant}-sdists         sdists prod
```

As of AIPCC-32489, production repos and base paths are **unsuffixed**; only test
repos carry a suffix. Both test and prod distributions are floating (always serve
latest). Confirmed in `src/rhai_pipeline/pulp_ops.py`
(`resolve_wheels_repo_and_path` / `resolve_sdists_repo_and_path`): production =
unsuffixed, test = `-test`; sdists prod = `-sdists`, sdists test = `-sdists-test`.

1. **Upload** (`uv run pulp upload`, opt-in `--dual-repo`): Discovers
   wheel/sdist artifacts, filters packages listed in deletion manifests, uploads
   via mTLS to the `-test` repo. Labels each artifact: accelerator, rhel_version,
   variant, product_version, ci_commit_sha.
2. **Promote** (`uv run pulp promote`, driven in CI by `bin/dual-repo-promote.sh`
   in `plan`/`apply` steps): computes `content-diff` (test − prod), runs AutoQA
   on candidates, qualifies via dependency resolution, and promotes passing
   wheels (with matching sdists) from the `-test` repo to the unsuffixed
   production repo. Promotion is additive; the obsolete `-prod` target suffix is
   rejected. Legacy single-repo releases still use `publish_config.yml` to pin
   `WHEEL_REPO_VERSION`/`SDIST_REPO_VERSION` production distribution versions.

**Repository naming (`src/rhai_pipeline/pulp_ops.py`):**
- Wheels prod repo/base path: `{product}-{version}-{variant}` /
  `{product}/{version}/{variant}` → index `rhoai/{version}/{variant}/simple/`
- Wheels test: `-test` suffix → `rhoai/{version}/{variant}-test/simple/`
- SDists prod: `{product}-{version}-{variant}-sdists`; test:
  `-sdists-test`

**Base-path override path** (when `PULP_BASE_PATH` is set, e.g. `torch-day0`):
`repo_name_from_base_path` flattens `/` → `-`, ignoring `PRODUCT_VERSION`. The
production repo is the unsuffixed base path and the test repo appends `-test`.
These two `torch-day0` variants therefore land at the concrete repos
`public-rhai/torch-2.14.0-cpu-ubi9/simple/` and
`public-rhai/torch-2.14.0-cuda13.0-ubi9/simple/` (prod), each with a `-test`
sibling (`…-cpu-ubi9-test`, `…-cuda13.0-ubi9-test`). Note the published repo
**drops** the `-torch-day0` collection segment — it is **not**
`torch-2.14.0-cpu-torch-day0-ubi9`. These do **not** use the default
per-product path.

**Authentication:** All Pulp operations use mTLS (`PULP_CERT_BASE64` /
`PULP_KEY_BASE64` CI/CD variables).

**Supporting tools:** `pulp autoqa` (trigger AutoQA + wait), `pulp catalog`
(export repo content as JSON), `pulp content-diff` (packages in test but not yet
in prod).

### Package Deletion System

YAML manifests in `package-deletions/{release}.yaml` declare packages to remove
(schema in `package-deletions/manifest-schema.json`). Deletion is enforced at
upload time (packages matching deletion patterns are skipped before upload,
preventing re-introduction even if builds re-run). MR validation runs a dry-run;
on merge the job executes actual Pulp deletion. `pulp delete` processes every
manifest in one CI run and probes Pulp per `(version, variant)`: if a `-test`
repo exists it is a dual-repo release (removes from `-test` and the unsuffixed
production repo, plus `-sdists-test`/`-sdists` when `delete_sdist: true`),
otherwise legacy layout (removes from the unsuffixed repo names,
`resolve_delete_repo_names`). Pre-AIPCC-32489 `-prod`/`-sdists-prod` repos are
also recognized for older releases. The system is idempotent.

### Version Branching

`uv run pulp copy` (`src/rhai_pipeline/pulp_copy.py`) copies existing Pulp
repositories to new repository names when creating a new product version (e.g.,
`--source-version 3.5 --dest-version 3.6 --dual-repo`). With `--dual-repo` it
creates all four repos per variant, seeded from the source version's prod
content. Variants dropped in the destination are skipped; the variant set for a
version comes from `supported_versions.yml`. This is a fast promotion path that
does not require re-running builds.

## Impact on Strategies

- This is the authoritative publish gate for all RHAI Python packages. No package
  reaches the customer-facing Pulp index without going through this pipeline.
- Adding a new collection or variant requires: (1) adding an entry to the
  `collections:` section in `ci-job-definitions.yml` at the monorepo root,
  (2) creating `rhai-pipeline/collections/{name}/{variant}/` directory structure
  with `requirements.txt`/`constraints.txt`, (3) running `make regen` to generate
  `.generated/rhai-{name}.yml`, (4) merging all three changesets together. The
  `variant-linter` CI job blocks any unrecognized accelerator type. To exclude an
  arch, add an `omit_jobs` entry whose first element is the **collection** key (a
  package name there is a no-op) — or, for a single package, use a PEP 508 marker
  in `requirements.txt`.
- All release branch jobs use `ENABLE_REPEATABLE_BUILD_MODE`. On a release branch,
  adding or updating a package requires explicit version pinning — the locked
  dependency graph will not re-resolve.
- Two CUDA versions (12.9 and 13.0) are maintained simultaneously. Adding a third
  is structurally straightforward but doubles builder resource consumption for
  that variant. `cuda12.9-ubi9` does not receive vLLM packages (it is not in the
  `rhaiis` collection; vLLM dropped CUDA 12 support upstream).
- ROCm 7.1 has been fully retired. Only ROCm 7.14 is built and published. ROCm
  7.14 pins torch 2.12.0 (one minor below the CUDA variants' 2.13.0 — an
  intentional per-variant split, not an oversight). RFEs referencing ROCm should
  specify 7.14 and note the different torch pin.
- Spyre carries IBM-proprietary packages:
  `vllm[tensorizer]==0.27.1+rhaiv.1.spyre` (IBM fork, `.spyre` suffix
  distinguishes it from the NeuralMagic fork; RHAI-688),
  `sendnn-inference==2.6.1` (IBM inference runtime),
  `torch-sendnn==1.3.1` (pre-built IBM wheel; no arch marker, covering x86_64/P),
  `torch-nnpa==1.5.0` (pre-built IBM wheel, s390x only),
  `ibm-fms==1.13.1` (Foundation Model Stack, built from source; both pinned and
  unpinned `ibm-fms` listed per AIPCC-27741), `spyremetrics==0.5.0` (ppc64le
  only, AIPCC-28704), and `ibm-aiu-smi==1.3.0` (ppc64le only, AIPCC-28706).
  Spyre's torch 2.11.0 pin (in `constraints-rules.txt`) is **unique to spyre** —
  it is not shared with any other variant. cpu and cuda variants pin 2.13.0;
  rocm pins 2.12.0; each variant's pin is independent.
- The public index URL structure (`rhoai/{version}/{variant}/simple/`, unsuffixed
  production as of AIPCC-32489; `-test` for test) is treated as a stable contract
  for air-gapped mirroring (`wget`-based). Breaking this structure requires
  coordinating all downstream consumers. Note that `torch-day0` variants deviate
  from this structure (version-pinned base path).
- Package deletion is idempotent and enforced at upload time. RFEs proposing
  package removal must go through the deletion manifest process; simply removing
  a package from `requirements.txt` does not remove it from Pulp.
- New packages must go through the `onboarding` collection first. The weekly
  graduation automation handles promotion to `rhai`, but it runs weekly, not on
  demand.

### ROCm Work Breakdown Patterns

When a strategy involves a ROCm variant update in the pipeline (e.g., new ROCm
version or ROCm package changes), the pipeline-side work decomposes into these
epics:

- **Update ROCm variant constraints** — torch pin, vllm pin, and ROCm-specific
  package versions in `constraints.txt` and `constraints-rules.txt` for the
  `rocm{version}-ubi9` variant.
- **Add or update ROCm-specific packages in collections** — `amd-quark`,
  `amd-aiter`, `tensorflow-rocm`, `flash-attn`, and any new AMD ecosystem packages
  in `collections/rhaiis/rocm{version}-ubi9/requirements.txt`.
- **Validate build and publish for the ROCm variant** — CI pipeline green, wheels
  uploaded to Pulp, promoted to the production repo, and the customer-facing index
  updated.

Strategies referencing ROCm pipeline updates should structure their Technical
Approach around these epics rather than describing the work as prose.

## Context

This overlay was created to capture the state of the RHAI wheel pipeline at the
3.6-EA1 release boundary and updated to reflect the current 3.6 (GA) state. The
pipeline repo is purely declarative — it contains no application code. Its
architecture (collection structure, variant matrix, Pulp publishing contract)
shapes what RHAI customers receive and constrains what RFEs can realistically
propose. This overlay allows the feasibility reviewer to evaluate whether a
proposed change is compatible with the existing index structure, build
infrastructure, and publishing workflow.

Updated 2026-09-23 by running the `update-rhai-pipeline-overlay` skill against a
fresh fondue monorepo checkout on `main` (commit `da4a35d`). All values are read
literally from current source. Notable changes vs. the prior (2026-09-22)
revision:

- **Builder resolution corrected** — rhai-pipeline builds against the fondue
  builder resolved **dynamically at tip-of-`main`**
  (`ci-${BUILDER_PRODUCT_VERSION}-${CI_MERGE_REQUEST_IID}`), **not** a fixed pin.
  The `v46.0.0` tag in `releases/builder-release.yaml` is fondue's own release
  manifest, not a pin held by rhai-pipeline. Base OS version `0.0-el9.8` (RHEL 9.8,
  from `builder/product-version.yml`) is reported separately.
- **`rhai` cpu-ubi9 team files** — count grew from 37 to **39**; new team files
  `team-data-connect-hub.txt` and `team-training-kubeflow.txt` added.
- **Confirmed unchanged from source:** product version 3.6; the five published
  variants (cpu, cuda13.0, cuda12.9, rocm7.14, spyre) in `publish_config.yml`;
  the collection set (model-opt, ogx, onboarding, rhai, rhai-innovation, rhaiis,
  torch-day0, torch-deps, vllm-deps/torch-2.11); torch pins (cpu/cuda 2.13.0,
  rocm 2.12.0, spyre 2.11.0, torch-day0 2.14.0); vLLM versions
  (cpu/cuda13.0/rocm7.14 = 0.28.0+rhaiv.1, gaudi 0.26.0, neuron 0.16.0+rhaiv.12,
  tpu 0.27.1, spyre 0.27.1+rhaiv.1.spyre); the Spyre IBM stack
  (`sendnn-inference 2.6.1`, `torch-sendnn 1.3.1`, `torch-nnpa 1.5.0`,
  `ibm-fms 1.13.1`, `spyremetrics 0.5.0`, `ibm-aiu-smi 1.3.0`); Pulp routing
  (`variant_overrides`: torch-day0 `PULP_BASE_PATH` → `torch-2.14.0-*`; rhaiis
  gaudi/neuron/tpu `PULP_DOMAIN: rhai`; vllm-deps/torch-2.11 collection-level
  `PULP_DOMAIN: rhai`); the effective `omit_jobs` entry `[ogx, cpu-ubi9, s390x]`
  and the two **dead** entries (`docling`, `sdg-hub`, s390x skip enforced by
  PEP 508 markers); unsuffixed production repos with `-test` distributions
  (AIPCC-32489); and version branching via `pulp copy` (`--dest-version 3.6`).

- **aiu-monitor / global-constraints:** verified `builder/collections/global-constraints.txt`
  still exists; it contains **no** `aiu-monitor<0.0.0` / `ibm-aiu-monitor<0.0.0`
  entries (removed in AIPCC-28729). aiu-monitor is not blocked via
  global-constraints.txt and is not carried as a wheel collection package;
  `torch-sendnn>=1.2.0` (AIPCC-14853) is the only Spyre-adjacent guard present.
</content>
