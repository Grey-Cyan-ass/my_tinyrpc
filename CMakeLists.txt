cmake_minimum_required(VERSION 3.0)
project(tinyrpc)

# --- 1. 基础设置 ---
# 生成 compile_commands.json (解决 VS Code 报红线关键)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# 启用汇编支持 (为了 coctx_swap.S)
enable_language(ASM)

# 设置 C++ 标准和编译选项
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -O0 -Wall -Wno-deprecated -Wno-unused-but-set-variable")

# 设置输出路径
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)

# [关键] 将项目根目录加入头文件搜索路径 (解决所有头文件找不到的问题)
include_directories(${PROJECT_SOURCE_DIR})

# --- 2. 依赖库查找 ---

# (A) Protobuf
find_package(Protobuf REQUIRED)
if (${PROTOBUF_FOUND})
    message(STATUS "Protobuf found: ${Protobuf_LIBRARIES}")
    include_directories(${Protobuf_INCLUDE_DIRS})
else ()
    message(FATAL_ERROR "Cannot find Protobuf")
endif ()

# (B) TinyXML
# 查找库文件 (libtinyxml.a 或 .so)
find_library(TINYXML_LIBRARY tinyxml)
# 查找头文件 (tinyxml.h) 所在的路径
find_path(TINYXML_INCLUDE_DIR tinyxml.h)

if (NOT TINYXML_LIBRARY OR NOT TINYXML_INCLUDE_DIR)
    message(FATAL_ERROR "Cannot find TinyXML. Please install libtinyxml-dev")
else()
    message(STATUS "TinyXML found: ${TINYXML_LIBRARY}, Include: ${TINYXML_INCLUDE_DIR}")
    include_directories(${TINYXML_INCLUDE_DIR})
endif()

# --- 3. 源码文件收集 ---

# 自动扫描各个目录下的源文件
aux_source_directory(${PROJECT_SOURCE_DIR}/tinyrpc/comm COMM)
aux_source_directory(${PROJECT_SOURCE_DIR}/tinyrpc/coroutine COR)
aux_source_directory(${PROJECT_SOURCE_DIR}/tinyrpc/net NET)
aux_source_directory(${PROJECT_SOURCE_DIR}/tinyrpc/net/http HTTP)
aux_source_directory(${PROJECT_SOURCE_DIR}/tinyrpc/net/tcp TCP)
aux_source_directory(${PROJECT_SOURCE_DIR}/tinyrpc/net/tinypb TINYPB)

# 指定汇编文件
set(COCTX ${PROJECT_SOURCE_DIR}/tinyrpc/coroutine/coctx_swap.S)

# --- 4. 生成核心库 (libtinyrpc.a) ---

add_library(tinyrpc ${COMM} ${COR} ${NET} ${HTTP} ${TCP} ${TINYPB} ${COCTX})

# 链接依赖库 (dl 和 pthread 是 Linux 系统库)
target_link_libraries(tinyrpc 
    ${Protobuf_LIBRARIES} 
    ${TINYXML_LIBRARY} 
    dl 
    pthread
)

# --- 5. 生成测试可执行文件 ---

# 定义通用的链接库列表
set(COMMON_LIBS tinyrpc ${Protobuf_LIBRARIES} ${TINYXML_LIBRARY} dl pthread)

# Test 1: tinypb_server
add_executable(test_tinypb_server 
    ${PROJECT_SOURCE_DIR}/testcases/test_tinypb_server.cc
    ${PROJECT_SOURCE_DIR}/testcases/test_tinypb_server.pb.cc
)
target_link_libraries(test_tinypb_server ${COMMON_LIBS})

# Test 2: tinypb_server_client
add_executable(test_tinypb_server_client 
    ${PROJECT_SOURCE_DIR}/testcases/test_tinypb_server_client.cc
    ${PROJECT_SOURCE_DIR}/testcases/test_tinypb_server.pb.cc
)
target_link_libraries(test_tinypb_server_client ${COMMON_LIBS})

# Test 3: http_server
add_executable(test_http_server 
    ${PROJECT_SOURCE_DIR}/testcases/test_http_server.cc
    # 注意：这里引用了 tinypb 的 pb.cc，确认这是你需要的吗？如果不需要请删除
    ${PROJECT_SOURCE_DIR}/testcases/test_tinypb_server.pb.cc 
)
target_link_libraries(test_http_server ${COMMON_LIBS})

# Test 4: coroutine
add_executable(test_coroutine 
    ${PROJECT_SOURCE_DIR}/testcases/test_coroutine.cc
)
target_link_libraries(test_coroutine ${COMMON_LIBS})

# --- 6. 安装指令 (可选) ---
# 如果不需要 `make install`，这部分可以忽略
install(TARGETS tinyrpc DESTINATION lib)