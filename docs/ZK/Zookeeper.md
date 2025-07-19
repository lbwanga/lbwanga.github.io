ZooKeeper 是一个开源的**分布式协调服务**，它的设计目标是将那些复杂且容易出错的分布式一致性服务封装起来，构成一个高效可靠的原语集，并以一系列简单易用的接口提供给用户使用。

ZooKeeper 为我们提供了高可用、高性能、稳定的分布式数据一致性解决方案，通常被用于实现诸如**数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master 选举、分布式锁和分布式队列**等功能。这些功能的实现主要依赖于 ZooKeeper 提供的 **数据存储+事件监听** 功能。

**特点**：

* **顺序一致性：** 从同一客户端发起的事务请求，最终将会严格地按照顺序被应用到 ZooKeeper 中去。
* **原子性：** 所有事务请求的处理结果在整个集群中所有机器上的应用情况是一致的，也就是说，要么整个集群中所有的机器都成功应用了某一个事务，要么都没有应用。
* **单一系统映像：** 无论客户端连到哪一个 ZooKeeper 服务器上，其看到的服务端数据模型都是一致的。
* **可靠性：** 一旦一次更改请求被应用，更改的结果就会被持久化，直到被下一次更改覆盖。
* **实时性：** 一旦数据发生变更，其他节点会实时感知到。每个客户端的系统视图都是最新的。
* **集群部署**：3~5 台（最好奇数台）机器就可以组成一个集群，每台机器都在内存保存了 ZooKeeper 的全部数据，机器之间互相通信同步数据，客户端连接任何一台机器都可以。
* **高可用：**如果某台机器宕机，会保证数据不丢失。集群中挂掉不超过一半的机器，都能保证集群可用。比如 3 台机器可以挂 1 台，5 台机器可以挂 2 台。



**应用场景**：

* **命名服务**：可以通过 ZooKeeper 的顺序节点生成全局唯一 ID。

* **数据发布/订阅**：通过 **Watcher 机制** 可以很方便地实现数据发布/订阅。当你将数据发布到 ZooKeeper 被监听的节点上，其他机器可通过监听 ZooKeeper 上节点的变化来实现配置的动态更新。

* **分布式锁**：通过创建唯一节点获得分布式锁，当获得锁的一方执行完相关代码或者是挂掉之后就释放锁。



## 重要概念

### 数据模型

ZooKeeper 是一个层次化的**多叉树形结构**，

这里面的每一个节点都被称为： **ZNode**，每个节点上都可以存储数据，每个节点还可以拥有 N 个子节点。

ZooKeeper 主要是用来协调服务的，而不是用来存储业务数据的，所以不要放比较大的数据在 znode 上，每个节点的数据大小上限是 1M 。

节点可以分为四大类：

* PERSISTENT 持久化节点

* EPHEMERAL 临时节点 ：-e

* PERSISTENT_SEQUENTIAL 持久化顺序节点 ：-s

* EPHEMERAL_SEQUENTIAL 临时顺序节点 ：-es

![](.\zk\数据模型.png)

### ZNode

我们通常是将 znode 分为 4 大类：

- **持久（PERSISTENT）节点**：一旦创建就一直存在即使 ZooKeeper 集群宕机，直到将其删除。
- **临时（EPHEMERAL）节点**：临时节点的生命周期是与 客户端会话（session） 绑定的，会话消失则节点消失。并且，**临时节点只能做叶子节点** ，不能创建子节点。
- **持久顺序（PERSISTENT_SEQUENTIAL）节点**：除了具有持久（PERSISTENT）节点的特性之外， 子节点的名称还具有顺序性。比如 `/node1/app0000000001`、`/node1/app0000000002` 。
- **临时顺序（EPHEMERAL_SEQUENTIAL）节点**：除了具备临时（EPHEMERAL）节点的特性之外，子节点的名称还具有顺序性

每个 znode 由 2 部分组成:

- **stat**：状态信息，包含了一个数据节点的所有状态信息的字段，包括事务 ID（cZxid）、节点创建时间（ctime） 和子节点个数（numChildren） 等等。
- **data**：节点存放的数据的具体内容



## 集群

### 集群角色

| 角色     | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| Leader   | 为客户端提供读和写的服务，负责投票的发起和决议，更新系统状态。 |
| Follower | 为客户端提供读服务，如果是写服务则转发给 Leader。参与选举过程中的投票。 |
| Observer | 为客户端提供读服务，如果是写服务则转发给 Leader。不参与选举过程中的投票，也不参与“过半写成功”策略。在不影响写性能的情况下提升集群的读性能。此角色于 ZooKeeper3.3 系列新增的角色。 |

### Leader 选举过程

当 Leader 服务器出现网络中断、崩溃退出与重启等异常情况时，就会进入 Leader 选举过程，这个过程会选举产生新的 Leader 服务器。

1、**Leader election（选举）**：节点在一开始都处于选举阶段，只要有一个节点得到超半数节点的票数，它就可以当选准 leader。

2、**Discovery（发现）**：在这个阶段，followers 跟准 leader 进行通信，同步 followers 最近接收的事务提议。

3、**Synchronization（同步）**：同步阶段主要是利用 leader 前一阶段获得的最新提议历史，同步集群中所有的副本。同步完成之后准 leader 才会成为真正的 leader。

4、**Broadcast（广播）**：到了这个阶段，ZooKeeper 集群才能正式对外提供事务服务，并且 leader 可以进行消息广播。同时如果有新的节点加入，还需要对新节点进行同步。



### 脑裂

**何为集群脑裂？**

对于一个集群，通常多台机器会部署在不同机房，来提高这个集群的可用性。保证可用性的同时，会发生一种机房间网络线路故障，导致机房间网络不通，而集群被割裂成几个小集群。这时候子集群各自选主导致“脑裂”的情况。

举例说明：比如现在有一个由 6 台服务器所组成的一个集群，部署在了 2 个机房，每个机房 3 台。正常情况下只有 1 个 leader，但是当两个机房中间网络断开的时候，每个机房的 3 台服务器都会认为另一个机房的 3 台服务器下线，而选出自己的 leader 并对外提供服务。若没有过半机制，当网络恢复的时候会发现有 2 个 leader。仿佛是 1 个大脑（leader）分散成了 2 个大脑，这就发生了脑裂现象。脑裂期间 2 个大脑都可能对外提供了服务，这将会带来数据一致性等问题。

**过半机制是如何防止脑裂现象产生的？**

ZooKeeper 的过半机制导致不可能产生 2 个 leader，因为少于等于一半是不可能产生 leader 的，这就使得不论机房的机器如何分配都不可能发生脑裂。



## ZAB协议

[ZooKeeper 与 Zab 协议 · Analyze - beihai blog](https://wingsxdu.com/posts/database/zookeeper/)

ZAB（ZooKeeper Atomic Broadcast，原子广播） 协议是为分布式协调服务 ZooKeeper 专门设计的一种支持崩溃恢复的原子广播协议。 在 ZooKeeper 中，主要依赖 ZAB 协议来实现分布式数据一致性，基于该协议，ZooKeeper 实现了一种主备模式的系统架构来保持集群中各个副本之间的数据一致性。



ZAB 协议包括两种基本的模式，分别是

- **崩溃恢复**：当整个服务框架在启动过程中，或是当 Leader 服务器出现网络中断、崩溃退出与重启等异常情况时，ZAB 协议就会进入恢复模式并选举产生新的 Leader 服务器。当选举产生了新的 Leader 服务器，同时集群中已经有过半的机器与该 Leader 服务器完成了状态同步之后，ZAB 协议就会退出恢复模式。
- **消息广播**：**当集群中已经有过半的 Follower 服务器完成了和 Leader 服务器的状态同步，那么整个服务框架就可以进入消息广播模式了。** 当一台同样遵守 ZAB 协议的服务器启动后加入到集群中时，如果此时集群中已经存在一个 Leader 服务器在负责进行消息广播，那么新加入的服务器就会自觉地进入数据恢复模式：找到 Leader 所在的服务器，并与其进行数据同步，然后一起参与到消息广播流程中去。



## 常用命令

服务端常用命令：

```sh
启动 ZooKeeper 服务: ./zkServer.sh start
查看 ZooKeeper 服务状态: ./zkServer.sh status
停止 ZooKeeper 服务: ./zkServer.sh stop 
重启 ZooKeeper 服务: ./zkServer.sh restart 
```

客户端常用命令：

```sh
连接服务端：./zkCli.sh -server ip:port
断开连接：quit
设置节点值：set /节点path value
查看帮助：help
删除单个结点：delete /节点path
显示目录下结点：ls 目录
删除带子结点的结点：deleteall /节点path
创建节点：create /节点path value
获取节点值：get /节点path
创建临时节点：create -e /节点path value
创建顺序节点：create -s /节点path value
查看节点详细信息：ls -s /节点path
监听节点变化：get -w /path
查看节点状态：stat /path
查看节点ACL权限：getAcl /path
设置节点ACL权限：setAcl /path acl
查看节点子节点数量：count /path
查看节点子节点数量并监听变化：count -w /path
```



## JavaAPI

Curator 是 Apache ZooKeeper 的Java客户端库。http://curator.apache.org/

常见的ZooKeeper Java API ：

* 原生Java API

* ZkClient

* Curator

**建立连接**

```java
/*
        参数1：server地址和端口
        参数2：会话超时时间 ms
        参数3：连接超时世间
        参数4：重试策略
*/
// 1.
CuratorFramework client = CuratorFrameworkFactory.newClient("121.41.226.62:2181", new ExponentialBackoffRetry(3000, 10));
client.start();
// 2.
CuratorFramework client = CuratorFrameworkFactory.builder()
    .connectString("121.41.226.62:2181")
    .retryPolicy(new ExponentialBackoffRetry(3000, 10))
    .namespace("wang") // 命名空间 该客户端下所有操作在该命名空间下
    .build();
client.start();
```



**添加节点**:

```java
// 1. 基本创建
client.create().forPath("/app1");
// 2. 带有数据
client.create().forPath("/app2", "haha".getBytes());
// 3. 设置节点类型
client.create().withMode(CreateMode.EPHEMERAL).forPath("/app3");
// 4. 创建多级节点
client.create().creatingParentsIfNeeded().forPath("/app4/1");
```



**删除节点**

```java
// 1. 删除单个节点
client.delete().forPath("/app1");
// 2. 删除带子节点的节点
client.delete().deletingChildrenIfNeeded().forPath("/app4/1");
// 3. 必须删除成功
client.delete().guaranteed().forPath("/app2");
// 4. 回调
client.delete().guaranteed().inBackground(new BackgroundCallback() {
    @Override
    public void processResult(CuratorFramework client, CuratorEvent event) throws Exception {
        System.out.println("删除 /wang");
    }
}).forPath("/wang");
```



**修改节点**

```java
// 1. 修改数据
client.setData().forPath("/app2", "haha".getBytes());
// 2. 根据版本修改
Stat stat = new Stat();
client.getData().storingStatIn(stat).forPath("/app2");
client.setData().withVersion(stat1.getVersion()).forPath("/app2", "wang".getBytes());
```



**查询节点**

```java
// 1. 查询数据
byte[] bytes = client.getData().forPath("/app2");

// 2. 查询子节点
List<String> app4 = client.getChildren().forPath("/app4");

// 3. 查询节点状态
Stat stat = new Stat();
client.getData().storingStatIn(stat).forPath("/app2");
```



**Watch事件监听**

ZooKeeper 允许用户在指定节点上注册一些Watcher，并且在一些特定事件触发的时候，ZooKeeper 服务端会将事件通知到感兴趣的客户端上去，该机制是 ZooKeeper 实现分布式协调服务的重要特性。

ZooKeeper 中引入了Watcher机制来实现了发布/订阅功能能，能够让多个订阅者同时监听某一个对象，当一个对象自身状态变化时，会通知所有订阅者。

ZooKeeper 原生支持通过注册Watcher来进行事件监听，但是其使用并不是特别方便需要开发人员自己反复注册Watcher，比较繁琐。

Curator引入了 Cache 来实现对 ZooKeeper 服务端事件的监听。

ZooKeeper提供了三种Watcher：

* NodeCache : 只是监听某一个特定的节点

* PathChildrenCache : 监控一个ZNode的子节点. 

* TreeCache : 可以监控整个树上的所有节点，类似于PathChildrenCache和NodeCache的组合

```java
// 监听指定节点
// 1). 创建NodeCache对象
NodeCache nodeCache = new NodeCache(client, "/app1");
// 2). 注册监听
nodeCache.getListenable().addListener(new NodeCacheListener() {
    @Override
    public void nodeChanged() throws Exception {
        byte[] data = nodeCache.getCurrentData().getData();
        System.out.println("/app1 改变为"+new String(data));
    }
});
// 3). 开启 如果设置为true则监听开启时加载缓存数据
nodeCache.start();
```

```java
// 监听指定节点的子节点
// 1). 创建PathChildrenCache对象
PathChildrenCache pathChildrenCache = new PathChildrenCache(client, "/app1", true);
// 2). 绑定监听器
pathChildrenCache.getListenable().addListener(new PathChildrenCacheListener() {
    @Override
    public void childEvent(CuratorFramework client, PathChildrenCacheEvent event) throws Exception {
        // 只监听子节点数据更新
        PathChildrenCacheEvent.Type type = event.getType();
        if(type== PathChildrenCacheEvent.Type.CHILD_UPDATED) {
            System.out.println(event.getData().getData());
        }
    }
});
// 3). 开启
pathChildrenCache.start();
```

```java
TreeCache treeCache = new TreeCache(client, "/app1");
treeCache.getListenable().addListener(new TreeCacheListener() {
    @Override
    public void childEvent(CuratorFramework client, TreeCacheEvent event) throws Exception {
        if(event.getType()== TreeCacheEvent.Type.NODE_UPDATED) {
            System.out.println(event.getData().getData());
        }
    }
});
treeCache.start();
```



**分布式锁实现**

核心思想：当客户端要获取锁，则创建节点，使用完锁，则删除该节点。

1. 客户端获取锁时，在lock节点下创建**临时顺序节点**。

2. 然后获取lock下面的所有子节点，客户端获取到所有的子节点之后，如果发现自己创建的子节点序号最小，那么就认为该客户端获取到了锁。使用完锁后，将该节点删除。

3. 如果发现自己创建的节点并非lock所有子节点中最小的，说明自己还没有获取到锁，此时客户端需要找到比自己小的那个节点，同时对其注册事件监听器，监听删除事件。

4. 如果发现比自己小的那个节点被删除，则客户端的Watcher会收到相应通知，此时再次判断自己创建的节点是否是lock子节点中序号最小的，如果是则获取到了锁，如果不是则重复以上步骤继续获取到比自己小的一个节点并注册监听。

在Curator中有五种锁方案：

* InterProcessSemaphoreMutex：分布式排它锁（非可重入锁）

* InterProcessMutex：分布式可重入排它锁

* InterProcessReadWriteLock：分布式读写锁

* InterProcessMultiLock：将多个锁作为单个实体管理的容器

* InterProcessSemaphoreV2：共享信号量

```java
InterProcessMutex lock = new InterProcessMutex(client, "/app");
lock.acquire();
lock.release();
```


