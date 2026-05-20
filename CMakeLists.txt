cmake_minimum_required(VERSION 4.0)

set(NAME boost)
set(VERSION 1.90.0)
set(EXT tar.bz2)
set(SHA256 49551aff3b22cbc5c5a9ed3dbc92f0e23ea50a0f7325b0d198b705e8ee3fc305)

project("cmeel-${NAME}" VERSION "${VERSION}")

# helper strings for Boost
string(REPLACE "." "_" VERSION_UNDERSCORES ${VERSION})

find_package(Python3 REQUIRED COMPONENTS Interpreter Development.Module)

configure_file(user-config.jam.in user-config.jam)

if(APPLE)
  set(ORIGIN "@loader_path")
else()
  set(ORIGIN "\$ORIGIN")
endif()

set(CONFIGURE_COMMAND "./bootstrap.sh" "--prefix=${CMAKE_INSTALL_PREFIX}")

set(BUILD_COMMAND
    "./b2" "hardcode-dll-paths=true" "dll-path='${ORIGIN}'" "link=shared"
    "python=3.${Python3_VERSION_MINOR}"
    "--user-config=${CMAKE_BINARY_DIR}/user-config.jam")

# ref https://github.com/numpy/numpy/releases
set(NPY_TARGET_VERSION "")
if(Python3_VERSION_MINOR LESS 13)
  # we don't support numpy < 2.0
  set(NPY_TARGET_VERSION "NPY_2_0_API_VERSION")
elseif(Python3_VERSION_MINOR LESS 14)
  # 3.13 <= python < 3.14 => runtime can't use numpy < 2.1
  set(NPY_TARGET_VERSION "NPY_2_1_API_VERSION")
elseif(Python3_VERSION_MINOR LESS 15)
  # 3.14 <= python < 3.15 => runtime can't use numpy < 2.3
  set(NPY_TARGET_VERSION "NPY_2_3_API_VERSION")
else()
  message(
    FATAL_ERROR "Python 3.${Python3_VERSION_MINOR} not yet supported by numpy")
endif()

add_compile_definitions("NPY_TARGET_VERSION=${NPY_TARGET_VERSION}")
set(_CFLAGS "-DNPY_TARGET_VERSION=${NPY_TARGET_VERSION}")

if(APPLE AND CMAKE_HOST_SYSTEM_PROCESSOR STREQUAL "arm64")
  set(_CFLAGS "${_CFLAGS} -arch arm64")
  set(BUILD_COMMAND ${BUILD_COMMAND} "toolset=clang-darwin" "target-os=darwin"
                    "architecture=arm" "abi=aapcs" "linkflags=-arch arm64")
endif()

set(CONFIGURE_COMMAND ${CONFIGURE_COMMAND} "cflags=${_CFLAGS}")
set(BUILD_COMMAND ${BUILD_COMMAND} "cflags=${_CFLAGS}")

message(STATUS "CONFIGURE_COMMAND: ${CONFIGURE_COMMAND}")
message(STATUS "BUILD_COMMAND: ${BUILD_COMMAND}")

include(ExternalProject)
ExternalProject_Add(
  ${NAME}
  URL "https://archives.boost.io/release/${VERSION}/source/${NAME}_${VERSION_UNDERSCORES}.${EXT}"
  URL_HASH "SHA256=${SHA256}"
  DOWNLOAD_EXTRACT_TIMESTAMP OFF
  BUILD_IN_SOURCE ON
  CONFIGURE_COMMAND ${CONFIGURE_COMMAND}
  BUILD_COMMAND ${BUILD_COMMAND}
  INSTALL_COMMAND ${BUILD_COMMAND} "install")

# dummy file for install target
install(FILES "README.md" DESTINATION "share/cmeel-${NAME}/")
