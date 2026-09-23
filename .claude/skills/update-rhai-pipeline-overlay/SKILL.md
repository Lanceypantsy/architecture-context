---
name: update-rhai-pipeline-overlay
description: Use when the rhai/pipeline repository has changed and the overlay file overlays/0020-rhai-pipeline.md needs to be refreshed with current product version, published variants, collection matrix, or Pulp publishing details.
user-invocable: true
allowed-tools: Read, Write, Bash(bash ${CLAUDE_SKILL_DIR}/scripts/fetch-repo.sh), Glob, Grep
---

# Update RHAI Pipeline Overlay

Refresh `overlays/0020-rhai-pipeline.md` with current information from the
`rhai/pipeline` repository.

## Overview

The overlay documents the RHAI wheel package index management system: which
variants are published, what collections exist, and how wheels flow from CI
builds to the customer-facing Pulp index. When the repo changes -- new variant
in `publish_config.yml`, new collection, updated product version, or changes to
the Pulp publishing workflow -- run this skill to update the overlay.

## Instructions

### Step 1: Locate or Clone the Repository

Run the fetch script from the root of the architecture-context repository:

```bash
bash ${CLAUDE_SKILL_DIR}/scripts/fetch-repo.sh
```

The script checks for a local fondue checkout at `../fondue/rhai-pipeline`. If
found, it prints that path and exits. Otherwise it clones or updates
`./tmp/fondue` from `https://gitlab.com/redhat/rhel-ai/wheels/fondue.git` and
prints `./tmp/fondue/rhai-pipeline`.

Use the printed path as `{REPO}` in all subsequent steps. The **fondue
monorepo root** is the parent of `{REPO}` (i.e. `dirname {REPO}` — `../fondue`
for a local checkout, `./tmp/fondue` for a clone). Refer to it as
`{FONDUE_ROOT}`. Some authoritative files live at `{FONDUE_ROOT}`, not inside
`{REPO}` — those paths are called out explicitly below.

### Step 2: Read the Key Files

Read the following files to extract current state. All paths are relative to
`{REPO}`:

**Version information:**
- `product-version.yml` -> `PRODUCT_VERSION` (the rhai-pipeline wheel-index
  product version, e.g. `3.6`)
- `supported_versions.yml` -> full version history and per-version variant lists
- `{FONDUE_ROOT}/builder/product-version.yml` -> `PRODUCT_VERSION` (e.g. `0.0-el9.8`)
  is the **OS/platform version** the builder targets (RHEL 9.8). Report this in the
  overlay as the base OS version, not as the builder release.
- **Builder resolution — rhai-pipeline does NOT pin a fixed builder release.**
  It builds against the fondue-monorepo builder resolved **dynamically at
  tip-of-`main`**, via the in-tree CI images whose `BUILDER_IMAGE_VERSION` is the
  string `ci-${BUILDER_PRODUCT_VERSION}-${CI_MERGE_REQUEST_IID}` (resolved only at
  pipeline runtime — not a tag). Report this dynamic resolution as the builder
  source. The CI trigger template `builder-image-version.yml` lives at
  `{FONDUE_ROOT}` (not under `{REPO}`); read it to confirm the resolution string,
  but do **not** report it as a concrete version tag.
- The tag in `{FONDUE_ROOT}/releases/builder-release.yaml` (e.g. `v46.0.0`; note:
  `builder/release.yaml` does **not** exist) is fondue's **own** release manifest —
  the builder declaring its current release. **Do not** report it as rhai-pipeline's
  pinned builder release; rhai-pipeline does not pin to it. Cite it only as context
  for what the current fondue release contains. Contrast `rhaiis/pipeline`, which
  *does* pin a fixed fondue release ref (see overlay `0021`).

**Published variant configuration:**
- `publish_config.yml` -> per-variant `VARIANT`, `WHEEL_REPO_VERSION`, and
  `SDIST_REPO_VERSION` entries (these are the variants currently wired for
  production publication)

**Build matrix and Pulp routing:**
- `{FONDUE_ROOT}/ci-job-definitions.yml` (at the **fondue monorepo root**, not
  inside `rhai-pipeline/`) -> the authoritative source for the CI matrix and
  per-collection / per-variant Pulp routing. Read this file, not any script
  inside `rhai-pipeline/bin/`. Key sections to extract:
  - `collections:` block -> `COLLECTIONS` (collection -> variants), `VARIANTS`
    (variant -> arches), `omit_jobs` list
  - **Effective arch set per (collection, variant) is computed, not copied.**
    For every collection/variant pair, start from `VARIANTS.<variant>.arches`
    (the base arch set) and subtract every `omit_jobs` entry matching
    `[<collection>, <variant>, <arch>]`. The remainder is the effective arch
    set. Recompute this from the current file for every pair — never carry the
    arch annotation over from the existing overlay. A narrowed set (fewer than
    the base arches) is valid **only** if a current `omit_jobs` entry removes
    those arches; if no matching `omit_jobs` entry exists, the variant builds on
    all of its base arches. When the effective set equals the base set, say so
    (e.g. "cpu (all 4 arches)") rather than restating a stale subset.
  - **An `omit_jobs` entry is only effective if its first element is a
    collection key** (a key under `collections:`). `regen-ci.py` skips a job
    only when the tuple `(collection, variant, arch)` matches, and its
    validator never checks the first element — so an entry naming a *package* or
    *product* (e.g. `docling`, `sdg-hub`) instead of a collection matches
    nothing and is **dead**. Before treating any `omit_jobs` entry as a real
    exclusion, confirm its first element appears as a collection key. Flag dead
    entries explicitly; do not present them as effective exclusions. Note that
    per-package arch exclusions are instead enforced by PEP 508 environment
    markers in the collection's `requirements*.txt` (e.g.
    `docling ; platform_machine != 's390x'`), which is a separate mechanism from
    `omit_jobs` — verify the marker before claiming a package is skipped on an
    arch.
  - `overrides._default` -> default `PULP_DOMAIN` (e.g. `public-rhai`)
  - `overrides.<collection>` -> collection-level routing exceptions (e.g.
    `vllm-deps/torch-2.11` -> `PULP_DOMAIN: rhai`)
  - `variant_overrides.<collection>.<variant>` -> per-variant routing overrides
    applied after collection-level overrides. Precedence:
    `_default -> <collection> -> variant_overrides.<collection>.<variant>`.
    **Enumerate every entry** under `variant_overrides` — iterate all
    collections and all variants beneath them; do not assume the set is limited
    to the `rhaiis` collection or to any previously documented list. For each
    entry, record the collection, the variant, and the exact override keys and
    values (e.g. `PULP_DOMAIN`, `PULP_BASE_PATH`, `PRODUCT_NAME`). Capture the
    rationale wherever a comment supplies one (e.g. AIPCC-28553 for the private
    `PULP_DOMAIN: rhai` overrides).

**Collection contents (spot-check):**
- For each collection subdirectory under `collections/`, read the directory
  listing to confirm which variant subdirs exist.
- For the `rhai` collection on `cpu-ubi9`, read
  `collections/rhai/cpu-ubi9/requirements/` directory listing to see the
  team-owned requirement files.
- For `onboarding`, `rhai-innovation`, `rhaiis`, `model-opt`, `ogx`,
  `aiu-monitor-deps` -- read the `requirements.txt` or directory listing for at
  least one representative variant.

**Key top-level package versions per variant:**

To identify release-defining packages, use these heuristics rather than a
hardcoded list:

- Read `collections/{collection}/{variant}/constraints-rules.txt` for each
  published variant in the `rhai` collection. Lines of the form `torch-X.Y.Z *`
  pin the torch version. Packages that appear in constraints-rules are
  release-defining because they anchor the dependency graph for their variant.
- Read `collections/rhaiis/{VARIANT}/requirements.txt` for each published
  variant. Packages with local version tags (e.g. `+rhaiv.N`, `+rhai19`,
  `+rhaiv.1.spyre`) are maintained as internal forks and are release-defining.
  Note the fork origin: `+rhai` tags indicate the NeuralMagic enterprise fork,
  plain upstream versions indicate `vllm-project/vllm`, and `.spyre` suffixes
  indicate IBM forks.
- Packages that have different versions across variants (visible by comparing
  requirements.txt files) are also release-defining because they drive
  per-variant scope decisions.
- Include these packages and their versions in the Published Variants table so
  RFE creators can identify major upgrades at a glance.

  **For `spyre-ubi9` specifically**, when reading
  `collections/rhaiis/spyre-ubi9/requirements.txt`, capture:
  - `vllm` exact version string (note the local version tag and fork origin)
  - `sendnn-inference` pinned version; check whether it differs by arch
  - `torch-sendnn` pinned version -- pre-built IBM wheel, not compiled by builder
  - `torch-nnpa` pinned version -- pre-built IBM wheel, s390x only
  - `ibm-fms` pinned version (also check `constraints.txt`)
  - `spyremetrics` and `ibm-aiu-smi` -- new wheels; capture version and arch
    restrictions if present
  - **Do not** list `aiu-monitor` / `ibm-aiu-monitor` as wheel collection packages.
    As of AIPCC-28729 the `aiu-monitor<0.0.0` / `ibm-aiu-monitor<0.0.0` guard lines
    were removed from `builder/collections/global-constraints.txt`; read the current
    file to confirm the actual set of global constraints. Do not state that aiu-monitor
    is blocked via global-constraints.txt — verify before making that claim.

### Step 3: Read the Current Overlay

Read `overlays/0020-rhai-pipeline.md` to understand the existing structure.
Identify the human-authored sections (Impact on Strategies, Context) that must
be preserved and updated, not replaced wholesale.

### Step 4: Update the Overlay

Rewrite `overlays/0020-rhai-pipeline.md` using the following approach:

**Preserve the YAML front matter** (`id`, `title`, `status`, `created`,
`affects`, `provenance`, `author`, `superseded_by`). Update `release` only if
the product version indicates a new RHOAI release.

**Fact section** -- Replace with fresh content derived from the files above.
This section must cover:

- **Header** -- Purpose of the repo (index management, not compilation);
  product name, product version, builder version, Pulp domain, public base URL
- **Published Variants table** -- One row per variant in `publish_config.yml`;
  columns: variant, architectures, key release-defining package versions (found
  using the heuristics from Step 2), public index path.
- **Collections and Variant Coverage table** -- One row per collection with
  purpose description and which variants it covers; derived from
  `{FONDUE_ROOT}/ci-job-definitions.yml` (the authoritative source per Step 2).
  Where a variant's arch coverage is annotated (e.g. `cpu (all 4 arches)` or
  `cpu (aarch64, x86_64)`), use the **effective arch set computed in Step 2**
  (`VARIANTS.<variant>.arches` minus matching `omit_jobs` entries) — recompute
  it, never copy the annotation from the existing overlay. Any subset shown must
  be justified by a current `omit_jobs` entry for that exact
  `(collection, variant, arch)`; if none exists, show the full base arch set.
  Note all routing exceptions from the same file:
  - **Collection-level**: `vllm-deps/torch-2.11` -> private domain `rhai`
    (`private.console.redhat.com`); used by the vLLM team for test builds
    during the transition to supplying pre-built wheels. Do not assert
    different deletion or promotion behavior -- the documentation does not
    establish that.
  - **Variant-level** (`variant_overrides`): list **every** entry present in
    the file, transcribed exactly (see Step 2) — do not treat the set as fixed.
    Known categories include private-domain overrides (`PULP_DOMAIN: rhai`,
    e.g. the `rhaiis` gaudi/neuron/tpu variants carrying vendor wheels that
    cannot be publicly redistributed, AIPCC-28553) and index-path overrides
    (`PULP_BASE_PATH`, e.g. `torch-day0` variants), but the file is the source
    of truth for what actually exists. Precedence:
    `_default -> overrides.<collection> ->
    variant_overrides.<collection>.<variant>`.
- **OMIT_JOBS** -- Explicit exclusions from the matrix with their reasons.
  Only **effective** entries (first element is a collection key, per Step 2)
  justify narrowing a variant's arch coverage in the table above: every reduced
  arch annotation must map to an effective entry, and every effective entry must
  be reflected as a reduced arch set in the corresponding collection row. If a
  previously-omitted `(collection, variant, arch)` combination is no longer
  listed, that variant now builds on the full base arch set — update the table
  accordingly. **List any dead entries separately and label them as such** (an
  entry whose first element is a package/product name, not a collection, so
  `regen-ci.py` never matches it — e.g. `docling`, `sdg-hub`). Do not describe a
  dead entry as a working exclusion; where the intended arch skip is actually
  achieved by a PEP 508 marker in `requirements*.txt`, say so and cite the
  marker.
- **Test jobs** -- Which collections have `ENABLE_TEST_JOBS: true`
- **The `rhai` Collection** -- Team ownership structure; list notable team files
  and their contents
- **Onboarding Pipeline** -- How new packages enter (onboarding -> graduation via
  weekly bot -> `rhai` collection)
- **Pipeline Flow** -- Stage list; trigger types; key checks-stage gates
  (variant-linter, verify-publish-config, validate-package-deletion-manifests)
- **Pulp Publishing Mechanics** -- Two-stage workflow (upload then publish);
  repository naming convention; authentication method; `-test` distribution
  auto-increment. Routing has **two independent dimensions**, both driven by
  `overrides`/`variant_overrides` (Step 2): `PULP_DOMAIN` (which Pulp instance —
  public `public-rhai` vs private `rhai`) and `PULP_BASE_PATH` (the path/repo
  name **within** a domain). The general rule -- all collections for a published
  variant go to the same public index at the default base path -- applies only
  to variants with **no** `PULP_DOMAIN` and **no** `PULP_BASE_PATH` override.
  Enumerate both kinds of exception from `variant_overrides`:
  - `PULP_DOMAIN` overrides (e.g. `vllm-deps` collection-level, and the `rhaiis`
    gaudi/neuron/tpu variants) redirect to the private `rhai` instance.
  - `PULP_BASE_PATH` overrides (e.g. `torch-day0` variants) keep the **public**
    `public-rhai` domain but publish to a **version-pinned base path** within it
    — the source values are `PULP_BASE_PATH: torch-2.14.0-cpu-ubi9` and
    `PULP_BASE_PATH: torch-2.14.0-cuda13.0-ubi9`, which yield the Pulp repos
    `public-rhai/torch-2.14.0-cpu-ubi9` and `public-rhai/torch-2.14.0-cuda13.0-ubi9`
    (each with a `-test` sibling). `dual-repo-promote.sh` derives the repo name
    directly from `PULP_BASE_PATH` (flattening any `/` to `-` — a no-op here, since
    these values contain no slash) and ignores `PRODUCT_VERSION` entirely. Copy the
    base path **verbatim** from source; do not synthesize it from the variant name.
    The published repo **drops** the `-torch-day0` collection segment — it is
    **not** `torch-2.14.0-cpu-torch-day0-ubi9`. These variants do **not** land at
    the default `rhoai/<PRODUCT_VERSION>/<variant>-<stage>` path.
    Do not describe them as using the default per-product path.
  Do not assert that deletion or promotion flows differ for the private-routed
  collections beyond what the source establishes.
- **Package Deletion System** -- Manifest-driven, enforced at upload time,
  idempotent
- **Version Branching** -- `copy_wheels.py` for EA->GA promotion

**Impact on Strategies section** -- Update to reflect current state. Must include:

- A bullet establishing this repo as the authoritative Pulp publish gate
- A bullet on the steps required to add a new collection or variant
- A bullet on `ENABLE_REPEATABLE_BUILD_MODE` implications for release branches
- A bullet on how many CUDA versions are maintained and the cost of adding one
- A bullet on ROCm version state (which is built vs which is published)
- A bullet on Spyre's IBM-proprietary wheel stack: the vLLM IBM fork (`.spyre`
  suffix), sendnn-inference (IBM's inference runtime), torch-sendnn (x86/P,
  pre-built from private index), torch-nnpa (Z only, pre-built from private
  index), ibm-fms (Foundation Model Stack, built from source). Torch version is
  shared with other variants via constraints-rules.txt. aiu-monitor is blocked
  from all wheel collections and lives in the base image instead.
- A bullet on the public URL as a stable contract for air-gapped mirroring
- A bullet on the idempotent deletion system and the onboarding graduation cadence

**ROCm Work Breakdown Patterns** -- Include this subsection to guide downstream
strategy generation. When a strategy involves a ROCm variant update in the
pipeline (e.g., new ROCm version or ROCm package changes), the pipeline-side
work decomposes into these epics:
- Update ROCm variant constraints (torch pin, vllm pin, ROCm-specific package
  versions in constraints.txt and constraints-rules.txt)
- Add or update ROCm-specific packages in collections (amd-quark, amd-aiter,
  tensorflow-rocm, flash-attn, and any new AMD ecosystem packages)
- Validate build and publish for the ROCm variant (CI pipeline green, wheels
  uploaded to Pulp, customer-facing index updated)

Strategies referencing ROCm pipeline updates should structure their Technical
Approach around these epics rather than describing the work as prose.

**Context section** -- Keep the rationale unchanged. Update the date and version
references to remain accurate.

### Step 5: Write the Updated File

Write the updated content to `overlays/0020-rhai-pipeline.md` using the Write
tool.

### Step 6: Report

Output a brief summary:

```
Updated overlays/0020-rhai-pipeline.md

Changes:
- [product version: old -> new]
- [builder version: old -> new]
- [any variants added/removed from publish_config.yml]
- [any collections added/removed]
- [any notable changes to the publishing workflow]

Repository used: {REPO} (./tmp/fondue/rhai-pipeline is not tracked by git)
```

## Notes

- **Trust assumption:** The fetch script validates the git remote origin against
  the allowlisted fondue repository. Both the HTTPS form
  (`https://gitlab.com/redhat/rhel-ai/wheels/fondue.git`) and the SSH form
  (`git@gitlab.com:redhat/rhel-ai/wheels/fondue.git`) are accepted, as they
  resolve to the same repository. Do not bypass the fetch script by supplying a
  path directly.
- `tmp/` is in `.gitignore`; the cloned repository is local only
- The script is idempotent: run it again any time the upstream repo changes
- Do not change the overlay `id` (0020) or `author` fields
- Preserve AIPCC ticket references when they are still accurate
- The Pulp version HREFs in `publish_config.yml` are long UUIDs -- include only
  the version number portion in the overlay, not the full HREF
- Do not commit any changes to the rhai/pipeline repository or to GitLab
