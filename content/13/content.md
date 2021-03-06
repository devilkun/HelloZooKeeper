好久没更新 ZK 的文章了，我想死你们啦。之前发布的 [HelloZooKeeper 系列文章](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA5MzYyNzQ0MQ==&action=getalbum&album_id=1709315979568037891#wechat_redirect)完结后，[项目](https://github.com/HelloGitHub-Team/HelloZooKeeper)收获了将近 600 个 star。这远远超过了我自己的预期，在这里感谢大家的支持～

后面会继续 ZooKeeper 的话题，通过单篇的形式就某个 ZK 的话题继续聊，今天我们先来看看 ZK 的节点类型。话不多说，我们进入今天的主题～

## 一、关于 ZK 的节点类型

大家如果刷过 ZK 相关面试题的话，就一定会刷到过 “ZK 有几种节点类型？”，大家通常背书的答案的话是：4 种！但其实 ZK （3.6.2）服务端支持 7 种节点类型，分别是：

- 持久
- 持久顺序
- 临时
- 临时顺序
- 容器
- 持久 TTL
- 持久顺序 TTL

这 7 种类型之前的文章中也有提到过，但是并没有展开讲。这次更新的单篇想要把这 7 种类型的节点，认认真真的讲一遍！Let's GO

<img src="./images/1.gif" style="zoom:67%;" />

## 二、简单介绍

### 2.1 持久、临时

**持久**不用我多说，是用的最多的一种类型，也是默认的节点类型，**临时**节点相较于**持久**节点来说，就是它会随着客户端会话结束而被删除，通常可以用在一些特定的场景，例如分布式锁释放，健康检查等。

### 2.2 持久顺序、临时顺序

这两种我放在一起介绍，因为他们相对于上面两种的特性就是 ZK 会自动在这两种节点之后增加一个数字的后缀，而路径 + 数字后缀是能保证唯一的，这数字后缀的应用场景可以实现诸如分布式队列，分布式公平锁等。

### 2.3 容器

**容器**节点是 3.5 以后新增的节点类型，只要在调用 `create` 方法时，指定 `CreateMode` 为 `CONTAINER` 即可创建**容器**的节点类型，**容器**节点的表现形式和持久节点是一样的，但是区别是 ZK 服务端启动后，会有一个单独的线程去扫描，所有的容器节点，当发现**容器**节点的子节点数量为 0 时，会自动删除该节点，除此之外和**持久**节点没有区别，官方注释给出的使用场景是 `Container nodes are special purpose nodes useful for recipes such as leader, lock, etc.` 说可以用在 leader 或者锁的场景中。

### 2.4 持久 TTL、持久顺序 TTL

关于**持久**和**顺序**这两个关键字，不用我再解释了，这两种类型的节点重点是后面的 **TTL**，**TTL** 是 `time to live` 的缩写，指带有存活时间，简单来说就是当该节点下面没有子节点的话，超过了 **TTL** 指定时间后就会被自动删除，特性跟上面的**容器**节点很像，只是**容器**节点没有超时时间而已，但是 **TTL** 启用是需要额外的配置（这个之前也有提过）配置是 `zookeeper.extendedTypesEnabled` 需要配置成 `true`，否则的话创建 **TTL** 时会收到 `Unimplemented` 的报错

## 三、原理介绍

单纯的**持久**和**临时**节点我就不介绍了，之前的系列文章有讲

### 3.1 顺序关键字

客户端创建一个**顺序**节点的时候，服务端得知当前节点是**顺序**节点的时候会自动给路径加上后缀，后缀就是父节点的 cversion，代表创建子节点的个数

```java
if (createMode.isSequential()) {
  	path = path + String.format(Locale.ENGLISH, "%010d", parentCVersion);
}
```

就是这么简单~

### 3.2 容器、TTL 关键字

这两种其实可以放在一起讲，服务端在启动的时候会额外启动一个定时任务线程，会定期的扫描所有的**容器**和 **TTL** 的节点，逐一判断子节点的数量以及一些相关配置，来决定是否删除，首先整个逻辑是在 `ContainerManager` 中，定时任务是由 `TimeTask` 实现的，相关的配置有

| 配置项                                 | 默认值        | 说明                                                         |
| -------------------------------------- | ------------- | ------------------------------------------------------------ |
| znode.container.checkIntervalMs        | 60000（毫秒） | 定时任务检查的间隔                                           |
| znode.container.maxPerMinute           | 10000         | 和上面的参数联合成为最小的检查间隔，每个节点间隔必须差 （60000 / 10000）毫秒（默认 6 毫秒）以上 |
| znode.container.maxNeverUsedIntervalMs | 0             | 如果配置不为 0 的话，当**容器**和 **TTL** 节点最后一次更新的时间和当前时间戳的差超过这个值的话，也会被删除 |

## 四、小结

- **持久**关键字：客户端不主动删除的话，节点数据会一直存在
- **临时**关键字：客户端连接断开后，节点数据会被一起删除
- **顺序**关键字：服务端会自动为该节点加数字后缀
- **容器**：服务端会定期扫描这些节点，当该节点下面没有子节点时（或其他条件时）服务端会自动删除节点
- **TTL**：需要额外配置才能启用，基本和**容器**相同，当超过 **TTL** 时间节点下面都没有再创建子节点时会被删除，但是当创建子节点会重置该超时时间

---

好久没更了，这次先挑一个简单的话题。我下面列了一些知识点，大家可以投票选出你想学的知识点：

- 只读服务端
- sync 命令
- ObserverMaster
- LocalSession

ZKr～


<img src="./images/2.gif" style="zoom:67%;" />