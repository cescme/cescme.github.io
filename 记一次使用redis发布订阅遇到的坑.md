---
title: 记一次使用redis发布订阅遇到的坑
date: 2016-12-21 22:22:02
tags: redis
categories: 框架
---

有一个计算排名的定时任务，处理过程中分主任务和子任务。主任务负责子任务的切分，如分配100个子任务，子任务执行具体的分布式任务，当所有子任务结束后，需要主任务汇总子任务处理的数据并入库。

公司目前提供的定时调度系统还不支持子任务执行成功通知主任务的机制，因此需要自己实现。经过调研，选择了redis自带的发布订阅机制，原因在于其轻量级，接入成本低。

Jedis提供了JedisPubSub抽象类，以下为实现的一个监听器，onMessage方法用于处理订阅的消息。在这里，RankListener用于汇总子任务执行后的数据汇总和入库操作，并在结束后取消订阅。

```java
    public class RankListener extends JedisPubSub {

        @Override
        public void onMessage(String channel, String message) {
            try {
                // 汇总数据
            } finally {
                this.unsubscribe(); // 执行完毕，取消订阅
            }
        }
    }
```

主任务每成功生成一个子任务，则将redis中一个键count加一。当所有子任务生成结束后，执行消息的订阅，通过subscribe方法使rankListener订阅CHANNEL通道的消息。

```java
    jedisClient.getClient().subscribe(rankListener, CHANNEL);
```

子任务在执行结束后，无论成功失败，都将count值减一。当发现count值为0时，表示所有子任务都已结束，则发布任务结束的消息。

```java
    try {
        // 正常业务逻辑
    } finally {
        // 标记任务结束
        if(jedisClient.decr("count") == 0) { // 若检测到所有任务执行完毕，发送完成通知
            jedisClient.getClient().publish(CHANNEL, "done");
        }
    }
```

此时，rankListener通过onMessage方法接收到done消息，执行汇总任务。

这个流程看似完善，也将信号的控制放入了finally代码块，确保了异常情况下能够执行汇总任务并取消订阅。

悲剧的是，异常不总是理想中的异常，比如意外宕机，就被我赶上了。

某次在子任务的执行过程中(假设有六台机器在执行，每台机器并发执行10个子任务)，运维由于操作失败将其中一台机器重启了。那重启就重启了呗，定时调度控制台随后就报有两个子任务执行失败了，而且由于这两个任务是在执行过程中宕机的，因此控制台已经不能手工重新执行了。

没办法，失败了那就只能整体重试了，于是点了主任务重新执行。细心的你可能已经发现会出什么问题了(为什么我没发现！！)，子任务发布done消息后，rankListener的onMessage方法居然执行了两次。不幸中的万幸，数据库表加了唯一约束，因此重复数据并没有入库，但报了一堆异常，报警邮件发到老大那里，脸色真不好看~0-0~。

到这里，已经能够比较明显地找到问题所在。在第一次执行宕机时，由于两个子任务意外中途挂掉，并没有执行count值减一的操作(即使是在finally代码块里)，因此也不可能发出done的消息，这就意味着rankListener无法收到消息，一直不会解除订阅，就一直挂在那里了。当重新执行主任务，又会有另一个线程进行订阅，当收到done消息后，两个线程同时执行onMessage方法，自然会出现重复报警。

明确了原因，要怎么解决呢？这里，我们进入jedis发布订阅的源码，看看jedis的订阅监听是怎样运作的，以及怎样结束监听。

```java
    // 订阅消息执行runWithAnyNode方法
    public void subscribe(final JedisPubSub jedisPubSub, final String... channels) {
        new JedisClusterCommand<Integer>(connectionHandler, maxRedirections) {
            @Override
            public Integer execute(Jedis connection) {
                connection.subscribe(jedisPubSub, channels);
                return 0;
            }
        }.runWithAnyNode();
    }
```

```java
    // JedisClusterCommand为抽象类
    public T runWithAnyNode() {
        Jedis connection = null;
        try {
            connection = connectionHandler.getConnection();
            return execute(connection); // 执行子类方法
        } catch (JedisConnectionException e) {
            throw e;
        } finally {
            releaseConnection(connection);
        }
    }
```

这里是一个很明显的模板方法模式，抽象父类调用子类具体的方法实现，而子类是一个匿名的内部内，最终还是调用了Jedis的subscribe方法。

```java
    public void subscribe(final JedisPubSub jedisPubSub, final String... channels) {
        client.setTimeoutInfinite();
        try {
            jedisPubSub.proceed(client, channels);
        } finally {
            client.rollbackTimeout();
        }
    }
```

```java
    public void proceed(Client client, String... channels) {
        this.client = client;
        client.subscribe(channels);
        client.flush();
        process(client);
    }

    private void process(Client client) {
        do {
            List<Object> reply = client.getRawObjectMultiBulkReply();
            final Object firstObj = reply.get(0);
            if (!(firstObj instanceof byte[])) {
                throw new JedisException("Unknown message type: " + firstObj);
            }
            final byte[] resp = (byte[]) firstObj;
            if (Arrays.equals(SUBSCRIBE.raw, resp)) {
                subscribedChannels = ((Long) reply.get(2)).intValue();
                final byte[] bchannel = (byte[]) reply.get(1);
                final String strchannel = (bchannel == null) ? null : SafeEncoder.encode(bchannel);
                onSubscribe(strchannel, subscribedChannels);
            } else if (Arrays.equals(UNSUBSCRIBE.raw, resp)) {
                subscribedChannels = ((Long) reply.get(2)).intValue();
                final byte[] bchannel = (byte[]) reply.get(1);
                final String strchannel = (bchannel == null) ? null : SafeEncoder.encode(bchannel);
                onUnsubscribe(strchannel, subscribedChannels);
            } else if (Arrays.equals(MESSAGE.raw, resp)) {
                final byte[] bchannel = (byte[]) reply.get(1);
                final byte[] bmesg = (byte[]) reply.get(2);
                final String strchannel = (bchannel == null) ? null : SafeEncoder.encode(bchannel);
                final String strmesg = (bmesg == null) ? null : SafeEncoder.encode(bmesg);
                onMessage(strchannel, strmesg);
            } else if (Arrays.equals(PMESSAGE.raw, resp)) {
                final byte[] bpattern = (byte[]) reply.get(1);
                final byte[] bchannel = (byte[]) reply.get(2);
                final byte[] bmesg = (byte[]) reply.get(3);
                final String strpattern = (bpattern == null) ? null : SafeEncoder.encode(bpattern);
                final String strchannel = (bchannel == null) ? null : SafeEncoder.encode(bchannel);
                final String strmesg = (bmesg == null) ? null : SafeEncoder.encode(bmesg);
                onPMessage(strpattern, strchannel, strmesg);
            } else if (Arrays.equals(PSUBSCRIBE.raw, resp)) {
                subscribedChannels = ((Long) reply.get(2)).intValue();
                final byte[] bpattern = (byte[]) reply.get(1);
                final String strpattern = (bpattern == null) ? null : SafeEncoder.encode(bpattern);
                onPSubscribe(strpattern, subscribedChannels);
            } else if (Arrays.equals(PUNSUBSCRIBE.raw, resp)) {
                subscribedChannels = ((Long) reply.get(2)).intValue();
                final byte[] bpattern = (byte[]) reply.get(1);
                final String strpattern = (bpattern == null) ? null : SafeEncoder.encode(bpattern);
                onPUnsubscribe(strpattern, subscribedChannels);
            } else {
                throw new JedisException("Unknown message type: " + firstObj);
            }
        } while (isSubscribed());
    }
```

最终还是执行了JedisPubSub的process方法，这个方法居然是一个循环，一直会尝试从redis集群拿数据，并根据数据类型做相应的处理。到这里清楚了，这个监听订阅并没有另起一个线程来做，主线程执行订阅方法其实就是进入了一个循环体在对消息做不断的轮询。这里我还是吐槽下这种实现，比如没有超时机制，无法控制监听线程的野蛮增长，多个线程处于这种死循环状态很容易耗尽CPU资源。好吧，本身人家redis也不是专业做消息队列的，只是提供了一种轻量级的实现，使用中遇到问题还是只能我们开发者自己担着，谁让我们贪便宜图省事呢。

问题还是要想办法解决。这个循环体唯一的退出条件是isSubscribe方法返回false，看看源码:

```java
    public boolean isSubscribed() {
        return subscribedChannels > 0;
    }
```

subscribedChannels，即订阅的通道数，需等于0才能返回false，那就找找哪里能对这个属性进行写操作，最终发现process循环体内部处理消息逻辑中会对其赋值。JedisPubSub中自带了unSubsrcibe方法，会向redis集群发送一个取消订阅的消息，而process循环体中会受到取消订阅的消息，并将subscribedChannels置为0，退出循环体，即实现取消监听。

好了，了解了取消监听的机制，那我们可以在主任务开始时首先执行一次rankListener的unSubscribe方法，将所有订阅者全部取消，这样持有rankListener对象的所有监听线程都能够退出死循环。

也许大家可能会说，这种订阅了消费一次就取消真的处理起来好麻烦，为何不专门弄一个线程来做监听，并不需要取消。这种方案也不是不行，但鉴于定时任务每天只跑一次，监听者也只需要处理一次订阅消息，专门弄一个线程一直死循环监听对CPU资源也是一种浪费，还是需要时激活一次然后释放，能够达到对资源的更有效的利用。