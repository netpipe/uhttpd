cmake_minimum_required(VERSION 2.6)

PROJECT(uhttpd C)

INCLUDE (CheckFunctionExists)

SET(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS "")
ADD_DEFINITIONS(-D_FILE_OFFSET_BITS=64 -Os -Wall -Werror -Wmissing-declarations --std=gnu99 -g3)

OPTION(TLS_SUPPORT "TLS support" ON)
OPTION(LUA_SUPPORT "Lua support" OFF)
OPTION(UBUS_SUPPORT "ubus support" ON)
OPTION(UCODE_SUPPORT "ucode support" OFF)

IF(APPLE)
  INCLUDE_DIRECTORIES(/opt/local/include)
  LINK_DIRECTORIES(/opt/local/lib)
ENDIF()
 INCLUDE_DIRECTORIES(./)
  INCLUDE_DIRECTORIES(./ubus)
   INCLUDE_DIRECTORIES(./ustream-ssl)
  LINK_DIRECTORIES(./libubus)
  LINK_DIRECTORIES(./libubox)
  
  
FIND_LIBRARY(LIBS crypt)
IF(LIBS STREQUAL "LIBS-NOTFOUND")
	SET(LIBS "")
ENDIF()

FIND_PATH(ubox_include_dir libubox/usock.h)
INCLUDE_DIRECTORIES(${ubox_include_dir})

SET(SOURCES main.c listen.c client.c utils.c file.c auth.c cgi.c relay.c proc.c plugin.c handler.c)
IF(TLS_SUPPORT)
	SET(SOURCES ${SOURCES} tls.c)
	ADD_DEFINITIONS(-DHAVE_TLS)
ENDIF()

CHECK_FUNCTION_EXISTS(getspnam HAVE_SHADOW)
IF(HAVE_SHADOW)
    ADD_DEFINITIONS(-DHAVE_SHADOW)
ENDIF()

FIND_LIBRARY(ubox NAMES ubox)
FIND_LIBRARY(json_script NAMES json_script)
FIND_LIBRARY(blobmsg_json NAMES blobmsg_json)

ADD_EXECUTABLE(uhttpd ${SOURCES})
FIND_LIBRARY(libjson NAMES json-c json)
TARGET_LINK_LIBRARIES(uhttpd ubox dl json_script blobmsg_json ${LIBS})

SET(PLUGINS "")
IF(LUA_SUPPORT)
	FIND_PROGRAM(PKG_CONFIG pkg-config)

	IF(NOT LUA_CFLAGS AND PKG_CONFIG)
		EXECUTE_PROCESS(
			COMMAND pkg-config --silence-errors --cflags lua5.1
			OUTPUT_VARIABLE LUA_CFLAGS
			OUTPUT_STRIP_TRAILING_WHITESPACE
		)
	ENDIF()

	IF(NOT LUA_LIBS AND PKG_CONFIG)
		EXECUTE_PROCESS(
			COMMAND pkg-config --silence-errors --libs lua5.1
			OUTPUT_VARIABLE LUA_LIBS
			OUTPUT_STRIP_TRAILING_WHITESPACE
		)
	ENDIF()

	IF(NOT LUA_LIBS)
		SET(LUA_LIBS "lua")
	ENDIF()

	SET(PLUGINS ${PLUGINS} uhttpd_lua)
	ADD_DEFINITIONS(-DHAVE_LUA ${LUA_CFLAGS})
	ADD_LIBRARY(uhttpd_lua MODULE lua.c)
	TARGET_LINK_LIBRARIES(uhttpd_lua ${LUA_LIBS} m dl)
ENDIF()

IF(UCODE_SUPPORT)
	SET(PLUGINS ${PLUGINS} uhttpd_ucode)
	ADD_DEFINITIONS(-DHAVE_UCODE)
	ADD_LIBRARY(uhttpd_ucode MODULE ucode.c)
	TARGET_LINK_LIBRARIES(uhttpd_ucode ucode ${libjson})
ENDIF()

IF(UBUS_SUPPORT)
 INCLUDE_DIRECTORIES(./)
  INCLUDE_DIRECTORIES(./ubus)
   INCLUDE_DIRECTORIES(./ustream-ssl)
  LINK_DIRECTORIES(./libubus)
  LINK_DIRECTORIES(./libubox)
	FIND_LIBRARY(ubus NAMES ubus)
	SET(PLUGINS ${PLUGINS} uhttpd_ubus)
	ADD_DEFINITIONS(-DHAVE_UBUS)
	ADD_LIBRARY(uhttpd_ubus MODULE ubus.c)
	TARGET_LINK_LIBRARIES(uhttpd_ubus ubus ubox blobmsg_json libjson)
ENDIF()

IF(PLUGINS)
	SET_TARGET_PROPERTIES(${PLUGINS} PROPERTIES
		PREFIX ""
	)
ENDIF()

INSTALL(TARGETS uhttpd ${PLUGINS}
	RUNTIME DESTINATION bin
	LIBRARY DESTINATION lib
)
