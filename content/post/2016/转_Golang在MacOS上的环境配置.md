
---
date: 2016-12-31T11:34:11+08:00
title: "Golang在MacOS上的环境配置"
description: ""
disqus_identifier: 1485833651969259294
slug: "Golangzai-Mac-OSshang-de-huan-jing-pei-zhi"
source: "https://segmentfault.com/a/1190000003933557"
tags: 
- homebrew 
- macos 
- golang 
categories:
- 编程语言与开发
---

`Golang`是Google开发的一种编译型，平行化，具有垃圾回收功能的编译语言,还和C一样有指针。\
`Golang`是天生的网络编程语言，学习使用`Golang`先从环境配置开始。\
环境要求:

-   `Homerbrew` 安装参考:<http://brew.sh/>

安装Golang
----------

`Golang`可以通过源代码自己编译安装<https://golang.org/project/>,为了管理和升级建议使用`Homerbrew`安装。\
使用命令行直接安装`Golang`:

> brew install go

    ==> Downloading https://homebrew.bintray.com/bottles/go-1.7.el_capitan.bottle.tar.gz
    ######################################################################## 100.0%
    ==> Pouring go-1.7.el_capitan.bottle.tar.gz
    ==> Caveats
    As of go 1.2, a valid GOPATH is required to use the `go get` command:
      https://golang.org/doc/code.html#GOPATH
    You may wish to add the GOROOT-based install location to your PATH:
      export PATH=$PATH:/usr/local/opt/go/libexec/bin
    ==> Summary
    🍺  /usr/local/Cellar/go/1.7: 6,435 files, 250.6M

从提示中可以看出需要设置`GOPATH`和`GOROOT`的环境变量，以及设置`PATH`.

配置GOPATH
----------

查看go 的环境变量设置的命令

> go env

        GOARCH="amd64"
        GOBIN=""
        GOEXE=""
        GOHOSTARCH="amd64"
        GOHOSTOS="darwin"
        GOOS="darwin"
        GOPATH=""
        GORACE=""
        GOROOT="/usr/local/Cellar/go/1.7.6/libexec"
        GOTOOLDIR="/usr/local/Cellar/go/1.7.6/libexec/pkg/tool/darwin_amd64"
        GO15VENDOREXPERIMENT=""
        CC="clang"
        GOGCCFLAGS="-fPIC -m64 -pthread -fno-caret-diagnostics -Qunused-arguments -fmessage-length=0 -fno-common"
        CXX="clang++"
        CGO_ENABLED="1"

需要设置的环境变量包括:`GOPATH` ,`GOBIN`
以及把GOBIN加入到`PATH`中,`GOROOT`变量默认已经设置好。

在`fishshell`设置GOPATH：

> `set -gx GOPATH /usr/local/Cellar/go/1.7.6`

在`bash`中设置：

> vim .bash\_profile

        export GOPATH=/usr/local/Cellar/go/1.7.6
        export GOBIN=$GOPATH/bin
        export PATH=$PATH:$GOBIN

使修改立刻生效:

> source .bash\_profile

再看下go的环境变量:

> go env

       GOARCH="amd64"
       GOBIN="/usr/local/Cellar/go/1.7.6/bin"
       GOEXE=""
       GOHOSTARCH="amd64"
       GOHOSTOS="darwin"
       GOOS="darwin"
       GOPATH="/usr/local/Cellar/go/1.7.6"
       GORACE=""
       GOROOT="/usr/local/Cellar/go/1.7.6/libexec"
       GOTOOLDIR="/usr/local/Cellar/go/1.7.6/libexec/pkg/tool/darwin_amd64"
       GO15VENDOREXPERIMENT=""
       CC="clang"
       GOGCCFLAGS="-fPIC -m64 -pthread -fno-caret-diagnostics -Qunused-arguments -fmessage-length=0 -fno-common"
       CXX="clang++"
       CGO_ENABLED="1"

环境变量设置完成!

