---
title: Springboot内嵌tomcat代码走读（二）
date: 2020-09-24 19:13:34
tags: 
- Java
- 源码阅读
- Spring
categories:
- 源码阅读
- Spring
---
离上次tomcat的代码走读过了很长时间了，这段时间主要是在自己造个类似web容器的轮子，用来加深NIO的理解。同时大致走读了下undertow相关的代码。今天继续tomcat的代码走读。主要内容是tomcat如何从接收请求到处理请求并回执。
<!--more-->

首先看下tomcat处理请求的线程模型。
tomcat处理请求主要有三种线程，这里就用线程名里的关键字来标识。
* Acceptor线程 主要是接收请求 。也是一个线程。
* ClientPoller线程主要是向selector中注册socket，并分发到处理线程中。这是一个线程。
* exec线程 主要负责处理真正的请求。调用servlet的service方法。这是一个线程池。
下面从源码层面看是如何处理一个请求的。

