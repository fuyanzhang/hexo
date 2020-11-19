---
title: dubbo spi机制源码阅读
date: 2020-07-17 09:52:54
tags:
  - Dubbo
  - Java
  - 源码阅读
  - rpc
categories:
  - 源码阅读
  - 技术
---

dubbo 的扩展能力很强大。他是通过扩展 Java 的 spi 机制得到的。

### Java Spi 机制介绍

SPI 是 Service Provider Interface 的缩写。是 jdk 专门针对类似插件扩展的一种功能。为定义的接口提供查找实现的机制。

java spi 的具体约定为:当服务的提供者，提供了服务接口的一种实现之后，在 jar 包的 META-INF/services/目录里同时创建一个以服务接口命名的文件。该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该 jar 包 META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入。 基于这样一个约定就能很好的找到服务接口的实现类，而不需要再代码里制定。jdk 提供服务实现查找的一个工具类：java.util.ServiceLoader

<!--more-->

#### 一个例子

定义一个接口：

```
package com.test.spi;
public interface JavaSpiDemoService {

   void sayHi(String msg);
}
```

接口的实现：

```
package com.test.spi;
public class EnJavaSpiDemoServiceImpl implements JavaSpiDemoService {
    @Override
    public void sayHi(String msg) {
        System.out.println("Hello..." +msg);
    }
}
```

另一个实现：

```
package com.test.spi;
public class ChJavaSpiDemoServiceImpl implements JavaSpiDemoService {
    @Override
    public void sayHi(String msg) {
        System.out.println("你好..."+msg);
    }
}
```

在 classpath 里新建文件 META-INF/services/com.test.spi.JavaSpiDemoService ，文件内容为：

```
com.test.spi.ChJavaSpiDemoServiceImpl
com.test.spi.EnJavaSpiDemoServiceImpl
```

写一个 main 方法，执行之。。。

```
package com.test.spi;

import java.util.Iterator;
import java.util.ServiceLoader;

/**
 * Author:yanzhang.fu
 * Date:2018/6/5
 * Description:
 * Modified By：
 **/
public class SpiMain {
    public static void main(String[] args) {
        ServiceLoader<JavaSpiDemoService> serviceLoader = ServiceLoader.load(JavaSpiDemoService.class);
        Iterator<JavaSpiDemoService> it = serviceLoader.iterator();
        while(it.hasNext()){
            JavaSpiDemoService service = it.next();
            service.sayHi("world");
        }
    }
}
```

执行结果如下：
![jdk spi执行结果.png](https://upload-images.jianshu.io/upload_images/11942209-879d17802d7692d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### dubbo spi 机制

dubbo 的 spi 机制思想来源于 jdk 的 spi 思想。对 jdk 的 spi 进行了改进。下面一段话是 dubbo 官网的一段描述。

```
Dubbo 的扩展点加载从 JDK 标准的 SPI (Service Provider Interface) 扩展点发现机制加强而来。

Dubbo 改进了 JDK 标准的 SPI 的以下问题：

JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。
如果扩展点加载失败，连扩展点的名称都拿不到了。比如：JDK 标准的 ScriptEngine，通过 getName() 获取脚本类型的名称，但如果 RubyScriptEngine 因为所依赖的 jruby.jar 不存在，导致 RubyScriptEngine 类加载失败，这个失败原因被吃掉了，和 ruby 对应不起来，当用户执行 ruby 脚本时，会报不支持 ruby，而不是真正失败的原因。
增加了对扩展点 IoC 和 AOP 的支持，一个扩展点可以直接 setter 注入其它扩展点。
```

加载插件的入口类是`ExtensionLoader`.通过`getExtensionLoader`方法获取插件的加载器。

```
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        if (type == null)
            throw new IllegalArgumentException("Extension type == null");
//只有是接口，才可以被扩展
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
        }
// 只有被SPI注解修饰，才算是可扩展的点
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type(" + type +
                    ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
        }
// 从缓存中获取加载器，没有，则新建一个加载器，然后放到缓存里，key为接口，value为加载器
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```

下面看`new ExtensionLoader<T>(type)`做了什么操作。

```
private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```

```
 public T getAdaptiveExtension() {
    // 缓存中获取
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
            if (createAdaptiveInstanceError == null) {
                synchronized (cachedAdaptiveInstance) {
                    instance = cachedAdaptiveInstance.get();
                    if (instance == null) {
                        try {
                          // 创建一个
                            instance = createAdaptiveExtension();
                            //设置代理对象
                            cachedAdaptiveInstance.set(instance);
                        } catch (Throwable t) {
                            createAdaptiveInstanceError = t;
                            throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                        }
                    }
                }
            } else {
                throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
            }
        }

        return (T) instance;
    }
```

```
private T createAdaptiveExtension() {
        try {
          //
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());
        } catch (Exception e) {
            throw new IllegalStateException("Can not create adaptive extension " + type + ", cause: " + e.getMessage(), e);
        }
    }
```

```
private Class<?> getAdaptiveExtensionClass() {
        //加载spi配置文件，接口的实现类
        getExtensionClasses();
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
      // 动态生成扩展实现的代理类
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }
```

下面先来看看 spi 配置文件是如何加载的。。。即`getExtensionClasses`的实现。

```
private Map<String, Class<?>> getExtensionClasses() {
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    //加载配置文件
                    classes = loadExtensionClasses();
                    //放入本地缓存
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
```

```
 private Map<String, Class<?>> loadExtensionClasses() {
    //可以指定默认实现
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if (defaultAnnotation != null) {
            String value = defaultAnnotation.value();
            if ((value = value.trim()).length() > 0) {
                String[] names = NAME_SEPARATOR.split(value);
                if (names.length > 1) {
                    throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                            + ": " + Arrays.toString(names));
                }
                if (names.length == 1) cachedDefaultName = names[0];
            }
        }

        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
      //dubbo里会从3个文件夹里加载配置。META-INF/dubbo/internal优先级最高，META-INF/dubbo/里文件次之，META-INF/services/里优先级最低。
    //即，如果internal里，就以internal里的为准，若没有，则找dubbo里的，还没有，再到services里找。
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
        loadDirectory(extensionClasses, DUBBO_DIRECTORY);
        loadDirectory(extensionClasses, SERVICES_DIRECTORY);
        return extensionClasses;
    }
```

看下`loadDirectory`的方法实现。

```
 private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir) {
        String fileName = dir + type.getName();
        try {
            Enumeration<java.net.URL> urls;
            ClassLoader classLoader = findClassLoader();
            if (classLoader != null) {
                urls = classLoader.getResources(fileName);
            } else {
                urls = ClassLoader.getSystemResources(fileName);
            }
            if (urls != null) {
                while (urls.hasMoreElements()) {
                    java.net.URL resourceURL = urls.nextElement();
                  //此处是真正的加载的地方
                    loadResource(extensionClasses, classLoader, resourceURL);
                }
            }
        } catch (Throwable t) {
            logger.error("Exception when load extension class(interface: " +
                    type + ", description file: " + fileName + ").", t);
        }
    }
```

```
private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader, java.net.URL resourceURL) {
        try {
            BufferedReader reader = new BufferedReader(new InputStreamReader(resourceURL.openStream(), "utf-8"));
            try {
                String line;
                while ((line = reader.readLine()) != null) {
                    final int ci = line.indexOf('#');
                    if (ci >= 0) line = line.substring(0, ci);
                    line = line.trim();
                    if (line.length() > 0) {
                        try {
                            String name = null;
                            int i = line.indexOf('=');
                            if (i > 0) {
                                name = line.substring(0, i).trim();
                                line = line.substring(i + 1).trim();
                            }
                            if (line.length() > 0) {
                                loadClass(extensionClasses, resourceURL, Class.forName(line, true, classLoader), name);
                            }
                        } catch (Throwable t) {
                            IllegalStateException e = new IllegalStateException("Failed to load extension class(interface: " + type + ", class line: " + line + ") in " + resourceURL + ", cause: " + t.getMessage(), t);
                            exceptions.put(line, e);
                        }
                    }
                }
            } finally {
                reader.close();
            }
        } catch (Throwable t) {
            logger.error("Exception when load extension class(interface: " +
                    type + ", class file: " + resourceURL + ") in " + resourceURL, t);
        }
    }
```

上面代码可以看到，是逐行加载配置的。已`=`分割，`=`之前的为 name，之后的为 line。当 line 不为空，则说明有实现的类。加载实现类到 classloader 中。

```
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
        if (!type.isAssignableFrom(clazz)) {
            throw new IllegalStateException("Error when load extension class(interface: " +
                    type + ", class line: " + clazz.getName() + "), class "
                    + clazz.getName() + "is not subtype of interface.");
        }
// 扩展接口是否被Adaptive注解修饰，说明该类是一个适配类，是，则将cachedAdaptiveClass 设为当前接口。
        if (clazz.isAnnotationPresent(Adaptive.class)) {
            if (cachedAdaptiveClass == null) {
                cachedAdaptiveClass = clazz;
            } else if (!cachedAdaptiveClass.equals(clazz)) {
                throw new IllegalStateException("More than 1 adaptive class found: "
                        + cachedAdaptiveClass.getClass().getName()
                        + ", " + clazz.getClass().getName());
            }
        // 是不是包装类，是，则将包装类放到cachedWrapperClasses中。
        } else if (isWrapperClass(clazz)) {
            Set<Class<?>> wrappers = cachedWrapperClasses;
            if (wrappers == null) {
                cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
                wrappers = cachedWrapperClasses;
            }
            wrappers.add(clazz);
        } else {

            clazz.getConstructor();
            if (name == null || name.length() == 0) {
                name = findAnnotationName(clazz);
                if (name.length() == 0) {
                    throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
                }
            }
            String[] names = NAME_SEPARATOR.split(name);
            if (names != null && names.length > 0) {
                Activate activate = clazz.getAnnotation(Activate.class);
                if (activate != null) {
                    cachedActivates.put(names[0], activate);
                }
                for (String n : names) {
                    if (!cachedNames.containsKey(clazz)) {
                        cachedNames.put(clazz, n);
                    }
                    Class<?> c = extensionClasses.get(n);
                    if (c == null) {
                        extensionClasses.put(n, clazz);
                    } else if (c != clazz) {
                        throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
                    }
                }
            }
        }
    }
```

上面的代码里，如果配置成 pluginA,pluginB=com.test.dubbo.xxxImpl，会出现什么情况呢。写代码验证一下。
_extensionClasses 里会有两个值，而 cachedNames 里只有一个值，通过接口查找插件的时候，可能会丢插件，不过没有影响，两个插件其实是一个东西_
以上，将加载配置文件的代码走读完成。主要是相当于一个初始化的过程。真正的处理还是`createAdaptiveExtensionClass`和`injectExtension`。
接下来，我们先看看`createAdaptiveExtensionClass`
若当前处理的插件不是一个适配类，即类不被 Adaptive 注解。则生成一个适配类。相当于动态代理。`createAdaptiveExtensionClass`里就是生成适配类的逻辑。
代码如下：

```
private Class<?> createAdaptiveExtensionClass() {
        // 生成适配类的代码。动态组装Java代码。是一个字符串
        String code = createAdaptiveExtensionClassCode();
        // 获取类加载器
        ClassLoader classLoader = findClassLoader();
        // 获取编译插件
        com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        //编译该Java代码并加载到JVM里
        return compiler.compile(code, classLoader);
    }
```

以 Procotol 插件为例，生成的代理类代码如下：

```
package com.alibaba.dubbo.rpc;

import com.alibaba.dubbo.common.extension.ExtensionLoader;

// 类名以插件接口名+Adaptive命名
public class Protocol$Adaptive implements com.alibaba.dubbo.rpc.Protocol {
    //未被Adaptive注解的方法，抛异常
    public void destroy() {
        throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

////未被Adaptive注解的方法，抛异常
    public int getDefaultPort() {
        throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }
//被Adaptive注解的方法,通过url里的标识取不同的插件。extName即为配置文件里key值。
//这里就能看到与jdk的spi的区别了
    public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }

    public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws com.alibaba.dubbo.rpc.RpcException {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }
}
```

上面的代理类其实就是通过 URL 来获取具体的插件进行不同的处理。
injectExtension 方法也就是比较简单的注入代理类的一些属性，并没有什么多的功能。

```
private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
                for (Method method : instance.getClass().getMethods()) {
                    if (method.getName().startsWith("set")
                            && method.getParameterTypes().length == 1
                            && Modifier.isPublic(method.getModifiers())) {
                        Class<?> pt = method.getParameterTypes()[0];
                        try {
                            String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                            Object object = objectFactory.getExtension(pt, property);
                            if (object != null) {
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("fail to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```

通过以上代码，就能通过 xml 里的配置，获取到相应的处理插件了。

了解了 dubbo 的微内核加插件机制，接下来看 dubbo 的代码就清晰简单多了。
