# The riot package API

``` r

library(riot)
```

This vignette describes the public API of
[{riot}](https://tractoverse.github.io/riot/) and the internal design of
the package. It is aimed at developers who want to extend riot or reuse
its internal machinery.

## Public API

The public API is deliberately small: two exported functions.

### `read_bundle()` — import bundles

`read_bundle(file, reference_file = NULL, bundle_data = list())` reads a
tractography file and returns a `bundle` (or a single `streamline` when
the file contains exactly one tract).

``` r

b <- read_bundle(system.file("extdata", "UF_left.vtk", package = "riot"))
#> Number of data points: 38697
#> Number of streamlines: 2042
#> ✔ The bundle stored in '/home/runner/work/_temp/Library/riot/extdata/UF_left.vtk' has been successfully imported.
class(b)
#> [1] "fiber::bundle" "S7_object"
```

The supported extensions are declared by the internal
`supported_formats()`:

``` r

riot:::supported_formats()
#> [1] "vtk" "vtp" "fds" "tck" "trk" "trx" "fib" "dpy"
```

Dispatch is by file extension:

- `.vtk`, `.vtp`, `.fds` — the C++ readers `ReadVTK()`, `ReadVTP()`,
  `ReadFDS()`;
- `.tck` — `read_mrtrix()`;
- `.trk` — `read_trk()`;
- `.trx`, `.fib`, `.dpy` — the DIPY Python bridge through `reticulate`.

For the DIPY-backed formats,
[`read_bundle()`](https://tractoverse.github.io/riot/reference/read_bundle.md)
requires a `reference_file` and initialises the Python bridge with the
internal `check_dipy()` helper.

### `write_bundle()` — export bundles

`write_bundle(x, file, reference_file = NULL)` writes a `bundle` (or a
single `streamline`) to a file, invisibly returning `x` so it can be
used in pipes. The target format is chosen from the file extension of
`file`.

``` r

out <- tempfile(fileext = ".vtk")
write_bundle(b, out)
#> Number of streamlines: 2042
#> Number of arrays: 0
#> ✔ The bundle stored in `b` has been successfully exported to '/tmp/Rtmp8Wwpd0/file1a9e7ff77c42.vtk'.
```

## Internal functions

The internal (non-exported) API is what the C++ layer and the format
readers are built on.

### Conversion between `bundle` objects and the C++ flat list

The C++ functions do not know about S7 objects. They exchange data
through a flat named list with the columns:

- `X`, `Y`, `Z` — point coordinates;
- `PointId` — 1-based position of the point within its streamline;
- `StreamlineId` — 1-based identifier of the streamline.

Additional columns hold per-point scalar attributes. Columns named by
`streamline_cols` are treated as per-streamline attributes instead.

Two internal functions convert between the two representations:

- `flat_list_to_bundle(lst, streamline_cols = character(0))` — converts
  the flat list returned by `ReadVTK()`, `ReadVTP()`, `ReadFDS()`, or
  the TCK/TRK readers into a `streamline` or `bundle`;
- `bundle_to_flat_list(x)` — converts a `bundle` or `streamline` back
  into the flat list expected by `WriteVTK()`, `WriteVTP()`, and
  `WriteFDS()`.

``` r

flat <- riot:::bundle_to_flat_list(b)
names(flat)[1:5]
#> [1] "X"            "Y"            "Z"            "PointId"      "StreamlineId"
```

The conversion is `O(points)` rather than `O(streamlines × points)`: it
uses `split(seq_along(sid), sid)` once to group point indices per
streamline instead of scanning all points for each streamline. This
matters for the TRK reader, which can process hundreds of thousands of
points.

### The MRtrix reader

`read_mrtrix()` handles `.tck` and `.tsf` files. It is split into:

- `read_header()` — parses the text header and derives the data offset,
  endianness, and element size;
- `read_tck()` — reads the binary track data, using `NaN` separators
  between streamlines and an `Inf` terminator at the end;
- `read_tsf()` — reads track-scalar data, matching scalars to
  streamlines.

`read_mrtrix()` merges `.tsf` scalar columns into the tracks when the
timestamps, streamline counts, and point counts match.

### The TRK reader

`read_trk()` reads the binary TrackVis format. It uses
`read_fixed_char_binary()` for null-terminated header strings and
`retrieve_trk_endianness()` to detect big/little endian by checking the
`hdr_size` field (1000 bytes) at offset 996. The bulk of the parsing
happens in the C++ function `ReadTRK()`, which reads the whole file body
in a single `fread()` call and pre-allocates the output vectors after a
counting pass.

## The C++ layer

Four C++ entry points are registered via cpp11 and re-exported as
internal R functions in `R/cpp11.R`:

| R function             | C++ function | Purpose                          |
|------------------------|--------------|----------------------------------|
| `ReadVTK(path)`        | `ReadVTK`    | read legacy `.vtk` polydata      |
| `ReadVTP(path)`        | `ReadVTP`    | read XML `.vtp` polydata         |
| `ReadFDS(path)`        | `ReadFDS`    | read medInria `.fds` XML bundle  |
| `ReadTRK(...)`         | `ReadTRK`    | read TrackVis `.trk` (fast path) |
| `WriteVTK(flat, path)` | `WriteVTK`   | write legacy `.vtk`              |
| `WriteVTP(flat, path)` | `WriteVTP`   | write XML `.vtp`                 |
| `WriteFDS(flat, path)` | `WriteFDS`   | write medInria `.fds`            |

The readers return the flat list described above; the writers accept it.
The VTK classes come from the `rvtk` package, which supplies
pre-compiled VTK headers and libraries at compile time.

The FDS reader/writer use the bundled TinyXML-2 parser (zlib license).
The bundled copy is modified to avoid `printf`/`stdout` in compliance
with the *Writing R Extensions* manual.

## The DIPY Python bridge

The formats `.trx`, `.fib`, and `.dpy` are handled by
[DIPY](https://docs.dipy.org/) through
[{reticulate}](https://rstudio.github.io/reticulate/). The internal
state is kept in two package-level globals (`io_stateful_tractogram`,
`io_streamline`), lazily initialised by `check_dipy()`. Reticulate’s
`py_require("dipy")` ensures that a Python environment with `dipy` is
available, provisioning it automatically when needed (reticulate \>=
1.41).

These code paths are skipped during coverage (`# nocov start`/`end`)
because they require a Python environment.

## Adding a new format

To add a new native format:

1.  Implement a reader that returns the flat list (columns `X`, `Y`,
    `Z`, `PointId`, `StreamlineId`, plus optional attribute columns)
    and, if writable, a writer that accepts it.
2.  Register the extension in `supported_formats()`.
3.  Add a branch in
    [`read_bundle()`](https://tractoverse.github.io/riot/reference/read_bundle.md)
    (and
    [`write_bundle()`](https://tractoverse.github.io/riot/reference/write_bundle.md)
    for writers).
4.  Add tests in `inst/tinytest/`; the TRK and TCK test files show how
    to build minimal binary fixtures without external tools.

The flat-list contract is the seam that keeps new formats decoupled from
the S7 data model: readers only need to produce the list, and
[`flat_list_to_bundle()`](https://tractoverse.github.io/riot/reference/flat_list_to_bundle.md)
takes care of the rest.
