cmake_minimum_required(VERSION 3.20)

project(TestVcpkgCache)

find_package(fmt CONFIG REQUIRED)

add_executable(main main.cpp)
target_link_libraries(main PRIVATE fmt::fmt)
