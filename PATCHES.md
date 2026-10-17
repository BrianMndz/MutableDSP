# Patch Log

## Policy

- Build-only: build-system changes that do not alter compiled DSP behavior.
- Behavior-neutral: source changes intended to preserve output.
- Behavior-changing: intentional changes to DSP behavior or public semantics.

Each module's files are first imported unchanged (see `UPSTREAM.md`); every
later change to an imported file is listed here under its module.

## Baseline import

Status: no modifications to imported DSP source.
Upstream commits: Eurorack `f4cfa769...`, stmlib `e3bd7c9c...`.

## Build files

`CMakeLists.txt` is not imported; upstream builds with Makefiles for the
STM32. These notes record why it is set up as it is.

### C++14 language requirement

- Origin: CloudsDSP commit `165ebde`, where it replaced a public `cxx_std_17`.
- Classification: build-only.
- Problem: a public `cxx_std_17` requirement forces every consumer to compile
  as C++17, but the Bela board's Clang 3.9 cannot build Clouds as C++17. That
  compiler rejects the array template argument in `granular_processor.h`
  (`src_filter_1x_2_45`) in C++1z mode as a pointer to a subobject, while
  accepting it in C++14 mode.
- Change: every library requires `cxx_std_14` publicly and compiles its own
  sources as C++14 (`CXX_STANDARD 14`, no extensions), as the board does.
  Consumers may still compile as C++17.
- Verification:
    - All imported sources pass a Clang 3.9 `-std=c++14` syntax check against
      Debian 9 armhf headers (CloudsDSP).
    - CloudsEngine's four quality renders were byte-identical before and after
      CloudsDSP's change; stereo/16-bit/32-kHz remains
      `9b9c09ee3a612eab0226b0e4214f7f10448d7d1be1b57d21f93d9566b06298e3`.
    - With Apple Clang `-O3` on arm64, all ten object files are
      byte-identical between this repository's libraries, CloudsDSP's
      `clouds_dsp` compiled as C++14, and `clouds_dsp` compiled as C++17.

### Shared stmlib target

- Classification: build-only.
- Change: stmlib's sources form their own library, `MutableDSP::stmlib`, which
  owns the include directory and `TEST=1`; each module's library links it.
  CloudsDSP compiled them into `clouds_dsp`.
- Reason: with a second module, two libraries each compiling stmlib would
  define its symbols twice.

## clouds

The patches below were made in CloudsDSP (`github.com/BrianMndz/CloudsDSP`)
and carried over unchanged; each names its CloudsDSP commit. The verification
renders are CloudsEngine's.

### Explicit initialization for portable object storage

- CloudsDSP commit: `1ffb365`.
- Classification: behavior-neutral.
- Problem: the upstream global processor relied on static zero-initialization.
- Changed files: `clouds/dsp/granular_processor.cc`,
  `clouds/dsp/grain.h`, `clouds/dsp/granular_sample_player.h`, and
  `clouds/dsp/fx/reverb.h`.
- Initialized state: processor state, feedback/tail buffers, reverb decay,
  grain quality, and grain phasor.
- Verification:
    - Density 0.75 ordinary and patterned automatic storage:
      `9b9c09ee3a612eab0226b0e4214f7f10448d7d1be1b57d21f93d9566b06298e3`.
    - Density 0.25 zero-filled and patterned automatic storage:
      `57959d7f661ad53da6c3830508cecec10e3be902c00933c4f1eb9ec1d970e140`.

### Explicit initialization for the stretch-mode correlator

- CloudsDSP commit: `f69c1f4`.
- Classification: behavior-neutral.
- Problem: `Correlator::Init()` leaves `size_` uninitialized, and
  `EvaluateSomeCandidates()` loops `(size_ >> 2) + 16` times. The module's
  static processor starts with zero; a processor in automatic storage starts
  with garbage, and the first `Prepare()` in stretch mode then loops for
  minutes.
- Changed file: `clouds/dsp/correlator.cc`.
- Initialized state: `increment_`, `size_`, `candidate_`, `best_score_`, and
  `trace_`. `done_` is already true after `Init()`, so the search evaluates no
  candidates until `StartSearch()`.
- Verification:
    - Each mode in each quality, rendered directly and after recording in
      granular mode (32 cases, with triggers, freeze, gate and swept knobs),
      gives byte-identical output from engine storage pre-filled with `0x00`,
      `0xA5` and `0x5A`.
    - CloudsEngine's four granular quality renders are unchanged; stereo/
      16-bit/32-kHz remains
      `9b9c09ee3a612eab0226b0e4214f7f10448d7d1be1b57d21f93d9566b06298e3`.

### Restored window start in stretch mode

- CloudsDSP commit: `f69c1f4`.
- Classification: behavior-changing relative to the imported upstream commit;
  restores the behavior of the module's released firmware.
- Problem: `Window::Start()` no longer sets `done_ = false`, so after
  `Window::Init()` every window stays done and stretch mode outputs silence.
  The released firmware (upstream `2d0a5dc`, "Clouds: latest firmware") set
  `done_ = false` twice. In March 2023 two separate cleanups each removed one
  of the duplicates (`fbb53ba` the second, `0e3756f` the first), and the merge
  `d1d8839` kept both removals.
- Changed file: `clouds/dsp/window.h` (restores the line as in `2d0a5dc`).
- Verification:
    - Stretch mode's wet output peaks at 3000-4700 for an 8000-peak test tone
      in every quality, against 16 before the change.
    - CloudsEngine's `clouds_engine_playback_mode_check` fails without the
      line and passes with it.
    - CloudsEngine's four granular quality renders are unchanged.

### Defined shift in the stretch-mode correlator

- CloudsDSP commit: `f69c1f4`.
- Classification: behavior-changing on hosts; restores the module's result.
- Problem: `Correlator::EvaluateNextCandidate()` computes
  `destination[i + 1] >> (32 - offset_bits)`, a shift by 32 whenever
  `offset_bits` is 0. C++ leaves that undefined. The module's Cortex-M4
  yields 0, so the term drops out; AArch64 reduces the amount modulo 32 and
  ORs in the whole word. On the Mac, Debug and Release builds rendered stretch
  mode differently in three of the four qualities.
- Changed file: `clouds/dsp/correlator.cc` (the term is skipped when
  `offset_bits` is 0).
- Verification:
    - CloudsEngine's CLI renders all four modes in all four qualities
      byte-identically in Debug and Release builds; UBSan reports nothing in
      stretch mode.
    - Granular, looping delay and spectral renders are unchanged.

### Known, not changed

- `GranularProcessor::Process()` interpolates `lut_xfade_in`/`lut_xfade_out`
  at index 16 when DRY/WET is fully wet, reading one element past each
  17-entry table and multiplying it by a fraction of exactly 0 (AddressSanitizer
  reports a global-buffer-overflow read). The neighbouring constants are
  finite, so the output is unaffected.
- UBSan also reports signed left shifts in `looping_sample_player.h` (the
  `<< 4` that extracts the fractional position), float-to-`uint16_t`
  conversions out of range in `FrameTransformation::SetPhases()` (phase
  arithmetic meant to wrap), and pointer arithmetic wrapping in `shy_fft.h`.
  All compile to the same wrapping result on the module and on the hosts;
  Debug and Release renders agree.
