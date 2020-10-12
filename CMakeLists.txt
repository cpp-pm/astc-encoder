cmake_minimum_required(VERSION 3.0)

include(cmake/HunterGate.cmake)
HunterGate(
    URL "https://github.com/cpp-pm/hunter/archive/v0.23.275.tar.gz"
    SHA1 "0952a54dc983ecbea09dfe8c53ee65d1e0e91232"
)

project(astc-encoder VERSION 2.0)

# Introduce variables:
# * CMAKE_INSTALL_LIBDIR
# * CMAKE_INSTALL_BINDIR
# * CMAKE_INSTALL_INCLUDEDIR
include(GNUInstallDirs)

option(BUILD_SHARED_LIBS "Build shared libraries" FALSE)
option(BUILD_APPS "Build astcenc and astcdec" FALSE)
set(VEC "sse2" CACHE STRING "Architecture to optimize for, must be one of (neon|sse2|sse4.2|avx2)")

if(MSVC AND BUILD_SHARED_LIBS)
  set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
endif()

if(VEC STREQUAL "neon")
  add_compile_definitions("ASTCENC_SSE=0;ASTCENC_AVX=0;ASTCENC_POPCNT=0;ASTCENC_VECALIGN=16")
elseif(VEC STREQUAL "sse2")
  add_compile_definitions("ASTCENC_SSE=20;ASTCENC_AVX=0;ASTCENC_POPCNT=0;ASTCENC_VECALIGN=16")
  if(MSVC)
    if(CMAKE_SIZEOF_VOID_P EQUAL 4)
      add_compile_options("/arch:SSE2") # Only needed on 32-bit/x86 machine targets.
    endif()
  else()
    add_compile_options("-mfpmath=sse;-msse2")
  endif()
elseif(VEC STREQUAL "sse4.2")
  add_compile_definitions("ASTCENC_SSE=42;ASTCENC_AVX=0;ASTCENC_POPCNT=1;ASTCENC_VECALIGN=16")
  if(NOT MSVC)
    add_compile_options("-mfpmath=sse;-msse4.2;-mpopcnt")
  endif()
elseif(VEC STREQUAL "avx2")
  add_compile_definitions("ASTCENC_SSE=42;ASTCENC_AVX=2;ASTCENC_POPCNT=1;ASTCENC_VECALIGN=32")
  if(MSVC)
    add_compile_options("/arch:AVX2")
  else()
    add_compile_options("-mfpmath=sse;-mavx2;-mpopcnt")
  endif()
else()
  message(FATAL_ERROR "Invalid VEC string provided!")
endif()

set(ASTC_ENCODER_SRC_LIST
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_averages_and_directions.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_block_sizes2.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_color_quantize.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_color_unquantize.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_compress_symbolic.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_compute_variance.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_decompress_symbolic.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_encoding_choice_error.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_entry.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_find_best_partitioning.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_ideal_endpoints_and_weights.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_image.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_integer_sequence.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_kmeans_partitioning.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_mathlib.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_mathlib_softfloat.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_partition_tables.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_percentile_tables.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_pick_best_endpoint_format.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_platform_isa_detection.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_quantization.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_symbolic_physical.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_weight_align.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_weight_quant_xfer_tables.cpp
)
set(ASTC_ENCODER_PUBLIC_HDR_LIST
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc.h
)
set(ASTC_ENCODER_PRIVATE_HDR_LIST
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_internal.h
  ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenc_mathlib.h
  ${ASTC_ENCODER_PUBLIC_HDR_LIST}
)

add_library(astcenc ${ASTC_ENCODER_SRC_LIST} ${ASTC_ENCODER_PRIVATE_HDR_LIST})
set_target_properties(astcenc PROPERTIES OUTPUT_NAME "astcenc-${VEC}")
set(TARGETS astcenc)

target_include_directories(astcenc INTERFACE
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/Source>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}>
)

if(BUILD_APPS)
  hunter_add_package(stb)
  find_package(stb CONFIG REQUIRED)

  hunter_add_package(tinyexr)
  find_package(tinyexr CONFIG REQUIRED)

  set(ASTC_ENCODER_CLI_SRC_LIST
    ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenccli_error_metrics.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenccli_image.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenccli_image_load_store.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenccli_platform_dependents.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenccli_toplevel.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenccli_toplevel_help.cpp
  )
  set(ASTC_ENCODER_CLI_PRIVATE_HDR_LIST
    ${CMAKE_CURRENT_SOURCE_DIR}/Source/astcenccli_internal.h
    ${ASTC_ENCODER_PRIVATE_HDR_LIST}
  )

  add_executable(astcenccli ${ASTC_ENCODER_CLI_SRC_LIST} ${ASTC_ENCODER_CLI_PRIVATE_HDR_LIST})
  target_link_libraries(astcenccli PRIVATE astcenc stb::stb tinyexr::tinyexr)
  set_target_properties(astcenccli PROPERTIES OUTPUT_NAME "astcenc-${VEC}")
  list(APPEND TARGETS astcenccli)

  add_executable(astcdeccli ${ASTC_ENCODER_CLI_SRC_LIST} ${ASTC_ENCODER_CLI_PRIVATE_HDR_LIST})
  target_link_libraries(astcdeccli PRIVATE astcenc stb::stb tinyexr::tinyexr)
  target_compile_definitions(astcdeccli PRIVATE "ASTCENC_DECOMPRESS_ONLY")
  set_target_properties(astcdeccli PROPERTIES OUTPUT_NAME "astcdec-${VEC}")
  list(APPEND TARGETS astcdeccli)
endif()

set(GENERATED_DIR "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(VERSION_CONFIG "${GENERATED_DIR}/${PROJECT_NAME}ConfigVersion.cmake")
set(PROJECT_CONFIG "${GENERATED_DIR}/${PROJECT_NAME}Config.cmake")

set(CONFIG_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}")

set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")
set(NAMESPACE "${PROJECT_NAME}::")

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    "${VERSION_CONFIG}" COMPATIBILITY ExactVersion
)

configure_package_config_file(
    "cmake/Config.cmake.in"
    "${PROJECT_CONFIG}"
    INSTALL_DESTINATION "${CONFIG_INSTALL_DIR}"
)

install(
    TARGETS ${TARGETS}
    EXPORT "${TARGETS_EXPORT_NAME}"
    INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
    RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
    LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
)

install(
    FILES ${ASTC_ENCODER_PUBLIC_HDR_LIST}
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}
)

install(
    FILES "${PROJECT_CONFIG}" "${VERSION_CONFIG}"
    DESTINATION "${CONFIG_INSTALL_DIR}"
)

install(
    EXPORT "${TARGETS_EXPORT_NAME}"
    NAMESPACE "${NAMESPACE}"
    DESTINATION "${CONFIG_INSTALL_DIR}"
)
