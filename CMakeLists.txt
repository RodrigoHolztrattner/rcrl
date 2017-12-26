cmake_minimum_required(VERSION 3.0.0)

project(rcrl)

set(OUTPUT_DIR ${PROJECT_BINARY_DIR}/bin/)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${OUTPUT_DIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${OUTPUT_DIR})
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${OUTPUT_DIR})

# latest c++ standards
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++1z -fvisibility=hidden")
else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /std:c++latest")
endif()

set(plugin_file ${PROJECT_BINARY_DIR}/plugin.cpp)
# touch the file so it exists
file(WRITE ${plugin_file} "")

set(additional_flags "")
if(MSVC AND NOT ${CMAKE_GENERATOR} STREQUAL "Ninja" AND NOT CMAKE_GENERATOR STREQUAL "NMake Makefiles")
    set(additional_flags "--config $(Configuration) -- /verbosity:quiet /consoleloggerparameters:PerformanceSummary")
endif()

#add_executable(host_app src/main.cpp src/visibility.h src/host_app.h)
add_executable(host_app src/main.cpp src/third_party/imgui/examples/opengl2_example/imgui_impl_glfw.cpp)
# so the plugin can link to the executable
set_target_properties(host_app PROPERTIES ENABLE_EXPORTS ON)
target_compile_definitions(host_app PRIVATE "CRCL_PLUGIN_FILE=\"${plugin_file}\"")
target_compile_definitions(host_app PRIVATE "CRCL_BUILD_FOLDER=\"${PROJECT_BINARY_DIR}\"")
target_compile_definitions(host_app PRIVATE "CRCL_HOST_APP")
target_compile_definitions(host_app PRIVATE "CRCL_ADDITIONAL_FLAGS=\"${additional_flags}\"")
target_include_directories(host_app PUBLIC src)
if(UNIX)
    target_link_libraries(host_app PRIVATE dl GL)
else()
    target_link_libraries(host_app PRIVATE opengl32)
endif()

# third party libs
set(GLFW_BUILD_DOCS     OFF CACHE BOOL "" FORCE)
set(GLFW_BUILD_TESTS    OFF CACHE BOOL "" FORCE)
set(GLFW_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(GLFW_INSTALL        OFF CACHE BOOL "" FORCE)
add_subdirectory(src/third_party/glfw)
add_library(imgui STATIC
    src/third_party/imgui/imgui.cpp
    src/third_party/imgui/imgui_draw.cpp
    src/third_party/imgui/imgui_demo.cpp
)

set_target_properties(glfw PROPERTIES FOLDER "third_party")
set_target_properties(imgui PROPERTIES FOLDER "third_party")

# link to third party libs
target_include_directories(host_app PRIVATE src/third_party/imgui)
target_link_libraries(host_app PRIVATE imgui glfw)

add_library(plugin SHARED EXCLUDE_FROM_ALL ${plugin_file})
# link with the host app so we can call stuff from it
target_link_libraries(plugin host_app)
# exclude it even for Visual Studio when building the whole solution (EXCLUDE_FROM_ALL is not enough)
set_target_properties(plugin PROPERTIES EXCLUDE_FROM_DEFAULT_BUILD 1)
# no 'lib' prefix for some compilers/platforms - simplifies my code
set_target_properties(plugin PROPERTIES PREFIX "")
if(MSVC)
    # we don't want .pdb files for the plugin because when we are within Visual Studio and debugging the application it locks
    # the .pdb for the original .dll and subsequent compiles fail even though we have loaded copies of the original .dll
    set_target_properties(plugin PROPERTIES LINK_FLAGS /DEBUG:NONE)
endif()