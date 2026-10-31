# CMake build for the PoissonRecon command-line tools.
#
# The bundled copies of zlib, libpng and libjpeg are not used: the libraries
# come from vcpkg (vcpkg.json). The shim headers in cmake/shims map the
# include paths used on Windows ("PNG/png.h", "JPEG/jpeglib.h") to them.
#
# The tools that need a network connection (PoissonReconClient and
# PoissonReconServer, which require Boost.Asio) are not built.

cmake_minimum_required(VERSION 3.21)
project(PoissonRecon LANGUAGES C CXX)

option(POISSON_FAST_MATH "Use -ffast-math (as the original Makefile)" ON)
option(POISSON_NATIVE_ARCH "Optimize for the build machine (-march=native)" OFF)
option(POISSON_USE_OPENMP "Use OpenMP if available" ON)
option(POISSON_USE_MIMALLOC "Use mimalloc for C++ allocations (operator new/delete)" ON)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
# Executables (and the OpenMP runtime on Windows) in one directory.
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()
# Debug information is only needed for profiling: configure with
# -DCMAKE_CXX_FLAGS_RELEASE="-O3 -funroll-loops -DRELEASE -g".
set(CMAKE_CXX_FLAGS_RELEASE "-O3 -funroll-loops -DRELEASE" CACHE STRING "")

find_package(ZLIB REQUIRED)
find_package(PNG REQUIRED)
find_package(JPEG REQUIRED)
find_package(Threads REQUIRED)
if(POISSON_USE_MIMALLOC)
    find_package(mimalloc CONFIG REQUIRED)
    if(TARGET mimalloc-static)
        set(POISSON_MIMALLOC_TARGET mimalloc-static)
    else()
        set(POISSON_MIMALLOC_TARGET mimalloc)
    endif()
endif()
if(POISSON_USE_OPENMP)
    if(WIN32)
        # Prefer LLVM's own libomp.lib/libomp.dll (redistributable) over the one
        # shipped with Visual Studio, which imports libomp140.x86_64.dll.
        get_filename_component(llvm_bin_dir "${CMAKE_CXX_COMPILER}" DIRECTORY)
        if(EXISTS "${llvm_bin_dir}/../lib/libomp.lib")
            get_filename_component(llvm_omp_lib "${llvm_bin_dir}/../lib/libomp.lib" ABSOLUTE)
            set(OpenMP_libomp_LIBRARY "${llvm_omp_lib}" CACHE FILEPATH "" FORCE)
        endif()
    endif()
    find_package(OpenMP COMPONENTS CXX)
endif()

# ---- Flags shared by all targets -------------------------------------------

add_library(poisson_options INTERFACE)
target_include_directories(poisson_options INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/Src
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/shims)
target_compile_options(poisson_options INTERFACE
    -Wno-deprecated -Wno-invalid-offsetof -Wno-dangling-else)
if(POISSON_FAST_MATH)
    target_compile_options(poisson_options INTERFACE -ffast-math)
endif()
if(POISSON_NATIVE_ARCH)
    target_compile_options(poisson_options INTERFACE -march=native)
endif()
if(WIN32)
    target_compile_definitions(poisson_options INTERFACE
        _CRT_SECURE_NO_WARNINGS _CRT_NONSTDC_NO_WARNINGS NOMINMAX WIN32_LEAN_AND_MEAN _USE_MATH_DEFINES)
endif()
target_link_libraries(poisson_options INTERFACE
    Threads::Threads ZLIB::ZLIB PNG::PNG JPEG::JPEG)
if(OpenMP_CXX_FOUND)
    target_link_libraries(poisson_options INTERFACE OpenMP::OpenMP_CXX)
endif()

# ---- Executables -----------------------------------------------------------

set(poisson_tools
    PoissonRecon SSDRecon SurfaceTrimmer PointInterpolant EDTInHeat
    ImageStitching AdaptiveTreeVisualization ChunkPLY PointsToDisks
    ScaleNormals Reconstruction.example)

foreach(tool IN LISTS poisson_tools)
    string(REPLACE "." "_" target ${tool})
    add_executable(${target} Src/${tool}.cpp)
    set_target_properties(${target} PROPERTIES OUTPUT_NAME ${tool})
    target_link_libraries(${target} PRIVATE poisson_options)
    if(POISSON_USE_MIMALLOC)
        target_sources(${target} PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/cmake/mimalloc_override.cc)
        target_link_libraries(${target} PRIVATE ${POISSON_MIMALLOC_TARGET})
    endif()
    list(APPEND poisson_targets ${target})
endforeach()

install(TARGETS ${poisson_targets} RUNTIME DESTINATION bin)

# On Windows the LLVM OpenMP runtime is a DLL that has to ship with the tools.
if(WIN32 AND OpenMP_CXX_FOUND)
    get_filename_component(compiler_dir "${CMAKE_CXX_COMPILER}" DIRECTORY)
    find_file(POISSON_LIBOMP_DLL libomp.dll HINTS "${compiler_dir}" NO_DEFAULT_PATH)
    if(POISSON_LIBOMP_DLL)
        add_custom_target(poisson_copy_libomp ALL
            COMMAND ${CMAKE_COMMAND} -E copy_if_different "${POISSON_LIBOMP_DLL}" "${CMAKE_BINARY_DIR}")
        add_dependencies(PoissonRecon poisson_copy_libomp)
        install(FILES "${POISSON_LIBOMP_DLL}" DESTINATION bin)
    else()
        message(WARNING "libomp.dll not found next to the compiler; copy it manually")
    endif()
endif()
