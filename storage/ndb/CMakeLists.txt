# Copyright (c) 2008, 2021, Oracle and/or its affiliates.
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License, version 2.0,
# as published by the Free Software Foundation.
#
# This program is also distributed with certain software (including
# but not limited to OpenSSL) that is licensed under separate terms,
# as designated in a particular file or component or in included license
# documentation.  The authors of MySQL hereby grant you an additional
# permission to link the program and your derivative works with the
# separately licensed software that they have included with MySQL.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License, version 2.0, for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA 02110-1301  USA

IF(NOT WITH_NDBCLUSTER)
  RETURN()
ENDIF()

# Add both MySQL and NDB cmake repositories to search path
SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH}
    ${CMAKE_SOURCE_DIR}/cmake
    ${CMAKE_SOURCE_DIR}/storage/ndb/cmake)
INCLUDE(ndb_add_target_property)
INCLUDE(ndb_add_test)
INCLUDE(ndb_add_executable)

#
# Add the ndbcluster plugin
#
SET(NDBCLUSTER_SOURCES
  plugin/ha_ndbcluster.cc
  plugin/ha_ndbcluster_cond.cc
  plugin/ha_ndbcluster_push.cc
  plugin/ha_ndbcluster_connection.cc
  plugin/ha_ndbcluster_binlog.cc
  plugin/ha_ndb_index_stat.cc
  plugin/ha_ndb_ddl_fk.cc
  plugin/ha_ndbinfo.cc
  plugin/ha_ndbinfo_sql.cc
  plugin/ndb_local_connection.cc
  plugin/ndb_share.cc
  plugin/ndb_thd.cc
  plugin/ndb_thd_ndb.cc
  plugin/ndb_global_schema_lock.cc
  plugin/ndb_mi.cc
  plugin/ndb_conflict_trans.cc
  plugin/ndb_anyvalue.cc
  plugin/ndb_ndbapi_util.cc
  plugin/ndb_binlog_extra_row_info.cc
  plugin/ndb_event_data.cc
  plugin/ndb_schema_object.cc
  plugin/ndb_schema_dist.cc
  plugin/ndb_component.cc
  plugin/ndb_repl_tab.cc
  plugin/ndb_conflict.cc
  plugin/ndb_tdc.cc
  plugin/ndb_log.cc
  plugin/ndb_binlog_thread.cc
  plugin/ndb_name_util.cc
  plugin/ndb_bitmap.cc
  plugin/ndb_table_map.cc
  plugin/ndb_dd.cc
  plugin/ndb_modifiers.cc
  plugin/ndb_dd_client.cc
  plugin/ndb_dd_schema.cc
  plugin/ndb_dd_table.cc
  plugin/ndb_dd_sdi.cc
  plugin/ndb_plugin_reference.cc
  plugin/ndb_server_hooks.cc
  plugin/ndb_binlog_client.cc
  plugin/ndb_metadata.cc
  plugin/ndb_dd_disk_data.cc
  plugin/ndb_dd_upgrade_table.cc
  plugin/ndb_binlog_hooks.cc
  plugin/ndb_util_table.cc
  plugin/ndb_schema_dist_table.cc
  plugin/ndb_fk_util.cpp
  plugin/ndb_schema_trans_guard.cc
  plugin/ndb_create_helper.cc
  plugin/ndb_apply_status_table.cc
  plugin/ndb_metadata_change_monitor.cc
  plugin/ndb_schema_result_table.cc
  plugin/ndb_metadata_sync.cc
  plugin/ndb_ddl_transaction_ctx.cc
  plugin/ndb_record_layout.cc
  plugin/ndb_sql_metadata_table.cc
  plugin/ndb_stored_grants.cc
  plugin/ndb_dd_fk.cc
  plugin/ndb_pfs_init.cc
  plugin/ndb_pfs_table.cc
  plugin/ndb_sync_pending_objects_table.cc
  plugin/ndb_sync_excluded_objects_table.cc
  plugin/ndb_dd_sync.cc
  plugin/ndb_upgrade_util.cc
  plugin/ndb_mysql_services.cc
)

NDB_ADD_TEST("ndb_bitmap-t" "plugin/ndb_bitmap.cc" LIBS ndbgeneral)

# Include directories used when building ha_ndbcluster
INCLUDE_DIRECTORIES(${CMAKE_SOURCE_DIR}/storage/ndb/include)

IF(SOLARIS_INTEL)
  # Sun Studio 12.6 on x86 machines creates a bogus code
  # when -xO2 is used and crashes the executable whenever
  # it calls std::condition_variable::wait_for()
  STRING_APPEND(CMAKE_CXX_FLAGS_RELWITHDEBINFO " -xO1")
  STRING_APPEND(CMAKE_CXX_FLAGS_RELEASE " -xO1")
  STRING_APPEND(CMAKE_CXX_FLAGS_MINSIZEREL " -xO1")
ENDIF()

IF(NOT WITHOUT_SERVER)
  MYSQL_ADD_PLUGIN(ndbcluster ${NDBCLUSTER_SOURCES} STORAGE_ENGINE
    DEFAULT STATIC_ONLY
    LINK_LIBRARIES ndbclient_static)

  # Sanity check that MYSQL_ADD_PLUGIN didn't decide to skip build
  IF (NOT WITH_NDBCLUSTER)
    MESSAGE(FATAL_ERROR "MYSQL_ADD_PLUGIN decided to skip build")
  ENDIF()
ENDIF()

# Disable strict-aliasing for storage/ndb
IF(MY_COMPILER_IS_GNU_OR_CLANG)
  STRING_APPEND(CMAKE_C_FLAGS   " -fno-strict-aliasing")
  STRING_APPEND(CMAKE_CXX_FLAGS " -fno-strict-aliasing")
ENDIF()

IF(NOT CMAKE_SIZEOF_VOID_P EQUAL 8)
  MESSAGE(FATAL_ERROR "Only 64-bit build of NDB is supported")
ENDIF()

INCLUDE(${CMAKE_CURRENT_SOURCE_DIR}/ndb_configure.cmake)

INCLUDE_DIRECTORIES(
  # NDB includes
  ${CMAKE_CURRENT_SOURCE_DIR}/include
  ${CMAKE_CURRENT_SOURCE_DIR}/include/util
  ${CMAKE_CURRENT_SOURCE_DIR}/include/portlib
  ${CMAKE_CURRENT_SOURCE_DIR}/include/debugger
  ${CMAKE_CURRENT_SOURCE_DIR}/include/transporter
  ${CMAKE_CURRENT_SOURCE_DIR}/include/kernel
  ${CMAKE_CURRENT_SOURCE_DIR}/include/mgmapi
  ${CMAKE_CURRENT_SOURCE_DIR}/include/mgmcommon
  ${CMAKE_CURRENT_SOURCE_DIR}/include/ndbapi
  ${CMAKE_CURRENT_SOURCE_DIR}/include/logger
  ${CMAKE_CURRENT_BINARY_DIR}/include
)

OPTION(WITH_NDB_TEST
  "Include the NDB Cluster ndbapi test programs" OFF)
IF(WITH_NDB_TEST)
  MESSAGE(STATUS "Building NDB test programs")
ELSE()
  # Turn on NDB tests automatically for gcov builds
  IF(ENABLE_GCOV AND NOT WIN32 AND NOT APPLE)
    MESSAGE(STATUS "Building NDB test programs for improved gcov coverage")
    SET(WITH_NDB_TEST ON)
    MESSAGE(STATUS "Building NdbApi examples for improved gcov coverage")
    SET(WITH_NDBAPI_EXAMPLES ON)
  ENDIF()
  # Turn on NDB tests automatically for valgrind builds
  IF(WITH_VALGRIND)
    MESSAGE(STATUS "Building NDB test programs for improved valgrind coverage")
    SET(WITH_NDB_TEST ON)
    MESSAGE(STATUS "Building NdbApi examples for improved valgrind coverage")
    SET(WITH_NDBAPI_EXAMPLES ON)
  ENDIF()
  # Turn on NDB tests automatically for "-devbld" debug builds
  # to increase test coverage
  IF(PACKAGE_SUFFIX STREQUAL "-devbld" AND
     CMAKE_BUILD_TYPE_UPPER STREQUAL "DEBUG")
    MESSAGE(STATUS "Building NDB test programs for \"-devbld\" debug build")
    SET(WITH_NDB_TEST ON)
    MESSAGE(STATUS "Building NdbApi examples for \"-devbld\" debug build")
    SET(WITH_NDBAPI_EXAMPLES ON)
  ENDIF()
  # Turn on build of NDB tests automatically for "-werror" builds
  IF(PACKAGE_SUFFIX STREQUAL "-werror")
    MESSAGE(STATUS "Building NDB test programs for \"-werror\" build")
    SET(WITH_NDB_TEST ON)
    MESSAGE(STATUS "Building NdbApi examples for \"-werror\" build")
    SET(WITH_NDBAPI_EXAMPLES ON)
  ENDIF()
ENDIF()

SET(DEBUG_FLAGS
  -DACC_SAFE_QUEUE
  -DAPI_TRACE
  -DARRAY_GUARD
  -DERROR_INSERT
  -DNDB_DEBUG
  -DVM_TRACE
  )
SET(DEBUG_FLAGS_STRING "")
FOREACH(flag ${DEBUG_FLAGS})
  STRING_APPEND(DEBUG_FLAGS_STRING " ${flag}")
ENDFOREACH()

IF(APPLE_ARM)
  # Do not add any flags, ndbmtd gets signal 10 = sigbus
  # Without any additional flags, ~10 out of 200 tests fail.
ELSE()
  STRING_APPEND(CMAKE_C_FLAGS_DEBUG "${DEBUG_FLAGS_STRING}")
  STRING_APPEND(CMAKE_CXX_FLAGS_DEBUG "${DEBUG_FLAGS_STRING}")
ENDIF()

OPTION(WITH_ERROR_INSERT
  "Enable error injection in NDB" OFF)
IF(WITH_ERROR_INSERT)
  ADD_DEFINITIONS(-DERROR_INSERT)
ENDIF()

OPTION(WITH_NDB_DEBUG
  "Enable special ndb NDB features(including ERROR_INSERT)" OFF)
IF(WITH_NDB_DEBUG)
  ADD_DEFINITIONS(${DEBUG_FLAGS})
ENDIF()

# Set extra compiler switches from WITH_NDB_CCFLAGS
SET(WITH_NDB_CCFLAGS "" CACHE STRING "Extra compiler flags for NDB")
IF(WITH_NDB_CCFLAGS)
  MESSAGE(STATUS
    "Appending extra compiler flags for NDB: \"${WITH_NDB_CCFLAGS}\"")
  STRING_APPEND(CMAKE_C_FLAGS   " ${WITH_NDB_CCFLAGS}")
  STRING_APPEND(CMAKE_CXX_FLAGS " ${WITH_NDB_CCFLAGS}")
ENDIF()

# Set extra compiler switches from environment variable NDB_EXTRA_FLAGS
SET(ndb_flags_env $ENV{NDB_EXTRA_FLAGS})
IF(ndb_flags_env)
  MESSAGE(STATUS "Appending extra compiler flags for NDB from "
                 "environment: \"${ndb_flags_env}\"")
  STRING_APPEND(CMAKE_C_FLAGS   " ${ndb_flags_env}")
  STRING_APPEND(CMAKE_CXX_FLAGS " ${ndb_flags_env}")
ENDIF()

# Moved here from configure.cmake
CHECK_FUNCTION_EXISTS (getrlimit HAVE_GETRLIMIT)

SET(HAVE_JAVA FALSE)
SET(HAVE_JDK FALSE)

IF(APPLE_ARM)
  SET(WITH_NDB_JAVA_DEFAULT OFF)
  MESSAGE(STATUS "Default value for WITH_NDB_JAVA set to OFF")
ELSE()
  SET(WITH_NDB_JAVA_DEFAULT ON)
ENDIF()

# Check if the default for WITH_NDB_JAVA should be overriden
# by environment variable
IF (DEFINED ENV{WITH_NDB_JAVA_DEFAULT})
  SET(WITH_NDB_JAVA_DEFAULT $ENV{WITH_NDB_JAVA_DEFAULT})
  MESSAGE(STATUS
    "Default value for WITH_NDB_JAVA set to ${WITH_NDB_JAVA_DEFAULT}")
ELSE()
  # Check if the default for WITH_NDB_JAVA should be
  # magically overriden
  SET(magic_value $ENV{PRODUCT_ID})
  IF (magic_value STREQUAL "bootstrap-cmake-tar-gz" OR
      magic_value STREQUAL "source-dist-tar-gz")
    SET(WITH_NDB_JAVA_DEFAULT OFF)
    MESSAGE(STATUS "Default value for WITH_NDB_JAVA set to 0 since "
                   "bootstrapping was detected")
  ENDIF()
  IF(NOT MYSQL_CLUSTER_VERSION)
    # Don't build the Java parts of NDB by default in MySQL Server
    # (this lowers the threshold for checking that things
    #  work also with engine = NDB)
    SET(WITH_NDB_JAVA_DEFAULT OFF)
    MESSAGE(STATUS "Default value for WITH_NDB_JAVA set to 0 in "
                   "MySQL Server version")
  ENDIF()
ENDIF()
OPTION(WITH_NDB_JAVA
  "Include NDB Cluster Java components" ${WITH_NDB_JAVA_DEFAULT})
IF(WITH_ASAN)
  MESSAGE(STATUS "Skip building ClusterJ for ASAN builds")
ELSEIF(WITH_NDB_JAVA)
  #
  # Check for Java and JDK needed by ndbjtie and clusterj
  #

  # Print value of JAVA_HOME if set
  IF(DEFINED ENV{JAVA_HOME})
    MESSAGE(STATUS "Looking for Java in JAVA_HOME=" $ENV{JAVA_HOME} " "
                   "and standard locations")
  ELSE()
    MESSAGE(STATUS "Looking for Java in standard locations")
  ENDIF()

  FIND_PACKAGE(Java 1.8 COMPONENTS Development)
  IF(NOT JAVA_FOUND)
    IF(DEFINED ENV{JAVA_HOME})
      # Could not find Java in the specific location set by JAVA_HOME
      # or in standard paths, don't search further
      MESSAGE(FATAL_ERROR "Could NOT find Java: neither in specified "
                          "JAVA_HOME=" $ENV{JAVA_HOME} " or standard location")
    ENDIF()

    #
    # Continue looking for Java in some additional
    # well known locations
    #

    # Prefer Java with same bit size as current build
    SET(_bit_suffix)
    IF(CMAKE_SIZEOF_VOID_P EQUAL 8)
      SET(_bit_suffix "-64")
    ENDIF()

    # Use well known standard base
    SET(_base_path /usr/local/java/)
    IF(WINDOWS)
      SET(_base_path C:\\java\\)
    ENDIF()

    # Search for version in specified order
    SET(_preferred_versions 1.8)

    FOREACH(_version ${_preferred_versions})
      SET(_path ${_base_path}jdk${_version}${_bit_suffix})
      MESSAGE(STATUS "Looking for Java in ${_path}...")
      SET(ENV{JAVA_HOME} ${_path})
      FIND_PACKAGE(Java ${_version} COMPONENTS Development)
      IF(JAVA_FOUND)
        # Found java, no need to search further
        MESSAGE(STATUS "Found Java in ${_path}")
        BREAK()
      ENDIF()
    ENDFOREACH()

    IF(NOT JAVA_FOUND)
      # Could not find Java in well known locations either
      MESSAGE(FATAL_ERROR "Could NOT find suitable version of Java")
    ENDIF()

  ENDIF()

  MESSAGE(STATUS "Java_VERSION: ${Java_VERSION}")
  MESSAGE(STATUS "Java_VERSION_STRING: ${Java_VERSION_STRING}")
  MESSAGE(STATUS "JAVA_RUNTIME: ${JAVA_RUNTIME}")
  MESSAGE(STATUS "JAVA_COMPILE: ${JAVA_COMPILE}")
  MESSAGE(STATUS "JAVA_ARCHIVE: ${JAVA_ARCHIVE}")
  NDB_REQUIRE_VARIABLE(JAVA_RUNTIME)
  NDB_REQUIRE_VARIABLE(JAVA_COMPILE)
  NDB_REQUIRE_VARIABLE(JAVA_ARCHIVE)

  # Help FindJNI by setting JAVA_HOME (if not already set)
  # to point at the java found above
  IF(NOT DEFINED ENV{JAVA_HOME})
    # Convert to realpath
    GET_FILENAME_COMPONENT(java_home ${JAVA_COMPILE} REALPATH)
    # Remove filename
    GET_FILENAME_COMPONENT(java_home ${java_home} PATH)
    # Remove dir
    GET_FILENAME_COMPONENT(java_home ${java_home} PATH)
    MESSAGE(STATUS "Setting JAVA_HOME=${java_home}")
    SET(ENV{JAVA_HOME} ${java_home})
  ENDIF()

  FIND_PACKAGE(JNI REQUIRED)
  MESSAGE(STATUS "JNI_FOUND: ${JNI_FOUND}")
  MESSAGE(STATUS "JNI_INCLUDE_DIRS: ${JNI_INCLUDE_DIRS}")
  MESSAGE(STATUS "JNI_LIBRARIES: ${JNI_LIBRARIES}")
  NDB_REQUIRE_VARIABLE(JNI_INCLUDE_DIRS)

  INCLUDE("${CMAKE_CURRENT_SOURCE_DIR}/config/type_JAVA.cmake")
  SET(HAVE_JAVA TRUE)
  SET(HAVE_JDK TRUE)
  SET(WITH_CLASSPATH ${WITH_CLASSPATH} CACHE STRING
    "Enable the classpath for MySQL Cluster Java Connector")
ELSE(WITH_NDB_JAVA)
  MESSAGE(STATUS "Excluding Cluster Java components")
ENDIF(WITH_ASAN)

ADD_SUBDIRECTORY(include)
ADD_SUBDIRECTORY(src)
ADD_SUBDIRECTORY(tools)
ADD_SUBDIRECTORY(test)
IF(WITH_NDB_TEST)
  ADD_SUBDIRECTORY(src/cw/cpcd)
ENDIF()
IF (HAVE_JAVA AND HAVE_JDK)
  ADD_SUBDIRECTORY(clusterj)
ENDIF()

SET(WITH_NDB_NODEJS_DEFAULT ON)

# Check if the default for WITH_NDB_NODEJS should be overriden
# by environment variable
IF(DEFINED ENV{WITH_NDB_NODEJS_DEFAULT})
  SET(WITH_NDB_NODEJS_DEFAULT $ENV{WITH_NDB_NODEJS_DEFAULT})
  MESSAGE(STATUS
    "Default value for WITH_NDB_NODEJS set to ${WITH_NDB_NODEJS_DEFAULT}")
ENDIF(DEFINED ENV{WITH_NDB_NODEJS_DEFAULT})

OPTION(WITH_NDB_NODEJS
  "Include NDB Cluster node.js components" ${WITH_NDB_NODEJS_DEFAULT})
IF(WITH_NDB_NODEJS)
  ADD_SUBDIRECTORY(nodejs)
ENDIF(WITH_NDB_NODEJS)

OPTION(WITH_NDBAPI_EXAMPLES "Build the NdbApi example programs" OFF)
IF(WITH_NDBAPI_EXAMPLES)
  MESSAGE(STATUS "Building the NdbApi example programs")
  ADD_SUBDIRECTORY(ndbapi-examples)
ENDIF()

ADD_SUBDIRECTORY(wix)
