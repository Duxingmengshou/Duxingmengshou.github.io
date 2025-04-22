---
title: 'hiredis编译（MSVC）'
date: 2024-8-10
permalink: /posts/hiredis编译（MSVC）/
tags:
  - C++
---
# hiredis编译（MSVC）

### hiredis源码

https://github.com/redis/hiredis

### cmake配置

这里我根据需要勾选了

ENABLE_ASYNC_TESTS

ENABLE_EXAMPLES

ENABLE_SSL

ENABLE_SSL_TESTS

修改了

CMAKE_INSTALL_PREFIX = H:\Programming\C++\hiredis\hiredis

同时注意

SSL相关库的位置是否正确

### hiredis编译

cmake没有能扫描到libevent库的位置，VS2022编译会出现问题（无法打开文件envent.lib）

将对应出错模块的包含目录中加入H:\Programming\C++\libevent\libevent\include

库目录中加入H:\Programming\C++\libevent\libevent\lib

链接器的输入-附加依赖项中改event.lib为libevent.lib，同时加入Iphlpapi.lib（编译libevent过程中报错引入的，不加入报错外部符号\__imp__if_nametoindex）

如果报错找不到event-config.h，那么就要去libevent源码编译目录中找到这个文件，复制到对应位置

### hiredis测试

CMakeLists

```cmake
set(HIREDIS_ROOT "H:/Programming/C++/hiredis/hiredis")
include_directories(${HIREDIS_ROOT}/include)
link_directories(${HIREDIS_ROOT}/lib)
set(HIREDIS_LIBRARIES
        hiredis_ssl.lib
        hiredis.lib
)

add_executable(${PROJECT_NAME}
        main.cpp
)

target_link_libraries(${PROJECT_NAME} ${HIREDIS_LIBRARIES})
```

C++代码

```CPP
#include <cstdio>
#include <cstdlib>
#include <hiredis/hiredis.h>
#include <WinSock2.h>

int main(int argc, char **argv) {
    redisContext *conn;
    redisReply *reply;
    if (argc < 2) {
        printf("Usage: example {instance_ip_address} 6379\n");
        exit(0);
    }
    const char *hostname = argv[1];
    const int port = atoi(argv[2]);
    struct timeval timeout = {1, 500000}; // 1.5 seconds
    conn = redisConnectWithTimeout(hostname, port, timeout);
    if (conn == nullptr || conn->err) {
        if (conn) {
            printf("Connection error: %s\n", conn->errstr);
            redisFree(conn);
        } else {
            printf("Connection error: can't allocate redis context\n");
        }
        exit(1);
    }
    /* AUTH */
    reply = (redisReply *) redisCommand(conn, "AUTH %s");//此处以及后面都要进行强制类型转换
    printf("AUTH: %s\n", reply->str);
    freeReplyObject(reply);

    /* Set */
    reply = (redisReply *) redisCommand(conn, "SET %s %s", "welcome", "Hello, DCS for Redis!");
    printf("SET: %s\n", reply->str);
    freeReplyObject(reply);

    /* Get */
    reply = (redisReply *) redisCommand(conn, "GET welcome");
    printf("GET welcome: %s\n", reply->str);
    freeReplyObject(reply);

    /* Disconnects and frees the context */
    redisFree(conn);
    return 0;
}

/*
结果：
.\CMake_check.exe 192.168.11.216 6379
AUTH: WRONGPASS invalid username-password pair
SET: NOAUTH Authentication required.        
GET welcome: NOAUTH Authentication required.
*/
```