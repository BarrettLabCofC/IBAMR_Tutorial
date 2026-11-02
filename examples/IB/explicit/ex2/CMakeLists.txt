add_executable(IB-ex2-2d example.cpp)
target_link_libraries(IB-ex2-2d PRIVATE IBAMR::IBAMR2d)
set_target_properties(IB-ex2-2d PROPERTIES OUTPUT_NAME main2d)
