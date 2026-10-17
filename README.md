# MutableDSP

Port of Mutable Instruments DSP to use in different platforms.

A curated fork of the [Mutable Instruments Eurorack](https://github.com/pichenettes/eurorack)
code: the DSP of selected modules and the one `stmlib` they share, in the
original layout, built as portable C++14 libraries for desktop, Bela and Daisy.
Hardware drivers and UI code are left out, apart from the few headers the DSP
includes (such as `clouds/drivers/debug_pin.h`).

## Modules

| Module | CMake target | Sources |
|---|---|---|
| stmlib | `MutableDSP::stmlib` | `stmlib/` (shared by every module) |
| Clouds | `MutableDSP::clouds` | `clouds/dsp`, `clouds/resources` |

## Using it

```cmake
add_subdirectory(Modules/MutableDSP EXCLUDE_FROM_ALL)
target_link_libraries(my_engine PUBLIC MutableDSP::clouds)
```

Include paths start at the repository root, as upstream:
`#include "clouds/dsp/granular_processor.h"`.

- `TEST=1` is a public definition of every target: it replaces ARM-only inline
  assembly with portable C. Code compiled outside CMake (for example on the
  Bela board) must define it too.
- `stmlib::Random` is process-global. Programs that run several modules share
  one random stream; seed it explicitly for reproducible output.

## Provenance and changes

- `UPSTREAM.md`: the upstream commits and the import policy.
- `UPSTREAM_FILES.txt`: every imported file.
- `PATCHES.md`: every change to an imported file, and the build notes.
- `LICENSE`: MIT, copyright Emilie Gillet.
