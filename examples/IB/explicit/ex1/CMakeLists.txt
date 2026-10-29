add_executable(IB-ex1 example.cpp)
target_link_libraries(IB-ex1 PRIVATE IBAMR::IBAMR2d)
set_target_properties(IB-ex1 PROPERTIES OUTPUT_NAME main2d)
