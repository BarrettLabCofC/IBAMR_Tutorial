add_executable(navier_stokes-ex1-2d example.cpp)
target_link_libraries(navier_stokes-ex1-2d PRIVATE IBAMR::IBAMR2d)
set_target_properties(navier_stokes-ex1-2d PROPERTIES OUTPUT_NAME main2d)
