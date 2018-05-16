CMAKE_MINIMUM_REQUIRED(VERSION 3.11)
PROJECT(GoodbyeDPI)

set(CMAKE_USE_RELATIVE_PATHS TRUE)

set(PACKAGE_NAME "GoodbyeDPI" CACHE STRING "Program name")
set(PACKAGE_VERSION ${VERSION})
set(PACKAGE_STRING "${PACKAGE_NAME} ${PACKAGE_VERSION}")

FIND_PACKAGE(Git)
if(GIT_FOUND)
  execute_process(
    COMMAND ${GIT_EXECUTABLE} describe --tags --dirty --long
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    OUTPUT_VARIABLE VERSION_FROM_GIT
    RESULT_VARIABLE GIT_RESULT
    ERROR_QUIET
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
  if(NOT GIT_RESULT EQUAL 0)
    message(WARNING "git returned error ${VERSION_FROM_GIT}")
    set(VERSION_FROM_GIT "unknown")
  endif()
else(GIT_FOUND)
  set(VERSION_FROM_GIT "v0.1.5")
endif(GIT_FOUND)


string(REGEX MATCH "^v?[0-9]+(\\.[0-9]+)+" VERSION ${VERSION_FROM_GIT})
string(REGEX MATCHALL "[0-9]+" PARSED_VER ${VERSION})

list(LENGTH PARSED_VER PARSED_VER_LEN)
if(PARSED_VER_LEN GREATER 0)
  list(GET PARSED_VER 0 VERSION_MAJOR)
else()
  set(VERSION_MAJOR 0)
endif()
if(PARSED_VER_LEN GREATER 1)
  list(GET PARSED_VER 1 VERSION_MINOR)
else()
  set(VERSION_MINOR 0)
endif()
if(PARSED_VER_LEN GREATER 2)
  list(GET PARSED_VER 2 VERSION_PATCH)
else()
  set(VERSION_PATCH 0)
endif()
if(PARSED_VER_LEN GREATER 3)
  list(GET PARSED_VER 3 VERSION_TWEAK)
else()
  set(VERSION_TWEAK 0)
endif()

set(VERSION_BIN "${VERSION_MAJOR}${VERSION_MINOR}${VERSION_PATCH}")
message(STATUS "GoodbyeDPI version: ${VERSION}")
message(STATUS "GoodbyeDPI bin version: ${VERSION_BIN}")

if(GIT_FOUND)
  execute_process(
    COMMAND ${GIT_EXECUTABLE} log -1 --pretty=format:%ct
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    OUTPUT_VARIABLE COMPTIME
    RESULT_VARIABLE GIT_RESULT
    ERROR_QUIET
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
  if(NOT GIT_RESULT EQUAL 0)
    set(COMPTIME "0000000000")
  endif()

  execute_process(
    COMMAND ${GIT_EXECUTABLE} log -1 --pretty=format:%D
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    OUTPUT_VARIABLE VERSION_EXPORT
    RESULT_VARIABLE GIT_RESULT
    ERROR_QUIET
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
  if(NOT GIT_RESULT EQUAL 0)
    set(VERSION_EXPORT "HEAD -> master")
  endif()
else(GIT_FOUND)
  set(COMPTIME "0000000000")
  set(VERSION_EXPORT "HEAD -> master")
endif(GIT_FOUND)
set(GoodbyeDPI_VERSION "${VERSION_EXPORT} ${VERSION_FROM_GIT}")
message(STATUS "version tag: ${GoodbyeDPI_VERSION}")

if(${CMAKE_SYSTEM_NAME} MATCHES "Darvin")
    set(MACOS TRUE)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    set(LINUX TRUE)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD")
    set(FREEBSD TRUE)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    set(WINDOWS TRUE)
endif()

set(Source_dir "${CMAKE_CURRENT_SOURCE_DIR}/src")
set(Utils_dir "${CMAKE_CURRENT_SOURCE_DIR}/src/utils")
set(WinDivert_dir "${CMAKE_CURRENT_SOURCE_DIR}/WinDivert")
set(CMake_Misc_Dir "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

file(GLOB_RECURSE SRCFILES "${Source_dir}/*.c" "${Source_dir}/*.cpp")

include_directories(GoodbyeDPI "${WinDivert_dir}/include")
if(MSVC)
    set(divertCompilerName "WDDK")
elseif(MINGW)
    set(divertCompilerName "MINGW")
endif()
set(divertLibraryPath "${WinDivert_dir}/install/${divertCompilerName}/${CMAKE_SYSTEM_PROCESSOR}/")
message(STATUS ${divertLibraryPath})

#making resource file
set(rc_script "${Source_dir}/goodbyedpi-rc.rc")
set(rc_script_proto "${CMake_Misc_Dir}/goodbyedpi-rc.rc.in")
set(license_file "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE")
file(READ ${license_file} LICENSE_TEXT)
string(REPLACE "\"" "\"\"" LICENSE_TEXT ${LICENSE_TEXT})
string(REPLACE "\r\n" "\n" LICENSE_TEXT ${LICENSE_TEXT})
string(REPLACE "\n" "\\r\\n" LICENSE_TEXT ${LICENSE_TEXT})
message(STATUS "generating resources: ${rc_script_proto} -> ${rc_script}")
configure_file(${rc_script_proto} ${rc_script})
file(GLOB_RECURSE resource_files "${Source_dir}/*.rc" "${Source_dir}/*.ico")


if(MSVC_IDE)
   file(GLOB_RECURSE HDRFILES "${Source_dir}/*.h")
endif()

# config.h
set(CONFIG_FILENAME "config.h")
set(CONFIG_FILE "${Source_dir}/${CONFIG_FILENAME}")
mark_as_advanced(CONFIG_FILE)
set(CONFIG_FILE_PROTO "${CMake_Misc_Dir}/${CONFIG_FILENAME}.in")

set(GoodbyeDPI_SERVICE_NAME "${PACKAGE_NAME}" CACHE STRING "Service name")

set(GoodbyeDPI_HOST_MAXLEN 253 CACHE INTEGER "Max length of a host")
set(GoodbyeDPI_MAX_FILTERS 4 CACHE INTEGER "Max length of a host")
set(GoodbyeDPI_MAX_PACKET_SIZE 9016 CACHE INTEGER "Max length of a host")
set(GoodbyeDPI_DEBUG ${GoodbyeDPI_DEBUG} CACHE BOOL "Enable debug output")

mark_as_advanced(GoodbyeDPI_SERVICE_NAME, GoodbyeDPI_HOST_MAXLEN)

message(STATUS "generating config: ${CONFIG_FILE_PROTO} -> ${CONFIG_FILE}")
configure_file(${CONFIG_FILE_PROTO} ${CONFIG_FILE})

add_executable(GoodbyeDPI "${SRCFILES}" "${HDRFILES}" "${resource_files}")
target_link_libraries(GoodbyeDPI "${divertLibraryPath}/WinDivert.dll")

if(MINGW)
    target_link_libraries(GoodbyeDPI "wsock32" "ws2_32" "ssp")
    set_target_properties(GoodbyeDPI PROPERTIES COMPILE_FLAGS "-Wall -Wextra -Wconversion -pie -Wformat -Werror=format-security -D_FORTIFY_SOURCE=2 -flto -mmitigate-rop -fstack-protector-strong -fno-common -fstack-check")
    #-mcet
    #-fsanitize=cfi
    set_target_properties(GoodbyeDPI PROPERTIES LINK_FLAGS "-Wl,-O1,--sort-common,--as-needed,--dynamicbase,--nxcompat,--export-all-symbols,-flto")
    
    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
        set_target_properties(GoodbyeDPI PROPERTIES LINK_FLAGS "-Wl,--image-base,0x140000000")
    endif()
elseif(MSVC)
  set_target_properties(GoodbyeDPI PROPERTIES COMPILE_FLAGS "/sdl /GS /SafeSEH /NXCOMPAT /dynamicbase /guard:cf /HIGHENTROPYVA")
  set_target_properties(GoodbyeDPI PROPERTIES LINK_FLAGS "/guard:cf")
else()
  message(FATAL_ERROR "The compiler is not supported")
endif()


set(RESOURCE_FILES
  resourcefile.txt
  appresourcedir/appres.txt
)


if(MSVC_IDE)
   source_group("res" FILES  ${resource_files})
endif()

if(MSVC)
  set_source_files_properties(${SRCFILES} PROPERTIES LANGUAGE CXX)
else(MSVC)
  #set_source_files_properties(${SRCFILES} PROPERTIES LANGUAGE CXX)
endif(MSVC)

#target_compile_features(GoodbyeDPI PUBLIC cxx_decltype_auto)