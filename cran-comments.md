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
  bundles VTK sources.
* Two new vignettes document the user-facing API and the internal design.

## Test environments

**Local:**
* macOS (arm64), R 4.6.2

**Continuous integration via GitHub Actions (`R-CMD-check.yaml`):**
* macOS latest, R release
* Windows latest, R release
* Ubuntu latest, R devel
* Ubuntu latest, R release
* Ubuntu latest, R oldrel-1

**Win-builder:**
* [win-builder](https://win-builder.r-project.org/) R release
* [win-builder](https://win-builder.r-project.org/) R-devel

**R-Hub (`rhub::check_for_cran()`):**
* 36 platforms (all standard R-hub platforms except `rchk`): Linux x86_64
  (R devel, release, oldrel, patched, fedora-clang, fedora-gcc), macOS
  (arm64 and x86_64, R release and devel), Windows x86_64 (R devel,
  release, oldrel), plus the extra `gcc13`/`gcc14` configurations.

## R CMD check results

Per platform run of `R CMD check --as-cran`:

0 errors | 0 warnings | 0 notes (local macOS and most CI platforms)

Remaining notes (only on some R-hub / win-builder configurations):

* *Possibly misspelled words in DESCRIPTION*: none expected — checked with
  `utils::aspell` and the `spelling` package.
* *Installed size is 5.8Mb* (some platforms): the package links pre-compiled
  static VTK libraries provided by `rvtk` at install time; these are not
  re-distributed with the tarball.

## Notes for CRAN reviewers

* All vendored/bundled third-party code is licensed for this use: the only
  vendored source is TinyXML-2 (zlib license, header retained, modified only
  to avoid `printf`/`stdout` per the *Writing R Extensions* manual). VTK is
  linked (not vendored) via the `rvtk` package under its BSD-3-Clause
  license. Provenance records are in `LICENSE.md` and `inst/brand/NOTICE`.
* DIPY-backed formats (`.trx`, `.fib`, `.dpy`) require an optional Python
  environment with the `dipy` package, provisioned automatically by
  `reticulate` on first use; they are covered by tests only when such an
  environment is available.
