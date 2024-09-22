#----------------------------------------
# 1. Preparing build:
#    rm -rf build
#    mkdir build && cd build
#
# 2. Select compiler and build it
# - Compile with Clang:
#    CC="clang" CXX="clang++" cmake ..
#    make
#
# - Compile with Intel C++ Compiler:
#    CC="icc" CXX="icpc" cmake ..
#    make
#
# - Compile with GCC Compiler:
#    cmake ..
#    make
#
# Use -DCMAKE_BUILD_TYPE=COMPAT_GLIBC
# to backward compatibility with oldest version of glibc 2.11
#
#----------------------------------------

cmake_minimum_required(VERSION 3.1)
project(metamod CXX)

option(DEBUG "Build with debug information." OFF)
option(USE_STATIC_LIBSTDC "Enables static linking libstdc++." OFF)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Avoid -rdynamic -fPIC options
set(CMAKE_SHARED_LIBRARY_CXX_FLAGS "")
set(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS "")

# Common compile and link flags
set(COMMON_COMPILE_FLAGS "-Wall -fno-exceptions -fno-builtin -Wno-unknown-pragmas -ffunction-sections -fdata-sections")
set(COMMON_LINK_FLAGS "")

if (DEBUG)
    set(COMMON_COMPILE_FLAGS "${COMMON_COMPILE_FLAGS} -g3 -O0 -ggdb")
else()
    set(COMMON_COMPILE_FLAGS "${COMMON_COMPILE_FLAGS} -g0 -O3 -fno-stack-protector")
    set(COMMON_LINK_FLAGS "${COMMON_LINK_FLAGS} -s")
endif()

# Check Intel C++ compiler
if ("$ENV{CXX}" MATCHES "icpc")
    set(COMMON_COMPILE_FLAGS "${COMMON_COMPILE_FLAGS} \
        -fasm-blocks -Qoption,cpp,--treat_func_as_string_literal_cpp\
        -inline-forceinline -no-ansi-alias")

    set(COMMON_LINK_FLAGS "${COMMON_LINK_FLAGS} -static-intel -no-intel-extensions")

    if (NOT DEBUG)
        set(COMMON_COMPILE_FLAGS "${COMMON_COMPILE_FLAGS} -ipo")
        set(COMMON_LINK_FLAGS "${COMMON_LINK_FLAGS} -ipo -diag-disable=11003")
    endif()
else()
    # Produce code optimized for the most common IA32/AMD64/EM64T processors.
    # As new processors are deployed in the marketplace, the behavior of this option will change.
    set(COMMON_COMPILE_FLAGS "${COMMON_COMPILE_FLAGS} \
        -mtune=generic -msse3\
        -fno-sized-deallocation -Wno-invalid-offsetof -Wno-ignored-attributes\
        -Wno-write-strings -Wno-strict-aliasing -fno-plt")

    # Check Clang compiler
    if (NOT "$ENV{CXX}" MATCHES "clang")
        set(COMMON_COMPILE_FLAGS "${COMMON_COMPILE_FLAGS} -Wno-maybe-uninitialized -Wno-unused-but-set-variable")

        # GCC >= 8.3
        if (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 8.0)
            set(COMMON_COMPILE_FLAGS "${COMMON_COMPILE_FLAGS} -Wno-stringop-truncation -Wno-format-truncation -Wno-class-memaccess")
        endif()
    endif()
endif()

# GCC >= 8.3
if (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 8.0)
    set(COMMON_COMPILE_FLAGS "${COMMON_COMPILE_FLAGS} -fcf-protection=none")
endif()

if (NOT DEBUG)
    set(COMMON_LINK_FLAGS "${COMMON_LINK_FLAGS} \
        -Wl,-gc-sections -Wl,--version-script=\"${PROJECT_SOURCE_DIR}/../version_script.lds\"")
endif()

if (CMAKE_BUILD_TYPE MATCHES COMPAT_GLIBC)
    set(COMMON_LINK_FLAGS "${COMMON_LINK_FLAGS} -Wl,--wrap=stat64")
endif()

set(PROJECT_SRC_DIR
    "${PROJECT_SOURCE_DIR}"
    "${PROJECT_SOURCE_DIR}/src"
    "${PROJECT_SOURCE_DIR}/include/engine"
    "${PROJECT_SOURCE_DIR}/include/common"
    "${PROJECT_SOURCE_DIR}/include/dlls"
    "${PROJECT_SOURCE_DIR}/include/game_shared"
    "${PROJECT_SOURCE_DIR}/include/pm_shared"
    "${PROJECT_SOURCE_DIR}/include/public"
)

set(PROJECT_PUBLIC_DIR
    "${PROJECT_SOURCE_DIR}/include/public"
    "${PROJECT_SOURCE_DIR}/include/public/metamod"
)

# i386 specific flags
set(I386_COMPILE_FLAGS "${COMMON_COMPILE_FLAGS} -m32 -U_FORTIFY_SOURCE")
set(I386_LINK_FLAGS "${COMMON_LINK_FLAGS} -m32")

# i686 specific flags
set(I686_COMPILE_FLAGS "${COMMON_COMPILE_FLAGS} -m32 -march=i686 -U_FORTIFY_SOURCE")
set(I686_LINK_FLAGS "${COMMON_LINK_FLAGS} -m32")

# Function to create target
function(create_target target_name compile_flags link_flags output_name)
    add_library(${target_name} SHARED ${appversion.sh})

    if (NOT TARGET appversion)
        add_custom_target(appversion DEPENDS COMMAND "${PROJECT_SOURCE_DIR}/version/appversion.sh" "${PROJECT_SOURCE_DIR}/..")
    endif()

    add_dependencies(${target_name} appversion)

    target_include_directories(${target_name} PRIVATE
        ${PROJECT_SRC_DIR}
        ${PROJECT_PUBLIC_DIR}
    )

    target_compile_definitions(${target_name} PRIVATE
        _LINUX
        LINUX
        NDEBUG
        METAMOD_CORE
        _GLIBCXX_USE_CXX11_ABI=0
        _stricmp=strcasecmp
        _strnicmp=strncasecmp
        _strdup=strdup
        _unlink=unlink
        _write=write
        _close=close
        _getcwd=getcwd
        _vsnprintf=vsnprintf
        _vsnwprintf=vswprintf
        _snprintf=snprintf
    )

    target_sources(${target_name} PRIVATE
        "src/api_info.cpp"
        "src/callback_jit.cpp"
        "src/commands_meta.cpp"
        "src/conf_meta.cpp"
        "src/dllapi.cpp"
        "src/engine_api.cpp"
        "src/game_support.cpp"
        "src/h_export.cpp"
        "src/osdep_linkent_linux.cpp"
        "src/log_meta.cpp"
        "src/mdebug.cpp"
        "src/mem_utils.cpp"
        "src/meta_rehlds_api.cpp"
        "src/metamod.cpp"
        "src/mextdll.cpp"
        "src/mlist.cpp"
        "src/mplayer.cpp"
        "src/mplugin.cpp"
        "src/mreg.cpp"
        "src/mutil.cpp"
        "src/precompiled.cpp"
        "src/public_amalgamation.cpp"
        "src/reg_support.cpp"
        "src/sdk_util.cpp"
        "src/studioapi.cpp"
        "src/sys_module.cpp"
        "src/utils.cpp"
        "src/glibc_compat.cpp"
    )

    target_link_libraries(${target_name} PRIVATE
        dl
        pthread
        $<$<CONFIG:COMPAT_GLIBC>:libc-2.11.1.so>
    )

    if (USE_STATIC_LIBSTDC)
        target_compile_definitions(${target_name} PRIVATE BUILD_STATIC_LIBSTDC)
        set(link_flags "${link_flags} -static-libgcc -static-libstdc++")
    endif()

    if (CMAKE_BUILD_TYPE MATCHES COMPAT_GLIBC)
        set(link_flags "${link_flags} \
            -Wl,-rpath,'$ORIGIN/.' \
            -L${PROJECT_SOURCE_DIR}/lib/linux32")
    endif()

    set_target_properties(${target_name} PROPERTIES
        OUTPUT_NAME ${output_name}
        PREFIX ""
        COMPILE_FLAGS ${compile_flags}
        LINK_FLAGS ${link_flags}
        POSITION_INDEPENDENT_CODE ON
    )
endfunction()

# Create i386 target
create_target(metamod_i386 "${I386_COMPILE_FLAGS}" "${I386_LINK_FLAGS}" "metamod_i386")

# Create i686 target
create_target(metamod_i686 "${I686_COMPILE_FLAGS}" "${I686_LINK_FLAGS}" "metamod_i686")