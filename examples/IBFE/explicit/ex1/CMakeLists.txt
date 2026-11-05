if(NOT IBAMR_HAVE_LIBMESH)
  message(FATAL_ERROR "The finite-element tutorials require IBAMR built with libMesh.")
endif()
include(CheckCXXSourceCompiles)
set(CMAKE_REQUIRED_LIBRARIES IBAMR::IBAMR2d)
check_cxx_source_compiles("#include <libmesh/libmesh_config.h>
#ifndef LIBMESH_HAVE_EXODUS_API
#error ExodusII support is required
#endif
int main() { return 0; }" TUTORIAL_LIBMESH_HAS_EXODUS)
unset(CMAKE_REQUIRED_LIBRARIES)
if(NOT TUTORIAL_LIBMESH_HAS_EXODUS)
  message(FATAL_ERROR "The finite-element tutorials require libMesh built with ExodusII output support.")
endif()

add_executable(IBFE-ex1 example.cpp)
target_link_libraries(IBFE-ex1 PRIVATE IBAMR::IBAMR2d)
set_target_properties(IBFE-ex1 PROPERTIES OUTPUT_NAME main2d)
