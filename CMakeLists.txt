CMAKE_MINIMUM_REQUIRED(VERSION 3.0)
include("cmake/HunterGate.cmake")
HunterGate(
    URL "https://github.com/ruslo/hunter/archive/v0.11.8.tar.gz"
    SHA1 "97e296c800ea792d98231ef007bfa1c4fca9ec6d"
)
PROJECT(SimpleAmqpClient VERSION 6.0.4 LANGUAGES C CXX)

# Follow all steps below in order to calculate new ABI version when updating the library
# NOTE: THIS IS UNRELATED to the actual project version
#
# 1. If the library source code has changed at all since the last update, then increment revision
# 2. If any interfaces have been added, removed, or changed since the last update, increment
#    current and set revision to 0.
# 3. If any interfaces have been added since the last public release, then increment age.
# 4. If any interfaces have been removed since the last public release, then set age to 0.

set(SAC_SOVERSION_CURRENT   6)
set(SAC_SOVERSION_REVISION  0)
set(SAC_SOVERSION_AGE       4)

math(EXPR SAC_SOVERSION_MAJOR "${SAC_SOVERSION_CURRENT} - ${SAC_SOVERSION_AGE}")
math(EXPR SAC_SOVERSION_MINOR "${SAC_SOVERSION_AGE}")
math(EXPR SAC_SOVERSION_PATCH "${SAC_SOVERSION_REVISION}")

set(SAC_VERSION ${SAC_SOVERSION_MAJOR}.${SAC_SOVERSION_MINOR}.${SAC_SOVERSION_PATCH})
set(SAC_SOVERSION ${SAC_SOVERSION_MAJOR})

file(STRINGS src/SimpleAmqpClient/Version.h _API_VERSION_MAJOR REGEX "^#define SIMPLEAMQPCLIENT_VERSION_MAJOR [0-9]+$")
file(STRINGS src/SimpleAmqpClient/Version.h _API_VERSION_MINOR REGEX "^#define SIMPLEAMQPCLIENT_VERSION_MINOR [0-9]+$")
file(STRINGS src/SimpleAmqpClient/Version.h _API_VERSION_PATCH REGEX "^#define SIMPLEAMQPCLIENT_VERSION_PATCH [0-9]+$")

string(REGEX MATCH "[0-9]+" _API_VERSION_MAJOR ${_API_VERSION_MAJOR})
string(REGEX MATCH "[0-9]+" _API_VERSION_MINOR ${_API_VERSION_MINOR})
string(REGEX MATCH "[0-9]+" _API_VERSION_PATCH ${_API_VERSION_PATCH})

set(SAC_APIVERSION ${_API_VERSION_MAJOR}.${_API_VERSION_MINOR}.${_API_VERSION_PATCH})

hunter_add_package(Boost COMPONENTS chrono system)
FIND_PACKAGE(Boost CONFIG REQUIRED chrono system)

hunter_add_package(rabbitmq-c)

FIND_PACKAGE(rabbitmq-c REQUIRED)
add_definitions(-DAMQP_STATIC)
option(ENABLE_SSL_SUPPORT "Enable SSL support." ON)

if (ENABLE_SSL_SUPPORT)
  add_definitions(-DSAC_SSL_SUPPORT_ENABLED)
endif()

INCLUDE_DIRECTORIES(BEFORE src
    ${CMAKE_CURRENT_BINARY_DIR})

option(BUILD_SHARED_LIBS "Build SimpleAmqpClient as a shared library" OFF)

SET(SAC_LIB_SRCS
    src/SimpleAmqpClient/SimpleAmqpClient.h

    src/SimpleAmqpClient/AmqpException.h
    src/AmqpException.cpp

    src/SimpleAmqpClient/Channel.h
    src/Channel.cpp

    src/SimpleAmqpClient/ChannelImpl.h
    src/ChannelImpl.cpp

    src/SimpleAmqpClient/BasicMessage.h
    src/BasicMessage.cpp

    src/SimpleAmqpClient/Util.h

    src/SimpleAmqpClient/AmqpResponseLibraryException.h
    src/AmqpResponseLibraryException.cpp

    src/SimpleAmqpClient/BadUriException.h
    src/SimpleAmqpClient/ConnectionClosedException.h
    src/SimpleAmqpClient/ConsumerTagNotFoundException.h

    src/SimpleAmqpClient/Envelope.h
    src/Envelope.cpp

    src/SimpleAmqpClient/MessageReturnedException.h
    src/MessageReturnedException.cpp

    src/SimpleAmqpClient/Table.h
    src/Table.cpp

    src/SimpleAmqpClient/TableImpl.h
    src/TableImpl.cpp
    )


ADD_LIBRARY(SimpleAmqpClient ${SAC_LIB_SRCS})
TARGET_LINK_LIBRARIES(SimpleAmqpClient Boost::chrono Boost::system rabbitmq-c::rabbitmq-static Crypt32 Ws2_32)

if (WIN32)
  set_target_properties(SimpleAmqpClient PROPERTIES VERSION ${SAC_VERSION} OUTPUT_NAME SimpleAmqpClient.${SAC_SOVERSION})
else ()
  set_target_properties(SimpleAmqpClient PROPERTIES VERSION ${SAC_VERSION} SOVERSION ${SAC_SOVERSION})
endif ()

option(ENABLE_TESTING "Enable smoke tests" ON)

if (ENABLE_TESTING)
    enable_testing()
    add_subdirectory(third-party/gtest-1.7.0)
    add_subdirectory(testing)
endif (ENABLE_TESTING)

INSTALL(TARGETS SimpleAmqpClient EXPORT SimpleAmqpClientTargets
    RUNTIME DESTINATION bin
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    INCLUDES DESTINATION include
    )

INSTALL(FILES
    src/SimpleAmqpClient/AmqpException.h
    src/SimpleAmqpClient/AmqpResponseLibraryException.h
    src/SimpleAmqpClient/BadUriException.h
    src/SimpleAmqpClient/BasicMessage.h
    src/SimpleAmqpClient/Channel.h
    src/SimpleAmqpClient/ConnectionClosedException.h
    src/SimpleAmqpClient/ConsumerCancelledException.h
    src/SimpleAmqpClient/ConsumerTagNotFoundException.h
    src/SimpleAmqpClient/Envelope.h
    src/SimpleAmqpClient/MessageReturnedException.h
    src/SimpleAmqpClient/SimpleAmqpClient.h
    src/SimpleAmqpClient/Table.h
    src/SimpleAmqpClient/Util.h
    src/SimpleAmqpClient/Version.h
    DESTINATION include/SimpleAmqpClient
    )

set(prefix ${CMAKE_INSTALL_PREFIX})
set(exec_prefix "\${prefix}")
set(libdir "\${exec_prefix}/lib")
set(includedir "\${prefix}/include")

set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(include_install_dir "include")
set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(targets_export_name "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

include(CMakePackageConfigHelpers)

configure_file("cmake/Config.cmake.in" "${project_config}" @ONLY)
    
# Note: PROJECT_VERSION is used as a VERSION
write_basic_package_version_file(
  "${version_config}" COMPATIBILITY SameMajorVersion
)

# Note: variable 'targets_export_name' used
configure_file("cmake/Config.cmake.in" "${project_config}" @ONLY)

install(
  FILES "${project_config}" "${version_config}"
  DESTINATION "${config_install_dir}"
)

install(
  EXPORT "${targets_export_name}"
  NAMESPACE "${namespace}"
  DESTINATION "${config_install_dir}"
  )
