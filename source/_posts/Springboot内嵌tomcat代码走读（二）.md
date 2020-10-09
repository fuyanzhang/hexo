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

首先看上面说的三个线程的启动。
```
 // Create worker collection
            if (getExecutor() == null) {
                //工作线程
                createExecutor();
            }

            initializeConnectionLatch();

            // Start poller thread
            //分发线程
            poller = new Poller();
            Thread pollerThread = new Thread(poller, getName() + "-ClientPoller");
            pollerThread.setPriority(threadPriority);
            pollerThread.setDaemon(true);
            pollerThread.start();

            //接收请求线程
            startAcceptorThread();
```
工作线程的源码如下：
```
    public void createExecutor() {
        internalExecutor = true;
        TaskQueue taskqueue = new TaskQueue();
        TaskThreadFactory tf = new TaskThreadFactory(getName() + "-exec-", daemon, getThreadPriority());
        executor = new ThreadPoolExecutor(getMinSpareThreads(), getMaxThreads(), 60, TimeUnit.SECONDS,taskqueue, tf);
        taskqueue.setParent( (ThreadPoolExecutor) executor);
    }
```
这段代码比较简单，主要是创建一个工作的线程池，线程池的初始线程数默认是10个，最大线程数默认是200。
下面看Acceptor的处理逻辑。
先上时序图：
![tomcat acceptor时序图](/images/tomcat_acceptor.png)
> 1、Acceptor在启动后，在run方法里调用accept方法，该方法会阻塞直到有连接到来。
> 2、在获取到channel后，将该channel封装成NioChannel，并注册到Poller中去。注册的过程就是简单的把NioChannel封装成的PollerEvent add到事件队列中。
> 至此，Acceptor完成了自己的使命了，接下来就是循环等待下一个请求的到来。
相关的源码如下，有些跟主体流程无关的代码就直接隐去了。
```
 @Override
    public void run() {

        // Loop until we receive a shutdown command
        while (endpoint.isRunning()) {
            try {
                //if we have reached max connections, wait
                endpoint.countUpOrAwaitConnection();

                // Endpoint might have been paused while waiting for latch
                // If that is the case, don't accept new connections
                if (endpoint.isPaused()) {
                    continue;
                }

                U socket = null;
                try {
                    //获取连接，线程会阻塞在这个地方，直到有新的连接请求过来。
                    socket = endpoint.serverSocketAccept();
                } catch (Exception ioe) {
                
                }
                // Successful accept, reset the error delay
                errorDelay = 0;

                // Configure the socket
                if (endpoint.isRunning() && !endpoint.isPaused()) {
                    // setSocketOptions里代码比较长，主要是将获取的channel封装成时间，注册到poller的事件队列里供Poller线程分发。
                    if (!endpoint.setSocketOptions(socket)) {
                        endpoint.closeSocket(socket);
                    }
                } else {
                    endpoint.destroySocket(socket);
                }
            } catch (Throwable t) {
             。。。
            }
            。。。
    }
```
下面来看下`setSocketOptions()`的实现
```
  /**
     * Process the specified connection.
     * @param socket The socket channel
     * @return <code>true</code> if the socket was correctly configured
     *  and processing may continue, <code>false</code> if the socket needs to be
     *  close immediately
     */
    @Override
    protected boolean setSocketOptions(SocketChannel socket) {
        NioSocketWrapper socketWrapper = null;
        try {
            // Allocate channel and wrapper
            NioChannel channel = null;
            if (nioChannels != null) {
                channel = nioChannels.pop();
            }
            if (channel == null) {
                SocketBufferHandler bufhandler = new SocketBufferHandler(
                        socketProperties.getAppReadBufSize(),
                        socketProperties.getAppWriteBufSize(),
                        socketProperties.getDirectBuffer());
                if (isSSLEnabled()) {
                    channel = new SecureNioChannel(bufhandler, selectorPool, this);
                } else {
                    channel = new NioChannel(bufhandler);
                }
            }
            NioSocketWrapper newWrapper = new NioSocketWrapper(channel, this);
            channel.reset(socket, newWrapper);
            connections.put(socket, newWrapper);
            socketWrapper = newWrapper;

            // Set socket properties
            // Disable blocking, polling will be used
            socket.configureBlocking(false);
            socketProperties.setProperties(socket.socket());

            socketWrapper.setReadTimeout(getConnectionTimeout());
            socketWrapper.setWriteTimeout(getConnectionTimeout());
            socketWrapper.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
            socketWrapper.setSecure(isSSLEnabled());
            poller.register(channel, socketWrapper);
            return true;
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            try {
                log.error(sm.getString("endpoint.socketOptionsError"), t);
            } catch (Throwable tt) {
                ExceptionUtils.handleThrowable(tt);
            }
            if (socketWrapper == null) {
                destroySocket(socket);
            }
        }
        // Tell to close the socket if needed
        return false;
    }
```
代码很简单，就不做细致的分析与注释了。
`register`方法代码:
```
      public void register(final NioChannel socket, final NioSocketWrapper socketWrapper) {
            socketWrapper.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.
            PollerEvent event = null;
            if (eventCache != null) {
                event = eventCache.pop();
            }
            if (event == null) {
                event = new PollerEvent(socket, OP_REGISTER);
            } else {
                event.reset(socket, OP_REGISTER);
            }
            addEvent(event);
        }
```
`addEvent`代码
```
        private void addEvent(PollerEvent event) {
            events.offer(event);
            if (wakeupCounter.incrementAndGet() == 0) {
                selector.wakeup();
            }
        }
```
到这里，Acceptor代码完成。请求已经接进来了。下面看Poller线程如何分发请求到worker线程池里。
首先看Poller的时序图：
![tomcat Poller时序图](/images/tomcat-poller.png)
> 1、poller线程run方法里，第一件事情是处理事件队列里的事件，将Acceptor里注册的channel注册到Poller中的selector中。
> 2、在selector中选取准备好的channel，循环进行处理。
> 3、将步骤二中选取的连接丢到工作线程池中进行处理。
至此，Poller的工作就完成了。接下来就是循环等待新的事件。
相关源码大致为:
`run()`的代码如下：
```
 /**
         * The background thread that adds sockets to the Poller, checks the
         * poller for triggered events and hands the associated socket off to an
         * appropriate processor as events occur.
         */
        @Override
        public void run() {
            // Loop until destroy() is called
            while (true) {

                boolean hasEvents = false;

                try {
                    if (!close) {
                        hasEvents = events();
                        if (wakeupCounter.getAndSet(-1) > 0) {
                            // If we are here, means we have other stuff to do
                            // Do a non blocking select
                            keyCount = selector.selectNow();
                        } else {
                            keyCount = selector.select(selectorTimeout);
                        }
                        wakeupCounter.set(0);
                    }
                  ......
                // Either we timed out or we woke up, process events first
                if (keyCount == 0) {
                    hasEvents = (hasEvents | events());
                }

                Iterator<SelectionKey> iterator =
                    keyCount > 0 ? selector.selectedKeys().iterator() : null;
                // Walk through the collection of ready keys and dispatch
                // any active event.
                while (iterator != null && iterator.hasNext()) {
                    SelectionKey sk = iterator.next();
                    NioSocketWrapper socketWrapper = (NioSocketWrapper) sk.attachment();
                    if (socketWrapper == null) {
                        iterator.remove();
                    } else {
                        iterator.remove();
                        //开始处理秀暗中的连接
                        processKey(sk, socketWrapper);
                    }
                }
    
            }

        }
```
`processKey()`的源码：
```
 protected void processKey(SelectionKey sk, NioSocketWrapper socketWrapper) {
            try {
                ...
                ...
                                } else if (!processSocket(socketWrapper, SocketEvent.OPEN_READ, true)) {
                                    closeSocket = true;
                                }
                            }
                           .....
                            if (closeSocket) {
                                cancelledKey(sk, socketWrapper);
                            }
                        }
                    }
                } else {
                    // Invalid key
                    cancelledKey(sk, socketWrapper);
                }
            } catch (CancelledKeyException ckx) {
             ....
        }

```
`processKey`主要功能是构造`processSocket`方法的参数。下面看`processSocket`相关的代码:
```
 public boolean processSocket(SocketWrapperBase<S> socketWrapper,
            SocketEvent event, boolean dispatch) {
        try {
            if (socketWrapper == null) {
                return false;
            }
            SocketProcessorBase<S> sc = null;
            if (processorCache != null) {
                sc = processorCache.pop();
            }
            if (sc == null) {
                sc = createSocketProcessor(socketWrapper, event);
            } else {
                sc.reset(socketWrapper, event);
            }
            Executor executor = getExecutor();
            if (dispatch && executor != null) {
                executor.execute(sc);
            } else {
                sc.run();
            }
        } catch (RejectedExecutionException ree) {
            getLog().warn(sm.getString("endpoint.executor.fail", socketWrapper) , ree);
            return false;
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            // This means we got an OOM or similar creating a thread, or that
            // the pool and its queue are full
            getLog().error(sm.getString("endpoint.process.fail"), t);
            return false;
        }
        return true;
    }

```
上面的方法就是向工作线程池里丢一个任务进行处理。丢进去的线程为`SocketProcessor`,该类继承自SocketProcessorBase,这个类实现了Runnable。
到这里，Poller相关的代码大致走完。下面就看worker线程里做了什么事情了。这里就正式的跟servlet打交道了。

我们主要看`SocketProcessor`类的`run()`方法的处理逻辑。
按照惯例，先上时序图。
![tomcat SocketProcessor时序图](/images/tomcat-worker.png)
从时序图上可以清楚的了解worker的工作流程。这里就不做详细的说明了。
关于servlet相关的处理，放在下篇文章里进行详细解读。
到此，tomcat接收请求的过程源码大体已经处理完成了。接下来就是servlet的处理了。

