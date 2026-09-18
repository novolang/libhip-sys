# libhip-sys

HIP is AMD's C API for running general-purpose computation on a
graphics card. A program uses it to select a card, allocate memory on
it, copy data to and from it, and wait for the work to finish. It is
documented in
[the HIP runtime API reference](https://rocm.docs.amd.com/projects/HIP/en/latest/doxygen/html/index.html).
This package declares thirteen of that library's entry points to
novo-lang, one declaration each.

Every function here is a declaration of a function in the HIP runtime.
The package contains no logic of its own, and it does nothing without
ROCm installed. The thirteen entry points are the ones a program needs
to move data to a card and back. The section "What is not included"
says what a program cannot do with them alone.

## What it is

A graphics card is a separate computer. It has its own memory, its own
processors, and no access to the memory of the machine it is plugged
into. Running a computation on it means three steps: copy the input into
the card's memory, run the computation, and copy the result back. This
package is the first and third steps.

**ROCm** is AMD's software stack for its cards. **HIP** is the part of
it a host program calls, and the library a program links against is
`libamdhip64.so`. There is no file called `libhip.so`. The package's
name follows the naming convention for a bindings package rather than
the file name of the library.

HIP's C API deliberately mirrors CUDA's. Every function below has a
CUDA counterpart that takes the same arguments in the same order. The
two names differ only in the prefix, `hip` where CUDA writes `cuda`.
That is what lets one piece of source compile for both kinds of card.

**Device memory** is the card's memory. `hip_malloc` reserves a block of
it and writes the address into a slot the caller supplies, where
`malloc` answers the address directly. The address means nothing to the
processor running the program. Only the card can read it, and only
through the runtime.

A **copy** moves bytes between the two memories. `hip_memcpy` takes a
destination address, a source address, a byte count and a direction. The
direction is the fourth argument. Getting it wrong is the common
mistake, because the runtime cannot tell a host address from a device
address by looking at it.

The runtime is **asynchronous** in places. A computation the card is
asked to run returns to the program immediately, and the program finds
out whether it succeeded later. `hip_device_synchronize` waits for
everything outstanding, and `hip_get_last_error` reports the failure of
something that has already returned.

An **error** is an integer. Zero is success. Every other value has a
name, such as `hipErrorOutOfMemory`, and a sentence of description, and
the runtime supplies both.

## Install

```
novo pkg add libhip-sys
```

Adding the package does not install the C library. AMD publishes the
current runtime in its own ROCm repository, which a machine has to add
before apt can see it:

```
sudo apt install rocm-hip-runtime
```

Ubuntu 24.04 carries an older runtime in `universe`, as
`libamdhip64-dev` at ROCm 5.7.1. That package supplies the
`libamdhip64.so` the linker asks for and needs no third-party
repository.

ROCm installs into `/opt/rocm`, where the linker does not look by
default. A program that links against it adds `/opt/rocm/lib` to its own
link flags.

A card and a kernel driver are separate from the runtime. A machine with
the runtime and no supported card links and runs, and the runtime
answers `hipErrorNoDevice`.

## Example

Eight bytes to the card and back:

```novo ignore
use libhip

fn main() [io, ffi]
    let count_slot = ptr.alloc_word()
    if libhip.hip_get_device_count(count_slot) != 0 or ptr.read_word(count_slot) == 0
        println("no HIP device")
        return

    let dev_slot = ptr.alloc_word()
    let host_in = ptr.alloc_word()
    let host_out = ptr.alloc_word()
    ptr.write_word(host_in, 42)

    let rc = libhip.hip_malloc(dev_slot, 8)
    if rc != 0
        println(ptr.read_str(libhip.hip_get_error_string(rc)))
        return
    let dev = ptr.read_word(dev_slot)

    // 1 is host to device, 2 is device to host.
    let _up = libhip.hip_memcpy(dev, host_in, 8, 1)
    let _down = libhip.hip_memcpy(host_out, dev, 8, 2)
    println("the card gave back ${ptr.read_word(host_out)}")

    let _freed = libhip.hip_free(dev)
    ptr.free(dev_slot)
    ptr.free(host_in)
    ptr.free(host_out)
```

The example is fenced as an illustration rather than a compiled block,
because compiling it would link against ROCm, which this repository does
not ship.

## What the package contains

| Module | Contents |
| --- | --- |
| `libhip` | Every entry point: the device calls, the memory calls, the copy, the barrier and the error calls. |

The four groups:

| Group | Entry points | What it does |
| --- | --- | --- |
| Device | 3 | Counts the cards, chooses one for the calling thread, and says which one is chosen. |
| Memory | 5 | Allocates and frees device memory, copies between the two memories, clears a block, and reports how much memory is left. |
| Synchronisation | 1 | Waits for everything the card has been asked to do. |
| Errors | 4 | Reads the last error, peeks at it, and turns a code into a name and a sentence. |

## The rules a user needs

1. **A device address is an `Int`, and the program may not read it.**
   `hip_malloc` answers an address in the card's memory. Passing it to
   anything but a HIP call reads memory that does not belong to the
   program.
2. **An out-parameter is the address of a caller-owned slot.**
   `hip_malloc`, `hip_get_device_count`, `hip_get_device` and
   `hip_mem_get_info` each write into a slot the caller supplies.
   `ptr.alloc_word` returns the address of an eight-byte slot, which is
   wide enough for all of them, and `ptr.free` releases it.
3. **Zero is success and every other code is a failure.** The name of
   the code comes from `hip_get_error_name` and the sentence from
   `hip_get_error_string`. Both answer the address of a C string the
   runtime owns. Read it with `ptr.read_str` and do not free it.
4. **The copy direction is an argument, and it is not checked.** 0 is
   host to host, 1 is host to device, 2 is device to host, 3 is device
   to device, and 4 asks the runtime to work it out from the two
   addresses. The runtime cannot tell the two kinds of address apart by
   inspection, so 1 with the arguments the wrong way round is a fault
   rather than a message.
5. **A byte count is a byte count.** Neither side checks the length of
   a buffer, so a copy that names more bytes than the source holds reads
   past it.
6. **The device belongs to the thread, not to the program.**
   `hip_set_device` chooses the card for the calling thread only. Every
   allocation and every copy after it belongs to that card, and an
   address from one card is not valid on another.
7. **A failure may arrive later than the call that caused it.** A card
   runs work asynchronously, so `hip_device_synchronize` and
   `hip_get_last_error` are where a fault surfaces.
   `hip_get_last_error` clears the error and `hip_peek_at_last_error`
   leaves it.
8. **Compare an error code against zero, not against a number.** HIP's
   error names match CUDA's and many of the values do. The enumeration
   is still HIP's own, so a program that hard-codes a number is relying
   on a coincidence.

## What is not included

- **Running anything.** A HIP computation is a *kernel*, written in HIP
  C++ and compiled by `hipcc`. There is no novo-lang syntax for one and
  no way to write one through this package. A program that needs a
  kernel compiles it with `hipcc` and calls it through its own foreign
  declaration. This package moves the data the kernel reads.
- **Streams and events.** `hipStream_t` and `hipEvent_t` are opaque
  handles the runtime passes by value. The calls that take them are the
  asynchronous copy, the asynchronous launch and the timing
  measurements. They are left out until there is a consumer that needs
  overlapping work.
- **The linear algebra library.** `hipblasSgemm` and its neighbours are
  in `libhipblas`, a second library. A binding package declares exactly
  one library, so hipBLAS belongs in a package of its own.
- **Device properties.** `hipGetDeviceProperties` fills in a structure
  of several hundred bytes. The novo-lang foreign function interface
  passes integers, floats and strings, so a structure has to be laid out
  by hand, and this one changes shape between ROCm versions.
  `hip_mem_get_info` answers the one property most programs ask for.
  The wavefront width, which a kernel must know and which is 32 on some
  AMD architectures and 64 on others, is in that structure.
- **Managed and pinned host memory.** `hipMallocManaged` and
  `hipHostMalloc` change how the two memories relate to each other, and
  a program that wants them wants the streams as well.

## Related packages

`libcuda-sys` is the same shape for NVIDIA cards. Its library is the
CUDA runtime, and its entry points are the same thirteen operations
under different names.

`std.gpu` in the standard library reaches a graphics card through
wgpu-native, and runs on AMD, NVIDIA, Intel and Apple hardware alike.
It is the right choice for a program that wants a graphics card rather
than an AMD one, and it installs nothing by hand.

There is no novo-lang replacement for this package and none is planned.
A graphics card is a piece of hardware whose published C API is the
vendor's own library. There is no format to reimplement.

## Tests

`tests/libhip_tests.nv` holds nine tests over the thirteen entry
points:

```
novo test tests/libhip_tests.nv
```

The suite links against the HIP runtime, so it needs ROCm installed.
Without it the link fails, naming `-lamdhip64`. `novo pkg build`
type-checks the declarations and needs nothing installed.

The tests do not need a card. Each one accepts both answers. On a
machine with a card the successful path is asserted to be
self-consistent, and on a machine without one the failure is asserted to
carry a name. The round trip through device memory writes eight bytes
up, reads them back, clears them and reads them back again.

The suite has never been run against an AMD card. Every test is written
to pass on a machine without one, and that is the only outcome anyone
has observed.

## Licence

Apache-2.0. See [LICENSE](LICENSE).

ROCm is distributed under the MIT licence, and installing it is the
reader's own step.
