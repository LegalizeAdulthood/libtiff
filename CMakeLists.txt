# CMake build for libtiff
# Run "cmake" to generate the build files for your platform
#
# Copyright © 2015 Open Microscopy Environment / University of Dundee
# Written by Roger Leigh <rleigh@codelibre.net>
#
# Permission to use, copy, modify, distribute, and sell this software and
# its documentation for any purpose is hereby granted without fee, provided
# that (i) the above copyright notices and this permission notice appear in
# all copies of the software and related documentation, and (ii) the names of
# Sam Leffler and Silicon Graphics may not be used in any advertising or
# publicity relating to the software without the specific, prior written
# permission of Sam Leffler and Silicon Graphics.
#
# THE SOFTWARE IS PROVIDED "AS-IS" AND WITHOUT WARRANTY OF ANY KIND,
# EXPRESS, IMPLIED OR OTHERWISE, INCLUDING WITHOUT LIMITATION, ANY
# WARRANTY OF MERCHANTABILITY OR FITNESS FOR A PARTICULAR PURPOSE.
#
# IN NO EVENT SHALL SAM LEFFLER OR SILICON GRAPHICS BE LIABLE FOR
# ANY SPECIAL, INCIDENTAL, INDIRECT OR CONSEQUENTIAL DAMAGES OF ANY KIND,
# OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS,
# WHETHER OR NOT ADVISED OF THE POSSIBILITY OF DAMAGE, AND ON ANY THEORY OF
# LIABILITY, ARISING OUT OF OR IN CONNECTION WITH THE USE OR PERFORMANCE
# OF THIS SOFTWARE.

cmake_minimum_required(VERSION 2.8.11) # b/c of use of BUILD_INTERFACE generator expression

# Default policy is from 2.8.9
cmake_policy(VERSION 2.8.9)
# Set MacOSX @rpath usage globally.
if (POLICY CMP0020)
  cmake_policy(SET CMP0020 NEW)
endif(POLICY CMP0020)
if (POLICY CMP0042)
  cmake_policy(SET CMP0042 NEW)
endif(POLICY CMP0042)
# Use new variable expansion policy.
if (POLICY CMP0053)
  cmake_policy(SET CMP0053 NEW)
endif(POLICY CMP0053)
if (POLICY CMP0054)
  cmake_policy(SET CMP0054 NEW)
endif(POLICY CMP0054)

# Read version information from configure.ac.
FILE(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/configure.ac" configure REGEX "^LIBTIFF_.*=")
foreach(line ${configure})
  foreach(var LIBTIFF_MAJOR_VERSION LIBTIFF_MINOR_VERSION LIBTIFF_MICRO_VERSION LIBTIFF_ALPHA_VERSION
          LIBTIFF_CURRENT LIBTIFF_REVISION LIBTIFF_AGE)
    if(NOT ${var} AND line MATCHES "^${var}=(.*)")
      set(${var} "${CMAKE_MATCH_1}")
      break()
    endif()
  endforeach()
endforeach()

math(EXPR SO_MAJOR "${LIBTIFF_CURRENT} - ${LIBTIFF_AGE}")
set(SO_MINOR "${LIBTIFF_AGE}")
set(SO_REVISION "${LIBTIFF_REVISION}")

message(STATUS "Building tiff version ${LIBTIFF_MAJOR_VERSION}.${LIBTIFF_MINOR_VERSION}.${LIBTIFF_MICRO_VERSION}${LIBTIFF_ALPHA_VERSION}")
message(STATUS "libtiff library version ${SO_MAJOR}.${SO_MINOR}.${SO_REVISION}")

set(BUILD_SHARED_LIBS ON CACHE BOOL "Build shared libraries")
set(CMAKE_C_STANDARD 99)

# Project version
project(tiff C)
set(VERSION "${LIBTIFF_MAJOR_VERSION}.${LIBTIFF_MINOR_VERSION}.${LIBTIFF_MICRO_VERSION}")
set(tiff_VERSION "${VERSION}")
set(tiff_VERSION_MAJOR "${LIBTIFF_MAJOR_VERSION}")
set(tiff_VERSION_MINOR "${LIBTIFF_MINOR_VERSION}")
set(tiff_VERSION_PATCH "${LIBTIFF_MICRO_VERSION}")

# the other tiff_VERSION_* variables are set automatically
set(tiff_VERSION_ALPHA "${LIBTIFF_ALPHA_VERSION}")
# Library version (unlike libtool's baroque scheme, WYSIWYG here)
set(SO_COMPATVERSION "${SO_MAJOR}")
set(SO_VERSION "${SO_MAJOR}.${SO_MINOR}.${SO_REVISION}")

# For autotools header compatibility
set(PACKAGE_NAME "LibTIFF Software")
set(PACKAGE_TARNAME "${PROJECT_NAME}")
set(PACKAGE_VERSION "${PROJECT_VERSION}${tiff_VERSION_ALPHA}")
set(PACKAGE_STRING "${PACKAGE_NAME} ${PACKAGE_VERSION}")
set(PACKAGE_BUGREPORT "tiff@lists.maptools.org")

include(GNUInstallDirs)
include(CheckCCompilerFlag)
include(CheckCSourceCompiles)
include(CheckIncludeFile)
include(CheckLibraryExists)
include(CheckTypeSize)
include(CheckSymbolExists)
enable_testing()

macro(current_date var)
  if(UNIX)
    execute_process(COMMAND "date" +"%Y%m%d" OUTPUT_VARIABLE ${var})
  endif()
endmacro()

current_date(RELEASE_DATE)

macro(extra_dist)
  foreach(file ${ARGV})
    file(RELATIVE_PATH relfile "${PROJECT_SOURCE_DIR}"
         "${CMAKE_CURRENT_SOURCE_DIR}/${file}")
    list(APPEND EXTRA_DIST "${relfile}")
  endforeach()
  set(EXTRA_DIST "${EXTRA_DIST}" PARENT_SCOPE)
endmacro()

set(EXTRA_DIST
  HOWTO-RELEASE
  autogen.sh
  libtiff-4.pc.in)

# These are annoyingly verbose, produce false positives or don't work
# nicely with all supported compiler versions, so are disabled unless
# explicitly enabled.
option(extra-warnings "Enable extra compiler warnings" OFF)

# This will cause the compiler to fail when an error occurs.
option(fatal-warnings "Compiler warnings are errors" OFF)

# Check if the compiler supports each of the following additional
# flags, and enable them if supported.  This greatly improves the
# quality of the build by checking for a number of common problems,
# some of which are quite serious.
if(CMAKE_C_COMPILER_ID STREQUAL "GNU" OR
   CMAKE_C_COMPILER_ID MATCHES "Clang")
  set(test_flags
      -Wall
      -Winline
      -W
      -Wformat-security
      -Wpointer-arith
      -Wdisabled-optimization
      -Wno-unknown-pragmas
      -Wdeclaration-after-statement
      -fstrict-aliasing)
  if(extra-warnings)
    list(APPEND test_flags
        -Wfloat-equal
        -Wmissing-prototypes
        -Wunreachable-code)
  endif()
  if(fatal-warnings)
    list(APPEND test_flags
         -Werror)
  endif()
elseif(CMAKE_C_COMPILER_ID STREQUAL "MSVC")
  set(test_flags)
  if(extra-warnings)
    list(APPEND test_flags
         /W4)
  else()
    list(APPEND test_flags
         /W3)
  endif()
  if (fatal-warnings)
    list(APPEND test_flags
         /WX)
  endif()
endif()

foreach(flag ${test_flags})
  string(REGEX REPLACE "[^A-Za-z0-9]" "_" flag_var "${flag}")
  set(test_c_flag "C_FLAG${flag_var}")
  CHECK_C_COMPILER_FLAG(${flag} "${test_c_flag}")
  if (${test_c_flag})
     set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${flag}")
  endif (${test_c_flag})
endforeach(flag ${test_flags})

if(MSVC)
  set(CMAKE_DEBUG_POSTFIX "d")
endif()

option(ld-version-script "Enable linker version script" ON)
# Check if LD supports linker scripts.
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/conftest.map" "VERS_1 {
        global: sym;
};

VERS_2 {
        global: sym;
} VERS_1;
")
set(CMAKE_REQUIRED_FLAGS_SAVE ${CMAKE_REQUIRED_FLAGS})
set(CMAKE_REQUIRED_FLAGS ${CMAKE_REQUIRED_FLAGS} "-Wl,--version-script=${CMAKE_CURRENT_BINARY_DIR}/conftest.map")
check_c_source_compiles("int main(void){return 0;}" HAVE_LD_VERSION_SCRIPT)
set(CMAKE_REQUIRED_FLAGS ${CMAKE_REQUIRED_FLAGS_SAVE})
file(REMOVE "${CMAKE_CURRENT_BINARY_DIR}/conftest.map")
if (ld-version-script AND HAVE_LD_VERSION_SCRIPT)
  set(HAVE_LD_VERSION_SCRIPT TRUE)
else()
  set(HAVE_LD_VERSION_SCRIPT FALSE)
endif()

# Find libm, if available
if(NOT MINGW)
  find_library(M_LIBRARY m)
endif()

check_include_file(assert.h    HAVE_ASSERT_H)
check_include_file(fcntl.h     HAVE_FCNTL_H)
check_include_file(io.h        HAVE_IO_H)
check_include_file(strings.h   HAVE_STRINGS_H)
check_include_file(sys/time.h  HAVE_SYS_TIME_H)
check_include_file(sys/types.h HAVE_SYS_TYPES_H)
check_include_file(unistd.h    HAVE_UNISTD_H)

check_symbol_exists(optarg "getopt.h" HAVE_DECL_OPTARG)
if (HAVE_DECL_OPTARG)
  set(HAVE_DECL_OPTARG 1)
else()
  set(HAVE_DECL_OPTARG 0)
endif()

# Disable deprecated features to ensure clean build
add_definitions(-DTIFF_DISABLE_DEPRECATED)

# Check type sizes
set(CMAKE_EXTRA_INCLUDE_FILES_SAVE ${CMAKE_EXTRA_INCLUDE_FILES})
set(CMAKE_EXTRA_INCLUDE_FILES ${CMAKE_EXTRA_INCLUDE_FILES} "stddef.h")
check_type_size("size_t" SIZEOF_SIZE_T)
set(CMAKE_EXTRA_INCLUDE_FILES ${CMAKE_EXTRA_INCLUDE_FILES_SAVE})

macro(report_values)
  foreach(val ${ARGV})
    message(STATUS "${val} set to ${${val}}")
  endforeach()
endmacro()

# C99 fixed-size integer types
set(TIFF_INT8_T "int8_t")
set(TIFF_UINT8_T "uint8_t")

set(TIFF_INT16_T "int16_t")
set(TIFF_UINT16_T "uint16_t")

set(TIFF_INT32_T "int32_t")
set(TIFF_UINT32_T "uint32_t")

set(TIFF_INT64_T "int64_t")
set(TIFF_UINT64_T "uint64_t")


if(SIZEOF_SIZE_T EQUAL 4)
  set(TIFF_SSIZE_T "int32_t")
elseif(SIZEOF_SIZE_T EQUAL 8)
  set(TIFF_SSIZE_T "int64_t")
else()
  message(FATAL_ERROR "Unsupported size_t size ${SIZEOF_SIZE_T}; please add support")
endif()

report_values(TIFF_SSIZE_T)

check_symbol_exists(mmap "sys/mman.h" HAVE_MMAP)
check_symbol_exists(setmode "unistd.h" HAVE_SETMODE)
check_symbol_exists(getopt "unistd.h;stdio.h" HAVE_GETOPT)

# CPU bit order
set(HOST_FILLORDER FILLORDER_MSB2LSB)
if(CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "i.*86.*" OR
   CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "amd64.*" OR
   # AMD64 on Windows
   CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "AMD64" OR
   CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "x86_64.*")
  set(HOST_FILLORDER FILLORDER_LSB2MSB)
endif()

# CPU endianness
include(TestBigEndian)
test_big_endian(HOST_BIG_ENDIAN)
if(HOST_BIG_ENDIAN)
  add_definitions(-DWORDS_BIGENDIAN)
endif()

# IEEE floating point
set(HAVE_IEEEFP 1)

report_values(CMAKE_HOST_SYSTEM_PROCESSOR HOST_FILLORDER
              HOST_BIG_ENDIAN HAVE_IEEEFP)

# Large file support
if (UNIX OR MINGW)
  # This might not catch every possibility catered for by
  # AC_SYS_LARGEFILE.
  add_definitions(-D_FILE_OFFSET_BITS=64)
  set(FILE_OFFSET_BITS 64)
endif()

# Documentation install directory (default to cmake project docdir)
set(LIBTIFF_DOCDIR "${CMAKE_INSTALL_FULL_DOCDIR}")

# Options to enable and disable internal codecs

option(ccitt "support for CCITT Group 3 & 4 algorithms" ON)
set(CCITT_SUPPORT ${ccitt})

option(packbits "support for Macintosh PackBits algorithm" ON)
set(PACKBITS_SUPPORT ${packbits})

option(lzw "support for LZW algorithm" ON)
set(LZW_SUPPORT ${lzw})

option(thunder "support for ThunderScan 4-bit RLE algorithm" ON)
set(THUNDER_SUPPORT ${thunder})

option(next "support for NeXT 2-bit RLE algorithm" ON)
set(NEXT_SUPPORT ${next})

option(logluv "support for LogLuv high dynamic range algorithm" ON)
set(LOGLUV_SUPPORT ${logluv})

# Option for Microsoft Document Imaging
option(mdi "support for Microsoft Document Imaging" ON)
set(MDI_SUPPORT ${mdi})

# ZLIB
set(ZLIB_SUPPORT FALSE)
find_package(ZLIB)
option(zlib "use zlib (required for Deflate compression)" ${ZLIB_FOUND})
if(zlib AND ZLIB_FOUND)
  set(ZLIB_SUPPORT TRUE)
endif()
set(ZIP_SUPPORT ${ZLIB_SUPPORT})

# libdeflate
set(LIBDEFLATE_SUPPORT FALSE)
find_path(DEFLATE_INCLUDE_DIR libdeflate.h)
set(DEFLATE_NAMES ${DEFLATE_NAMES} deflate libdeflate libdeflatestatic)
find_library(DEFLATE_LIBRARY NAMES ${DEFLATE_NAMES})
if (DEFLATE_INCLUDE_DIR AND DEFLATE_LIBRARY)
  set(DEFLATE_FOUND TRUE)
  set(DEFLATE_LIBRARIES ${DEFLATE_LIBRARY})
  message(STATUS "Found libdeflate: ${DEFLATE_LIBRARY}")
else()
  set(DEFLATE_FOUND FALSE)
endif()
option(libdeflate "use libdeflate (optional for faster Deflate support, still requires zlib)" ${DEFLATE_FOUND})
if (libdeflate AND DEFLATE_FOUND AND ZIP_SUPPORT)
  set(LIBDEFLATE_SUPPORT TRUE)
endif()
if(DEFLATE_FOUND AND NOT ZIP_SUPPORT)
  message(WARNING "libdeflate available but zlib is not. libdeflate cannot be used")
endif()

# Option for Pixar log-format algorithm
# Pixar log format
set(PIXARLOG_SUPPORT FALSE)
option(pixarlog "support for Pixar log-format algorithm (requires Zlib)" ${ZLIB_SUPPORT})
if(pixarlog AND ZLIB_SUPPORT)
  set(PIXARLOG_SUPPORT TRUE)
endif()

# JPEG
set(JPEG_SUPPORT FALSE)
find_package(JPEG)
option(jpeg "use libjpeg (required for JPEG compression)" ${JPEG_FOUND})
if (jpeg AND JPEG_FOUND)
   set(JPEG_SUPPORT TRUE)
endif()

# Old-jpeg
set(OJPEG_SUPPORT FALSE)
option(old-jpeg "support for Old JPEG compression (read-only)" ${JPEG_SUPPORT})
if (old-jpeg AND JPEG_SUPPORT)
  set(OJPEG_SUPPORT TRUE)
endif()

# JBIG-KIT
set(JBIG_SUPPORT FALSE)
find_path(JBIG_INCLUDE_DIR jbig.h)
set(JBIG_NAMES ${JBIG_NAMES} jbig libjbig)
find_library(JBIG_LIBRARY NAMES ${JBIG_NAMES})
if (JBIG_INCLUDE_DIR AND JBIG_LIBRARY)
  set(JBIG_FOUND TRUE)
  set(JBIG_LIBRARIES ${JBIG_LIBRARY})
else()
  set(JBIG_FOUND FALSE)
endif()
option(jbig "use ISO JBIG compression (requires JBIT-KIT library)" ${JBIG_FOUND})
if (jbig AND JBIG_FOUND)
  set(JBIG_SUPPORT TRUE)
endif()

set(CMAKE_REQUIRED_INCLUDES_SAVE ${CMAKE_REQUIRED_INCLUDES})
set(CMAKE_REQUIRED_INCLUDES ${CMAKE_REQUIRED_INCLUDES} ${JBIG_INCLUDE_DIR})
check_symbol_exists(jbg_newlen "jbig.h" HAVE_JBG_NEWLEN)
set(CMAKE_REQUIRED_INCLUDES ${CMAKE_REQUIRED_INCLUDES_SAVE})

# liblzma2
set(LZMA_SUPPORT FALSE)
find_package(LibLZMA)
option(lzma "use liblzma (required for LZMA2 compression)" ${LZMA_FOUND})
if (lzma AND LIBLZMA_FOUND)
  set(LZMA_SUPPORT TRUE)
endif()

# libzstd
set(ZSTD_SUPPORT FALSE)
find_path(ZSTD_INCLUDE_DIR zstd.h)
find_library(ZSTD_LIBRARY NAMES zstd zstd_static)
if (ZSTD_INCLUDE_DIR AND ZSTD_LIBRARY)
  check_library_exists ("${ZSTD_LIBRARY}" ZSTD_decompressStream "" ZSTD_RECENT_ENOUGH)
  if (ZSTD_RECENT_ENOUGH)
    set(ZSTD_FOUND TRUE)
    set(ZSTD_LIBRARIES ${ZSTD_LIBRARY})
    message(STATUS "Found ZSTD library: ${ZSTD_LIBRARY}")
  else()
    message(WARNING "Found ZSTD library, but not recent enough. Use zstd >= 1.0")
  endif()
else()
  set(ZSTD_FOUND FALSE)
endif()
option(zstd "use libzstd (required for ZSTD compression)" ${ZSTD_FOUND})
if (zstd AND ZSTD_FOUND)
  set(ZSTD_SUPPORT TRUE)
endif()

# libwebp
set(WEBP_SUPPORT FALSE)
find_path(WEBP_INCLUDE_DIR webp/decode.h)
find_library(WEBP_LIBRARY NAMES webp)
if (WEBP_INCLUDE_DIR AND WEBP_LIBRARY)
  set(WEBP_LIBRARIES ${WEBP_LIBRARY})
  set(WEBP_FOUND TRUE)
  message(STATUS "Found WEBP library: ${WEBP_LIBRARY}")
else()
  set(WEBP_FOUND FALSE)
endif()
option(webp "use libwebp (required for WEBP compression)" ${WEBP_FOUND})
if (webp AND WEBP_FOUND)
  set(WEBP_SUPPORT TRUE)
endif()

# 8/12-bit jpeg mode
set(JPEG12_INCLUDE_DIR JPEG12_INCLUDE_DIR-NOTFOUND CACHE PATH "Include directory for 12-bit libjpeg")
set(JPEG12_LIBRARY JPEG12_LIBRARY-NOTFOUND CACHE FILEPATH "12-bit libjpeg library")
set(JPEG_DUAL_MODE_8_12 FALSE)
if (JPEG12_INCLUDE_DIR AND JPEG12_LIBRARY)
  set(JPEG12_LIBRARIES ${JPEG12_LIBRARY})
  set(JPEG12_FOUND TRUE)
else()
  set(JPEG12_FOUND FALSE)
endif()
option(jpeg12 "enable libjpeg 8/12-bit dual mode (requires separate 12-bit libjpeg build)" ${JPEG12_FOUND})
if (jpeg12 AND JPEG12_FOUND)
  set(JPEG_DUAL_MODE_8_12 TRUE)
  set(LIBJPEG_12_PATH "${JPEG12_INCLUDE_DIR}/jpeglib.h")
endif()

# C++ support
option(cxx "Enable C++ stream API building (requires C++ compiler)" ON)
set(CXX_SUPPORT FALSE)
if (cxx)
  enable_language(CXX)
  set(CXX_SUPPORT TRUE)
endif()

# OpenGL and GLUT
find_package(OpenGL)
find_package(GLUT)
set(HAVE_OPENGL FALSE)
if(OPENGL_FOUND AND OPENGL_GLU_FOUND AND GLUT_FOUND)
  set(HAVE_OPENGL TRUE)
endif()
# Purely to satisfy the generated headers:
check_include_file(GL/gl.h HAVE_GL_GL_H)
check_include_file(GL/glu.h HAVE_GL_GLU_H)
check_include_file(GL/glut.h HAVE_GL_GLUT_H)
check_include_file(GLUT/glut.h HAVE_GLUT_GLUT_H)
check_include_file(OpenGL/gl.h HAVE_OPENGL_GL_H)
check_include_file(OpenGL/glu.h HAVE_OPENGL_GLU_H)

# Win32 IO
set(win32_io FALSE)
if(WIN32)
  set(win32_io TRUE)
endif()

set(USE_WIN32_FILEIO ${win32_io})

# Orthogonal features

# Strip chopping
option(strip-chopping "strip chopping (whether or not to convert single-strip uncompressed images to mutiple strips of specified size to reduce memory usage)" ON)
set(TIFF_DEFAULT_STRIP_SIZE 8192 CACHE STRING "default size of the strip in bytes (when strip chopping is enabled)")

set(STRIPCHOP_DEFAULT)
if(strip-chopping)
  set(STRIPCHOP_DEFAULT TRUE)
  if(TIFF_DEFAULT_STRIP_SIZE)
    set(STRIP_SIZE_DEFAULT "${TIFF_DEFAULT_STRIP_SIZE}")
  endif()
endif()

# Defer loading of strip/tile offsets
option(defer-strile-load "enable deferred strip/tile offset/size loading (also available at runtime with the 'D' flag of TIFFOpen())" OFF)
set(DEFER_STRILE_LOAD ${defer-strile-load})

# CHUNKY_STRIP_READ_SUPPORT
option(chunky-strip-read "enable reading large strips in chunks for TIFFReadScanline() (experimental)" OFF)
set(CHUNKY_STRIP_READ_SUPPORT ${chunky-strip-read})

# SUBIFD support
set(SUBIFD_SUPPORT 1)

# Default handling of ASSOCALPHA support.
option(extrasample-as-alpha "the RGBA interface will treat a fourth sample with no EXTRASAMPLE_ value as being ASSOCALPHA. Many packages produce RGBA files but don't mark the alpha properly" ON)
if(extrasample-as-alpha)
  set(DEFAULT_EXTRASAMPLE_AS_ALPHA 1)
endif()

# Default handling of YCbCr subsampling support.
# See Bug 168 in Bugzilla, and JPEGFixupTestSubsampling() for details.
option(check-ycbcr-subsampling "enable picking up YCbCr subsampling info from the JPEG data stream to support files lacking the tag" ON)
if (check-ycbcr-subsampling)
  set(CHECK_JPEG_YCBCR_SUBSAMPLING 1)
endif()

# Generate pkg-config file
set(prefix "${CMAKE_INSTALL_PREFIX}")
set(exec_prefix "${CMAKE_INSTALL_PREFIX}")
set(libdir "${CMAKE_INSTALL_FULL_LIBDIR}")
set(includedir "${CMAKE_INSTALL_FULL_INCLUDEDIR}")
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libtiff-4.pc.in
               ${CMAKE_CURRENT_BINARY_DIR}/libtiff-4.pc)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libtiff-4.pc
        DESTINATION "${CMAKE_INSTALL_FULL_LIBDIR}/pkgconfig")

# Includes used by libtiff (and tests)
if(ZLIB_INCLUDE_DIRS)
  list(APPEND TIFF_INCLUDES ${ZLIB_INCLUDE_DIRS})
endif()
if(DEFLATE_INCLUDE_DIR)
  list(APPEND TIFF_INCLUDES ${DEFLATE_INCLUDE_DIR})
endif()
if(JPEG_INCLUDE_DIR)
  list(APPEND TIFF_INCLUDES ${JPEG_INCLUDE_DIR})
endif()
if(JPEG12_INCLUDE_DIR)
  list(APPEND TIFF_INCLUDES ${JPEG12_INCLUDE_DIR})
endif()
if(JBIG_INCLUDE_DIR)
  list(APPEND TIFF_INCLUDES ${JBIG_INCLUDE_DIR})
endif()
if(LIBLZMA_INCLUDE_DIRS)
  list(APPEND TIFF_INCLUDES ${LIBLZMA_INCLUDE_DIRS})
endif()
if(ZSTD_INCLUDE_DIR)
  list(APPEND TIFF_INCLUDES ${ZSTD_INCLUDE_DIR})
endif()
if(WEBP_INCLUDE_DIR)
  list(APPEND TIFF_INCLUDES ${WEBP_INCLUDE_DIR})
endif()

# Libraries required by libtiff
set(TIFF_LIBRARY_DEPS)
if(NOT MINGW AND M_LIBRARY)
  list(APPEND TIFF_LIBRARY_DEPS "m")
endif()
if(ZLIB_LIBRARIES)
  list(APPEND TIFF_LIBRARY_DEPS ${ZLIB_LIBRARIES})
endif()
if(DEFLATE_LIBRARIES)
  list(APPEND TIFF_LIBRARY_DEPS ${DEFLATE_LIBRARIES})
endif()
if(JPEG_LIBRARIES)
  list(APPEND TIFF_LIBRARY_DEPS ${JPEG_LIBRARIES})
endif()
if(JPEG12_LIBRARIES)
  list(APPEND TIFF_LIBRARY_DEPS ${JPEG12_LIBRARIES})
endif()
if(JBIG_LIBRARIES)
  list(APPEND TIFF_LIBRARY_DEPS ${JBIG_LIBRARIES})
endif()
if(LIBLZMA_LIBRARIES)
  list(APPEND TIFF_LIBRARY_DEPS ${LIBLZMA_LIBRARIES})
endif()
if(ZSTD_LIBRARIES)
  list(APPEND TIFF_LIBRARY_DEPS ${ZSTD_LIBRARIES})
endif()
if(WEBP_LIBRARIES)
  list(APPEND TIFF_LIBRARY_DEPS ${WEBP_LIBRARIES})
endif()

report_values(TIFF_INCLUDES TIFF_LIBRARY_DEPS)

# Process subdirectories
add_subdirectory(port)
add_subdirectory(libtiff)
add_subdirectory(tools)
add_subdirectory(test)
add_subdirectory(contrib)
add_subdirectory(build)
add_subdirectory(man)
add_subdirectory(html)

#message(STATUS "EXTRA_DIST: ${EXTRA_DIST}")

message(STATUS "")
message(STATUS "Libtiff is now configured for ${host}")
message(STATUS "")
message(STATUS "  Installation directory:             ${prefix}")
message(STATUS "  Documentation directory:            ${LIBTIFF_DOCDIR}")
message(STATUS "  C compiler:                         ${CMAKE_C_COMPILER}")
message(STATUS "  C++ compiler:                       ${CMAKE_CXX_COMPILER}")
message(STATUS "  Build shared libraries:             ${BUILD_SHARED_LIBS}")
message(STATUS "  Enable linker symbol versioning:    ${HAVE_LD_VERSION_SCRIPT}")
message(STATUS "  Support Microsoft Document Imaging: ${mdi}")
message(STATUS "  Use win32 IO:                       ${USE_WIN32_FILEIO}")
message(STATUS "")
message(STATUS " Support for internal codecs:")
message(STATUS "  CCITT Group 3 & 4 algorithms:       ${ccitt}")
message(STATUS "  Macintosh PackBits algorithm:       ${packbits}")
message(STATUS "  LZW algorithm:                      ${lzw}")
message(STATUS "  ThunderScan 4-bit RLE algorithm:    ${thunder}")
message(STATUS "  NeXT 2-bit RLE algorithm:           ${next}")
message(STATUS "  LogLuv high dynamic range encoding: ${logluv}")
message(STATUS "")
message(STATUS " Support for external codecs:")
message(STATUS "  ZLIB support:                       Requested:${zlib} Availability:${ZLIB_FOUND} Support:${ZLIB_SUPPORT}")
if(ZLIB_SUPPORT)
  message(STATUS "  libdeflate support:                 Requested:${libdeflate} Availability:${DEFLATE_FOUND} Support:${LIBDEFLATE_SUPPORT}")
else()
  message(STATUS "  libdeflate support:                 Requested:${libdeflate} Availability:${DEFLATE_FOUND} Support:${LIBDEFLATE_SUPPORT} (Depends on ZLIB Support)")
endif()
if(ZLIB_SUPPORT)
  message(STATUS "  Pixar log-format algorithm:         Requested:${pixarlog} Availability:${ZLIB_FOUND} Support:${PIXARLOG_SUPPORT}")
else()
  message(STATUS "  Pixar log-format algorithm:         Requested:${pixarlog} Availability:${ZLIB_FOUND} Support:${PIXARLOG_SUPPORT} (Depends on ZLIB Support)")
endif()
message(STATUS "  JPEG support:                       Requested:${jpeg} Availability:${JPEG_FOUND} Support:${JPEG_SUPPORT}")
if(JPEG_SUPPORT)
  message(STATUS "  Old JPEG support:                   Requested:${old-jpeg} Availability:${JPEG_SUPPORT} Support:${OJPEG_SUPPORT}")
else()
  message(STATUS "  Old JPEG support:                   Requested:${old-jpeg} Availability:${JPEG_SUPPORT} Support:${OJPEG_SUPPORT} (Depends on JPEG Support)")
endif()
message(STATUS "  JPEG 8/12 bit dual mode:            Requested:${jpeg12} Availability:${JPEG12_FOUND} Support:${JPEG_DUAL_MODE_8_12}")
message(STATUS "  ISO JBIG support:                   Requested:${jbig} Availability:${JBIG_FOUND} Support:${JBIG_SUPPORT}")
message(STATUS "  LZMA2 support:                      Requested:${lzma} Availability:${LIBLZMA_FOUND} Support:${LZMA_SUPPORT}")
message(STATUS "  ZSTD support:                       Requested:${zstd} Availability:${ZSTD_FOUND} Support:${ZSTD_SUPPORT}")
message(STATUS "  WEBP support:                       Requested:${webp} Availability:${WEBP_FOUND} Support:${WEBP_SUPPORT}")
message(STATUS "")
message(STATUS "  C++ support:                        ${cxx} (requested) ${CXX_SUPPORT} (availability)")
message(STATUS "")
# message(STATUS "  X Athena Widgets support:           ${HAVE_XAW}")
message(STATUS "  OpenGL support:                     ${HAVE_OPENGL}")
message(STATUS "")
