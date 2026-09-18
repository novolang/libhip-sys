# Changelog

All notable changes to libhip-sys are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-09-18

The documentation and comments in plain prose; no declaration changed.

### Corrected against the HIP runtime API reference

- `hipGetErrorString` and `hipGetErrorName` answer the address of a C
  string. Every other entry point answers a `hipError_t`.
- `hipGetDeviceCount` answers `hipErrorNoDevice` on a machine the
  runtime finds no card on.
- `hipMemcpyKind` defines `hipMemcpyHostToHost` as 0, so the
  directions are 0 to 4. Any other value answers
  `hipErrorInvalidMemcpyDirection`.
- `hipDeviceSynchronize` blocks until the device has finished and
  reports a failure when one of those tasks failed. It does not
  promise the last one.
- `hipMalloc` writes the address into a slot the caller supplies,
  where `malloc` answers the address directly.
- The HIP runtime is reachable from this distribution's archive as
  `libamdhip64-dev`, as well as from AMD's own ROCm repository.

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
- `tests/libhip_tests.nv` — nine tests over the thirteen entry points.
  Each accepts both a machine with a card and a machine without one.

### Where the surface came from

The inference programs in this repository make eight runtime calls:
`hipMalloc`, `hipFree`, `hipMemcpy`, `hipMemset`, `hipGetDeviceCount`,
`hipSetDevice`, `hipGetLastError` and `hipDeviceSynchronize`. Five more
are here for three reasons. An error code cannot be reported without a
name and a description. The read half of `hipSetDevice` belongs beside
the write half. Whether the weights fit is the question asked before the
first upload.

The package is the counterpart of libcuda-sys, entry point for entry
point, in the same order.

### The wrapped library

`wraps = "libamdhip64"`. That is the file the loader opens, and there is
no `libhip.so`. The package's name follows the naming convention for a
bindings package rather than the file name of the library it binds.

### Unverified

No AMD card and no ROCm installation were available for this release.
The declarations are checked against the HIP runtime API reference, and
the test suite has never been linked. The package is unmeasured until
someone runs it on an AMD card.

### Named as missing

**A way to run a kernel.** This package moves data to a card and back.
The computation itself is written in HIP C++ and compiled by `hipcc`,
and there is no novo-lang path to one.

**The wavefront width.** A kernel that reduces across one wavefront must
know whether it is 32 or 64 lanes wide. The answer is a field of
`hipGetDeviceProperties`, which fills in a structure the caller
supplies. Until that structure is expressible, a program reads the width
some other way.
