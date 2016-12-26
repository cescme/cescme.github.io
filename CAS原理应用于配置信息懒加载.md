---
title: CAS原理应用于配置信息懒加载
date: 2016-12-06 20:12:34
categories: 并发编程
tags: [CAS,并发]
---

### CAS基本原理

我们常用的syncronized关键字用于构建同步代码块，其本质是悲观锁，即只允许一个线程获取锁抢占到共享资源，其他线程都阻塞，直到锁被释放。而乐观锁多采用自旋的形式不断尝试获取锁，优势在于减少了线程上下文切换的开销，在一定的应用场景中能够取得较好的性能。如ReentrantLock、AtomicInteger等Concurrent包里常用的类其底层原理都是通过乐观锁机制实现。

以AtomicInteger中incrementAndGet方法的实现简述一下CAS(compare and swap)原理

```java
    public final int incrementAndGet() {
        for (;;) {
            int current = get(); // 获取当前value
            int next = current + 1;
            /* value值若没变化，则赋新值，并返回true；若变化，说明在get到当前值
            和进行cas操作之间，有其他线程已对value值进行修改，此时当前线程不能
            依据已持有的value值进行计算，否则将出现混乱，因此需要自旋尝试，直
            到出现前一种情况。*/
            if (compareAndSet(current, next)) 
                return next;
        }
    }
```

AtomicInteger对象中持有一个int型的value属性，表示当前值。

compareAndSet的源码如下:

```java
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
```

```java
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
      try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
      } catch (Exception ex) { throw new Error(ex); }
    }
```

可以看到，CAS的是sun.misc.Unsafe提供的compareAndSwapInt方法，其本质是执行了机器指令级别的原子操作，确保了其高效性。通过反射获取value属性的偏移量，此方式并不建议在我们的业务代码中使用，即不直接引用UnSafe类，而是可以利用Concurrent提供的包装类。

### 配置信息懒加载的优化

在业务中有这样的需求：从数据库或文件中拉取的配置信息至应用JVM堆内缓存，当配置有修改时，需要基于用户请求进行配置文件的懒加载更新。其实现流程如下：

修改配置 --> redis中升级版本号 --> 用户请求到达，检测到版本号变化 --> 拉取更新的配置

这里的懒加载体现在，当配置信息更新后，并不提供一个后门接口来刷新配置缓存，而是基于用户请求被动拉取。第一个察觉到版本信息变化的请求顺带去拉取新的配置信息，而其他请求或无法感知，或感知到但不进行拉取操作。这里需要处理的就是高并发下的同步，即确保只有一个请求线程进行拉取操作，对其他请求的性能影响尽量最低。

第一反应是加同步锁syncronized

```java
    void syncronized syncConfig(String newVersion) {
        if(!localVersion.equals(newVersion)) {
            queryConfig();// 进行配置拉取，耗时操作
            localVersion = newVersion;
        }
    }
```

很明显，缺陷在于，所有用户请求线程都会参与锁的竞争，即使此时缓存并没有更新，并出现大量阻塞，这一个点可能会对用户整体的请求性能产生很大影响。

于是，进一步的改进方案出现了，双重检查锁定

```java
    void syncConfig(String newVersion) {
        if(!localVersion.equals(newVersion)) {
            syncronized(object) {
                if(!localVersion.equals(newVersion)) {
                    queryConfig();// 进行配置拉取，耗时操作
                    localVersion = newVersion;
                }
            }
        }
    }
```

进入同步块之前首先检查一次版本是否一致，这可以使绝大多数线程避开锁竞争。然而这并没有万事大吉，在出现版本更新时，难以避免仍有部分线程加入到锁竞争行列，出现阻塞，影响请求响应时间。

我们期望的是只有一个线程去执行配置信息拉取，其他线程无阻塞。那么上文提到的CAS可以帮上忙。

```java
    private AtomicReference<String> ar = new AtomicReference<String>(localVersion);

    void syncConfig(String newVersion) {
        if(!localVersion.equals(newVersion)) {
            String currentVersion = localVersion;
            if(ar.compareAndSet(currentVersion, newVersion)) {
                queryConfig();// 进行配置拉取，耗时操作
            }
        }
    }
```

相对于AtomicInteger，AtomicReference提供了更加通用的处理逻辑，提供泛型支持。

外层的版本比较同样可以挡住绝大部分线程；CAS操作保证了只有一个线程能够更新版本号并进行配置拉取操作，其他线程只是进行一次CAS操作，开销极小，且无阻塞。

### 后记

以上的配置信息懒加载的方案本身就是存在缺陷的，因为引入了资源竞争，给用户请求带来了额外开销，也带了一定风险。理想情况下，应该是开一个后门接口，在控制台配置更改后主动触发缓存的更新；或者应用消息队列进行监听，当更新事件到达，对缓存进行更新。本文只论述CAS的应用场景，并不保证具体业务方案的合理性。