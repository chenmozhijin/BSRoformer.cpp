cmake_minimum_required(VERSION 3.17)
project(bs_roformer_cpp VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

#================================================
# Build Options
#================================================

option(GGML_CUDA "Enable CUDA backend" ON)
option(BSR_BUILD_TESTS "Build tests" OFF)
option(BSR_BUILD_CLI "Build CLI application" ON)

#================================================
# Dependencies - GGML (Flexible Resolution)
#================================================

# Strategy: Allow ggml to be shared across multiple projects
# 1. Check if ggml target already exists (e.g., from parent project like whisper.cpp)
# 2. If not, try to find ggml via CMAKE_PREFIX_PATH or GGML_DIR
# 3. If not found, use local ggml (submodule or sibling directory)

if(NOT TARGET ggml)
    # Try to find ggml package first (for system-wide or parent project installation)
    find_package(ggml QUIET CONFIG)
    
    if(NOT ggml_FOUND)
        # ggml not found as package, look for source directory
        # Priority 1: GGML_DIR variable (explicitly set by user or parent project)
        # Priority 2: Submodule in ggml/
        # Priority 3: Sibling directory ../ggml
        
        if(DEFINED GGML_DIR)
            set(GGML_PATH "${GGML_DIR}")
            message(STATUS "Using GGML from GGML_DIR: ${GGML_PATH}")
        elseif(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/ggml/CMakeLists.txt")
            set(GGML_PATH "${CMAKE_CURRENT_SOURCE_DIR}/ggml")
            message(STATUS "Using GGML from submodule: ${GGML_PATH}")
        elseif(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/../ggml/CMakeLists.txt")
            set(GGML_PATH "${CMAKE_CURRENT_SOURCE_DIR}/../ggml")
            message(STATUS "Using GGML from sibling directory: ${GGML_PATH}")
        else()
            message(FATAL_ERROR 
                "ggml not found. Please either:\n"
                "  1. Add ggml as submodule: git submodule add https://github.com/ggerganov/ggml.git\n"
                "  2. Clone ggml to parent directory: cd .. && git clone https://github.com/ggerganov/ggml.git\n"
                "  3. Set GGML_DIR to point to ggml source: cmake -DGGML_DIR=/path/to/ggml\n"
                "  4. Let parent project provide ggml target"
            )
        endif()
        
        # Add ggml as subdirectory
        add_subdirectory(${GGML_PATH} ggml EXCLUDE_FROM_ALL)
    else()
        message(STATUS "Using GGML from installed package")
    endif()
else()
    message(STATUS "Using GGML target from parent project")
endif()

#================================================
# Core Library
#================================================

set(BSR_SOURCES
    src/model.cpp
    src/utils.cpp
    src/inference.cpp
)

add_library(bs_roformer STATIC ${BSR_SOURCES})

target_include_directories(bs_roformer PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
target_include_directories(bs_roformer PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/src
)

target_link_libraries(bs_roformer PUBLIC ggml)
if(GGML_CUDA AND TARGET ggml-cuda)
    # Fix for CI: Link against CUDA stubs if the driver is not present
    # This prevents errors like "libcuda.so.1 needed by ... not found" during linking
    find_package(CUDAToolkit REQUIRED)
    if(TARGET CUDA::cuda_driver)
        target_link_libraries(bs_roformer PUBLIC CUDA::cuda_driver)
        message(STATUS "Linked against CUDA driver stubs")
    endif()
endif()

# Compiler options
if(MSVC)
    target_compile_options(bs_roformer PRIVATE /W3 /utf-8)
else()
    target_compile_options(bs_roformer PRIVATE -Wall -Wextra)
endif()

# OpenMP support
find_package(OpenMP)
if(OpenMP_CXX_FOUND)
    target_link_libraries(bs_roformer PUBLIC OpenMP::OpenMP_CXX)
    target_compile_definitions(bs_roformer PUBLIC USE_OPENMP)
    message(STATUS "OpenMP: ENABLED")
else()
    message(STATUS "OpenMP: NOT FOUND")
endif()

#================================================
# DLL Copy Helper (Windows)
#================================================

function(bsr_copy_ggml_runtime_dlls target_name)
    if(NOT WIN32)
        return()
    endif()

    if(CMAKE_VERSION VERSION_GREATER_EQUAL "3.21")
        add_custom_command(TARGET ${target_name} POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
                $<TARGET_RUNTIME_DLLS:${target_name}>
                $<TARGET_FILE_DIR:${target_name}>
            COMMAND_EXPAND_LISTS
            COMMENT "Copying runtime DLLs for ${target_name}"
        )
        return()
    endif()

    set(ggml_dll_targets ggml ggml-base ggml-cpu ggml-cuda ggml-vulkan)
    set(runtime_dll_files)

    foreach(dll_target IN LISTS ggml_dll_targets)
        if(NOT TARGET ${dll_target})
            continue()
        endif()

        get_target_property(dll_target_type ${dll_target} TYPE)
        if(dll_target_type STREQUAL "SHARED_LIBRARY" OR dll_target_type STREQUAL "MODULE_LIBRARY")
            list(APPEND runtime_dll_files $<TARGET_FILE:${dll_target}>)
        endif()
    endforeach()

    if(runtime_dll_files)
        add_custom_command(TARGET ${target_name} POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
                ${runtime_dll_files}
                $<TARGET_FILE_DIR:${target_name}>
            COMMAND_EXPAND_LISTS
            COMMENT "Copying GGML runtime DLLs for ${target_name}"
        )
    endif()
endfunction()

#================================================
# CLI Application
#================================================

if(BSR_BUILD_CLI)
    # audio.cpp implements AudioFile utilities (using dr_wav)
    add_executable(bs_roformer-cli 
        cli/main.cpp 
        src/audio.cpp
    )
    target_link_libraries(bs_roformer-cli PRIVATE bs_roformer)
    target_include_directories(bs_roformer-cli PRIVATE 
        src 
        third_party
    )
    
    if(MSVC)
        target_compile_options(bs_roformer-cli PRIVATE /W3 /utf-8)
    endif()
    # argv and fopen take the process code page; UTF-8 lets paths outside the ANSI code page through
    if(WIN32)
        target_sources(bs_roformer-cli PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/cli/utf8.manifest)
    endif()

    bsr_copy_ggml_runtime_dlls(bs_roformer-cli)
endif()

#================================================
# Tests (Optional)
#================================================

if(BSR_BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
    message(STATUS "Tests: ENABLED")
else()
    message(STATUS "Tests: DISABLED (use -DBSR_BUILD_TESTS=ON to enable)")
endif()
