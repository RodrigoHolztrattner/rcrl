cmake_minimum_required(VERSION 3.1.1 FATAL_ERROR)

############
# INCLUDES #
############
include(FetchContent)

################
# DEPENDENCIES #
################

macro(PrintFetchMessage target_string)
	string(CONCAT end_string "Fetching " ${target_string})
	message(STATUS "")
	message(STATUS "################################")
	message(STATUS "# ${end_string}")
	message(STATUS "################################")
	message(STATUS "")
endmacro()

# tiny-process-library
FetchContent_Declare(
  tinyprocesslibrary
  GIT_REPOSITORY https://github.com/RodrigoHolztrattner/tiny-process-library
)

FetchContent_GetProperties(tinyprocesslibrary)
if(NOT tinyprocesslibrary_POPULATED)
  PrintFetchMessage("Tiny Process Library")
  FetchContent_Populate(tinyprocesslibrary)
  add_subdirectory(${tinyprocesslibrary_SOURCE_DIR} ${tinyprocesslibrary_BINARY_DIR})
endif()

##########
# CONFIG #
##########

set(PROJECT_NAME "RCRL")

###########
# PROJECT #
###########

SET(CMAKE_CXX_FLAGS "/Zi /sdl /MP /Oi /Gy /permissive- /GR- /TP /EHsc")
SET(CMAKE_CXX_FLAGS_DEBUG "/ZI /sdl /MP /JMC /permissive- /GR- /TP /EHsc /MDd /Od")

set(RCRL_FILES "src/rcrl")
include_directories(RCRL_FILES)

# Gather all files
file (GLOB_RECURSE PACKET_SOURCE_FILES ${RCRL_FILES}/*.cpp ${RCRL_FILES}/*.cc) 
file (GLOB_RECURSE PACKET_HEADER_FILES ${RCRL_FILES}/*.hpp ${RCRL_FILES}/*.h) 
set (ALL_RCRL_FILES ${PACKET_SOURCE_FILES} ${PACKET_HEADER_FILES})

# Setup the source groups
foreach(FILE ${ALL_RCRL_FILES}) 

    # Get the directory of the source file
    get_filename_component(PARENT_DIR "${FILE}" DIRECTORY)

    # Remove common directory prefix to make the group
    string(REPLACE "${CMAKE_CURRENT_SOURCE_DIR}/src/rcrl" "" GROUP "${PARENT_DIR}")

    # Make sure we are using windows slashes
    string(REPLACE "/" "\\" GROUP "${GROUP}")

    source_group("${GROUP}" FILES "${FILE}")
	
endforeach()

add_library(${PROJECT_NAME} STATIC
  ${ALL_RCRL_FILES})

target_include_directories(${PROJECT_NAME} PUBLIC ${tinyprocesslibrary_SOURCE_DIR})
target_link_libraries(${PROJECT_NAME} tiny-process-library)

# defines needed for RCRL integration
target_compile_definitions(${PROJECT_NAME} PRIVATE "RCRL_PLUGIN_FILE=\"${plugin_file}\"")
target_compile_definitions(${PROJECT_NAME} PRIVATE "RCRL_PLUGIN_NAME=\"plugin\"")
target_compile_definitions(${PROJECT_NAME} PRIVATE "RCRL_BUILD_FOLDER=\"${PROJECT_BINARY_DIR}\"")
target_compile_definitions(${PROJECT_NAME} PRIVATE "RCRL_BIN_FOLDER=\"$<TARGET_FILE_DIR:${PROJECT_NAME}>/\"")
target_compile_definitions(${PROJECT_NAME} PRIVATE "RCRL_EXTENSION=\"${CMAKE_SHARED_LIBRARY_SUFFIX}\"")
if(${CMAKE_GENERATOR} MATCHES "Visual Studio" OR ${CMAKE_GENERATOR} MATCHES "Xcode")
    target_compile_definitions(${PROJECT_NAME} PRIVATE "RCRL_CONFIG=\"$<CONFIG>\"")
endif()
# unimportant - to construct a path to the fonts in the third party imgui folder
target_compile_definitions(${PROJECT_NAME} PRIVATE "CMAKE_SOURCE_DIR=\"${CMAKE_SOURCE_DIR}\"")

###########
# INSTALL #
###########

include(GNUInstallDirs)

install(TARGETS ${PROJECT_NAME}
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  COMPONENT library)

set_target_properties(${PROJECT_NAME} PROPERTIES
  CXX_STANDARD 17
	CXX_STANDARD_REQUIRED ON)