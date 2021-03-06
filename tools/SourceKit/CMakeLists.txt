include(CheckSymbolExists)

# Append our own modules to the module path.
list(APPEND CMAKE_MODULE_PATH
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules")

if (DARWIN_TOOLCHAIN_VERSION)
  set(SOURCEKIT_VERSION_STRING "${DARWIN_TOOLCHAIN_VERSION}")
else()
  set(SOURCEKIT_VERSION_STRING "${SWIFT_COMPILER_VERSION}")
endif()

set(SOURCEKIT_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
set(SOURCEKIT_BINARY_DIR ${SWIFT_BINARY_DIR})

set(SOURCEKIT_RUNTIME_OUTPUT_INTDIR "${SWIFT_RUNTIME_OUTPUT_INTDIR}")
set(SOURCEKIT_LIBRARY_OUTPUT_INTDIR "${SWIFT_LIBRARY_OUTPUT_INTDIR}")

check_symbol_exists(dispatch_block_create "dispatch/dispatch.h" HAVE_DISPATCH_BLOCK_CREATE)
configure_file(
  ${SOURCEKIT_SOURCE_DIR}/include/SourceKit/Config/config.h.cmake
  ${SOURCEKIT_BINARY_DIR}/include/SourceKit/Config/config.h)

set(SOURCEKIT_DEPLOYMENT_OS "${SWIFT_HOST_VARIANT}")
set(SOURCEKIT_DEPLOYMENT_TARGET "${SWIFT_SDK_${SWIFT_HOST_VARIANT_SDK}_DEPLOYMENT_VERSION}")

swift_is_installing_component(sourcekit-inproc SOURCEKIT_INSTALLING_INPROC)

if("${CMAKE_SYSTEM_NAME}" STREQUAL "Darwin" AND NOT CMAKE_CROSSCOMPILING)
  set(CMAKE_OSX_SYSROOT "${SWIFT_SDK_${SWIFT_HOST_VARIANT_SDK}_PATH}")
  set(CMAKE_OSX_ARCHITECTURES "${SWIFT_HOST_VARIANT_ARCH}")
  set(CMAKE_OSX_DEPLOYMENT_TARGET "")
endif()

# Now include AddSwiftSourceKit
include(AddSwiftSourceKit)

if("${CMAKE_SYSTEM_NAME}" STREQUAL "Darwin")
  # Choose a deployment target if none was set.
  set(SOURCEKIT_DEPLOYMENT_TARGET "" CACHE STRING
      "Deployment target for SourceKit.")
  if (NOT SOURCEKIT_DEPLOYMENT_TARGET)
    execute_process(COMMAND sw_vers -productVersion
                    OUTPUT_VARIABLE SOURCEKIT_DEPLOYMENT_TARGET
                    OUTPUT_STRIP_TRAILING_WHITESPACE)
    string(REGEX MATCH "[0-9]+\\.[0-9]+" SOURCEKIT_DEPLOYMENT_TARGET ${SOURCEKIT_DEPLOYMENT_TARGET})
  endif()

  # Sadly there are two OS naming conventions.
  # xcrun SDK name:   macosx iphoneos iphonesimulator (+ "internal" or version)
  # -mOS-version-min: macosx ios      ios-simulator

  if (SOURCEKIT_DEPLOYMENT_OS MATCHES "^iphoneos")
    set(version_min_os "ios")
    set(triple_os "ios")
    set(SOURCEKIT_DEFAULT_TARGET_SDK "IOS")
  elseif (SOURCEKIT_DEPLOYMENT_OS MATCHES "^iphonesimulator")
    set(version_min_os "ios-simulator")
    set(triple_os "ios")
    set(SOURCEKIT_DEFAULT_TARGET_SDK "IOS_SIMULATOR")
  elseif (SOURCEKIT_DEPLOYMENT_OS MATCHES "^macosx")
    set(version_min_os "macosx")
    set(triple_os "macosx")
    set(SOURCEKIT_DEFAULT_TARGET_SDK "OSX")
  else()
    message(FATAL_ERROR "Can't understand SOURCEKIT_DEPLOYMENT_OS '${SOURCEKIT_DEPLOYMENT_OS}';")
  endif()

  # Add deployment target to C/C++ compiler and linker flags.
  # FIXME: CMAKE_OSX_DEPLOYMENT_TARGET falls over when used for iOS versions.
  if (XCODE)
    if (${SOURCEKIT_DEPLOYMENT_OS} MATCHES "^macosx")
      set(CMAKE_XCODE_ATTRIBUTE_MACOSX_DEPLOYMENT_TARGET ${SOURCEKIT_DEPLOYMENT_TARGET})
    else()
      set(CMAKE_XCODE_ATTRIBUTE_IPHONEOS_DEPLOYMENT_TARGET ${SOURCEKIT_DEPLOYMENT_TARGET})
    endif()
  else()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -m${version_min_os}-version-min=${SOURCEKIT_DEPLOYMENT_TARGET}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -m${version_min_os}-version-min=${SOURCEKIT_DEPLOYMENT_TARGET}")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -m${version_min_os}-version-min=${SOURCEKIT_DEPLOYMENT_TARGET}")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -m${version_min_os}-version-min=${SOURCEKIT_DEPLOYMENT_TARGET}")
  endif()
endif()

include_directories(BEFORE
  ${SOURCEKIT_BINARY_DIR}/include
  ${SOURCEKIT_SOURCE_DIR}/include
)

if("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux")
  if(SWIFT_BUILD_SOURCEKIT)
    include(ExternalProject)
    ExternalProject_Add(libdispatch
                        SOURCE_DIR
                          "${SWIFT_PATH_TO_LIBDISPATCH_SOURCE}"
                        BINARY_DIR
                          "${SWIFT_PATH_TO_LIBDISPATCH_BUILD}"
                        CMAKE_ARGS
                          -DCMAKE_C_COMPILER=${PATH_TO_CLANG_BUILD}/bin/clang
                          -DCMAKE_CXX_COMPILER=${PATH_TO_CLANG_BUILD}/bin/clang++
                          -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
                        BUILD_BYPRODUCTS
                          ${SWIFT_PATH_TO_LIBDISPATCH_BUILD}/src/${CMAKE_SHARED_LIBRARY_PREFIX}dispatch${CMAKE_SHARED_LIBRARY_SUFFIX})

    include_directories(AFTER
                          ${SWIFT_PATH_TO_LIBDISPATCH_SOURCE}/src/BlocksRuntime)
    link_directories(${SWIFT_PATH_TO_LIBDISPATCH_BUILD})
  endif()

  include_directories(AFTER ${SWIFT_PATH_TO_LIBDISPATCH_SOURCE})

  add_library(dispatch UNKNOWN IMPORTED)
  set_target_properties(dispatch
                        PROPERTIES
                          IMPORTED_LOCATION ${SWIFT_PATH_TO_LIBDISPATCH_BUILD}/src/${CMAKE_SHARED_LIBRARY_PREFIX}dispatch${CMAKE_SHARED_LIBRARY_SUFFIX})

  add_library(swiftCore SHARED IMPORTED)
  set_target_properties(swiftCore PROPERTIES
                        IMPORTED_LOCATION ${SOURCEKIT_BINARY_DIR}/lib/swift/linux/libswiftCore.so)

  set(SOURCEKIT_NEED_EXPLICIT_LIBDISPATCH TRUE)
endif()

add_subdirectory(lib)
add_subdirectory(tools)
