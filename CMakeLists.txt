cmake_minimum_required (VERSION 3.28)
set(CMAKE_DISABLE_SOURCE_CHANGES ON)
set(CMAKE_DISABLE_IN_SOURCE_BUILD ON)

# ----- Start icu project ----- 

if (POLICY CMP0135)
	# policy CMP0135 will use OLD behavior for consistency
	cmake_policy(SET CMP0135 OLD)
endif()

# Check if icu is being used directly or via add_subdirectory, but allow overriding
if(NOT DEFINED ICU_MASTER_PROJECT)
		if(CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
				set(ICU_MASTER_PROJECT ON)
		else()
				set(ICU_MASTER_PROJECT OFF)
		endif()
endif()

# ----- Compiler config ----- 

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

include(FetchContent)
FetchContent_Declare(
	icu	
	GIT_REPOSITORY https://github.com/unicode-org/icu
	GIT_TAG main
	GIT_SHALLOW ON
	GIT_PROGRESS ON
	CONFIGURE_COMMAND ""
	BUILD_COMMAND ""
)
FetchContent_MakeAvailable(icu)

include(cmake/utils.cmake)
icu_extract_version()

project (icu-cmake VERSION ${ICU_VERSION} LANGUAGES CXX)
message(STATUS "Build icu: ${ICU_VERSION}")


# Set the compiler to support utf8
if (CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
	add_compile_options(-finput-charset=UTF-8)
elseif (CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
	add_compile_options(/utf-8)
endif()

if(MSVC)
	string(APPEND CMAKE_CXX_FLAGS " /Zc:__cplusplus /MP")
endif()


# ----- Setting -----

option(ICU_BUILD_UC "Build icu::uc" ON)
option(ICU_BUILD_I18N "Build icu::i18n" ON)
option(ICU_BUILD_IO "Build icu::io" ON)
option(ICU_BUILD_DEBUG_INFO "Open debug info" OFF)
option(ICU_INSTALL "Generate the install target" ${ICU_MASTER_PROJECT})


# ----- Install setting -----

option(ICU_NO_INSTALL "Using libicu without installation" OFF)

# ----- Info -----

message(STATUS "ICU source path: ${icu_SOURCE_DIR}")
message(STATUS "Is build icu::uc: ${ICU_BUILD_UC}")
message(STATUS "Is build icu::i18n: ${ICU_BUILD_I18N}")
message(STATUS "Is build icu::io: ${ICU_BUILD_IO}")
message(STATUS "Is build icu shared library: ${BUILD_SHARED_LIBS}")
message(STATUS "Is open debug info: ${ICU_BUILD_DEBUG_INFO}")
message(STATUS "Is generate the install target: ${ICU_INSTALL}")


# ----- Find source file -----

set(STUBDATA_SRC "${icu_SOURCE_DIR}/icu4c/source/stubdata/stubdata.cpp")
if(ICU_BUILD_UC)
	append_prefix_to_file_list("${icu_SOURCE_DIR}/icu4c/source/common/sources.txt" UC_SOURCE_FILE_LIST "${icu_SOURCE_DIR}/icu4c/source/common/")
	if(ICU_BUILD_DEBUG_INFO)
		foreach(file ${UC_SOURCE_FILE_LIST})
			message("icu::uc source file: ${file}")
		endforeach()
	endif()
endif()
if(ICU_BUILD_I18N)
	append_prefix_to_file_list("${icu_SOURCE_DIR}/icu4c/source/i18n/sources.txt" I18N_SOURCE_FILE_LIST "${icu_SOURCE_DIR}/icu4c/source/i18n/")
	if(ICU_BUILD_DEBUG_INFO)
		foreach(file ${I18N_SOURCE_FILE_LIST})
			message("icu::i18n source file: ${file}")
		endforeach()
	endif()
endif()
if(ICU_BUILD_IO)
	append_prefix_to_file_list("${icu_SOURCE_DIR}/icu4c/source/io/sources.txt" IO_SOURCE_FILE_LIST "${icu_SOURCE_DIR}/icu4c/source/io/")
	if(ICU_BUILD_DEBUG_INFO)
		foreach(file ${IO_SOURCE_FILE_LIST})
			message("icu::io source file: ${file}")
		endforeach()
	endif()
endif()


# ----- Build -----

add_library(icu INTERFACE)

# add library
if(ICU_BUILD_UC)
	add_library(uc ${UC_SOURCE_FILE_LIST} ${STUBDATA_SRC})
	add_library(icu::uc ALIAS uc)
	target_compile_definitions(uc PRIVATE U_COMMON_IMPLEMENTATION)
endif()
if(ICU_BUILD_I18N)
	add_library(i18n ${I18N_SOURCE_FILE_LIST})
	add_library(icu::i18n ALIAS i18n)
	target_compile_definitions(i18n PRIVATE U_I18N_IMPLEMENTATION)
endif()
if(ICU_BUILD_IO)
	add_library(io ${IO_SOURCE_FILE_LIST})
	add_library(icu::io ALIAS io)
	target_compile_definitions(io PRIVATE U_IO_IMPLEMENTATION)
endif()

# set target properties
if (NOT BUILD_SHARED_LIBS)
	target_compile_definitions(uc PRIVATE U_STATIC_IMPLEMENTATION)
	set_target_properties(
		uc
		PROPERTIES
		INTERFACE_COMPILE_DEFINITIONS "U_STATIC_IMPLEMENTATION")
	target_compile_definitions(i18n PRIVATE U_STATIC_IMPLEMENTATION)
	set_target_properties(
		i18n
		PROPERTIES
		INTERFACE_COMPILE_DEFINITIONS "U_STATIC_IMPLEMENTATION")
	target_compile_definitions(io PRIVATE U_STATIC_IMPLEMENTATION)
	set_target_properties(
		io
		PROPERTIES
		INTERFACE_COMPILE_DEFINITIONS "U_STATIC_IMPLEMENTATION")
endif()	

# ----- Set include dir -----

# Search for .h files in the directory
file(GLOB HEADER_FILES "${icu_SOURCE_DIR}/icu4c/source/common/unicode/*.h")
 
# Iterate over each .h file and copy to the installation directory
foreach(header ${HEADER_FILES})
	# Copy file to CMAKE_CURRENT_BINARY_DIR
	file(COPY ${header} DESTINATION "${CMAKE_CURRENT_BINARY_DIR}/include/unicode")
endforeach()
# Search for .h files in the directory
file(GLOB HEADER_FILES "${icu_SOURCE_DIR}/icu4c/source/i18n/unicode/*.h")

# Iterate over each .h file and copy to the installation directory
foreach(header ${HEADER_FILES})
	# Copy file to CMAKE_CURRENT_BINARY_DIR
	file(COPY ${header} DESTINATION "${CMAKE_CURRENT_BINARY_DIR}/include/unicode")
endforeach()
# Search for .h files in the directory
file(GLOB HEADER_FILES "${icu_SOURCE_DIR}/icu4c/source/io/unicode/*.h")

# Iterate over each .h file and copy to the installation directory
foreach(header ${HEADER_FILES})

	# Copy file to CMAKE_CURRENT_BINARY_DIR
	file(COPY ${header} DESTINATION "${CMAKE_CURRENT_BINARY_DIR}/include/unicode")
endforeach()

target_include_directories(uc PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/include")
target_include_directories(uc PUBLIC "$<BUILD_INTERFACE:${icu_SOURCE_DIR}/icu4c/source/common>"
	"$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>")
target_include_directories(i18n PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/include")
target_include_directories(i18n PUBLIC "$<BUILD_INTERFACE:${icu_SOURCE_DIR}/icu4c/source/i18n>"
	"$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>")
target_include_directories(io PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/include")
target_include_directories(io PUBLIC "$<BUILD_INTERFACE:${icu_SOURCE_DIR}/icu4c/source/io>"
	"$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>")


# ----- Link -----

add_dependencies(i18n uc)
target_link_libraries(i18n uc)
add_dependencies(io uc i18n)
target_link_libraries(io uc i18n)


# ----- Install -----

if(ICU_INSTALL)
	message(STATUS "icu generating install")
	set(project_config_in "${CMAKE_CURRENT_LIST_DIR}/cmake/ICUConfig.cmake.in")
	set(project_config_out "${CMAKE_CURRENT_BINARY_DIR}/ICUConfig.cmake")
	set(version_config_file "${CMAKE_CURRENT_BINARY_DIR}/ICUConfigVersion.cmake")
	set(config_targets_file "ICUConfigTargets.cmake")
	set(export_dest_dir "${CMAKE_INSTALL_LIBDIR}/cmake/icu")

	# Install include files
	install(DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/include/unicode" DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")

	# Install target
	install(
		TARGETS uc
		EXPORT ICU
		LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
		ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
		RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
		INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
	)
	
	install(
		TARGETS i18n
		EXPORT ICU
		LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
		ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
		RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
		INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
	)
	
	install(
		TARGETS io
		EXPORT ICU
		LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
		ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
		RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
		INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
	)
	
	# Export target
	export(EXPORT ICU NAMESPACE icu:: FILE "${config_targets_file}")

	#Install ICUTargets.cmake
	install(EXPORT ICU DESTINATION "${export_dest_dir}" NAMESPACE icu:: FILE "${config_targets_file}")
	
	#Install CMake config files
	include(CMakePackageConfigHelpers)

	configure_package_config_file("${project_config_in}" "${project_config_out}" INSTALL_DESTINATION "${config_targets_file}")
	write_basic_package_version_file("${version_config_file}" COMPATIBILITY SameMajorVersion)

	#Install ICU-config.cmake
	install(FILES "${project_config_out}" "${version_config_file}" DESTINATION "${export_dest_dir}")

endif()
