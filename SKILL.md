---
name: easybuild-easyconfigs
description: Create, migrate, review, debug, validate, and maintain EasyBuild easyconfigs and patches, including containerized web launchers and Open OnDemand integration. Use for .eb files, toolchain bumps, dependency recipes, vendoring, binary installations, checksum or offline-build failures, sanity checks, and repository commit/push work.
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
- Keep every line in every easyconfig (`*.eb`) at or below 120 characters. Check the complete file, including
  unchanged lines, whenever creating or modifying an easyconfig.
- Treat E501 fixes as formatting-only changes: preserve values, ordering, checksums, URLs, and commits exactly.
  Use the reported lines as starting points, then scan the complete file because wrapping shifts later line numbers.
- For long Cargo checksum mappings and git-source crate tuples, match nearby multiline precedent rather than splitting
  string values or regenerating dependency data. Prefer forms such as:

  ```python
  {
      'very-long-crate-archive-name.tar.gz':
      'sha256',
  },
  (
      'crate', '1.2.3', 'https://example.invalid/repository',
      'commit',
  ),
  ```

- Use standard source constants such as `SOURCELOWER_TAR_GZ` when the source follows the standard naming convention.
- Keep single-entry `sources` and `checksums` lists on one line. Use multiline form for multiple artifacts or entries that are not readable when compact.
- Keep checksum filename comments on the checksum line.
- Keep patches minimal, descriptive, reusable across toolchains when their changes are source-version-specific, and authored:

```text
Author: Emik Lin (HKUMed CPOS)
```

- Match the patch filename style used by neighboring recipes, typically
  `%(name)s-%(version)s-fix-<issue>.patch`.
- Keep each patch focused on one concern. Separate fixes to the application source from fixes to bundled dependencies.
- Preserve the existing patch order and append new patches unless a dependency between patches requires otherwise.
  Keep checksums in the same order as sources and patches.
- Prefer patch-free configuration through documented build options when practical.
- Do not add symlink farms, vendored copies, or patches unless the build system requires them.
- Do not patch upstream packaging merely to install an auxiliary or historical script. If the unmodified installer provides the official entry points and passes the build and sanity checks, retain it unchanged.
- Add explicit sanity paths only when they improve coverage beyond the easyblock defaults.
- Use meaningful import and CLI sanity commands.
- In `modextrapaths`, an empty relative path means the installation root. Retain `'PATH': ['']` when the primary
  executable is installed directly under `%(installdir)s`; removing it makes the command unavailable after module
  load. Do not expose internal directories such as `util` unless their programs are intended as public entry points.

## Handle common recipe families

### PythonPackage and PythonBundle

- Use `PythonPackage` for one primary Python distribution.
- Use `PythonBundle` when installing ordered extensions or when the application needs bundled PyPI dependencies.
- For current `PythonPackage` recipes, rely on easyblock defaults for pip installation, dependency-download failure,
  and pip sanity checks. Do not add the legacy `download_dep_fail` assignment unless deliberately overriding a
  default with evidence.
- Do not add `use_pip = True` or `sanity_pip_check = True` to `PythonPackage` or `PythonBundle` recipes; both are
  enabled by default. Set either option only when deliberately overriding its default, with a documented reason.
- Inspect the EasyBuild command trace before setting `prebuildopts` or `preinstallopts`. A pip-based `PythonPackage`
  commonly has a no-op build step and builds its wheel during `pip install .`; put compile-time environment settings
  in `preinstallopts` unless the trace confirms a separate build command. A command-prefix assignment does not persist
  into a later phase.
- Treat upstream's manual `python -m build` followed by `pip install dist/*.whl` as one valid frontend workflow, not a
  requirement to reproduce two phases. The default local pip installation performs the PEP 517 wheel build and install;
  add `build` only when the selected workflow actually invokes it.
- Read how upstream parses build-control environment variables before assigning them. Values such as `enable`,
  `disable`, and `system` are selectors, not installation prefixes; do not replace `system` with `$EBROOT...` unless
  upstream explicitly accepts a path. EasyBuild dependency modules already expose their headers and libraries.
- Distinguish an optimized bundled dependency from a different system-compatible implementation. For example, an
  upstream `system` mode that links `-lz` does not consume a native zlib-ng module providing `libz-ng`; verify headers,
  library names, symbol compatibility, and compatibility-mode configuration before externalizing it. When bundled
  dependencies are intentional, omit conflicting EasyBuild dependencies and confirm the installed extension does not
  dynamically link the unwanted system library.
- Follow nearby current recipes on source URL declarations. Omit an explicit `PYPI_SOURCE` when the standard PyPI
  sdist and implicit source handling are sufficient.
- Order `exts_list` so build backends and dependencies are installed before their consumers.
- Specify nonstandard module names with `modulename`, including distributions whose import name differs because a
  hyphen becomes an underscore, such as `igv-reports` importing as `igv_reports`.
- Specify nonstandard sdist names with `source_tmpl` or `sources`.
- Add backend packages such as `pdm-backend`, `hatchling`, `poetry`, `maturin`, or `scikit-build-core` as build dependencies or earlier extensions when required by `pyproject.toml`.
- Do not assume pip can resolve missing dependencies during an offline EasyBuild build.
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

### Bioconductor R packages

- Determine the Bioconductor release corresponding to the target R version from the official release table. Do not
  use the newest Bioconductor release when the requested toolchain uses an older R generation.
- Confirm that matching `R` and `R-bundle-Bioconductor` easyconfigs exist locally before selecting versions.
- Read the release-specific `PACKAGES` or `PACKAGES.gz` index to obtain the exact package version, dependency
  metadata, and `NeedsCompilation` value.
- Use the closest current recipe for the package as the structural template. Retain auxiliary extensions that are not
  supplied by the matching R or Bioconductor bundles.
- Express the release consistently:

```python
versionsuffix = '-R-%(rver)s'
local_biocver = '3.20'

dependencies = [
    ('R', '4.4.2'),
    ('R-bundle-Bioconductor', local_biocver, versionsuffix),
]
```

- Fetch `%(name)s_%(version)s.tar.gz` from the release-specific Bioconductor `bioc/src/contrib` location. Keep the
  Bioconductor archive and data-package URLs used by nearby recipes when building a bundle.
- Hash the exact release artifact. If the primary Bioconductor endpoint is temporarily unavailable, use the official
  Bioconductor archive object store for the same release artifact rather than substituting a different source.
- Use `RPackage` for extensions in a Bundle recipe and preserve the extension ordering required by package
  dependencies. Rely on extension sanity checks unless explicit paths or commands add useful coverage.
- Validate the robot graph and, for a new package version or changed Bioconductor release, run an isolated build when
  feasible because compiled R extensions and bundle contents can change across releases.

### Rust and Cargo Python packages

- Obtain dependencies from the exact `Cargo.lock` used by the build.
- Use an existing `Cargo.lock` unchanged. Run `cargo generate-lockfile` only when the selected source does not provide
  a lockfile.
- From the Rust project root containing the applicable lockfile, generate the crate list after loading EasyBuild:

```bash
module load EasyBuild
# Run only when Cargo.lock is absent:
cargo generate-lockfile
python3 -m easybuild.easyblocks.generic.cargo . > list_of_crates.txt
```

- Re-run the crate-list helper whenever the source version, manifests, or lockfile changes.
- Compare `list_of_crates.txt` byte-for-byte with the easyconfig crate list. Preserve the generated order, versions,
  sources, and checksums exactly; do not omit entries or add speculative versions.
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

### Apptainer web applications behind Open OnDemand

- Treat the launcher as the integration layer for an immutable container. Extract upstream configuration templates at
  installation time, generate per-session copies at runtime, and bind those copies into the container. Give concurrent
  sessions unique sockets, PID files, temporary directories, and supervisor state.
- Trace every browser-visible URL producer when serving an application below an Open OnDemand proxy prefix such as
  `/rnode/HOST/PORT`: HTML asset URLs, the bundler public path, client-side router basename, API server, data-source
  catalogue, export/share endpoints, and stored or generated view configurations. Fixing only HTML `href` and `src`
  attributes does not prevent JavaScript from navigating to an origin-root path.
- Distinguish a router basename from a route. Inspect the pinned frontend source before assigning it. If the router
  basename is `/rnode/HOST/PORT` and the application route is `/app`, the browser entry point is
  `/rnode/HOST/PORT/app`; including `/app` in both commonly produces a doubled or unmatched route.
- Keep frontend API URLs under the same OOD prefix unless the application explicitly supports a different public API
  origin. For applications whose default view contains data-source URLs, generate an OOD-specific view configuration
  rather than relying on origin-root values such as `/api/v1`.
- A successful Apptainer bind proves only filesystem visibility. Data services may also require a database/catalogue
  registration. Verify both the stored path and a representative catalogue API response before concluding that the
  frontend can discover the data.
- Diagnose proxy-host failures at the header boundary. Django can reject a comma-separated `X-Forwarded-Host` as
  syntactically invalid even with permissive `ALLOWED_HOSTS`. Do not blindly replace the public forwarded host with
  nginx `$host`, which may be an internal compute-node hostname. When duplicate proxy values are unavoidable, retain
  the first valid forwarded host and fall back only when the header is absent, for example:

  ```nginx
  map $http_x_forwarded_host $ood_forwarded_host {
      ~^[[:space:]]*(?<first_forwarded_host>[^,[:space:]]+) $first_forwarded_host;
      default $host;
  }
  ```

  Pass the sanitized value explicitly to the application protocol, such as
  `uwsgi_param HTTP_X_FORWARDED_HOST $ood_forwarded_host;`.
- Do not equate an open TCP port or a running nginx process with application readiness. Database migrations, fixture
  loading, and uWSGI startup may finish later. Have the OOD readiness hook poll a small representative API request with
  a timeout and fail clearly if the application process exits or the deadline is reached.
- Validate the generated configuration, not only the templates: run the launcher shell syntax check and server config
  test, syntax-check generated frontend configuration, request a representative API endpoint with the real duplicated
  proxy headers, confirm client navigation retains the OOD prefix, and confirm registered data appears in the catalogue.

### Perl extensions

- Retain the dependency structure of the closest recipe.
- Add `Perl-bundle-CPAN` when build scripts require standard CPAN helpers such as `File::Which`.
- When an application tarball must install CPAN extensions into the same prefix, use a `Bundle` with the application
  as a `Tarball` component, `exts_defaultclass = 'PerlModule'`, and one shared filter:

```python
exts_filter = ("perldoc -lm %(ext_name)s ", "")
```

- Avoid repeating an identical templated `exts_filter` inside every Perl extension entry when one shared filter
  applies to all entries.
- Export bundled Perl modules with the prevailing repository form:

```python
modextrapaths = {'PERL5LIB': 'lib/perl5/site_perl/%(perlver)s/'}
```

- Treat another application's bundled Perl extensions as its private implementation detail. If multiple
  applications require the same modules, prefer standalone Perl easyconfigs used as direct dependencies. Otherwise,
  keep each application's required extensions local rather than relying accidentally on a transitive `PERL5LIB`.
- For installed Perl entry points, add a toolchain-matched `Perl` runtime dependency and set
  `fix_perl_shebang_for`; the dependency alone does not fix a hardcoded `/usr/bin/perl` shebang.
- Inspect the installed files and list the actual Perl entry points explicitly instead of using a broad `bin/*` glob
  when practical.
- Use raw Python strings or escaped backslashes for regular-expression commands in easyconfigs to avoid invalid
  escape-sequence warnings without changing the command.
- Do not replace a precise missing Perl module diagnosis with unrelated build changes.

### Legacy build systems

- Prefer external dependencies over bundled stale copies when the upstream build supports them.
- Add include-directory symlinks only when the source hard-codes a `third/` vendor layout.
- If a patch makes the build system discover external libraries normally, remove redundant symlinks.
- Use `maxparallel = 1` only for demonstrated race conditions or non-parallel-safe builds.
- Do not embed a patch heredoc inside another patch. If a bundled dependency is unpacked only during configure, keep
  the normal extraction workflow and make small, explicit post-extraction edits in the installer script.
- Use direct unified-diff hunks when the target files already exist during EasyBuild's patch step.
- Run through configure when a bundled dependency is unpacked there; `--stop=patch` cannot validate changes made
  after extraction.
- Do not patch compiler warnings unless they are promoted to errors or otherwise stop the build. Start with the first
  actionable error.

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
   Treat a compiled Python extension copied across multiple Python minor versions and compiler generations as a
   compatibility-sensitive migration; build every requested ABI when feasible and exercise a representative data path.
5. Confirm imports, executables, version output, extension sanity, and `pip check` when a build or matching installed
   module is available. Do not require these runtime checks for the robot-only simple copy-forward path.
6. Check every created or modified easyconfig for lines longer than 120 characters:

   ```bash
   awk 'length($0) > 120 {print FILENAME ":" FNR ":" length($0); bad=1} END {exit bad}' path/to/recipe.eb
   ```

7. Run `git diff --check`; normalize patch-file blank context lines if needed, then recompute the patch checksum.
8. Review `git status` and the exact diff.

Treat `--check-contrib` failures caused by a missing local checker such as `pycodestyle` as an environment issue, but do not use that to excuse recipe parse, style, or build failures.

## Commit safely

- Commit and push only when the user explicitly requests it.
- Stage only files belonging to the task.
- Recheck the staged diff with `git diff --cached --check`.
- Use a concise commit message naming the software, version, and toolchain when useful.
- Push the current branch and confirm `HEAD` matches its upstream.
- Never discard unrelated dirty-worktree changes.
