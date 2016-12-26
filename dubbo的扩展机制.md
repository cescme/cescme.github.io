---
title: dubbo的扩展机制
date: 2016-12-20 20:49:19
tags: dubbo
categories: 框架
---

dubbo被广泛接纳的其中的一个重要原因就是其强大灵活的扩展机制，开发者可自定义插件嵌入dubbo，实现灵活的业务需求。扩展机制的实现是利用了SPI规范，SPI规定了实现的基本原理，但不提供具体实现。dubbo利用SPI注解自己实现了一套SPI的插件加载机制，这其中主要是由ExtensionLoader扩展加载器完成的，本文就其中的实现机制进行简述。

### ExtensionLoader初始化

```java
    private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```

ExtensionLoader充当插件工厂角色，提供了一个私有的构造器。其入参type为扩展接口类型。dubbo通过SPI注解定义了可扩展的接口，如Filter、Transporter等。每个类型的扩展对应一个ExtensionLoader。SPI的value参数决定了默认的扩展实现。

当扩展类型是ExtensionFactory时，不指定objectFactory，否则初始化ExtensionFactory的ExtensionLoader并获取一个扩展适配器。扩展适配器将在下文讲述。

### 配置文件扫描

dubbo默认依次扫描META-INF/dubbo/internal/、META-INF/dubbo/、META-INF/services/三个classpath目录下的配置文件。配置文件以具体扩展接口全名命名，如com.alibaba.dubbo.rpc.Filter，内容如下：

```
    echo=com.alibaba.dubbo.rpc.filter.EchoFilter
    generic=com.alibaba.dubbo.rpc.filter.GenericFilter
    genericimpl=com.alibaba.dubbo.rpc.filter.GenericImplFilter
    token=com.alibaba.dubbo.rpc.filter.TokenFilter
    accesslog=com.alibaba.dubbo.rpc.filter.AccessLogFilter
    activelimit=com.alibaba.dubbo.rpc.filter.ActiveLimitFilter
    classloader=com.alibaba.dubbo.rpc.filter.ClassLoaderFilter
    context=com.alibaba.dubbo.rpc.filter.ContextFilter
    consumercontext=com.alibaba.dubbo.rpc.filter.ConsumerContextFilter
    exception=com.alibaba.dubbo.rpc.filter.ExceptionFilter
    executelimit=com.alibaba.dubbo.rpc.filter.ExecuteLimitFilter
    deprecated=com.alibaba.dubbo.rpc.filter.DeprecatedFilter
    compatible=com.alibaba.dubbo.rpc.filter.CompatibleFilter
    timeout=com.alibaba.dubbo.rpc.filter.TimeoutFilter
    monitor=com.alibaba.dubbo.monitor.support.MonitorFilter
    validation=com.alibaba.dubbo.validation.filter.ValidationFilter
    cache=com.alibaba.dubbo.cache.filter.CacheFilter
    trace=com.alibaba.dubbo.rpc.protocol.dubbo.filter.TraceFilter
    future=com.alibaba.dubbo.rpc.protocol.dubbo.filter.FutureFilter
```

等号前为扩展名，其后为扩展实现类全路径名。
ExtensionLoader中loadFile方法加载配置文件。
1.逐行读取配置文件，提取出扩展名或扩展类路径
2.利用Class.forName方法进行类加载

```java
     Class<?> clazz = Class.forName(line, true, classLoader);
```

3.处理Adaptive注解，若存在则将该实现类保存至cachedAdaptiveClass属性

```java
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        if(cachedAdaptiveClass == null) {
            cachedAdaptiveClass = clazz;
        } else if (! cachedAdaptiveClass.equals(clazz)) {
            throw new IllegalStateException("More than 1 adaptive class found:"    + cachedAdaptiveClass.getClass().getName()
                    + ", " + clazz.getClass().getName());
        }
    }
```

4.尝试获取参数类型为当前扩展类型的构造器方法，若成功则表明存在该扩展的封装类型，将封装类型存入wrappers集合；否则转入第五步

```java
    try {
        clazz.getConstructor(type); // 扩展类型参数的构造器是封装器的约定特征，目前dubbo中默认的只有Filter和Listener的封装器
        Set<Class<?>> wrappers = cachedWrapperClasses;
        if (wrappers == null) {
            cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
            wrappers = cachedWrapperClasses;
        }
        wrappers.add(clazz);
    } catch (NoSuchMethodException e) {
        // 第五步
    }
```

5.处理active注解，将扩展名对应active注解存入cachedActivates

```java
    Activate activate = clazz.getAnnotation(Activate.class);
    if (activate != null) {
        cachedActivates.put(names[0], activate);
    }
```

### 扩展适配器

在dubbo扩展中，适配器模式被广泛使用，其作用在于为同一扩展类型下的多个扩展实现的调用提供路由功能，如指定优先级等。dubbo提供了两种方式来生成扩展适配器：

1.静态扩展
所谓的静态扩展就是提前通过编码的形式确定扩展的具体实现，且该实现类由Adaptive注解标注，如AdaptiveCompiler。在加载配置文件的loadFile方法中，已经描述过处理该类型扩展的逻辑。

2.动态扩展
动态扩展即通过动态代理生成扩展类的动态代理类，在dubbo中是通过javassist技术生成的。与传统的jdk动态代理、cglib不同，javassist提供封装后的API对字节码进行间接操作，简单易用，不关心具体字节码，灵活性更高，且处理效率也较高，是dubbo默认的编译器。

首先，ExtensionLoader会调用createAdaptiveExtensionClassCode方法为当前扩展类型生成适配器的源码，其处理要点如下：

- 检查是否至少有一个方法有Adaptive注解，若不存在则抛出异常，即要完成动态代理，必须有方法标注了Adaptive注解
- 对于有Adaptive注解的方法，判断其入参中是否有URL类型的参数，或者复杂参数中是否有URL类型的属性，若没有则抛出异常。这里体现出了为什么dubbo要提供动态适配器生成机制。dubbo中的URL总线提供了服务的全部信息，而开发者可以定义差异化的服务配置，因此生成的URL差异化也较大，若全部靠用户硬编码静态适配器的话效率太低。有了动态代理，dubbo可以根据URL参数动态地生成适配器的适配逻辑，确定扩展实现的获取优先级。因此，URL作为参数直接或间接传入是必须的，否则失去了动态生成的凭据。

```java
    int urlTypeIndex = -1;
    for (int i = 0; i < pts.length; ++i) {
        if (pts[i].equals(URL.class)) {
            urlTypeIndex = i;
            break;
        }
    }
    // 有类型为URL的参数
    if (urlTypeIndex != -1) {
        // Null Point check
        String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"url == null\");",
                        urlTypeIndex);
        code.append(s);
        
        s = String.format("\n%s url = arg%d;", URL.class.getName(), urlTypeIndex); 
        code.append(s);
    }
    // 参数没有URL类型
    else {
        String attribMethod = null;
        
        // 找到参数的URL属性
        LBL_PTS:
        for (int i = 0; i < pts.length; ++i) {
            Method[] ms = pts[i].getMethods();
            for (Method m : ms) {
                String name = m.getName();
                if ((name.startsWith("get") || name.length() > 3)
                        && Modifier.isPublic(m.getModifiers())
                        && !Modifier.isStatic(m.getModifiers())
                        && m.getParameterTypes().length == 0
                        && m.getReturnType() == URL.class) {
                    urlTypeIndex = i;
                    attribMethod = name;
                    break LBL_PTS;
                }
            }
        }
        if(attribMethod == null) {
            throw new IllegalStateException("fail to create adative class for interface " + type.getName()
                    + ": not found url parameter or url attribute in parameters of method " + method.getName());
        }
        // 后续处理...
    }
```

- 对于无Adaptive注解的方法，调用时直接抛出异常
- 根据adaptive注解的value数组值，及SPI注解定义的默认扩展名，确定适配逻辑，即扩展获取的优先级，这里不再罗列代码，下面为一个具体生成的适配器源码：

```java
    package com.alibaba.dubbo.remoting;

    import com.alibaba.dubbo.common.extension.ExtensionLoader;

    public class Transporter$Adpative implements com.alibaba.dubbo.remoting.Transporter {

        public com.alibaba.dubbo.remoting.Client connect(com.alibaba.dubbo.common.URL arg0, com.alibaba.dubbo.remoting.ChannelHandler arg1) throws com.alibaba.dubbo.common.URL {
            if (arg0 == null) throw new IllegalArgumentException("url == null");
            com.alibaba.dubbo.common.URL url = arg0;
            String extName = url.getParameter("client", url.getParameter("transporter", "netty")); // 处理顺序
            if(extName == null) 
                throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" + url.toString() + ") use keys([client, transporter])");
            com.alibaba.dubbo.remoting.Transporter extension = (com.alibaba.dubbo.remoting.Transporter)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.Transporter.class).getExtension(extName);
            return extension.connect(arg0, arg1);
        }

        public com.alibaba.dubbo.remoting.Server bind(com.alibaba.dubbo.common.URL arg0, com.alibaba.dubbo.remoting.ChannelHandler arg1) throws com.alibaba.dubbo.common.URL {
            if (arg0 == null) throw new IllegalArgumentException("url == null");
            com.alibaba.dubbo.common.URL url = arg0;
            String extName = url.getParameter("server", url.getParameter("transporter", "netty")); // 处理顺序
            if(extName == null) 
                throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" + url.toString() + ") use keys([server, transporter])");
            com.alibaba.dubbo.remoting.Transporter extension = (com.alibaba.dubbo.remoting.Transporter)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.Transporter.class).getExtension(extName);
            return extension.bind(arg0, arg1);
        }
    }
```

可以看到，核心逻辑是获取扩展名extName，以bind方法为例，其获取优先级是server，transporter，netty，可参见URL的getParameter方法源码。其中netty是Transporter接口的SPI注解确定的默认值，而server和transporter是bind方法的Adaptive注解定义的。

```java
    @SPI("netty")
    public interface Transporter {

        @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
        Server bind(URL url, ChannelHandler handler) throws RemotingException;

        @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
        Client connect(URL url, ChannelHandler handler) throws RemotingException;

    }
```

拿到扩展名后，再从ExtensionLoader获取到扩展实例，调用具体的bind方法。

源码生成后，ExtensionLoader再调用默认的JavassitCompiler进行编译和类加载，其具体实现原理不在本文讨论范围，有机会的话后续会介绍这部分内容。

ExtensionLoader提供了获取扩展适配器的方法，优先查看是否有静态适配器，否则会使用动态适配器。

### 封装类

dubbo中存在一种对于扩展的封装类，其功能是将各扩展实例串联起来，形成扩展链，比如过滤器链，监听链。当调用ExtensionLoader的getExtension方法时，会做拦截处理，如果存在封装器，则返回封装器实现，而将真实实现通过构造方法注入到封装器中。

```java
    T instance = (T) EXTENSION_INSTANCES.get(clazz);
    if (instance == null) {
        EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
        instance = (T) EXTENSION_INSTANCES.get(clazz);
    }
    injectExtension(instance);
    Set<Class<?>> wrapperClasses = cachedWrapperClasses;
    if (wrapperClasses != null && wrapperClasses.size() > 0) {
        for (Class<?> wrapperClass : wrapperClasses) {
            instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
        }
    }
```

至于封装器是怎样组织起调用链的，可以参见[dubbo filter链实现逻辑](https://cescme.github.io/2016/12/02/dubbo%20filter%E9%93%BE%E5%AE%9E%E7%8E%B0%E9%80%BB%E8%BE%91/)

这里有个injectExtension方法，其作用是：如果当前扩展实例存在其他的扩展属性，则通过反射调用其set方法设置扩展属性。该扩展属性是适配器类型，也是通过ExtensionLoader获取的。

### 后记

ExtensionLoader作为一个IOC插件容器，为dubbo的插件体系运作提供了保障，可以说是dubbo中的核心。掌握了其基本原理，才有助于我们更好地分析dubbo源码。后续的dubbo源码分析中也会以此为基础展开介绍。