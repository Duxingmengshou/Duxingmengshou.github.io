---
title: 'OpenSSL、libevent编译（MSVC）'
date: 2024-8-10
permalink: /posts/OpenSSL、libevent编译（MSVC）/
tags:
  - C++
---
# OpenSSL、libevent编译（MSVC）

## OpenSSL

### Strawberry Perl安装

https://strawberryperl.com/

```powershell
perl -v

#结果
Locale 'Chinese (Simplified)_China.936' is unsupported, and may crash the interpreter.

This is perl 5, version 38, subversion 2 (v5.38.2) built for MSWin32-x64-multi-thread

Copyright 1987-2023, Larry Wall

Perl may be copied only under the terms of either the Artistic License or the
GNU General Public License, which may be found in the Perl 5 source kit.

Complete documentation for Perl, including FAQ lists, should be found on
this system using "man perl" or "perldoc perl".  If you have access to the
Internet, point your browser at https://www.perl.org/, the Perl Home Page.
```

### NSAM安装

https://www.nasm.us/

```powershell
nasm -v

#结果
NASM version 2.16.01 compiled on Jun  1 2023
```

### OpenSSL源码

https://openssl-library.org/

### MSVC工具解释

x86 Native Tools Command Prompt - Sets the environment to use 32-bit, x86-native tools to build 32-bit, x86-native code.
x64 Native Tools Command Prompt - Sets the environment to use 64-bit, x64-native tools to build 64-bit, x64-native code.
x86_x64 Cross Tools Command Prompt - Sets the environment to use 32-bit, x86-native tools to build 64-bit, x64-native code.
x64_x86 Cross Tools Command Prompt - Sets the environment to use 64-bit, x64-native tools to build 32-bit, x86-native code.

翻译成人话就是

x86 用32位的编译器编译出32位程序
x64 用64位的编译器编译出64位程序
x86_x64 用32位的编译器编译出64位程序
x64_x86 用64位的编译器编译出32位程序

### OpenSSL编译

生成配置

```powershell
perl Configure VC-WIN64A --prefix=H:\Programming\C++\OpenSSL\openssl
```

执行编译

```powershell
nmake
```

编译安装

```powershell
nmake install #安装，包含文档（HTML）
nmake install_sw #安装，不带文档
```

检查

```powershell
openssl version -a

#结果
OpenSSL 3.3.1 4 Jun 2024 (Library: OpenSSL 3.3.1 4 Jun 2024)
built on: Fri Aug  9 15:46:37 2024 UTC
platform: VC-WIN64A
options:  bn(64,64)
compiler: cl  /Zi /Fdossl_static.pdb /Gs0 /GF /Gy /MD /W3 /wd4090 /nologo /O2 -DL_ENDIAN -DOPENSSL_PIC -D"OPENSSL_BUILDING_OPENSSL" -D"OPENSSL_SYS_WIN32" -D"WIN32_LEAN_AND_MEAN" -D"UNICODE" -D"_UNICODE" -D"_CRT_SECURE_NO_DEPRECATE" -D"_WINSOCK_DEPRECATED_NO_WARNINGS" -D"NDEBUG"
OPENSSLDIR: "C:\Program Files\Common Files\SSL"
ENGINESDIR: "H:\Programming\C++\OpenSSL\openssl\lib\engines-3"
MODULESDIR: "H:\Programming\C++\OpenSSL\openssl\lib\ossl-modules"
Seeding source: os-specific
CPUINFO: OPENSSL_ia32cap=0xfedaf383ffcbffff:0x27ab
```

## libevent

### libevent源码

https://libevent.org/

### libevent编译

```powershell
nmake /f Makefile.nmake clean
nmake /f Makefile.nmake OPENSSL_DIR=H:\Programming\C++\OpenSSL\openssl
```

### 未声明的标识符错误

`error C2065: “UINT32_MAX”: 未声明的标识符`

源码根目录`minheap-internal.h`第38行加入`#include "stdint.h"`

### 无法打开输入文件libeay32.lib错误

源码根目录打开`\test\Makefile.nmake`文件，第6行将`ssleay32.lib`和`libeay32.lib`改为`libcrypto.lib`、`libssl.lib`

### 无法解析的外部符号__imp__if_nametoindex

源码根目录打开`\test\Makefile.nmake`文件，第39行添加`Iphlpapi.lib`库依赖

### 后续使用中出现无法解释的外部符号

源码编译依赖了`Iphlpapi.lib`，后续使用`libevent.lib`也需要引入这个依赖
自行将include和lib结尾文件复制安装，include/event2中还需包含`event-config.h`文件，同样是搜索复制到对应目录