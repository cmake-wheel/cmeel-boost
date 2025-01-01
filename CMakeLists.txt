cmake_minimum_required(VERSION 3.24)

set(NAME boost)
set(VERSION 1.87.0)
set(EXT tar.bz2)
set(SHA256 af57be25cb4c4f4b413ed692fe378affb4352ea50fbe294a11ef548f4d527d89)

project(cmeel-${NAME} VERSION ${VERSION})

# helper strings for Boost
string(REPLACE "." "_" VERSION_UNDERSCORES ${VERSION})

find_package(Python3 REQUIRED COMPONENTS Interpreter Development.Module)

configure_file(python.jam.in python.jam)

if(APPLE)
  set(ORIGIN "@loader_path")
else()
  set(ORIGIN "\$ORIGIN")
endif()

set(CONFIGURE_COMMAND "./bootstrap.sh" "--prefix=${CMAKE_INSTALL_PREFIX}")

set(BUILD_COMMAND "./b2" "hardcode-dll-paths=true" "dll-path='${ORIGIN}'"
                  "link=shared" "python=3.${Python3_VERSION_MINOR}")

if(APPLE AND CMAKE_HOST_SYSTEM_PROCESSOR STREQUAL "arm64")
  set(CONFIGURE_COMMAND ${CONFIGURE_COMMAND} "cxxflags=-arch arm64"
                        "cflags=-arch arm64")
  set(BUILD_COMMAND
      ${BUILD_COMMAND} "toolset=clang-darwin" "target-os=darwin"
      "architecture=arm" "abi=aapcs" "cxxflags=-arch arm64"
      "cflags=-arch arm64" "linkflags=-arch arm64")
endif()

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
  # append python configuration to generated project-config.jam
  COMMAND
    ${Python3_EXECUTABLE} "-c"
    "with open('${CMAKE_BINARY_DIR}/python.jam', 'a+') as a, open('project-config.jam') as b: a.write(b.read())"
  COMMAND ${CMAKE_COMMAND} "-E" "rename" "${CMAKE_BINARY_DIR}/python.jam"
          "project-config.jam"
  BUILD_COMMAND ${BUILD_COMMAND}
  INSTALL_COMMAND ${BUILD_COMMAND} "install")

# dummy file for install target
install(FILES README.md DESTINATION share/cmeel-${NAME}/)
