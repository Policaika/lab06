cmake_minimum_required(VERSION 3.10)
project(formatter_ex)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if (NOT TARGET formatter_ex)
	add_library(formatter_ex STATIC ${CMAKE_CURRENT_SOURCE_DIR}/formatter_ex.cpp)
	target_link_libraries(formatter_ex formatter)
endif()

target_include_directories(formatter_ex PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
