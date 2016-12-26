---
title: dubbo filter链实现逻辑
date: 2016-12-02 17:35:21
categories: 框架
tags: [DUBBO,装饰器模式]
---

dubbo作为目前流行的开源rpc服务治理框架，其最大的优势之一就是具备高扩展性，这得益于其基于SPI的服务发现机制和动态字节码生成技术。用户实现dubbo提供的扩展接口，在classpath路径下简单配置实现类的全路径，dubbo就能够发现并加载该实现类，并组装到自身的处理逻辑中。

我们在实际业务开发中，使用最多的可能就是对Filter接口进行扩展，在服务调用链路中嵌入我们自身的处理逻辑，如日志打印、调用耗时统计等。

com.alibaba.dubbo.rpc.Filter接口的源码如下

```java
    @SPI
    public interface Filter {
        Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException;
    }
```

dubbo自带超时过滤器TimeoutFilter实现如下

```java
    @Activate(group = Constants.PROVIDER)
    public class TimeoutFilter implements Filter {

        private static final Logger logger = LoggerFactory.getLogger(TimeoutFilter.class);

        public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
            long start = System.currentTimeMillis();
            Result result = invoker.invoke(invocation);
            long elapsed = System.currentTimeMillis() - start;
            if (invoker.getUrl() != null
                    && elapsed > invoker.getUrl().getMethodParameter(invocation.getMethodName(),
                            "timeout", Integer.MAX_VALUE)) {
                if (logger.isWarnEnabled()) {
                    logger.warn("invoke time out. method: " + invocation.getMethodName()
                            + "arguments: " + Arrays.toString(invocation.getArguments()) + " , url is "
                            + invoker.getUrl() + ", invoke elapsed " + elapsed + " ms.");
                }
            }
            return result;
        }
        
    }
```

那dubbo是怎样将众多的Filter实例组织成一个链条的呢？实现代码在ProtocolFilterWrapper封装器中。该封装器实现了Protocol接口，并提供了一个参数类型为Protocol的构造方法。dubbo依据这个构造方法识别出封装器，并将该封装器作为其他Protocol接口实现的代理。

```java
    public class ProtocolFilterWrapper implements Protocol {

        private final Protocol protocol;

        // 带参数构造器，ExtensionLoad通过该构造器识别封装器
        public ProtocolFilterWrapper(Protocol protocol){
            if (protocol == null) {
                throw new IllegalArgumentException("protocol == null");
            }
            this.protocol = protocol;
        }

        public int getDefaultPort() {
            return protocol.getDefaultPort();
        }

        // 对提供方服务暴露进行封装，组装filter调用链
        public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
            if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
                return protocol.export(invoker);
            }
            return protocol.export(buildInvokerChain(invoker, Constants.SERVICE_FILTER_KEY, Constants.PROVIDER));
        }

        // 对消费方服务引用进行封装，组装filter调用链
        public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
            if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                return protocol.refer(type, url);
            }
            return buildInvokerChain(protocol.refer(type, url), Constants.REFERENCE_FILTER_KEY, Constants.CONSUMER);
        }

        public void destroy() {
            protocol.destroy();
        }

        // 构造filter调用链
        private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
            Invoker<T> last = invoker;
            List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
            if (filters.size() > 0) {
                for (int i = filters.size() - 1; i >= 0; i --) {
                    final Filter filter = filters.get(i);
                    final Invoker<T> next = last;
                    last = new Invoker<T>() {

                        public Class<T> getInterface() {
                            return invoker.getInterface();
                        }

                        public URL getUrl() {
                            return invoker.getUrl();
                        }

                        public boolean isAvailable() {
                            return invoker.isAvailable();
                        }

                        // 关键代码，单向链表指针传递
                        public Result invoke(Invocation invocation) throws RpcException {
                            return filter.invoke(next, invocation);
                        }

                        public void destroy() {
                            invoker.destroy();
                        }

                        @Override
                        public String toString() {
                            return invoker.toString();
                        }
                    };
                }
            }
            return last;
        }
    }
```

这里的关键代码在buildInvokerChain方法，参数invoker为实际的服务(对于消费方而言，就是服务的动态代理)。从ExtensionLoader获取到已经过排序的Filter列表(排序规则可参见ActivateComparator)，然后开始倒序组装。

这里是个典型的装饰器模式，不过装饰器链条上的每个节点都是一个匿名内部类Invoker实例。每个节点invoker持有一个Filter引用，一个下级invoker节点引用以及实际调用的invoker实例(虽然持有但并不实际调用，仅仅是提供获取实际invoker相关参数的功能，如getInterface，getUrl等方法)。通过invoke方法，invoker节点将下级节点传递给当前的filter进行调用。filter在执行invoke方法时，就会触发下级节点invoker调用其invoke方法，实现调用的向下传递。当到达最后一级invoker节点，即实际服务invoker，即可执行真实业务逻辑。

这条调用链的每个节点都为真实的invoker增加了自定义的功能，在整个链条上不断丰富功能，是典型的装饰器模式。