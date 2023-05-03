---
title: Linux里编译Openssl动态链接库
# description: 这是一个副标题
date: 2022-10-10
slug: openssl-build-so-in-linux
image: openssl.jpg
categories:
    - openssl
---

最近写Go需要用到SM系列国密算法来确保合规，图方便直接用了Openssl实现。Linux版本可以直接引用自动化环境自带的库，Win版本就只能单独编译一套Openssl运行库了。虽然在Win端可以使用MinGW编译，但可能会碰到缺少\_imp\_\_\*系列函数的问题，可能不支持128位数字，也不方便做Devops。这里记下在Linux环境交叉编译方法。

环境

*   Ubuntu 20.04
*   OpenSSL：官方1.1.1h

准备操作

如果是内外环境，比如开发机无法直连，先设置下出口代理

```shell
export http_proxy="...."
export https_proxy="...."
```

从OpenSSL官网获取最新的tar.gz包，解压，假定解压到/tmp/workDir/openssl

```shell
workDir="/opt/workDir/openssl"
mkdir -p "${workDir}"
wget -qO- https://www.openssl.org/source/latest.tar.gz | tar -xz --strip-components=1 -C "${workDir}"
```

设置输出目录，用来存放编译产物

```shell
distDir="${workDir}/dist"
mkdir -p "${distDir}"
```

安装交叉编译环境

```shell
sudo apt-get install mingw-w64
```

开始编译

首先需要修改下配置文件，让Openssl导出所有API并去除编译环境限制

```shell
sed -i 's/:.dll.a/ -Wl,--export-all -shared:.dll.a/g' "${workDir}/Configure"
sed -i 's,.\*target already defined.\*,$target=$\_;,g' "${workDir}/Configure"
```

配置编译选项：

```shell
cd "${workDir}"
configArgs="shared no-dso no-unit-test no-zlib no-ssl2 no-ssl3 -pthread -D\_REENTRANT enable-ec\_nistp\_64\_gcc\_128"

#32位：
./configure --cross-compile-prefix="i686-w64-mingw32-" mingw --prefix="${distDir}" ${configArgs}

#64位：
./config --cross-compile-prefix="x86\_64-w64-mingw32-" mingw64 --prefix="${distDir}" ${configArgs}
```

编译搞起！(12替换成编译线程数)

```shell
make build\_libs -j 12
make install\_dev
```

编译产物

编译完成后，/opt/workDir/openssl/dist目录应该是这样的结构

*   bin
    *   libcrypto-1\_1-x64.dll
    *   libssl-1\_1-x64.dll
*   include
    *   openssl
        *   (好多.c和.h文件)
*   lib
    *   pkgconfig
        *   libcrypto.pc
        *   libssl.pc
        *   openssl.pc
    *   libcrypto.a
    *   libcrypto.dll.a
    *   libssl.a
    *   libssl.dll.a

使用方法

编译Go代码的时候设置环境变量：

`PKG\_CONFIG\_PATH=/opt/workDir/openssl/dist/lib/pkgconfig`

Go代码中：

```go
// #cgo linux windows pkg-config: libssl libcrypto
// #cgo windows CFLAGS: -DWIN32\_LEAN\_AND\_MEAN
import "C"

```

分发时将/opt/workDir/openssl/dist/bin下的两个dll放到程序目录一同打包