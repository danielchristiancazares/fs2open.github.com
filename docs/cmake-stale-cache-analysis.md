# FSO CMake Stale Cache State Analysis

## Executive Summary

The FSO build system has **several CMake patterns that cause stale cache state**, requiring `rm -rf build` for correct incremental builds. The most critical issues are:

1. **Critical bug**: Shader compilation has a typo causing broken dependency tracking
2. **Version headers**: No dependency tracking on `version.cmake` changes
3. **SIMD detection**: TRY_RUN results cached across CPU changes
4. **Prebuilt version**: Cached internally without change detection on source file

---

## 🔴 CRITICAL: Shader Compilation Bug

**File**: `code/shaders.cmake:48`

```cmake
foreach (_shader ${SHADERS})    # Loop variable is _shader
    ...
    add_custom_command(OUTPUT "${_spirvFile}"
        COMMAND glslc "${_shader}" ...
        MAIN_DEPENDENCY "${shader}"   # BUG: Missing underscore!
    )
```

**Impact**: The `MAIN_DEPENDENCY` references undefined variable `${shader}` instead of `${_shader}`. This means:
- CMake doesn't know shaders are dependencies
- Modifying `.vert` or `.frag` files **won't trigger rebuilds**
- Only clean builds produce correct results

---

## 🔴 CRITICAL: Shader DEPFILE Only for Ninja

**File**: `code/shaders.cmake:39-42`

```cmake
set(DEPFILE_PARAM)
if (CMAKE_GENERATOR STREQUAL "Ninja")
    set(DEPFILE_PARAM DEPFILE "${_depFile}")
endif ()
```

**Impact**: The glslc compiler generates `.d` dependency files for shader includes (e.g., `gamma.sdr`), but only Ninja can use them. Visual Studio, Xcode, and Unix Makefiles won't rebuild when shader include files change.

---

## 🟠 HIGH: Version Headers Missing Dependency Tracking

**File**: `cmake/generateHeaders.cmake:6-7`

```cmake
CONFIGURE_FILE(${CMAKE_CURRENT_LIST_DIR}/project.h.in ${GENERATED_SOURCE_DIR}/project.h)
CONFIGURE_FILE(${CMAKE_CURRENT_LIST_DIR}/fred_rc.h.in ${GENERATED_SOURCE_DIR}/fred_rc.h)
```

**File**: `cmake/version.cmake:4-7`

```cmake
if (EXISTS "${PROJECT_SOURCE_DIR}/version_override.cmake")
    include("${PROJECT_SOURCE_DIR}/version_override.cmake")
endif()
```

**Issues**:
1. `configure_file()` only regenerates when the `.in` template timestamp changes
2. If `version.cmake` or `version_override.cmake` changes, headers won't regenerate
3. No `CMAKE_CONFIGURE_DEPENDS` property set for these files

**Impact**: Version number changes in `version.cmake` won't propagate to built binaries without cache clear.

---

## 🟠 HIGH: SIMD Detection Cached via TRY_RUN

**File**: `cmake/util.cmake:191-207`

```cmake
function(detect_simd_instructions _out_var)
    TRY_RUN(RUN_RESULT COMPILE_RESULT "${CMAKE_BINARY_DIR}/temp"
            "${CMAKE_SOURCE_DIR}/cmake/cpufeatures.cpp"
            RUN_OUTPUT_VARIABLE FEATURE_OUTPUT)
```

**File**: `cmake/toolchain.cmake:8,19`

```cmake
set(FORCED_NATIVE_SIMD_INSTRUCTIONS ON CACHE BOOL "...")
set(FORCED_SIMD_INSTRUCTIONS "" CACHE STRING "...")
```

**Issues**:
1. `TRY_RUN()` results are automatically cached by CMake
2. If build directory is moved to different machine, stale CPU detection persists
3. `FORCED_NATIVE_SIMD_INSTRUCTIONS` is a CACHE BOOL that persists

**Impact**: Binary may be compiled with wrong SIMD flags for target CPU.

---

## 🟠 HIGH: Prebuilt Library Version Caching

**File**: `lib/prebuilt.cmake:95`

```cmake
set(DOWNLOADED_PREBUILT_VERSION "${PREBUILT_VERSION_NAME}" CACHE INTERNAL "")
```

**File**: `lib/prebuilt.cmake:15-19`

```cmake
if ("${DOWNLOADED_PREBUILT_VERSION}" STREQUAL "${PREBUILT_VERSION_NAME}")
    # Libraries already downloaded and up-to-date
    set(${OUT_VAR} "${PREBUILT_LIB_DIR}" PARENT_SCOPE)
    return()
endif()
```

**Issue**: The check compares cached version against hardcoded `PREBUILT_VERSION_NAME`. This works correctly **IF** the cache is cleared. However:
- The `PREBUILT_VERSION_NAME` is in `prebuilt.cmake` line 2
- No `CMAKE_CONFIGURE_DEPENDS` forces reconfiguration when this file changes

---

## 🟡 MEDIUM: TARGET_COPY_FILES Accumulation

**File**: `cmake/util.cmake:187`

```cmake
SET(TARGET_COPY_FILES ${TARGET_COPY_FILES} ${file} CACHE INTERNAL "" FORCE)
```

**Issue**: Files are appended to cache with FORCE. If CMakeLists.txt is re-processed multiple times (e.g., configuration changes), duplicates can accumulate.

---

## 🟡 MEDIUM: Environment Variables Not Cached

**Files**: `cmake/toolchain-gcc.cmake:60-64`, `cmake/toolchain-clang.cmake:50-54`

```cmake
if(DEFINED ENV{CFLAGS})
    set(C_BASE_FLAGS $ENV{CFLAGS})
endif()
```

**Issue**: If `CFLAGS`/`CXXFLAGS`/`LDFLAGS` environment variables change, there's no automatic reconfiguration trigger.

---

## ✅ GOOD: Source File Listing

The main FSO codebase uses **explicit file listing** via `add_file_folder()` in `code/source_groups.cmake`, avoiding GLOB-related staleness issues. This is best practice.

---

## Summary Table

| Issue | Severity | File:Line | Root Cause |
|-------|----------|-----------|------------|
| Shader MAIN_DEPENDENCY typo | **CRITICAL** | `code/shaders.cmake:48` | `${shader}` instead of `${_shader}` |
| Shader DEPFILE Ninja-only | **CRITICAL** | `code/shaders.cmake:39-42` | Include deps not tracked for other generators |
| Version header deps | HIGH | `cmake/generateHeaders.cmake:6-7` | No CMAKE_CONFIGURE_DEPENDS |
| TRY_RUN SIMD caching | HIGH | `cmake/util.cmake:196` | Results cached across machines |
| Prebuilt version check | MEDIUM | `lib/prebuilt.cmake:2,95` | File changes don't trigger reconfigure |
| TARGET_COPY_FILES accumulation | MEDIUM | `cmake/util.cmake:187` | CACHE INTERNAL with FORCE |
| Env var flag changes | LOW | `toolchain-*.cmake` | No dependency tracking |

---

## Recommended Fixes

### 1. Shader typo fix
**File**: `code/shaders.cmake:48`
```cmake
# Change from:
MAIN_DEPENDENCY "${shader}"
# To:
MAIN_DEPENDENCY "${_shader}"
```

### 2. Version dependency tracking
**File**: `cmake/generateHeaders.cmake` (add at top)
```cmake
set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS
    "${CMAKE_SOURCE_DIR}/cmake/version.cmake")
```

### 3. Version override tracking
**File**: `cmake/version.cmake` (modify existing block)
```cmake
if(EXISTS "${PROJECT_SOURCE_DIR}/version_override.cmake")
    set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS
        "${PROJECT_SOURCE_DIR}/version_override.cmake")
    include("${PROJECT_SOURCE_DIR}/version_override.cmake")
endif()
```

### 4. Shader includes for non-Ninja generators
**File**: `code/shaders.cmake` (add DEPENDS clause)
```cmake
# Collect known shader includes
file(GLOB _shaderIncludes "${LEGACY_SHADER_DIR}/*.sdr")

add_custom_command(OUTPUT "${_spirvFile}"
    COMMAND ${CMAKE_COMMAND} -E make_directory "${_depFileDir}"
    COMMAND glslc "${_shader}" -o "${_spirvFile}" --target-env=vulkan1.0 -O -g "-I${SHADER_DIR}"
        "-I${LEGACY_SHADER_DIR}" -MD -MF "${_depFile}" -MT "${_relativeSpirvPath}" -Werror -x glsl
    MAIN_DEPENDENCY "${_shader}"
    DEPENDS ${_shaderIncludes}  # Add explicit depends for non-Ninja
    COMMENT "Compiling shader ${_fileName}"
    ${DEPFILE_PARAM}
)
```

### 5. SIMD cache invalidation (optional)
**File**: `cmake/toolchain.cmake` (add before detect_simd_instructions call)
```cmake
# Force re-detection if this is a different machine
string(SHA256 _machine_hash "${CMAKE_SYSTEM_NAME}${CMAKE_SYSTEM_PROCESSOR}${CMAKE_HOST_SYSTEM_NAME}")
if(NOT "${_machine_hash}" STREQUAL "${CACHED_MACHINE_HASH}")
    unset(DETECTED_SIMD_INSTRUCTIONS CACHE)
    set(CACHED_MACHINE_HASH "${_machine_hash}" CACHE INTERNAL "")
endif()
```
