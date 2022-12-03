cmake_minimum_required(VERSION 3.16)

PROJECT(QuantFabric)

# 输出so至build目录
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/build/)
# 输出APP至build目录
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/build/)
# 输出静态库至build目录
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/build/)

add_subdirectory(XWatcher)
add_subdirectory(XTrader)
add_subdirectory(XServer)
add_subdirectory(XRiskJudge)
add_subdirectory(XMarketCenter)
add_subdirectory(XDataPlayer)
add_subdirectory(Tools)
