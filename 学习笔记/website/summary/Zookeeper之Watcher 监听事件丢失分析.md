## Zookeeper 之 Watcher 监听事件丢失分析

#### 背景

zk下不是所有的监听事件都会发送到客户端。比如连续更改一个节点的内容、创建节点再马上删除节点。

#### 案例

```java
import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.cache.NodeCache;
import org.apache.curator.framework.recipes.cache.NodeCacheListener;
import org.apache.curator.retry.ExponentialBackoffRetry;


public class CuratorNodeCacheTest {

    public static void main(String[] args) throws Exception {

        CuratorFramework client = getClient();
        String path = "/p1";
        final NodeCache nodeCache = new NodeCache(client,path);
        nodeCache.start();
        nodeCache.getListenable().addListener(new NodeCacheListener() {
            @Override
            public void nodeChanged() throws Exception {
                System.out.println("监听事件触发");
                System.out.println("重新获得节点内容为：" + new String(nodeCache.getCurrentData().getData()));
            }
        });
        client.setData().forPath(path,"111".getBytes());
        client.setData().forPath(path,"222".getBytes());
        client.setData().forPath(path,"333".getBytes());
        client.setData().forPath(path,"444".getBytes());
        client.setData().forPath(path,"555".getBytes());
        client.setData().forPath(path,"666".getBytes());
        Thread.sleep(15000);

    }

    private static CuratorFramework getClient(){
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000,3);
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString("127.0.0.1:2181")
                .retryPolicy(retryPolicy)
                .sessionTimeoutMs(6000)
                .connectionTimeoutMs(3000)
                .namespace("demo")
                .build();
        client.start();
        return client;
    }
}
```

执行结果：

```
监听事件触发
重新获得节点内容为：333
监听事件触发
重新获得节点内容为：555
监听事件触发
重新获得节点内容为：666
```

重复执行多次打印结果不同，但最后一个都会打印出 “666” 这个内容。

#### 分析

zookeeper 原生 API 注册 Watcher 需要反复注册，即 Watcher 触发之后就需要重新进行注册。另外，客户端断开之后重新连接到服务器也是需要一段时间。这就导致了 zookeeper 客户端不能够接收全部的 zookeeper 事件。zookeeper 保证的是**数据的最终一致性**。因此，对于此问题需要特别注意，在不要对 zookeeper 事件进行强依赖。

#### zookeeper 机制的特点

zookeeper 的 getData()，getChildren() 和 exists() 方法都可以注册 watcher 监听。而监听有以下几个特性：

- 一次性触发（one-time trigger） 当数据改变的时候，那么一个 Watch 事件会产生并且被发送到客户端中。但是客户端只会收到一次这样的通知，如果以后这个数据再次发生改变的时候，之前设置 Watch 的客户端将不会再次收到改变的通知，因为 Watch 机制规定了它是一个一次性的触发器。 当设置监视的数据发生改变时，该监视事件会被发送到客户端，例如，如果客户端调用了 getData(“/znode1”, true) 并且稍后 /znode1 节点上的数据发生了改变或者被删除了，客户端将会获取到 /znode1 发生变化的监视事件，而如果 /znode1 再一次发生了变化，除非客户端再次对 /znode1 设置监视，否则客户端不会收到事件通知。
- 发送给客户端（Sent to the client） 这个表明了 Watch 的通知事件是从服务器发送给客户端的，是异步的，这就表明不同的客户端收到的 Watch 的时间可能不同，但是 ZooKeeper 有保证：当一个客户端在看到 Watch 事件之前是不会看到结点数据的变化的。例如：A=3，此时在上面设置了一次 Watch，如果 A 突然变成 4 了，那么客户端会先收到 Watch 事件的通知，然后才会看到 A=4。

Zookeeper 客户端和服务端是通过 Socket 进行通信的，由于网络存在故障，所以监视事件很有可能不会成功地到达客户端，监视事件是异步发送至监视者的，Zookeeper 本身提供了保序性 (ordering guarantee)：即客户端只有首先看到了监视事件后，才会感知到它所设置监视的 znode 发生了变化 (a client will never see a change for which it has set a watch until it first sees the watch event). 网络延迟或者其他因素可能导致不同的客户端在不同的时刻感知某一监视事件，但是不同的客户端所看到的一切具有一致的顺序。

- 被设置了 watch 的数据（The data for which the watch was set） 这是指节点发生变动的不同方式。你可以认为 ZooKeeper 维护了两个 watch 列表：data watch 和 child watch。getData() 和 exists() 设置 data watch，而 getChildren() 设置 child watch。或者，可以认为 watch 是根据返回值设置的。getData() 和 exists() 返回节点本身的信息，而 getChildren() 返回 子节点的列表。因此，setData() 会触发 znode 上设置的 data watch（如果 set 成功的话）。一个成功的 create() 操作会触发被创建的 znode 上的数据 watch，以及其父节点上的 child watch。而一个成功的 delete() 操作将会同时触发一个 znode 的 data watch 和 child watch（因为这样就没有子节点了），同时也会触发其父节点的 child watch。

Watch 由 client 连接上的 ZooKeeper 服务器在本地维护。这样可以减小设置、维护和分发 watch 的开销。当一个客户端连接到一个新的服务器上时，watch 将会被以任意会话事件触发。当与一个服务器失去连接的时候，是无法接收到 watch 的。而当 client 重新连接时，如果需要的话，所有先前注册过的 watch，都会被重新注册。通常这是完全透明的。只有在一个特殊情况下，watch 可能会丢失：对于一个未创建的 znode 的 exist watch，如果在客户端断开连接期间被创建了，并且随后在客户端连接上之前又删除了，这种情况下，这个 watch 事件可能会被丢失。

#### ZooKeeper 对 Watch 提供了什么保障

对于 watch，ZooKeeper 提供了这些保障：

- Watch 与其他事件、其他 watch 以及异步回复都是有序的。 ZooKeeper 客户端库保证所有事件都会按顺序分发。
- 客户端会保障它在看到相应的 znode 的新数据之前接收到 watch 事件。// 这保证了在 process() 再次利用 zk client 访问时数据是存在的
- 从 ZooKeeper 接收到的 watch 事件顺序一定和 ZooKeeper 服务所看到的事件顺序是一致的。

#### 关于 Watch 的一些值得注意的事情

- Watch 是一次性触发器，如果得到了一个 watch 事件，而希望在以后发生变更时继续得到通知，应该再设置一个 watch。
- 因为 watch 是一次性触发器，而获得事件再发送一个新的设置 watch 的请求这一过程会有延时，所以无法确保看到了所有发生在 ZooKeeper 上的 一个节点上的事件。所以请处理好在这个时间窗口中可能会发生多次 znode 变更的这种情况。（可以不处理，但至少要意识到这一点）。// 也就是说, 在 process() 中如果处理得慢而没有注册 new watch 时, 在这期间有其它事件出现时是不会通知!!
- 一个 watch 对象或一个函数 / 上下文对，为一个事件只会被通知一次。比如，如果同一个 watch 对象在同一个文件上分别通过 exists 和 getData 注册了两次，而这个文件之后被删除了，这时这个 watch 对象将只会收到一次该文件的 deletion 通知。// 同一个 watch 注册同一个节点多次只会生成一个 event。
- 当从一个服务器上断开时（比如服务器出故障了），在再次连接上之前，将无法获得任何 watch。请使用这些会话事件来进入安全模式：在 disconnected 状态下将不会收到事件，所以程序在此期间应该谨慎行事。

#### 总结

 curator 帮开发人员封装了重复注册监听的过程，但是内部依旧需要重复进行注册，而在第一个 watcher 触发第二个 watcher 还未注册成功的间隙，进行节点数据的修改，显然无法收到 watcher 事件。