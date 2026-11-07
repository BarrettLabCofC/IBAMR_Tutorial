add_executable(IBFE-ex2-2d example.cpp)
target_link_libraries(IBFE-ex2-2d PRIVATE IBAMR::IBAMR2d)
set_target_properties(IBFE-ex2-2d PROPERTIES OUTPUT_NAME main2d)
