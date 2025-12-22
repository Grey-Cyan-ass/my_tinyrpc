cmake_minimum_required(VERSION 3.0)
project(tinyrpc)

# 设置安装路径为源码根目录 (方便调试)
SET(CMAKE_INSTALL_PREFIX ${PROJECT_SOURCE_DIR})

# 启用汇编语言支持 (为了 coctx_swap.S)
enable_language(ASM)

# 编译选项
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -O0 -std=c++11 -Wall -Wno-deprecated -Wno-unused-but-set-variable")

# 输出目录定义
set(PATH_LIB lib)
set(PATH_BIN bin)
set(PATH_TESTCASES testcases)

# [关键 1] 将项目根目录加入头文件搜索路径
# 解决了 "tinyrpc/comm/config.h file not found" 的问题
include_directories(${PROJECT_SOURCE_DIR})

# 自动查找源码文件
aux_source_directory(${PROJECT_SOURCE_DIR}/tinyrpc/comm COMM)
aux_source_directory(${PROJECT_SOURCE_DIR}/tinyrpc/coroutine COR)
aux_source_directory(${PROJECT_SOURCE_DIR}/tinyrpc/net NET)
aux_source_directory(${PROJECT_SOURCE_DIR}/tinyrpc/net/http HTTP)
aux_source_directory(${PROJECT_SOURCE_DIR}/tinyrpc/net/tcp TCP)
aux_source_directory(${PROJECT_SOURCE_DIR}/tinyrpc/net/tinypb TINYPB)

# 指定汇编文件
set(COXTX ${PROJECT_SOURCE_DIR}/tinyrpc/coroutine/coctx_swap.S)

# 生成核心库 libtinyrpc.a (或 .so)
add_library(tinyrpc ${COMM} ${COR} ${NET} ${HTTP} ${TCP} ${TINYPB} ${COXTX})
install(TARGETS tinyrpc DESTINATION ${PATH_LIB})

# --- 依赖库配置 (修复重点) ---

# 1. Protobuf 配置
find_package(Protobuf REQUIRED)
if (${PROTOBUF_FOUND})
    message(STATUS "Protobuf found: ${Protobuf_LIBRARIES}")
    # [关键 2] 必须包含 Protobuf 的头文件路径
    include_directories(${Protobuf_INCLUDE_DIRS})
else ()
    message(FATAL_ERROR "Cannot find Protobuf")
endif ()

# 2. TinyXML 配置 (修复重点)
# [关键 3] 不要硬编码 /usr/lib/libtinyxml.a，容器里可能叫 libtinyxml.so 或者是路径不同
# 直接查找库文件，让 CMake 去寻找正确的位置
find_library(TINYXML_LIBRARY tinyxml)
if (NOT TINYXML_LIBRARY)
    message(FATAL_ERROR "Cannot find TinyXML library. Please install libtinyxml-dev")
endif()

# 统一链接库列表
# 注意：使用 ${Protobuf_LIBRARIES} (复数) 而不是 ${Protobuf_LIBRARY}
set(LIBS
    tinyrpc
    ${Protobuf_LIBRARIES}
    ${TINYXML_LIBRARY}
    dl
    pthread
)

# --- 测试用例构建 ---

# test_tinypb_server
set(test_tinypb_server_src
    ${PROJECT_SOURCE_DIR}/${PATH_TESTCASES}/test_tinypb_server.cc
    ${PROJECT_SOURCE_DIR}/${PATH_TESTCASES}/test_tinypb_server.pb.cc
)
add_executable(test_tinypb_server ${test_tinypb_server_src})
target_link_libraries(test_tinypb_server ${LIBS})
install(TARGETS test_tinypb_server DESTINATION ${PATH_BIN})

# test_tinypb_server_client
set(test_tinypb_server_client_src
    ${PROJECT_SOURCE_DIR}/${PATH_TESTCASES}/test_tinypb_server_client.cc
    ${PROJECT_SOURCE_DIR}/${PATH_TESTCASES}/test_tinypb_server.pb.cc
)
add_executable(test_tinypb_server_client ${test_tinypb_server_client_src})
target_link_libraries(test_tinypb_server_client ${LIBS})
install(TARGETS test_tinypb_server_client DESTINATION ${PATH_BIN})

# test_http_server
set(test_http_server_src
    ${PROJECT_SOURCE_DIR}/${PATH_TESTCASES}/test_http_server.cc
    ${PROJECT_SOURCE_DIR}/${PATH_TESTCASES}/test_tinypb_server.pb.cc
)
add_executable(test_http_server ${test_http_server_src})
target_link_libraries(test_http_server ${LIBS})
install(TARGETS test_http_server DESTINATION ${PATH_BIN})

# test_coroutine
set(test_coroutine_src
    ${PROJECT_SOURCE_DIR}/${PATH_TESTCASES}/test_coroutine.cc
)
add_executable(test_coroutine ${test_coroutine_src})
target_link_libraries(test_coroutine ${LIBS})
install(TARGETS test_coroutine DESTINATION ${PATH_BIN})

# 安装头文件 (这些目录下的 CMakeLists.txt 需要正确配置 install 指令)
add_subdirectory(tinyrpc/comm)
add_subdirectory(tinyrpc/coroutine)
add_subdirectory(tinyrpc/net)
add_subdirectory(tinyrpc/net/http)
add_subdirectory(tinyrpc/net/tcp)
add_subdirectory(tinyrpc/net/tinypb)