# Copyright (c) 2015, 2016 Oracle and/or its affiliates. All rights reserved.
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; version 2 of the License.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA  02110-1301 USA


INCLUDE(${MYSQL_CMAKE_SCRIPT_DIR}/compile_flags.cmake)

IF(SSL_DEFINES)
  ADD_DEFINITIONS(${SSL_DEFINES})
ENDIF()

# Note that no COMPONENTS option is to be used for header only components,
# it would try to find a library
SET(Boost_USE_STATIC_LIBS OFF)
ADD_DEFINITIONS(-DBOOST_ALL_NO_LIB -DBOOST_NO_AUTO_PTR)

MESSAGE(STATUS "Using Boost headers from ${BOOST_INCLUDE_DIR}")

SET(XPLUGIN_LOG_PROTOBUF 1 CACHE STRING "Enable logging of protobuf messages")

INCLUDE_DIRECTORIES(${PROTOBUF_INCLUDE_DIRS})
INCLUDE(mysqlx_protobuf.cmake)

IF(NOT PROTOBUF_FOUND)
  MESSAGE(WARNING "Disabling mysqlx plugin.")
ELSE()
  INCLUDE(source_files.cmake)

  INCLUDE(mysqlx_error.cmake)

  SET(GENERATED_LITE_PROTO_FILES)
  # Generate protobuf-lite versions of proto files.
  FOREACH(PROTOBUF_MYSQLX_FILE ${PROTOBUF_MYSQLX_FILES})
    GET_FILENAME_COMPONENT(MYSQLX_NAME_WD ${PROTOBUF_MYSQLX_FILE} NAME)
    LIST(APPEND GENERATED_LITE_PROTO_FILES
      ${CMAKE_CURRENT_BINARY_DIR}/protobuf_lite/${MYSQLX_NAME_WD})
    ADD_CUSTOM_COMMAND(
      OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/protobuf_lite/${MYSQLX_NAME_WD}"
      COMMAND ${CMAKE_COMMAND}
              -D CURRENT_BINARY_DIR="${CMAKE_CURRENT_BINARY_DIR}"
              -D PROTO_FILE="${PROTOBUF_MYSQLX_FILE}"
              -D PROTO_FILE_FLAGS="PROTOBUF_LITE"
              -P ${CMAKE_CURRENT_SOURCE_DIR}/process_protobuf_file.cmake
      DEPENDS ${PROTOBUF_MYSQLX_FILE})
    SET_SOURCE_FILES_PROPERTIES(
      "${CMAKE_CURRENT_BINARY_DIR}/protobuf_lite/${MYSQLX_NAME_WD}"
      PROPERTIES GENERATED TRUE)
  ENDFOREACH()

  # Be sure to convert all proto files to protobuf_lite before running protoc
  ADD_CUSTOM_TARGET(GenLiteProtos ALL DEPENDS ${GENERATED_LITE_PROTO_FILES})

  MYSQLX_PROTOBUF_GENERATE_CPP(
    PROTO_SRCS PROTO_HDRS
    PROTO_LITE_SRCS PROTO_LITE_HDRS
    ${PROTOBUF_MYSQLX_FILES})

  MY_CHECK_CXX_COMPILER_FLAG("-Wignored-qualifiers" HAVE_NO_IGNORED_QUALIFIERS)
  IF(HAVE_NO_IGNORED_QUALIFIERS)
    ADD_COMPILE_FLAGS(${PROTO_SRCS} COMPILE_FLAGS
      "-Wno-ignored-qualifiers")
  ENDIF()

  MY_CHECK_CXX_COMPILER_FLAG("-Wsign-compare" CXX_HAVE_SIGN_COMPARE)
  IF(CXX_HAVE_SIGN_COMPARE)
    ADD_COMPILE_FLAGS(${PROTO_SRCS} COMPILE_FLAGS
      "-Wno-sign-compare")
  ENDIF()

  MY_CHECK_CXX_COMPILER_FLAG("-Wunused-variable" CXX_HAVE_UNUSED_VARIABLE)
  IF(CXX_HAVE_UNUSED_VARIABLE)
    ADD_COMPILE_FLAGS(${PROTO_SRCS} COMPILE_FLAGS
      "-Wno-unused-variable")
  ENDIF()

  INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/protocol/errors
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/ngs/include
    ${CMAKE_SOURCE_DIR}/include #temporary
    ${CMAKE_CURRENT_SOURCE_DIR}/include/mysql
    ${SSL_INCLUDE_DIRS}
    ${ZLIB_INCLUDE_DIR}
  )

  INCLUDE_DIRECTORIES(SYSTEM
    ${BOOST_INCLUDE_DIR}
    ${PROTOBUF_INCLUDE_DIRS}
    ${LIBEVENT_INCLUDE_DIR}
  )

  IF(XPLUGIN_NO_LITE_PROTOBUF)
    SET(xplugin_PROTOBUF_SRC ${PROTO_SRCS} ${PROTO_HDRS})
    SET(xplugin_PROTOBUF_DIR "${CMAKE_CURRENT_BINARY_DIR}/protobuf")
    SET(xplugin_PROTOBUF_LIB ${PROTOBUF_LIBRARY})
  ELSE()
    SET(xplugin_PROTOBUF_SRC ${PROTO_LITE_SRCS} ${PROTO_LITE_HDRS})
    SET(xplugin_PROTOBUF_DIR "${CMAKE_CURRENT_BINARY_DIR}/protobuf_lite")
    SET(xplugin_PROTOBUF_LIB ${PROTOBUF_LITE_LIBRARY})
  ENDIF()

  IF(MSVC)
    ADD_COMPILE_FLAGS(${PROTO_SRCS} ${PROTO_LITE_SRCS} COMPILE_FLAGS "/wd4018")
  ENDIF(MSVC)

  # Built *without* -DMYSQL_DYNAMIC_PLUGIN
  ADD_CONVENIENCE_LIBRARY(xplugin ${xplugin_SRC} ${xplugin_HDRS} ${xplugin_PROTOBUF_SRC})
  ADD_COMPILE_FLAGS(
    src/xpl_plugin.cc
    src/xpl_performance_schema.cc
    ${xplugin_SRC} ${xplugin_PROTOBUF_SRC}
    COMPILE_FLAGS "-I${xplugin_PROTOBUF_DIR}")

  TARGET_LINK_LIBRARIES(xplugin ${LIBEVENT_LIBRARY} ${LIBEVENT_LIBRARIES})

  # These are built *with* -DMYSQL_DYNAMIC_PLUGIN
  MYSQL_ADD_PLUGIN(mysqlx
    src/xpl_plugin.cc
    src/xpl_performance_schema.cc
    LINK_LIBRARIES xplugin ${xplugin_PROTOBUF_LIB}
    MODULE_ONLY MODULE_OUTPUT_NAME "mysqlx")

  RESTRICT_SYMBOL_SOURCE("${MYSQLX_PROJECT_DIR}/ngs/src/mysql41_hash.cc")

  ADD_DEPENDENCIES(xplugin GenError)
  IF (WITH_LIBEVENT STREQUAL "bundled")
    ADD_DEPENDENCIES(xplugin ${LIBEVENT_LIBRARY})
  ENDIF()

  IF(XPLUGIN_LOG_PROTOBUF)
    MESSAGE(STATUS "MYSQLX - Text log of protobuf messages enabled")
    ADD_DEFINITIONS(-DXPLUGIN_LOG_PROTOBUF)
  ENDIF()

  IF(XPLUGIN_LOG_DEBUG OR WITH_DEBUG)
    MESSAGE(STATUS "MYSQLX - Enabled debug traces")
    ADD_DEFINITIONS(-DXPLUGIN_LOG_DEBUG)
  ENDIF()

  IF(XPLUGIN_NO_LITE_PROTOBUF)
    MESSAGE(STATUS "MYSQLX - Using protobuf instead protobuf-litle")
    ADD_DEFINITIONS(-DUSE_MYSQLX_FULL_PROTO)
  ENDIF()

  INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR}/mysqlxtest_src/common)

  FILE(GLOB mysqlxtest_HDRS
    "${CMAKE_CURRENT_SOURCE_DIR}/mysqlxtest_src/*.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/mysqlxtest_src/common/*.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/ngs/include/ngs_common/*.h"
  )

  FILE(GLOB mysqlxtest_SRC
    "${CMAKE_CURRENT_SOURCE_DIR}/mysqlxtest_src/*.cc"
    "${CMAKE_CURRENT_SOURCE_DIR}/mysqlxtest_src/common/*.cc"
    "${CMAKE_CURRENT_SOURCE_DIR}/ngs/src/protocol_authentication.cc"
    "${CMAKE_CURRENT_SOURCE_DIR}/ngs/src/mysql41_hash.cc"
  )

  SET(mysqlxtest_SRC ${mysqlxtest_SRC} ${PROTO_SRCS})

  MYSQL_ADD_EXECUTABLE(mysqlxtest ${mysqlxtest_SRC})

  ADD_DEPENDENCIES(mysqlxtest GenError)

  MY_CHECK_CXX_COMPILER_FLAG("-Wunused-but-set-parameter" HAVE_UNUSED_BUT_SET)
  IF(HAVE_UNUSED_BUT_SET)
    ADD_COMPILE_FLAGS(${mysqlxtest_SRC} COMPILE_FLAGS
      "-Wno-unused-but-set-parameter")
  ENDIF()

  MY_CHECK_CXX_COMPILER_FLAG("-Wunused-parameter" HAVE_UNUSED_PARAMETER)
  IF(HAVE_UNUSED_PARAMETER)
    ADD_COMPILE_FLAGS(${mysqlxtest_SRC} COMPILE_FLAGS "-Wno-unused-parameter")
  ENDIF()

  GET_PROPERTY(OLD_COMPILE_FLAGS TARGET mysqlxtest PROPERTY COMPILE_FLAGS)
  SET_PROPERTY(TARGET mysqlxtest PROPERTY COMPILE_FLAGS "${OLD_COMPILE_FLAGS} -I${CMAKE_CURRENT_BINARY_DIR}/protobuf")

    GET_TARGET_PROPERTY(mysqlxtest_compile_definitions
      mysqlxtest COMPILE_DEFINITIONS)
  IF(mysqlxtest_compile_definitions)
    SET_TARGET_PROPERTIES(mysqlxtest PROPERTIES COMPILE_DEFINITIONS
      "${mysqlxtest_compile_definitions};NGS_DISABLE_LOGGING;USE_MYSQLX_FULL_PROTO")
  ELSE()
    SET_TARGET_PROPERTIES(mysqlxtest PROPERTIES COMPILE_DEFINITIONS
      "NGS_DISABLE_LOGGING;USE_MYSQLX_FULL_PROTO")
  ENDIF()

  TARGET_LINK_LIBRARIES(mysqlxtest mysys strings perconaserverclient)
  TARGET_LINK_LIBRARIES(mysqlxtest ${PROTOBUF_LIBRARY})

  IF( WIN32 )
    ADD_DEFINITIONS(-DMYSQLXTEST_EXPORTS)
  ENDIF()

ENDIF()
