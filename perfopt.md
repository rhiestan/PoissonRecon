# Performance optimizations (`perfopt` branch)

This fork of [PoissonRecon](https://github.com/mkazhdan/PoissonRecon) adds a
CMake build, some performance optimizations and one correctness fix. The work
lives on the `perfopt` branch (the default branch of this fork); `master`
tracks the upstream repository, so it can be synchronized with it.

Timings below were measured on a 32-thread Windows machine (Clang 22, default
OpenMP settings, `--depth 10`) on two test point clouds: "kermit" (843k
points) and "sceaux" (6M points).

## Build and CI

- CMake, Ninja, Clang and vcpkg (zlib, libpng, libjpeg-turbo, mimalloc) build
  `PoissonRecon`, `SSDRecon`, `SurfaceTrimmer`, `PointInterpolant`,
  `EDTInHeat`, `ImageStitching`, `AdaptiveTreeVisualization`, `ChunkPLY`,
  `PointsToDisks`, `ScaleNormals` and `Reconstruction.example`. Presets:
  `windows-clang`, `linux-clang`, `macos-clang`. The distributed
  client/server tools (they need Boost.Asio) are not built.
- The vendored image libraries are replaced by the vcpkg ones through small
  shim headers. `PNG.inl` uses `png_get_image_width` instead of a private
  struct member, which current libpng does not allow.
- GitHub Actions builds on Windows, Linux and macOS and starts the tool.
  Pushing a tag packages all tools (with the OpenMP runtime) and publishes a
  GitHub release.

## Optimizations

- **`NestedVector::resize`** takes its mutex only when the vector really has
  to grow. While the point data fields were built, all threads contended on it
  (over 20 CPU seconds in the lock).
- **PLY reading**: items are read without the locking `fread` and copied
  directly when the type in the file equals the type in memory.
- **mimalloc** for all C++ allocations.
- Together, compared with the original code (both with the fix below): kermit
  3.4 s to 2.6 s (about 23% faster), sceaux 18.0 s to 16.5 s (about 8%).
  Single-threaded (`--parallel 2`), sceaux went from 7.8 s to 4.85 s with
  byte-identical output (measured before the fix below).

## Correctness fix: bounding box under `-ffast-math`

The build uses `-ffast-math` (as the original Makefile), which implies finite
math only, so `std::numeric_limits<Real>::infinity()` is undefined behavior
(Clang warns with `-Wnan-infinity-disabled`). The initial values of the
extents (`PointExtent`, `FEMTree` bounding box, `L2Energy`) now use
`max()`/`lowest()`. Before, Clang treated the infinite initial values as 0,
which gave wrong extents (for kermit a minimum z of 0 instead of 2.18) and
therefore an offset, oversized reconstruction cube.

**This changes the results.** The cube now fits the data, so at the same
`--depth` the result can be more detailed and slower: for sceaux the output
grows from 8.5 MB to 63 MB, and the run time of the original code from 7.4 s
to 18.0 s.
