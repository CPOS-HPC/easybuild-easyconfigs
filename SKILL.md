---
name: easybuild-easyconfigs
description: Create, migrate, review, debug, validate, and maintain EasyBuild easyconfig recipes and patches. Use for .eb files, toolchain bumps, dependency recipes, PythonBundle/PythonPackage extensions, CargoPythonBundle vendoring, binary tarball installations, checksum failures, offline Cargo or pip errors, sanity checks, and repository commit/push work.
---

# EasyBuild Easyconfigs

Create recipes that match this repository, resolve through its dependency graph, and survive an actual EasyBuild installation.

## Follow the core workflow

### 1. Inspect before editing

- Read the target recipe and the closest recipes by software, version, toolchain, and easyblock.
- Search with `rg` or `find`; do not invent repository conventions from memory.
- Inspect `git status` and preserve unrelated user changes.
- Prefer the newest relevant local recipe as the structural template.
- When the user identifies a specific reference recipe, retain its dependency set and ordering unless upstream requirements justify a change.
- Check upstream release metadata, source contents, lockfiles, and package requirements when they can have changed.

For a simple copy-forward migration where the software version, pinned commit, sources, patches, and checksums are
unchanged:

- Copy the closest recipe and update only the requested toolchain and generation-matched dependencies.
- Do not re-download or re-hash unchanged artifacts.
- Confirm the source, commit, version, patches, and checksums remain identical with a focused diff.
- Run the robot dry run and `git diff --check`. Reserve an isolated build for demonstrated compatibility risk or a
  prior build failure.

### 2. Select the source artifact deliberately

- Distinguish PyPI sdists, GitHub release assets, GitHub-generated tag archives, crates.io archives, and vendor binary tarballs. Their checksums differ even for the same version.
- Use PyPI’s sdist checksum when a `PythonBundle` extension will fetch from PyPI.
- Use GitHub sources when the build needs repository-only files such as `Cargo.lock`, subprojects, tests, or generated metadata absent from the PyPI sdist.
- Prefer stable tags or releases. For untagged software, pin a full commit and use its commit date as the version when that matches repository precedent.
- When upstream metadata declares a release version but no corresponding tag exists, retain that declared version and pin a full commit that still represents it.
- Check the default branch tip immediately before finalizing a commit-based recipe. Do not confuse the commit that introduced a file with the repository’s latest commit.
- Never use a commit that is unavailable from the declared repository.
- Download and hash the exact artifact referenced by the recipe.
- Add filename comments only when a checksum list contains multiple artifacts. Put each filename comment after its checksum value. Do not add comments to single-entry one-line checksum lists:

```python
checksums = [
    'source-sha256',  # package-1.2.3.tar.gz
    'patch-sha256',  # package-1.2.3_fix-build.patch
]
```

### 3. Choose the dependency layer and toolchain

- Use `GCCcore` for compiler-independent or lightly compiled libraries that fit the core stack.
- Use `GCC` for compiled software needing the full compiler but no MPI or BLAS/LAPACK stack.
- Use `gfbf` for BLAS/LAPACK-linked scientific software without MPI.
- Use `foss` when MPI or the complete scientific stack is required.
- Match dependencies to the target toolchain generation; do not mix Python ABI generations.
- Verify every dependency recipe exists with robot resolution.
- Prefer `pkgconf` over the legacy `pkg-config` dependency.
- Treat dependency versions as a compatibility set. Check upstream constraints and run `pip check`.
- Do not drop optional-looking dependencies during a bump without checking whether the reference recipe intentionally included them.

Known generation anchors in this repository include:

| Toolchain | GCCcore | Typical Python |
|---|---:|---:|
| 2023b | 13.2.0 | 3.11.5 |
| 2024a | 13.3.0 | 3.12.3 |
| 2025a | verify locally | verify locally |
| 2025b | 14.3.0 | verify locally |
| 2026.1 | 15.2.0 | 3.14.2 |

Always confirm anchors from local easyconfigs instead of treating this table as exhaustive.

### 4. Implement with repository conventions

- Preserve official capitalization in `name`, filenames, modules, homepage, and descriptions.
- Follow the field order of the closest current recipe.
- Use standard source constants such as `SOURCELOWER_TAR_GZ` when the source follows the standard naming convention.
- Keep single-entry `sources` and `checksums` lists on one line. Use multiline form for multiple artifacts or entries that are not readable when compact.
- Keep checksum filename comments on the checksum line.
- Keep patches minimal, descriptive, reusable across toolchains when their changes are source-version-specific, and authored:

```text
Author: Emik Lin (HKUMed CPOS)
```

- Prefer patch-free configuration through documented build options when practical.
- Do not add symlink farms, vendored copies, or patches unless the build system requires them.
- Do not patch upstream packaging merely to install an auxiliary or historical script. If the unmodified installer provides the official entry points and passes the build and sanity checks, retain it unchanged.
- Add explicit sanity paths only when they improve coverage beyond the easyblock defaults.
- Use meaningful import and CLI sanity commands.

## Handle common recipe families

### PythonPackage and PythonBundle

- Use `PythonPackage` for one primary Python distribution.
- Use `PythonBundle` when installing ordered extensions or when the application needs bundled PyPI dependencies.
- Order `exts_list` so build backends and dependencies are installed before their consumers.
- Specify nonstandard module names with `modulename`.
- Specify nonstandard sdist names with `source_tmpl` or `sources`.
- Add backend packages such as `pdm-backend`, `hatchling`, `poetry`, `maturin`, or `scikit-build-core` as build dependencies or earlier extensions when required by `pyproject.toml`.
- Do not assume pip can resolve missing dependencies during an offline EasyBuild build.
- For `PythonBundle`, do not add `sanity_pip_check = True`; it is enabled by default. Set this option only when deliberately overriding the default, with a documented reason.
- Validate both imports and console entry points.
- When assessing a Python 2 project for Python 3, distinguish a mechanically converted script from package-wide compatibility. Compile all installed scripts and test a representative data path before declaring the package compatible.
- Keep experimental Python 3 conversion changes out of a requested Python 2 recipe unless they are required for that build.

When pip reports a constraint conflict, pin versions to the application’s declared range rather than disabling `pip check`. Examples include:

- `biopython<1.86`
- `importlib-metadata<9`
- `pandas<3`
- minimum `typing_extensions`
- exact ranges for `click`, `mypy`, or `ruff`

When a package exposes no import matching its distribution name, set the correct `modulename` or disable only that extension’s import sanity check with evidence.

### Rust and Cargo Python packages

- Obtain dependencies from the exact `Cargo.lock` used by the build.
- Generate the crate list with the EasyBuild Cargo helper after loading EasyBuild:

```bash
module load EasyBuild
python3 -m easybuild.easyblocks.generic.cargo path/to/Cargo.lock
```

- Re-run the helper whenever the source version or lockfile changes.
- Keep every locked crate version needed by every built subpackage. Do not add versions speculatively.
- Preserve `Cargo.lock`; do not delete it to force resolution.
- Treat lockfile checksum conflicts as evidence that crate versions or sources were mixed.
- For git dependencies, vendor the exact reachable revision or a stable tagged archive and add its archive checksum.
- Prefer upstream source archives containing the correct lockfile over large patches that reconstruct dependency metadata.
- When several extensions share one vendor directory, ensure the union of their lockfile dependencies is present.
- Check the minimum supported Rust version. If a crate requires a newer compiler, prefer an application release or dependency version compatible with the repository Rust toolchain.

Interpret common failures precisely:

- `can't checkout ... offline`: a git dependency was not vendored.
- `candidate versions found which didn't match`: the exact locked crate version is missing.
- `checksum ... changed between lock files`: incompatible lockfiles or crate sources were merged.
- `requires rustc X`: update Rust or select a compatible application/dependency release.

### Prebuilt binary tarballs

- Follow an existing binary recipe for the same vendor or a comparable package such as modkit.
- Use the exact vendor download URL and checksum.
- Confirm the extracted top-level directory before setting `start_dir`.
- Sanity-check the executable and a harmless command such as `--help` or `--version`.
- Put required post-install data downloads in the recipe only when the user explicitly requests them and the installed program supports the command, for example:

```python
postinstallcmds = ['dorado download --model all']
```

Account for the fact that post-install downloads require network access and can be large.

### Perl extensions

- Retain the dependency structure of the closest recipe.
- Add `Perl-bundle-CPAN` when build scripts require standard CPAN helpers such as `File::Which`.
- Do not replace a precise missing Perl module diagnosis with unrelated build changes.

### Legacy build systems

- Prefer external dependencies over bundled stale copies when the upstream build supports them.
- Add include-directory symlinks only when the source hard-codes a `third/` vendor layout.
- If a patch makes the build system discover external libraries normally, remove redundant symlinks.
- Use `maxparallel = 1` only for demonstrated race conditions or non-parallel-safe builds.

## Keep API compatibility in view

- A matching Python version does not guarantee application compatibility.
- Inspect the APIs the application calls before choosing a newer dependency.
- Do not force a newer major version merely because its recipe exists.
- Example: amplimap 0.4.20 calls the legacy `snakemake.snakemake(...)` API and its cluster arguments; Snakemake 8 removed that API. Retain Snakemake 7 or omit the newer toolchain recipe unless the integration is substantially rewritten and tested.
- Reuse one source-version patch across toolchains only when all patched APIs and dependency bounds remain valid for both stacks.

## Diagnose from the first actionable error

- For easyconfig parse errors, inspect the reported line and nearby `%` formatting, dictionaries, tuples, and strings.
- For missing checksums, compare the filename EasyBuild requests with the filename-to-checksum mapping.
- For metadata backend errors such as `Cannot import 'pdm.backend'`, add or correctly order the build backend.
- For `pip check` errors, update the declared dependency set; do not hide the check.
- For SSL failures, distinguish a local certificate-chain problem from a missing upstream mirror. Prefer a verified alternate source only when one exists.
- For missing imports after install, inspect the installed site-packages directory and upstream package layout before changing sanity checks.
- For native build errors, inspect compiler, Python ABI, Rust MSRV, BLAS/MPI toolchain, and build-backend compatibility separately.

## Validate proportionally

Always load EasyBuild before invoking `eb`:

```bash
module load EasyBuild
```

Run a robot dry run:

```bash
eb path/to/recipe.eb \
  --dry-run \
  --robot=/path/to/easybuild/easyconfigs \
  --minimal-toolchains
```

For new recipes and risky migrations, perform an isolated installation under `/tmp`:

```bash
eb path/to/recipe.eb \
  --robot=/path/to/easybuild/easyconfigs \
  --sourcepath=/tmp/task-sources:/software/easybuild/sources \
  --buildpath=/tmp/task-build \
  --installpath-software=/tmp/task-install \
  --installpath-modules=/tmp/task-modules \
  --minimal-toolchains
```

If EasyBuild’s downloader lacks network access, prefetch exact sources into the temporary source tree and verify them with `sha256sum`.

Before handoff:

1. Verify every new or changed source and patch checksum. For a simple copy-forward migration, compare unchanged
   checksum values with the reference recipe instead of downloading and hashing the artifacts again.
2. Run `patch --dry-run` for new or changed patches, or when the source version changed. Do not repeat it for an
   unchanged source-and-patch pair in a simple copy-forward migration.
3. Run the EasyBuild robot dry run.
4. Run a full isolated build for new recipes and risky migrations when feasible. For a simple copy-forward migration,
   the robot dry run is sufficient unless compatibility concerns or a prior failure justify a build.
5. Confirm imports, executables, version output, extension sanity, and `pip check` when a build or matching installed
   module is available. Do not require these runtime checks for the robot-only simple copy-forward path.
6. Run `git diff --check`; normalize patch-file blank context lines if needed, then recompute the patch checksum.
7. Review `git status` and the exact diff.

Treat `--check-contrib` failures caused by a missing local checker such as `pycodestyle` as an environment issue, but do not use that to excuse recipe parse, style, or build failures.

## Commit safely

- Commit and push only when the user explicitly requests it.
- Stage only files belonging to the task.
- Recheck the staged diff with `git diff --cached --check`.
- Use a concise commit message naming the software, version, and toolchain when useful.
- Push the current branch and confirm `HEAD` matches its upstream.
- Never discard unrelated dirty-worktree changes.
