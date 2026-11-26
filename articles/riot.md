# Importing and exporting bundles with riot

``` r

library(riot)
```

## What riot does

[{riot}](https://tractoverse.github.io/riot/) (R Inputs/Outputs for
Tractography) reads and writes tractography data: bundles of streamlines
(3-D polylines) reconstructed by fibre-tracking algorithms applied to
diffusion MRI data. It is the input/output layer of the
[{tractoverse}](https://github.com/tractoverse/tractoverse) package
family.

The central functions are:

- [`read_bundle()`](https://tractoverse.github.io/riot/reference/read_bundle.md)
  — import a bundle from a file in one of the supported formats;
- [`write_bundle()`](https://tractoverse.github.io/riot/reference/write_bundle.md)
  — export a bundle to a file.

Bundles live in R as the S7 classes `streamline` and `bundle` defined by
the [{fiber}](https://tractoverse.github.io/fiber/) package:

- a **streamline** holds one fibre tract: `@points` is an $`n \times 3`$
  numeric matrix with columns `X`, `Y`, `Z`; `@point_data` is a named
  list of per-point scalar attributes (e.g. fractional anisotropy
  sampled at every point); `@streamline_data` is a named list of
  per-streamline scalars (e.g. a tract-level weight);
- a **bundle** is an ordered collection of streamlines in
  `@streamlines`, plus optional bundle-level metadata in `@bundle_data`.

[`read_bundle()`](https://tractoverse.github.io/riot/reference/read_bundle.md)
returns a single `streamline` when the file contains exactly one tract
and a `bundle` otherwise.

## Supported formats

[`read_bundle()`](https://tractoverse.github.io/riot/reference/read_bundle.md)
reads:

| Format | Extension | Notes |
|----|----|----|
| [VTK](https://vtk.org) legacy & XML | `.vtk`, `.vtp` | native, compiled in C++ |
| [medInria](https://med.inria.fr) | `.fds` | native, compiled in C++ |
| [MRtrix](https://mrtrix.readthedocs.io/en/latest/getting_started/image_data.html) | `.tck`, `.tsf` | native, pure R |
| [TrackVis](https://trackvis.org/docs/?subsect=fileformat) | `.trk` | native, hybrid R/C++ |
| TrackVis TRX | `.trx` | via ‘DIPY’ (Python) |
| DIPY bundles | `.dpy` | via ‘DIPY’ (Python) |
| FIB | `.fib` | via ‘DIPY’ (Python) |

[`write_bundle()`](https://tractoverse.github.io/riot/reference/write_bundle.md)
writes `.vtk`, `.vtp`, and `.fds` natively, and `.trk`, `.tck`, `.trx`,
`.fib`, and `.dpy` through ‘DIPY’.

The formats read or written through [DIPY](https://docs.dipy.org/)
require a Python environment with the `dipy` package and a reference
image (see below).

## Example data

riot ships a few small example bundles in `inst/extdata`:

``` r

extdata <- system.file("extdata", package = "riot")
list.files(extdata)
#> [1] "AF_left.tck" "CCMid.trk"   "UF_left"     "UF_left.fds" "UF_left.vtk"
#> [6] "UF_left.vtp"
```

## Reading a bundle

The most direct way to import a bundle is
[`read_bundle()`](https://tractoverse.github.io/riot/reference/read_bundle.md)
with the path to a file:

``` r

uf_left_vtk <- read_bundle(system.file(
  "extdata",
  "UF_left.vtk",
  package = "riot"
))
#> Number of data points: 38697
#> Number of streamlines: 2042
#> ✔ The bundle stored in '/home/runner/work/_temp/Library/riot/extdata/UF_left.vtk' has been successfully imported.
uf_left_vtk
#> 
#> ── Object of class `fiber::bundle()` with 2042 streamlines and [9–33] points per streamline. ──
#> 
#> • Point attributes: none
#> • Streamline attributes: none
#> • Bundle attributes: none
#> 
```

The result is a `bundle` (or a `streamline` if the file contains a
single tract). Its `@streamlines` slot holds one `streamline` per tract:

``` r

length(uf_left_vtk)
#> [1] 2042
nrow(uf_left_vtk@streamlines[[1L]]@points)
#> [1] 27
```

You can read any of the supported formats the same way:

``` r

uf_left_vtp <- read_bundle(system.file(
  "extdata",
  "UF_left.vtp",
  package = "riot"
))
#> Number of data points: 38697
#> Number of streamlines: 2042
#> ✔ The bundle stored in '/home/runner/work/_temp/Library/riot/extdata/UF_left.vtp' has been successfully imported.
uf_left_fds <- read_bundle(system.file(
  "extdata",
  "UF_left.fds",
  package = "riot"
))
#> Number of data points: 38697
#> Number of streamlines: 2042
#> ✔ The bundle stored in '/home/runner/work/_temp/Library/riot/extdata/UF_left.fds' has been successfully imported.
af_left_tck <- read_bundle(system.file(
  "extdata",
  "AF_left.tck",
  package = "riot"
))
#> ✔ The bundle stored in '/home/runner/work/_temp/Library/riot/extdata/AF_left.tck' has been successfully imported.
cc_mid_trk <- read_bundle(system.file("extdata", "CCMid.trk", package = "riot"))
#> ✔ The bundle stored in '/home/runner/work/_temp/Library/riot/extdata/CCMid.trk' has been successfully imported.
```

### Attaching bundle-level metadata at import time

The `bundle_data` argument lets you attach a named list of metadata
directly at import time. It is stored in the `@bundle_data` slot of the
returned `bundle`:

``` r

uf_left_meta <- read_bundle(
  system.file("extdata", "UF_left.vtk", package = "riot"),
  bundle_data = list(subject = "sub-01", hemisphere = "left")
)
#> Number of data points: 38697
#> Number of streamlines: 2042
#> ✔ The bundle stored in '/home/runner/work/_temp/Library/riot/extdata/UF_left.vtk' has been successfully imported.
uf_left_meta@bundle_data
#> $subject
#> [1] "sub-01"
#> 
#> $hemisphere
#> [1] "left"
```

When the file contains a single streamline, `bundle_data` is ignored
with a warning (a `streamline` has no `@bundle_data` slot).

### Formats that need a reference image

Formats read through DIPY (`.trx`, `.fib`, `.dpy`) do not store the
image space they live in. You must pass a reference image to position
the bundle correctly:

``` r

read_bundle("path/to/bundle.trx", reference_file = "path/to/reference.nii.gz")
```

This requires `reticulate` and a Python environment with the `dipy`
package. On first use, reticulate provisions `dipy` automatically in an
ephemeral virtual environment (reticulate \>= 1.41).

## Transforming a bundle

A `bundle` is a plain list of `streamline` objects, so you can use the
usual list tools or the geometry helpers provided by **fiber**. For
example, to keep only the first 10 streamlines:

``` r

small <- uf_left_vtk
small@streamlines <- small@streamlines[1:10]
length(small)
#> [1] 10
```

## Writing a bundle

[`write_bundle()`](https://tractoverse.github.io/riot/reference/write_bundle.md)
exports a `bundle` (or a single `streamline`) to a file. The format is
chosen from the file extension:

``` r

out <- file.path(tempdir(), "UF_left_copy.vtk")
write_bundle(uf_left_vtk, out)
#> Number of streamlines: 2042
#> Number of arrays: 0
#> ✔ The bundle stored in `uf_left_vtk` has been successfully exported to '/tmp/RtmpDdaXJa/UF_left_copy.vtk'.

# Round-trip: read it back
uf_left_copy <- read_bundle(out)
#> Number of data points: 38697
#> Number of streamlines: 2042
#> ✔ The bundle stored in '/tmp/RtmpDdaXJa/UF_left_copy.vtk' has been successfully imported.
all.equal(uf_left_vtk, uf_left_copy)
#> [1] TRUE
```

The same works for `.vtp` and `.fds`. Writing the DIPY-backed formats
(`.trk`, `.tck`, `.trx`, `.fib`, `.dpy`) requires a reference image:

``` r

write_bundle(uf_left_vtk, "path/to/bundle.trk",
             reference_file = "path/to/reference.nii.gz")
```

Some formats keep additional attributes attached to streamlines or
points (e.g. `.trk` and `.trx`); others (e.g. `.tck` and `.dpy`) do not.

## Where to go next

- The [riot API
  vignette](https://tractoverse.github.io/riot/articles/riot-api.md)
  describes the package’s internal functions and the design of the C++
  layer.
- The [{fiber}](https://tractoverse.github.io/fiber/) package provides
  the `streamline` and `bundle` classes.
- The other tractoverse packages ({rtists} for visualization, {bot} for
  optimal transport of bundles, {toad} for atlases) build on the same
  data model.
