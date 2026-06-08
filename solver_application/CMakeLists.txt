cmake_minimum_required(VERSION 3.10)
project(solver)

add_executable(solver ${CMAKE_CURRENT_SOURCE_DIR}/equation.cpp)

target_link_libraries(solver formatter_ex solver_lib)

