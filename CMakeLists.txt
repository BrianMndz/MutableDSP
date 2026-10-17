cmake_minimum_required(VERSION 3.24)

project(MutableDSP VERSION 0.1.0 LANGUAGES CXX)

# One static library per module, all linking the single shared stmlib target,
# so two modules in one program never bring two copies of stmlib's symbols.
# Consumers that add this directory with EXCLUDE_FROM_ALL build only the
# modules they link.

# Applies the settings every library in this repository shares.
function(mutabledsp_library target)
    # C++14 is enough for this code and lets consumers build with older
    # compilers such as the Bela board's Clang 3.9. A public C++17 requirement
    # would force every consumer, including C++14 ones, to compile as C++17.
    # CXX_STANDARD also compiles the library's own sources as C++14 on every
    # host, as the board does.
    target_compile_features(${target} PUBLIC cxx_std_14)
    set_target_properties(${target} PROPERTIES
        CXX_STANDARD 14
        CXX_EXTENSIONS OFF
    )
endfunction()

# --- stmlib ---------------------------------------------------------------------

add_library(mutabledsp_stmlib STATIC
    stmlib/dsp/atan.cc
    stmlib/dsp/units.cc
    stmlib/utils/random.cc
)

add_library(MutableDSP::stmlib ALIAS mutabledsp_stmlib)

mutabledsp_library(mutabledsp_stmlib)

# Repository root: lets consumers include "stmlib/..." and "<module>/...".
target_include_directories(mutabledsp_stmlib PUBLIC
    ${CMAKE_CURRENT_LIST_DIR}
)

# TEST=1 must be PUBLIC: stmlib and module headers branch on it (it replaces
# ARM-only inline assembly with portable C), so every translation unit that
# includes them needs the same definition as these libraries.
target_compile_definitions(mutabledsp_stmlib PUBLIC
    TEST=1
)

# --- clouds ---------------------------------------------------------------------

add_library(mutabledsp_clouds STATIC
    clouds/resources.cc
    clouds/dsp/correlator.cc
    clouds/dsp/granular_processor.cc
    clouds/dsp/mu_law.cc
    clouds/dsp/pvoc/frame_transformation.cc
    clouds/dsp/pvoc/phase_vocoder.cc
    clouds/dsp/pvoc/stft.cc
)

add_library(MutableDSP::clouds ALIAS mutabledsp_clouds)

mutabledsp_library(mutabledsp_clouds)

target_link_libraries(mutabledsp_clouds PUBLIC MutableDSP::stmlib)
