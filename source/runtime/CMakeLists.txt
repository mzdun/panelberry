cmake_minimum_required(VERSION 3.4)
project(runtime C CXX)

set(USE_SANDBOX OFF)
find_package(CEF)

add_subdirectory(${CEF_LIBCEF_DLL_WRAPPER_PATH} libcef_dll)

if (UNIX)
set(LAUNCHER_DIRSEP "/")
set(LAUNCHER_PATHSEP ":")
else()
set(LAUNCHER_DIRSEP "\\\\")
set(LAUNCHER_PATHSEP ";")
endif()

CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/version.h.in ${CMAKE_CURRENT_BINARY_DIR}/version.h)
CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/paths_runtime.h.in ${CMAKE_CURRENT_BINARY_DIR}/paths_runtime.h)

include_directories(${CMAKE_CURRENT_SOURCE_DIR})
include_directories(${CMAKE_CURRENT_BINARY_DIR})


set(CEF_TARGET runtime)

add_custom_command(
	OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/template_contents.h"
	WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
	COMMAND python "../../tools/make_template.py"
			--header
			"--version=${TOPLEVEL_VERSION_MAJOR}.${TOPLEVEL_VERSION_MINOR}.${TOPLEVEL_VERSION_PATCH}${TOPLEVEL_VERSION_STABILITY}+${TOPLEVEL_VERSION_BUILD}"
			-o "${CMAKE_CURRENT_BINARY_DIR}/template_contents.h"
	MAIN_DEPENDENCY "${CMAKE_CURRENT_SOURCE_DIR}/../../tools/make_template.py"
	DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/../res/kiosk.svg"
	COMMENT "Building response template header"
)

set(SRCS
	runtime.cc
	runtime.h
	embed.cc
	embed.h
	kiosk.cc
	kiosk.h
	template.cc
	template.h
	"${CMAKE_CURRENT_BINARY_DIR}/template_contents.h"
)
set(SRCS_LINUX
	main_runtime_linux.cc
	embed_linux.cc
	runtime_linux.cc
)
set(SRCS_WINDOWS
	runtime.exe.manifest
	runtime.rc
	main_runtime_win.cc
	embed_win.cc
	runtime_win.cc
)
APPEND_PLATFORM_SOURCES(SRCS)

ADD_LOGICAL_TARGET("libcef_lib" "${CEF_LIB_DEBUG}" "${CEF_LIB_RELEASE}")
SET_CEF_TARGET_OUT_DIR()

# List of CEF binary files.
set(CEF_BINBLOB_FILES
	natives_blob.bin
	snapshot_blob.bin
)

# List of CEF resource files.
set(CEF_RESOURCE_FILES
	cef.pak
	cef_100_percent.pak
	cef_200_percent.pak
	cef_extensions.pak
	devtools_resources.pak
)

set(CEF_LOCALES_FILES locales)

set(CEF_BINRES_FILES
	icudtl.dat
)

#
# Linux configuration.
#

if(OS_LINUX)
set(CEF_BINARY_FILES
	chrome-sandbox
	libcef.so
)

  # Executable target.
  add_executable(runtime ${SRCS})

  # Set rpath so that libraries can be placed next to the executable.
  set_target_properties(runtime PROPERTIES INSTALL_RPATH "$ORIGIN")
  set_target_properties(runtime PROPERTIES BUILD_WITH_INSTALL_RPATH TRUE)

  # Set SUID permissions on the chrome-sandbox target.
  # SET_LINUX_SUID_PERMISSIONS(runtime "${CEF_TARGET_OUT_DIR}/chrome-sandbox")
endif()

#
# Windows configuration.
#

if(OS_WINDOWS)
  # List of CEF binary files.
  set(CEF_BINARY_FILES
    chrome_elf.dll
    d3dcompiler_43.dll
    d3dcompiler_47.dll
    libcef.dll
    libEGL.dll
    libGLESv2.dll
    )
  # Executable target.
  add_executable(runtime WIN32 ${SRCS})

  # if(USE_SANDBOX)
  #   # Logical target used to link the cef_sandbox library.
  #   ADD_LOGICAL_TARGET("cef_sandbox_lib" "${CEF_SANDBOX_LIB_DEBUG}" "${CEF_SANDBOX_LIB_RELEASE}")
  #   target_link_libraries(runtime cef_sandbox_lib ${CEF_SANDBOX_STANDARD_LIBS})
  # endif()

  # Add the custom manifest files to the executable.
  add_custom_command(
    TARGET runtime
    POST_BUILD
    COMMAND "mt.exe" -nologo
            -manifest \"${CMAKE_CURRENT_SOURCE_DIR}/runtime.exe.manifest\" \"${CMAKE_CURRENT_SOURCE_DIR}/compatibility.manifest\"
            -outputresource:"${CMAKE_CURRENT_BINARY_DIR}/runtime.exe"\;\#1
    COMMENT "Adding manifest..."
    )
endif()

COPY_FILES(runtime "${CEF_RESOURCE_FILES}" "${CEF_RESOURCE_DIR}" "${PROJECT_BINARY_DIR}/../../${RESURCE_CMAKE_BINARY_DIR}")
COPY_FILES(runtime "${CEF_LOCALES_FILES}" "${CEF_RESOURCE_DIR}" "${PROJECT_BINARY_DIR}/../../${LOCALE_CMAKE_BINARY_DIR}")
COPY_FILES(runtime "${CEF_BINARY_FILES}" "${CEF_BINARY_DIR}" "${PROJECT_BINARY_DIR}/../../${CEF_CMAKE_BINARY_DIR}")
COPY_FILES(runtime "${CEF_BINBLOB_FILES}" "${CEF_BINARY_DIR}" "${PROJECT_BINARY_DIR}/../../${CEFDATA_CMAKE_BINARY_DIR}")
COPY_FILES(runtime "${CEF_BINRES_FILES}" "${CEF_RESOURCE_DIR}" "${PROJECT_BINARY_DIR}/../../${CEFDATA_CMAKE_BINARY_DIR}")

SET_EXECUTABLE_TARGET_PROPERTIES(runtime)
add_dependencies(runtime libcef_dll_wrapper)
target_link_libraries(runtime libcef_lib libcef_dll_wrapper ${CEF_STANDARD_LIBS})
set_target_properties(runtime PROPERTIES CXX_STANDARD 14)
add_dependencies(runtime kiosk-apps)

macro(INSTALL_CEF_FILES file_list source_dir target_dir)
  foreach(FILENAME ${file_list})
    set(source_file ${source_dir}/${FILENAME})
    get_filename_component(target_name ${FILENAME} NAME)
    set(target_file ${target_dir}/${target_name})
    get_filename_component(target_dir ${target_file} DIRECTORY)
    if(IS_DIRECTORY ${source_file})
		install(DIRECTORY "${source_file}" DESTINATION "${target_dir}")
    else()
		install(FILES "${source_file}" DESTINATION "${target_dir}")
    endif()
  endforeach()
endmacro()

install(TARGETS runtime DESTINATION "${SHARE_INST}")
INSTALL_CEF_FILES("${CEF_RESOURCE_FILES}" "${CEF_RESOURCE_DIR}" "${RES_INST}")
INSTALL_CEF_FILES("${CEF_LOCALES_FILES}" "${CEF_RESOURCE_DIR}" "${LOCALE_INST}")
INSTALL_CEF_FILES("${CEF_BINARY_FILES}" "${CEF_BINARY_DIR}" "${CEF_INST}")
INSTALL_CEF_FILES("${CEF_BINBLOB_FILES}" "${CEF_BINARY_DIR}" "${DATA_INST}")
INSTALL_CEF_FILES("${CEF_BINRES_FILES}" "${CEF_RESOURCE_DIR}" "${DATA_INST}")

PRINT_CEF_CONFIG()
