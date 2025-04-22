# gRPC CMakeLists配置

### 单个protobuf文件

首先需要按照gRPC的示例程序中的CMakeLists去查找当前系统的库位置。

```cmake
# 查找 gRPC 和 Protobuf
include("C:/Program Files (x86)/grpc/cmake/common.cmake")
```

这个`common.cmake`原本是在源码目录的`/examples/cpp/cmake`下面，我为了方便就移动到了编译安装gRPC的目录位置。

对于单个protobuf文件，是按照示例程序的中的更改得到的。对于项目名称以及其他基本参数，有如下定义。

```cmake
cmake_minimum_required(VERSION 3.26)
set(ProjectName 项目名称)
project(${ProjectName})

set(CMAKE_CXX_STANDARD 17)
```

```cmake
# 查找 gRPC 和 Protobuf
include("C:/Program Files (x86)/grpc/cmake/common.cmake")
# 添加protobuf源文件
get_filename_component(math_proto "./protos/math.proto" ABSOLUTE)
get_filename_component(math_proto_path "${math_proto}" PATH)

message(${_GRPC_CPP_PLUGIN_EXECUTABLE})

set(math_proto_srcs "${CMAKE_CURRENT_BINARY_DIR}/math.pb.cc")
set(math_proto_hdrs "${CMAKE_CURRENT_BINARY_DIR}/math.pb.h")
set(math_grpc_srcs "${CMAKE_CURRENT_BINARY_DIR}/math.grpc.pb.cc")
set(math_grpc_hdrs "${CMAKE_CURRENT_BINARY_DIR}/math.grpc.pb.h")
add_custom_command(
        OUTPUT "${math_proto_srcs}" "${math_proto_hdrs}" "${math_grpc_srcs}" "${math_grpc_hdrs}"
        COMMAND ${_PROTOBUF_PROTOC}
        ARGS --grpc_out "${CMAKE_CURRENT_BINARY_DIR}"
        --cpp_out "${CMAKE_CURRENT_BINARY_DIR}"
        -I "${math_proto_path}"
        --plugin=protoc-gen-grpc="${_GRPC_CPP_PLUGIN_EXECUTABLE}"
        "${math_proto}"
        DEPENDS "${math_proto}")

include_directories("${CMAKE_CURRENT_BINARY_DIR}")
add_library(math_grpc_proto
        ${math_grpc_srcs}
        ${math_grpc_hdrs}
        ${math_proto_srcs}
        ${math_proto_hdrs})
target_link_libraries(math_grpc_proto
        ${_REFLECTION}
        ${_GRPC_GRPCPP}
        ${_PROTOBUF_LIBPROTOBUF})

add_executable(${ProjectName} main.cpp)

target_link_libraries(${ProjectName}
        math_grpc_proto
        absl::flags
        absl::flags_parse
        ${_REFLECTION}
        ${_GRPC_GRPCPP}
        ${_PROTOBUF_LIBPROTOBUF})
```

### 多个protobuf文件，自动化生成

```cmake
# 查找 gRPC 和 Protobuf
include("C:/Program Files (x86)/grpc/cmake/common.cmake")
# 定义生成目录
set(PROTO_DIR
        ./protos/src
)
# 定义protobuf文件
set(PROTO_FILES
        ./protos/math.proto
        ./protos/sub.proto
)
# 定义当前项目使用的ProtoBuf链接库名称，随便定义即可
set(PROTO_LIB
        grpc_proto
)
# 定义文件生成目录
get_filename_component(PROTO_ABS_DIR "${PROTO_DIR}" ABSOLUTE)
# 生成源文件和头文件的列表
set(PROTO_SRC_FILES "")
set(PROTO_HDR_FILES "")
set(GRPC_SRC_FILES "")
set(GRPC_HDR_FILES "")
# 遍历每个 Protobuf 文件
foreach(PROTO_FILE ${PROTO_FILES})
    get_filename_component(PROTO_ABS_PATH "${PROTO_FILE}" ABSOLUTE)
    get_filename_component(PROTO_DIR "${PROTO_ABS_PATH}" PATH)
    # 生成源文件和头文件的名称
    get_filename_component(PROTO_NAME "${PROTO_FILE}" NAME_WE)
    set(PROTO_SRC "${PROTO_ABS_DIR}/${PROTO_NAME}.pb.cc")
    set(PROTO_HDR "${PROTO_ABS_DIR}/${PROTO_NAME}.pb.h")
    set(GRPC_SRC "${PROTO_ABS_DIR}/${PROTO_NAME}.grpc.pb.cc")
    set(GRPC_HDR "${PROTO_ABS_DIR}/${PROTO_NAME}.grpc.pb.h")
    # 添加到源文件列表
    list(APPEND PROTO_SRC_FILES "${PROTO_SRC}")
    list(APPEND PROTO_HDR_FILES "${PROTO_HDR}")
    list(APPEND GRPC_SRC_FILES "${GRPC_SRC}")
    list(APPEND GRPC_HDR_FILES "${GRPC_HDR}")
    # 添加自定义命令
    add_custom_command(
            OUTPUT "${PROTO_SRC}" "${PROTO_HDR}" "${GRPC_SRC}" "${GRPC_HDR}"
            COMMAND ${_PROTOBUF_PROTOC}
            ARGS --grpc_out "${PROTO_ABS_DIR}"
            --cpp_out "${PROTO_ABS_DIR}"
            -I "${PROTO_DIR}"
            --plugin=protoc-gen-grpc="${_GRPC_CPP_PLUGIN_EXECUTABLE}"
            "${PROTO_ABS_PATH}"
            DEPENDS "${PROTO_ABS_PATH}"
    )
endforeach()
# 创建库
add_library(${PROTO_LIB}
        ${PROTO_SRC_FILES}
        ${PROTO_HDR_FILES}
        ${GRPC_SRC_FILES}
        ${GRPC_HDR_FILES}
)
target_link_libraries(${PROTO_LIB}
        ${_REFLECTION}
        ${_GRPC_GRPCPP}
        ${_PROTOBUF_LIBPROTOBUF})
        
        
# 创建可执行文件####################################################################
add_executable(${PROJECT_NAME} main.cpp)



# 链接库
target_link_libraries(${PROJECT_NAME}
        ${PROTO_LIB}
        absl::flags
        absl::flags_parse
        ${_REFLECTION}
        ${_GRPC_GRPCPP}
        ${_PROTOBUF_LIBPROTOBUF}
)
```

最后只需定义protos文件目录以及设置生成头文件目录，就能每次在编译的时候自动化的完成这一系列操作。