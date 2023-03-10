cmake_minimum_required(VERSION 2.6)

PROJECT(ubus C)
ADD_DEFINITIONS(-Os -Wall -Werror --std=gnu99 -g3 -Wmissing-declarations)

  INCLUDE_DIRECTORIES(../libubox)
  LINK_DIRECTORIES(../libubox)
  
OPTION(BUILD_LUA "build Lua plugin" OFF)
OPTION(BUILD_EXAMPLES "build examples" ON)
OPTION(ENABLE_SYSTEMD "systemd support" ON)

SET(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS "")
SET(UBUS_UNIX_SOCKET "/var/run/ubus.sock")
SET(UBUS_MAX_MSGLEN 1048576)

ADD_DEFINITIONS( -DUBUS_UNIX_SOCKET="${UBUS_UNIX_SOCKET}")
ADD_DEFINITIONS( -DUBUS_MAX_MSGLEN=${UBUS_MAX_MSGLEN})

IF(APPLE)
  INCLUDE_DIRECTORIES(/opt/local/include)
  LINK_DIRECTORIES(/opt/local/lib)
ENDIF()

ADD_LIBRARY(ubus SHARED libubus.c libubus-io.c libubus-obj.c libubus-sub.c libubus-req.c libubus-acl.c)
TARGET_LINK_LIBRARIES(ubus ubox)

ADD_EXECUTABLE(ubusd ubusd.c ubusd_id.c ubusd_obj.c ubusd_proto.c ubusd_event.c ubusd_acl.c ubusd_monitor.c)
TARGET_LINK_LIBRARIES(ubusd ubox blobmsg_json ${json})

find_library(json NAMES json-c json)
ADD_EXECUTABLE(cli cli.c)
SET_TARGET_PROPERTIES(cli PROPERTIES OUTPUT_NAME ubus)
TARGET_LINK_LIBRARIES(cli ubus ubox blobmsg_json ${json})

ADD_SUBDIRECTORY(lua)
ADD_SUBDIRECTORY(examples)

INSTALL(TARGETS ubus cli
	LIBRARY DESTINATION lib
	RUNTIME DESTINATION bin
)
INSTALL(TARGETS ubusd
	RUNTIME DESTINATION sbin
)

INSTALL(FILES ubusmsg.h ubus_common.h libubus.h DESTINATION include)

# FIXME: this works but certainly can be done better:
SET(UBUSD_BINARY "${CMAKE_INSTALL_PREFIX}/sbin/ubusd")

# do this after the installs so we have the proper paths
IF(ENABLE_SYSTEMD)
  INCLUDE(FindPkgConfig)
  PKG_CHECK_MODULES(SYSTEMD systemd)
  IF(SYSTEMD_FOUND)
    ADD_SUBDIRECTORY(systemd)
  ENDIF()
ENDIF()
