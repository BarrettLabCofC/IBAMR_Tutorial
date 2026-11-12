if(NOT IBAMR_HAVE_LIBMESH)
  message(FATAL_ERROR "Tutorial 14 requires IBAMR built with libMesh.")
endif()

include(CheckCXXSourceCompiles)
set(CMAKE_REQUIRED_LIBRARIES IBAMR::IBAMR2d)
check_cxx_source_compiles("#include <libmesh/libmesh_config.h>
#ifndef LIBMESH_HAVE_TRIANGLE
#error Triangle support is required
#endif
int main() { return 0; }" TUTORIAL_LIBMESH_HAS_TRIANGLE)
unset(CMAKE_REQUIRED_LIBRARIES)
if(NOT TUTORIAL_LIBMESH_HAS_TRIANGLE)
  message(FATAL_ERROR "Tutorial 14 requires libMesh built with Triangle support.")
endif()

add_executable(IIM-ex0-2d example.cpp)
target_link_libraries(IIM-ex0-2d PRIVATE IBAMR::IBAMR2d)
set_target_properties(IIM-ex0-2d PROPERTIES OUTPUT_NAME main2d)
