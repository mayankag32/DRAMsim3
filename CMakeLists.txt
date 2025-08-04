cmake_minimum_required(VERSION 3.0.0)
project(dramsim3)

set(default_build_type "Release")
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to '${default_build_type}' as none was specified.")
  set(CMAKE_BUILD_TYPE "${default_build_type}" CACHE
      STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
    "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

# MSVC-specific compiler options
if(MSVC)
  add_compile_options(/W4 /wd4100 /wd4201 /wd4505 /permissive-)
  add_compile_definitions(_CRT_SECURE_NO_WARNINGS)
  set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)  # Auto-export DLL symbols
endif()

add_library(inih INTERFACE)
target_include_directories(inih INTERFACE ext/headers)

add_library(format INTERFACE)
target_include_directories(format INTERFACE ext/fmt/include)
target_compile_definitions(format INTERFACE FMT_HEADER_ONLY=1)

# argparsing library, only used in main program not the library
add_library(args INTERFACE)
target_include_directories(args INTERFACE ext/headers)

add_library(json INTERFACE)
target_include_directories(json INTERFACE ext/headers)

# Main DRAMSim Lib
add_library(dramsim3 SHARED
    src/bankstate.cc
    src/channel_state.cc
    src/command_queue.cc
    src/common.cc
    src/configuration.cc
    src/controller.cc
    src/dram_system.cc
    src/hmc.cc
    src/refresh.cc
    src/simple_stats.cc
    src/timing.cc
    src/memory_system.cc
)

if (THERMAL)
    # Windows-specific library setup
    if(MSVC)
        # Adjust SuperLU library name for Windows
        find_library(SUPERLU
            NAMES superlu_mt_OPENMP
            HINTS ${PROJECT_SOURCE_DIR}/ext/SuperLU_MT_3.1/lib/
        )
        
        # Find BLAS libraries for Windows
        find_package(OpenMP REQUIRED)
        set(BLAS_LIBRARIES "blas.lib;lapack.lib" CACHE STRING "BLAS libraries")
    else()
        find_package(BLAS REQUIRED)
        find_package(OpenMP REQUIRED)
        find_library(SUPERLU
            NAME superlu_mt_OPENMP libsuperlu_mt_OPENMP
            HINTS ${PROJECT_SOURCE_DIR}/ext/SuperLU_MT_3.1/lib/
        )
    endif()

    target_link_libraries(dramsim3
        PRIVATE ${SUPERLU} ${BLAS_LIBRARIES} OpenMP::OpenMP_C
    )
    target_sources(dramsim3
        PRIVATE src/thermal.cc src/sp_ienv.c src/thermal_solver.c
    )
    
    # MSVC-compatible defines
    target_compile_definitions(dramsim3 PRIVATE THERMAL _LONGINT Add_)

    add_executable(thermalreplay src/thermal_replay.cc)
    target_link_libraries(thermalreplay dramsim3 inih)
    target_compile_definitions(thermalreplay PRIVATE THERMAL _LONGINT Add_)
endif (THERMAL)

if (CMD_TRACE)
    target_compile_definitions(dramsim3 PRIVATE CMD_TRACE)
endif (CMD_TRACE)

if (ADDR_TRACE)
    target_compile_definitions(dramsim3 PRIVATE ADDR_TRACE)
endif (ADDR_TRACE)

target_include_directories(dramsim3 INTERFACE src)
if(NOT MSVC)
    target_compile_options(dramsim3 PRIVATE -Wall)
endif()
target_link_libraries(dramsim3 PRIVATE inih format)
set_target_properties(dramsim3 PROPERTIES
    LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}
    RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}  # For DLL on Windows
    CXX_STANDARD 11
    CXX_STANDARD_REQUIRED YES
    CXX_EXTENSIONS NO
)

# trace CPU, .etc
add_executable(dramsim3main src/main.cc src/cpu.cc)
target_link_libraries(dramsim3main PRIVATE dramsim3 args)
set_target_properties(dramsim3main PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}  # For EXE on Windows
    CXX_STANDARD 11
    CXX_STANDARD_REQUIRED YES
    CXX_EXTENSIONS NO
)

# Unit testing
add_library(Catch INTERFACE)
target_include_directories(Catch INTERFACE ext/headers)

add_executable(dramsim3test EXCLUDE_FROM_ALL
    tests/test_config.cc
    tests/test_dramsys.cc
    tests/test_hmcsys.cc
)
target_link_libraries(dramsim3test Catch dramsim3)
target_include_directories(dramsim3test PRIVATE src/)
set_target_properties(dramsim3test PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}  # For EXE on Windows
)

# Use generator expression for cross-platform executable path
add_custom_command(
    TARGET dramsim3test POST_BUILD
    COMMAND $<TARGET_FILE:dramsim3test>
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
)