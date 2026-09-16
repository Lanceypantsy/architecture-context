---
name: update-aipcc-base-images-overlay
description: Use when the AIPCC base image repository has changed and the overlay file overlays/0017-aipcc-base-images.md needs to be refreshed with current accelerator support information, versions, or architecture details.
user-invocable: true
allowed-tools: Read, Write, Bash(bash ${CLAUDE_SKILL_DIR}/scripts/fetch-base-images-repo.sh), Bash(python3 ${REPO}/bin/generate-platform-docs.py), Glob, Grep
---

# Update AIPCC Base Images Overlay

Refresh `overlays/0017-aipcc-base-images.md` with current information from the
AIPCC base images repository (`images/base/` inside the fondue monorepo).

## Overview

The overlay documents the accelerator variants (CPU, CUDA, ROCm, Gaudi, Spyre,
Neuron, TPU, Rubin) built from `images/base/`. It is used to evaluate RFEs that
propose changes to accelerator support. When the repository changes, run this
skill to update the overlay.

**Dependency management model (as of 3.6):** Package dependencies are declared
in `context/<variant>/rpms.in.yaml` and resolved into a hermetic lockfile at
`context/<variant>/rpms.lock.yaml` using `rpm-lockfile-prototype`. Konflux
(via Cachi2) fetches packages directly from the locked CDN URLs during hermetic
builds. Routine version drift is handled automatically by MintMaker
(`refresh-rpm-lockfiles` Renovate preset). Manual lockfile regeneration is only
needed when adding or removing packages from `rpms.in.yaml`.

## Instructions

### Step 1: Locate the Repository

Run the fetch script from the root of the architecture-context repository:

```bash
bash ${CLAUDE_SKILL_DIR}/scripts/fetch-base-images-repo.sh
```

The script checks for a local fondue checkout at `../fondue/images/base`. If
found, it prints that path and exits. Otherwise it clones or updates
`./tmp/fondue` from `https://gitlab.com/redhat/rhel-ai/wheels/fondue.git` and
prints `./tmp/fondue/images/base`.

Use the printed path as `{REPO}` in all subsequent steps.

### Step 2: Read the Key Files

All paths are relative to `{REPO}`.

**Common build configuration:**
- `build-args/argfile.conf` -> `APP_BASE_IMAGE` (base OS pin), `INDEX_VERSION`
  (RHOAI product version), `REPO_VERSION` (RHEL AI RPM repo version),
  `PYTHON_VERSION` default, `INDEX_BASE_URL` (default public package index URL),
  `TEST_INDEX_SUFFIX`

**Per-variant Python package index** -- read each `build-args/<variant>-*.conf`
for `INDEX_BASE_URL`, `INDEX_VARIANT`, `DISTRIBUTION_SCOPE`, and whether
`regen-skip: INDEX_BASE_URL` is present (indicating a deliberate override).
There are three index patterns:

- **Public RHEL AI index** (`https://packages.redhat.com/api/pypi/public-rhai/rhoai`):
  CPU, CUDA, ROCm, Rubin. These images embed a single `index-url` in `pip.conf`
  and `uv.toml` pointing to this index.
- **Private RHAIIS index** (`https://private.console.redhat.com/api/pypi/rhai/rhaiis`):
  Gaudi, Neuron, TPU. Marked `regen-skip: INDEX_BASE_URL` and
  `DISTRIBUTION_SCOPE=private`. The private index **replaces** the public index
  entirely -- pip and uv are configured with a single `index-url` (not
  `extra-index-url`), so there is no fallback to the public index. These variants
  use a private index because their wheels contain non-redistributable content
  (Habanalabs, AWS Neuron SDK, Google TPU/Torch-XLA). Downstream product
  container builds consume this index directly.
- **Torch Day 0 variants**: use the public host (`packages.redhat.com/api/pypi/public-rhai`)
  with a different index path structure (`torch-2.X.Y-{variant}`).
- **Spyre**: public index URL is set but carries a `# NOTE: index does not exist`
  comment -- Spyre has no Python wheel index. All Spyre Python dependencies come
  from RPMs (Spyre IBM SDK) or pre-built IBM wheels served via the builder's
  private Spyre PyPI (a separate mechanism from this index).

**Per-variant configuration** (one directory per variant under `context/`):
- `context/<variant>/rpms.in.yaml` -- **primary source of truth for package
  content.** Contains:
  - `contentOrigin.repofiles`: which `.repo` files feed the resolver (RHEL EUS,
    RHELAI, and any accelerator-specific repo)
  - `arches`: list of target architectures for this variant
  - `packages`: flat list of package names (bare names) plus arch-scoped entries
    using `arches: {only: [...]}`. For Spyre, IBM SDK RPMs are expressed as
    versioned package names (e.g. `ibm-aiu-toolbox-e2e-1.3.0`) -- these are the
    authoritative version pins (AIPCC-29839).
- `context/<variant>/rpms.lock.yaml` -- hermetic lockfile produced by
  `rpm-lockfile-prototype`. Contains per-arch closures with full CDN URL,
  sha256/sha512 checksum, size, name, and EVR for every resolved package.
  Read to confirm resolved EVRs for key packages (e.g. CUDA driver RPMs, ROCm
  SDK version, Spyre IBM RPMs). Do not use this file to enumerate the *declared*
  package list -- use `rpms.in.yaml` for that.

**Build args per variant** (read for hardware-specific details):
- `build-args/cuda12.9-*.conf`, `build-args/cuda13.0-*.conf`,
  `build-args/cuda13.2-*.conf` -> CUDA toolkit version, `TORCH_CUDA_ARCH_LIST`
- `build-args/rocm7.*.conf` -> ROCm version
- `build-args/spyre-*.conf`, `build-args/gaudi-*.conf`, `build-args/tpu-*.conf`,
  `build-args/neuron-*.conf` -> other variant-specific build args

**Variant inventory** (authoritative list of variants currently built):
- `build-args/` directory listing -> one conf file per variant (e.g.
  `cuda13.0-el9.8-app.conf`); `argfile.conf` is the common base
- Cross-check with `context/` directory listing to confirm per-variant lockfiles

**Documentation generator** (supplementary -- use to verify tables):
```bash
python3 {REPO}/bin/generate-platform-docs.py
```
Run from `{REPO}`. This script reads the conf files and produces a human-readable
accelerator summary table. Use it to validate what you read from the source
files, not as the sole source of truth.

**Lockfile tooling** (for context -- do not execute):
- `bin/hermetic-generate-lockfiles.sh` -> how lockfiles are regenerated manually
  (only needed when adding/removing packages); reads `rpm-lockfile-prototype`
- `bin/lint-lockfiles.py` -> static linter run in CI (arch alignment, integrity,
  coupling: if `rpms.in.yaml` changed, `rpms.lock.yaml` must also change)
- `bin/merge-arch-locks.py` -> merges per-arch lockfile outputs

### Step 3: Extract Current State per Variant

For each variant in `context/`, read `rpms.in.yaml` to determine:
- Target architectures (`arches:`)
- Repo sources (`contentOrigin.repofiles`)
- Notable declared packages (especially any version-pinned names, arch-scoped
  packages, and packages with JIRA-referenced comments)

For Spyre specifically, record all `ibm-*` versioned package names from
`rpms.in.yaml` -- these are the IBM SDK version pins replacing the old conf-file
approach (AIPCC-29839).

For CUDA variants, read `build-args/cuda*.conf` for the CUDA toolkit version
and `TORCH_CUDA_ARCH_LIST`.

### Step 4: Read the Current Overlay

Read `overlays/0017-aipcc-base-images.md` to understand the existing structure.
Identify the "Impact on Strategies" and "Context" sections, which must be
preserved and updated -- not replaced wholesale.

### Step 5: Update the Overlay

Rewrite `overlays/0017-aipcc-base-images.md` using the following approach:

**Preserve the YAML front matter** (`id`, `title`, `status`, `created`,
`affects`, `release`, `provenance`, `author`, `superseded_by`). Update
`release` only if the repository clearly targets a new RHEL AI release.

**Fact section** -- Replace entirely with fresh content. This section must cover:

- What the base images are and how downstream teams use them
- **Common foundation** -- base OS image pin (from `argfile.conf`), Python
  version, RHEL AI repo version, package index version and URL, container
  layout, environment metadata (labels, env vars), helper scripts
- **Dependency management model** -- describe the `rpms.in.yaml` / `rpms.lock.yaml`
  pattern: `rpms.in.yaml` declares packages and arch scoping;
  `rpm-lockfile-prototype` resolves the hermetic lockfile; Konflux/Cachi2
  fetches from locked CDN URLs at build time; MintMaker handles routine drift
  automatically; manual regeneration required only when adding/removing packages
- **Accelerator Summary** table with columns:
  `Accelerator | Version | Status | Python | RHEL | aarch64 | ppc64le | s390x | x86_64`
- **Status Legend** -- Active / In development / Disabled / Retired
- One subsection per accelerator variant covering: status, `rpms.in.yaml` path,
  target architectures, container image name, driver/SDK version (from build-args
  conf or lock), Python package index (public RHEL AI, private RHAIIS, Torch Day 0,
  or none), `DISTRIBUTION_SCOPE`, and notable declared packages (version-pinned or
  arch-scoped)
- For **Spyre**: list the IBM SDK RPM version pins from `rpms.in.yaml` and note
  that these are the single source of truth per AIPCC-29839; note which packages
  are arch-conditional (`only: [...]`)
- **Retired Accelerators** subsection for anything removed from the repo

**Impact on Strategies section** -- Update to reflect current state. Must include:

- A bullet that all RHAI components using accelerator-specific Python libraries
  must use these base images
- A bullet on the lockfile-based hermetic build model: package content is fixed
  at lockfile commit time; RFEs adding packages must account for the
  `rpms.in.yaml` -> lockfile regeneration -> CI validation cycle
- Bullets for each non-Active variant (in-development, disabled, retired)
  explaining what strategies must or must not assume
- A bullet about the number of concurrent CUDA versions and the driver-version
  dependency each introduces
- A bullet about Spyre's arch-conditional IBM RPM stack and that version pins
  live in `rpms.in.yaml` (not in wheel pipeline requirements files)
- A bullet explicitly distinguishing the two Python package index patterns:
  (a) public RHEL AI index for CPU/CUDA/ROCm/Rubin -- the default for most
  strategies; (b) private RHAIIS index for Gaudi/Neuron/TPU -- **replaces** the
  public index entirely (single `index-url`, no fallback) because those wheels
  are non-redistributable; downstream product builds consume the private index
  directly. Strategies targeting Gaudi, Neuron, or TPU must not assume the
  public RHEL AI index is available or sufficient for those variants.

**Context section** -- Keep the rationale unchanged. Update the date and any
version references so it remains accurate.

### Step 6: Write the Updated File

Write the updated content to `overlays/0017-aipcc-base-images.md` using the
Write tool.

### Step 7: Report

Output a brief summary:

```
Updated overlays/0017-aipcc-base-images.md

Changes:
- [list any accelerators added, removed, or with changed status/versions]
- [note any changes to common foundation (base OS, Python, RHEL AI repo)]
- [note any Spyre IBM SDK RPM version changes]
- [note any lockfile model changes if relevant]

Repository used: {REPO}
```

## Notes

- **Trust assumption:** The fetch script validates the git remote origin against
  the allowlisted fondue repository. Both HTTPS and SSH forms are accepted.
  Do not bypass the fetch script.
- The `rpms.in.yaml` + `rpms.lock.yaml` pattern replaced the old per-variant
  conf-file approach as of 3.6. When reading older overlays or conf files,
  treat them as historical -- the lockfiles are authoritative now.
- `tmp/` is in `.gitignore`; any cloned repository is local only
- Do not change the overlay `id` (0017) or `author` fields
- Preserve JIRA ticket references (e.g. AIPCC-29839, AIPCC-29840) in the Fact
  section when they are still accurate; remove them if the underlying issue
  is resolved
- Do not commit changes to the fondue repository or to GitLab
