# Changelog

All notable changes to libhip-sys are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-09-15

The first release: thirteen entry points of the HIP runtime library,
one `@ffi` declaration each, and no logic.

### Added

- `libhip` — the whole surface, in four groups.
  - The device: `hip_get_device_count`, `hip_set_device` and
    `hip_get_device`.
  - Memory: `hip_malloc`, `hip_free`, `hip_memcpy`, `hip_memset` and
    `hip_mem_get_info`.
  - Synchronisation: `hip_device_synchronize`.
  - Errors: `hip_get_last_error`, `hip_peek_at_last_error`,
    `hip_get_error_string` and `hip_get_error_name`.
- `tests/libhip_tests.nv` — nine tests over the signatures. Each accepts
  both a machine with a card and a machine without one.

### Where the surface came from

The eight runtime calls that the inference programs in this project make
are `hipMalloc`, `hipFree`, `hipMemcpy`, `hipMemset`,
`hipGetDeviceCount`, `hipSetDevice`, `hipGetLastError` and
`hipDeviceSynchronize`. Five more are here because an error code cannot
be reported without a name and a description, because the read half of
`hipSetDevice` belongs beside the write half, and because whether the
weights fit is the question asked before the first upload.

The package is the counterpart of libcuda-sys, entry point for entry
point, in the same order.

### The wrapped library

`wraps = "libamdhip64"`. That is the file the loader opens; there is no
`libhip.so`. The package's name follows the naming convention for a
bindings package rather than the file name of the library it binds.

### Not a `0.0.x` interface release

An interface release is the shape whose every `pub fn` body is a
`todo()`. Every `pub fn` here is an `@ffi` declaration with no body, so
`novo pkg publish` reads the package as a release with bodies and
refuses a `0.0.x` version for it. The first release of a bindings
package is therefore `0.1.0`.

### Unverified

No AMD card and no ROCm installation were available when this package
was written. The declarations were checked against the HIP runtime API
reference, and the test suite has never been linked. Treat the whole
package as unmeasured until someone runs it on real hardware.

### Named as missing

**A way to run a kernel.** This package moves data to a card and back.
The computation itself is written in HIP C++ and compiled by `hipcc`,
and there is no novo-lang path to one.

**The wavefront width.** A kernel that reduces across one wavefront must
know whether it is 32 or 64 lanes wide, and the answer is a field of
`hipGetDeviceProperties`, which needs a structure passed by value. Until
that is expressible, a program reads the width some other way.
