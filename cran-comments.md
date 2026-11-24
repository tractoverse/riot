## Resubmission

This is a resubmission. riot 1.1.0 was archived from CRAN; this release
(v2.0.0) is a major update that modernises the package:

* The S7 data model (classes `streamline` and `bundle`) has moved to the
  [fiber](https://CRAN.R-project.org/package=fiber) package (now a CRAN
  dependency) and riot re-exports it.
* `read_tractogram()` / `write_tractogram()` were renamed to
  `read_bundle()` / `write_bundle()`.
* VTK support is now delegated to the
  [rvtk](https://CRAN.R-project.org/package=rvtk) package (now on CRAN),
  which supplies pre-compiled VTK libraries at install time; riot no longer
  bundles VTK sources. riot requires rvtk (>= 0.2.0), which introduced
  per-module link flag filtering.
* Two new vignettes document the user-facing API and the internal design.

## Test environments

**Local:**
* macOS (arm64), R 4.6.2

**Continuous integration via GitHub Actions (`R-CMD-check.yaml`):**
* macOS latest, R release, Homebrew VTK
* Windows latest, R release
* Ubuntu latest, R devel
* Ubuntu latest, R release
* Ubuntu latest, R oldrel-1
* Ubuntu latest, R release, VTK built from source

"Installed size" notes are expected: the package links pre-compiled static
VTK libraries supplied by `rvtk` at install time; these are not
re-distributed with the tarball.

## R CMD check results

* Local: `R CMD check --as-cran` — 0 errors, 0 warnings, 0 notes (one HTML
  note is an artifact of Tidy not being available locally and does not occur
  on CRAN or CI).
* GitHub Actions `R-CMD-check`: 0 errors, 0 warnings across the matrix above
  (`--no-manual`, `--compact-vignettes=gs+qpdf`).
* R-hub Windows (R-devel): 0 errors, 0 warnings, 0 notes. The rvtk 0.2.0
  dependency is now fully rolled out on CRAN (previous r-patched and
  r-release-linux check rows were stale during the rollout).
* R-hub sanitizer legs (`clang-asan`, `clang-ubsan`) were run on the merged
  UB fix but stalled in R-hub's linux-container queue; the fix was instead
  verified locally with UBSan (see below).

## Notes for CRAN reviewers

* The only vendored/bundled third-party source is TinyXML-2 (zlib license,
  license header retained, modified only to avoid `printf`/`stdout` per the
  *Writing R Extensions* manual). VTK is linked (not vendored) via the
  `rvtk` package under its BSD-3-Clause license. Provenance records are in
  `LICENSE.md` and `inst/brand/NOTICE`.
* `cran-comments.md` previously noted that win-builder and R-hub checks for
  the Windows and Linux release flavors were not completed before submission:
  the rvtk dependency has a newer release (0.2.0) whose CRAN binary rollout
  was in progress at submission time, and early win-builder/R-hub runs with
  those flavors hit installation issues caused by the previous rvtk release
  on CRAN (0.1.3), not by riot. The rvtk rollout is now complete (all CRAN
  check flavors OK) and the R-hub Windows leg passes with the `local::.`
  workaround in place.
* A null-pointer issue in the VTK reader (string point-data arrays) found by
  the clang-asan/clang-ubsan legs was fixed. The fix was verified locally on
  macOS (arm64): building the pre-fix snapshot with `-fsanitize=undefined`
  reproduces the exact failure (`runtime error: member call on null pointer
  of type 'vtkAbstractArray'` at `bundleReaders.cpp:221`), while the
  post-fix snapshot (PR #20) passes the same string-array reproducer and the
  full test suite (128/128) under UBSan with no reports.
* CRAN's `r-devel-linux-x86_64-fedora-clang` flavor compiles packages with
  clang, which targets libc++, while the pre-built VTK static archives rvtk
  supplies are built with GCC/libstdc++. To remove that ABI risk, rvtk now
  ships a clang/libc++-built variant of the Linux x86_64 archive and its
  `configure` selects it automatically when the build compiler targets
  libc++. riot's CI runs a dedicated leg that installs rvtk and riot with a
  clang + libc++ toolchain and asserts the libc++ archive is the one that was
  linked (see the `rvtk-check` workflow).
* DIPY-backed formats (`.trx`, `.fib`, `.dpy`) require an optional Python
  environment with the `dipy` package, provisioned automatically by
  `reticulate` on first use; they are covered by tests only when such an
  environment is available.
