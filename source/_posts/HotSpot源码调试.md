---
title: HotSpot源码调试
date: 2020-11-18 09:36:22
tags:
  - Java
  - 源码阅读
categories:
  - 源码阅读
  - 技术
---

我们平时在工作中，处处离不开 JAVA，那么 java 到底是怎么玩的呢，这里从 Java 源码层面进行一个走读。本文主要介绍 java 源码编译，使用的 java 源码是 openjdk 的当前 master 分支，bootjdk 使用的是 jdk15.0.1，在 Ubuntu16.04 LTS

<!--more-->

# 编译

### 下载代码及相关依赖

现在 openjdk 源码已经托管到 GitHub 上了，在 GitHub 上下载 master 分支代码
`git clone https://github.com/openjdk/jdk.git`
编译 openjdk 需要一个 boot-jdk，执行源码的 configure 脚本，里边会有提示，openjdk16 推荐的 boot-jdk 版本为 14 和 15，这里我用的是 jdk15，在 openjdk 官网上下载 jdk15 的安装包。`https://download.java.net/java/GA/jdk15.0.1/51f4f36ad4ef43e39d0dfdbaf6549e32/9/GPL/openjdk-15.0.1_linux-x64_bin.tar.gz`

### 编译

关于编译，GitHub 上的官方文档也有一些，但是比较粗略，官方文档`https://github.com/openjdk/jdk/blob/master/doc/building.md`。这里我就直接贴出我的编译命令。

```
./configure --with-target-bits=64 --disable-warnings-as-errors --with-boot-jdk=/home/fuyanzhang/xiaomi/study/java/jdk-15.0.1
make images
```

当在执行`configure`命令时，如果有依赖包没有，会有一些提示。由于我前面编译过 jdk14 的源码，所有的相关依赖包已经装过了，所以这次编译 jdk16 时比较顺利。如果有缺包报错，缺啥包就装啥包就 OK 了。
接下来就是等待。当出现如下代码时，说明已经完成编译了。

```
Stopping sjavac server
Finished building target 'images' in configuration 'linux-x86_64-server-release'
```

### 验证

进入编译的包中
cd build/linux-x86_64-server-release/jdk/bin
执行如下命令：
./java -version
出现下面的结果，说明编译没问题。

```
openjdk version "16-internal" 2021-03-16
OpenJDK Runtime Environment (build 16-internal+0-adhoc.fuyanzhang.jdk)
OpenJDK 64-Bit Server VM (build 16-internal+0-adhoc.fuyanzhang.jdk, mixed mode)
```

也可以按照官网的验证方法，跑基础的测试用例。
`make run-test-tier1`

至此，openjdk 源码编译完成了。

# 调试源码

能编译 openjdk 源码不是我们的最终目的，我们最终的目的是通过 jdk 源码的阅读，更深入的理解 java 知识，这就需要我们搭建一套可以 debug 的 java 源码环境。这里我用 vscode 作为 ide 工具进行搭建。
软件准备这里就不说了。直接进入主题。
1、导入 jdk 源码。
2、点击 vscode 左侧的 run 按钮，进入 debug 设置，开始配置 debug 参数，即 launch.json。我的设置如下：

```
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "jdk16 debug",
            "type": "cppdbg",
            "request": "launch",
            "program": "/home/fuyanzhang/xiaomi/study/java/jdk/build/linux-x86_64-server-release/jdk/bin/java",
            "args": ["Test"],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [
                {"name":"JAVA_HOME","value":"/home/fuyanzhang/xiaomi/study/java/jdk/build/linux-x86_64-server-release/jdk/"},
                {"name":"CLASSPATH","value":".:/home/fuyanzhang/xiaomi/study/java/jdk/test/mytest"}
            ],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        }
    ]
}
```

简单介绍一下上面的参数意思：
program：配置 java 的入口，即我们平时使用的 java 命令。
args： java 命令的参数，即我们自己写的业务代码入口类，也就是 main 方法所在的类。这里我的类名是 Test,主要实现的是打印一个`hello world`
environment:设置 JAVA_HOME 及 args 中类所在的位置。
其他的参数使用默认值即可。
如果保存之后，没有出现 debug 窗口，则需要 reload 一下 C/C++插件。
正常情况下会出现如图的窗口：
![debug窗口](/images/debug_openjdk16.png)
点击`jdk16 debug`边上的三角，就开始 debug 了。
随便在 java 源码中打几个端点，执行 debug，就能看到如图的 debug 信息了。
![debug窗口](/images/debug_openjdk_ing.png)
到此，源码阅读环境搭建就 OK 了，可徜徉在 java 源码的海洋里了。have fun ！！！
