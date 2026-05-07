cmake_minimum_required(VERSION 3.14)

set(CMAKE_VERBOSE_MAKEFILE off)

project(
    piper_phonemize
    VERSION 1.2.0
    DESCRIPTION "Phonemization library for Piper text to speech system"
    HOMEPAGE_URL "https://github.com/rhasspy/piper-phonemize"
    LANGUAGES CXX
)

if(MSVC)
    # Force compiler to use UTF-8 for IPA constants
    add_compile_options("$<$<C_COMPILER_ID:MSVC>:/utf-8>")
    add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")

elseif(NOT APPLE)
    # Linux flags
    string(APPEND CMAKE_CXX_FLAGS " -Wall -Wextra -Wl,-rpath,'$ORIGIN'")
    string(APPEND CMAKE_C_FLAGS " -Wall -Wextra")
endif()
set(SUBMODULE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/lib")
if(ANDROID)
  set(ANDROID_EXTERNAL_ARG
      -DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}
      -DANDROID_ABI=${ANDROID_ABI}
      -DANDROID_PLATFORM=${ANDROID_PLATFORM}
      -DANDROID_NDK=${ANDROID_NDK}
      -DANDROID_STL=${ANDROID_STL}
      -DCMAKE_ANDROID_STL_TYPE=${CMAKE_ANDROID_STL_TYPE}
      -DCMAKE_SYSTEM_NAME=Android
      -DCMAKE_SYSTEM_VERSION=${CMAKE_SYSTEM_VERSION}
  )
endif()
add_library(
    piper_phonemize SHARED
    src/phonemize.cpp
    src/phoneme_ids.cpp
    src/tashkeel.cpp
    src/shared.cpp
)

set_target_properties(piper_phonemize PROPERTIES
    VERSION ${PROJECT_VERSION}
    SOVERSION ${PROJECT_VERSION_MAJOR}
)

# ---- onnxruntime ---

# Look for onnxruntime files in <root>/lib
if(NOT DEFINED ONNXRUNTIME_DIR)
    if(NOT DEFINED ONNXRUNTIME_VERSION)
        set(ONNXRUNTIME_VERSION "1.14.1")
    endif()

    if(WIN32)
        # Windows x86-64
        set(ONNXRUNTIME_PREFIX "onnxruntime-win-x64-${ONNXRUNTIME_VERSION}")
        set(ONNXRUNTIME_EXT "zip")
    elseif (APPLE)
        if(CMAKE_SYSTEM_PROCESSOR STREQUAL x86_64)
            # MacOS x86-64
            set(ONNXRUNTIME_PREFIX "onnxruntime-osx-x86_64-${ONNXRUNTIME_VERSION}")
        elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL arm64)
            # MacOS Apple Silicon
            set(ONNXRUNTIME_PREFIX "onnxruntime-osx-arm64-${ONNXRUNTIME_VERSION}")
        else()
            message(FATAL_ERROR "Unsupported architecture for onnxruntime")
        endif()

        set(ONNXRUNTIME_EXT "tgz")
    elseif (ANDROID)
      set(ONNXRUNTIME_VERSION "1.14.0") # Android doesnt have the 1.14.1 patch released for it because it didn't fix anything on Android
      set(ONNXRUNTIME_PREFIX "onnxruntime-android-${ONNXRUNTIME_VERSION}")
      set(ONNXRUNTIME_EXT "aar")
    else()
        if(CMAKE_SYSTEM_PROCESSOR STREQUAL x86_64)
            # Linux x86-64
            set(ONNXRUNTIME_PREFIX "onnxruntime-linux-x64-${ONNXRUNTIME_VERSION}")
        elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL aarch64)
            # Linux ARM 64-bit
            set(ONNXRUNTIME_PREFIX "onnxruntime-linux-aarch64-${ONNXRUNTIME_VERSION}")
        elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL armv7l)
            # Linux ARM 32-bit
            set(ONNXRUNTIME_PREFIX "onnxruntime-linux-arm32-${ONNXRUNTIME_VERSION}")
            set(ONNXRUNTIME_URL "https://github.com/synesthesiam/prebuilt-apps/releases/download/v1.0/onnxruntime-linux-arm32-${ONNXRUNTIME_VERSION}.tgz")
        else()
            message(FATAL_ERROR "Unsupported architecture for onnxruntime")
        endif()

        set(ONNXRUNTIME_EXT "tgz")
    endif()

    if(NOT DEFINED ONNXRUNTIME_URL)
      if(NOT ANDROID)
        set(ONNXRUNTIME_URL "https://github.com/microsoft/onnxruntime/releases/download/v${ONNXRUNTIME_VERSION}/${ONNXRUNTIME_PREFIX}.${ONNXRUNTIME_EXT}")
      else()
        # Android does not have the 1.14.1 patch release because nothing was changed for it
        set(ONNXRUNTIME_URL "https://repo1.maven.org/maven2/com/microsoft/onnxruntime/onnxruntime-android/${ONNXRUNTIME_VERSION}/${ONNXRUNTIME_PREFIX}.${ONNXRUNTIME_EXT}")
      endif()
    endif()

    set(ONNXRUNTIME_FILENAME "${ONNXRUNTIME_PREFIX}.${ONNXRUNTIME_EXT}")
    set(ONNXRUNTIME_DIR "${CMAKE_CURRENT_LIST_DIR}/lib/${ONNXRUNTIME_PREFIX}")

    if(NOT EXISTS "${ONNXRUNTIME_DIR}")
        if(NOT EXISTS "download/${ONNXRUNTIME_FILENAME}")
            # Download onnxruntime release
            message("Downloading ${ONNXRUNTIME_URL}")
            file(DOWNLOAD "${ONNXRUNTIME_URL}" "download/${ONNXRUNTIME_FILENAME}")
        endif()
        # Extract .zip or .tgz to a directory like lib/onnxruntime-linux-x64-1.14.1/
        if(NOT ANDROID)
          file(ARCHIVE_EXTRACT INPUT "download/${ONNXRUNTIME_FILENAME}" DESTINATION "${CMAKE_CURRENT_LIST_DIR}/lib")
        else() # Special handling for the AAR format on Android
          file(ARCHIVE_EXTRACT INPUT "download/${ONNXRUNTIME_FILENAME}" DESTINATION "${CMAKE_CURRENT_LIST_DIR}/lib/${ONNXRUNTIME_PREFIX}")
        endif()
    endif()
endif()

# ---- espeak-ng ---

if(NOT DEFINED ESPEAK_NG_DIR)
    set(ESPEAK_NG_DIR "${CMAKE_CURRENT_BINARY_DIR}/ei")
    set(ESPEAK_SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/espeak-ng-src")

    set(ESPEAK_ARGS
      -DCMAKE_INSTALL_PREFIX:PATH=${ESPEAK_NG_DIR}
      -DUSE_ASYNC:BOOL=OFF
      -DBUILD_SHARED_LIBS:BOOL=ON
      -DUSE_MBROLA:BOOL=OFF
      -DUSE_LIBSONIC:BOOL=OFF
      -DUSE_LIBPCAUDIO:BOOL=OFF
      -DUSE_KLATT:BOOL=OFF
      -DUSE_SPEECHPLAYER:BOOL=OFF
      -DEXTRA_cmn:BOOL=ON
      -DEXTRA_ru:BOOL=ON
      -DCMAKE_C_FLAGS="-D_FILE_OFFSET_BITS=64"
    )
    include(ExternalProject)
    ExternalProject_Add(
        espeak_ng_source
        PREFIX "${CMAKE_CURRENT_BINARY_DIR}/source"
        URL "https://github.com/rhasspy/espeak-ng/archive/0f65aa301e0d6bae5e172cc74197d32a6182200f.zip"
        SOURCE_DIR "${ESPEAK_SOURCE_DIR}"
        CONFIGURE_COMMAND ""
        BUILD_COMMAND ""
        INSTALL_COMMAND ""
    )

    if(ANDROID)
        ExternalProject_Add(
            espeak_ng_host
            PREFIX "${CMAKE_CURRENT_BINARY_DIR}/eh" # EspeakHost
            DEPENDS espeak_ng_source
            DOWNLOAD_COMMAND ""
            SOURCE_DIR "${ESPEAK_SOURCE_DIR}"
            CMAKE_ARGS ${ESPEAK_ARGS} -DCMAKE_INSTALL_PREFIX:PATH=${ESPEAK_NG_DIR}h
            BUILD_COMMAND ${CMAKE_MAKE_PROGRAM} espeak-ng-data
        )
        list(APPEND ESPEAK_ARGS
          ${ANDROID_EXTERNAL_ARG}
        )
        set(ESPEAK_BUILD_CMD BUILD_COMMAND ${CMAKE_MAKE_PROGRAM} espeak-ng espeak-ng-bin)
        set(ESPEAK_INSTALL_COMMAND INSTALL_COMMAND ${CMAKE_MAKE_PROGRAM} install/fast)
        set(ESPEAK_NG_DEPENDS DEPENDS espeak_ng_host)
    endif()
    ExternalProject_Add(
        espeak_ng_external
        ${ESPEAK_NG_DEPENDS}
        PREFIX "${CMAKE_CURRENT_BINARY_DIR}/e"
        DEPENDS espeak_ng_source
        DOWNLOAD_COMMAND ""
        SOURCE_DIR "${ESPEAK_SOURCE_DIR}"
        CMAKE_ARGS ${ESPEAK_ARGS}
        ${ESPEAK_BUILD_CMD}
        ${ESPEAK_INSTALL_COMMAND}
    )
    add_dependencies(piper_phonemize espeak_ng_external)
endif()


# ---- Declare library ----

target_include_directories(
    piper_phonemize PUBLIC
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src>"
    ${ESPEAK_NG_DIR}/include
)

target_link_directories(
    piper_phonemize PUBLIC
    ${ESPEAK_NG_DIR}/lib
)

# Android has different paths due to being AAR instead of the normal install distribution.
if(NOT ANDROID)
  set(ONNX_LIB_PATH "${ONNXRUNTIME_DIR}/lib")
  set(ONNX_INCLUDE_PATH "${ONNXRUNTIME_DIR}/include")
else()
  set(SUPPORTED_ABIS "arm64-v8a;armeabi-v7a;x86_64;x86")
  if(NOT ANDROID_ABI IN_LIST SUPPORTED_ABIS)
      message(FATAL_ERROR 
          "\nFATAL: Unsupported Android ABI: '${ANDROID_ABI}'.\n"
          "Must be one of: ${SUPPORTED_ABIS}"
      )
  endif()
  set(ONNX_LIB_PATH "${ONNXRUNTIME_DIR}/jni/${ANDROID_ABI}/") # Make sure this ends with a "/" else it gets installed to lib/<abi>/libonnxruntime.so
  set(ONNX_INCLUDE_PATH "${ONNXRUNTIME_DIR}/headers/") # Make sure this ends with a "/" else it gets installed to include/headers/<headers>.h
  if(NOT EXISTS "${ONNX_LIB_PATH}")
      message(FATAL_ERROR "Could not find ONNX library folder at: ${ONNX_LIB_PATH}")
  endif()
endif()
message(INFO "Using lib path ${ONNX_LIB_PATH}")
message(INFO "Using header path ${ONNX_INCLUDE_PATH}")
target_link_directories(piper_phonemize PUBLIC "${ONNX_LIB_PATH}")
target_include_directories(piper_phonemize PUBLIC "${ONNX_INCLUDE_PATH}")

target_link_libraries(
    piper_phonemize
    espeak-ng
    onnxruntime
)

target_compile_features(piper_phonemize PUBLIC cxx_std_17)

# ---- Declare executable ----

add_executable(piper_phonemize_exe src/main.cpp src/phoneme_ids.cpp)

if(NOT WIN32)
    set_property(TARGET piper_phonemize_exe PROPERTY OUTPUT_NAME piper_phonemize)
endif()

target_compile_features(piper_phonemize_exe PUBLIC cxx_std_17)

target_include_directories(
    piper_phonemize_exe PUBLIC
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src>"
    ${ESPEAK_NG_DIR}/include
)

target_link_directories(
    piper_phonemize_exe PUBLIC
    ${ESPEAK_NG_DIR}/lib
)

target_link_libraries(piper_phonemize_exe PUBLIC
    piper_phonemize
    espeak-ng
)

# ---- Declare test ----

include(CTest)
enable_testing()
add_executable(test_piper_phonemize src/test.cpp src/phoneme_ids.cpp)
add_test(
    NAME test_piper_phonemize
    COMMAND test_piper_phonemize "${ESPEAK_NG_DIR}/share/espeak-ng-data" "${CMAKE_SOURCE_DIR}/etc/libtashkeel_model.ort"
)

target_compile_features(test_piper_phonemize PUBLIC cxx_std_17)

target_include_directories(
    test_piper_phonemize PUBLIC
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src>"
    ${ESPEAK_NG_DIR}/include
)

target_link_directories(
    test_piper_phonemize PUBLIC
    ${ESPEAK_NG_DIR}/lib
)

target_link_libraries(test_piper_phonemize PUBLIC
    piper_phonemize
    espeak-ng
)

# ---- Declare install targets ----

include(GNUInstallDirs)

install(
    TARGETS piper_phonemize
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR})

install(
    DIRECTORY ${CMAKE_SOURCE_DIR}/src/
    DESTINATION include/piper-phonemize
    FILES_MATCHING
    PATTERN "*.h"
    PATTERN "*.hpp")

install(
    TARGETS piper_phonemize_exe
    ARCHIVE DESTINATION ${CMAKE_INSTALL_BINDIR})

install(
    FILES ${CMAKE_SOURCE_DIR}/etc/libtashkeel_model.ort
    TYPE DATA)

# Dependencies
install(
    DIRECTORY ${ESPEAK_NG_DIR}/
    DESTINATION ${CMAKE_INSTALL_PREFIX})

if(ANDROID)
  install(
    DIRECTORY "${ESPEAK_NG_DIR}h/share/"
    DESTINATION "share"
  )
endif()

install(
    DIRECTORY ${ONNX_INCLUDE_PATH}
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

install(
    DIRECTORY ${ONNX_LIB_PATH}
    DESTINATION ${CMAKE_INSTALL_LIBDIR})
