cmake_minimum_required(VERSION 3.10)
project(formatter)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if (NOT TARGET formatter)
	add_library(formatter STATIC ${CMAKE_CURRENT_SOURCE_DIR}/formatter.cpp)
endif()

target_include_directories(formatter PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
