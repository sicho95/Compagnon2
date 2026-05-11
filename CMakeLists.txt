cmake_minimum_required(VERSION 3.16)

# Custom partition table
set(PARTITION_TABLE_CSV "${CMAKE_CURRENT_SOURCE_DIR}/partitions/compagnon2.csv")

include($ENV{IDF_PATH}/tools/cmake/project.cmake)

project(compagnon2)
