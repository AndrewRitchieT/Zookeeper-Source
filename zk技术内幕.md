





![1567648000148](E:\github\zookeeper\images\1567648000148.png)



zk 数据存储结构与标准的Unix 文件系统非常相似，都是在根节点下挂很多子节点。zk 中没有引入传统文件系统中目录与文件的概念，而是使用了称为 znode 的数据节点概念。 znode 是 zk 中数据的最小单元，每个 znode 上都可以保存数据，同时还可以挂载子节点，形成一个树形化命名空间。





## 节点类型



持久节点 

持久顺序节点 

临时节点：临时节点的生命周期与客户端的会话绑定在一起。临时节点不能有子节点， 即临时节点只能是叶子节点

临时顺序节点



## 节点状态





cZxid：Created Zxid，表示当前 znode 被创建时的事务 ID 

ctime：Created Time，表示当前 znode 被创建的时间 

mZxid：Modified Zxid，表示当前 znode 最后一次被修改时的事务 ID

mtime：Modified Time，表示当前 znode 最后一次被修改时的时间 

pZxid：表示当前 znode 的子节点列表最后一次被修改时的事务 ID。

注意，只能是其子 节点列表变更了才会引起 pZxid 的变更，子节点内容的修改不会影响 pZxid。

cversion：Children Version，表示子节点的版本号。该版本号用于充当乐观锁。 

dataVersion：表示当前 znode 数据的版本号。该版本号用于充当乐观锁。 

aclVersion：表示当前 znode 的权限 ACL 的版本号。该版本号用于充当乐观锁。

ephemeralOwner：若当前 znode 是持久节点，则其值为 0；若当前 znode 为临时节点，则其值为创 建该节点的会话的 SessionID。当会话消失后，会根据 SessionID 来查找与该会话相关的 临时节点进行删除。

dataLength：当前 znode 中存放的数据的长度。

numChildren：当前 znode 所包含的子节点的个数。





## 会话

会话是 zk 中最重要的概念之一，客户端与服务端之间的任何交互操作都与会话相关。 ZooKeeper 客户端启动时，首先会与 zk 服务器建立一个 TCP 长连接。连接一旦建立，客户端会话的生命周期也就开始了。



常见的会话状态有三种： 

CONNECTING：连接中。客户端要创建连接，其首先会在客户端创建一个 zk 对象，代表 服务器。其会采用轮询的方式对服务器列表逐个尝试连接，直到连接成功。不过，为了 对 Server 进行负载均衡，其会首先对服务器列表进行打散操作，然后再轮询。

CONNECTED：已经连接

CLOSED：连接已经关闭



## 会话连接事件

客户端与服务端的长连接失效后，客户端将进行重连。在重连过程中客户端会产生三种 会话连接事件：

CONNECTION_LOSS：连接丢失。因为网络抖动等原因导致连接中断，在客户端会引发连 接丢失事件。该事件会触发客户端逐个尝试重新连接服务器，直到连接成功，或超时。

SESSION_MOVED：会话转移。当连接丢失后，在 SessionTimeout 内重连成功，则 SessionId 是不变的。若两次连接上的 Server 不是同一个，则会引发会话转移事件。该事件会引
发客户端更新本地 zk 对象中的相关信息。

SESSION_EXPIRED：会话失效。若客户端在 SessionTimeout 内没有连接成功，则服务器 会将该会话进行清除，并向Client发送通知。但在Client收到通过之前，又连接上了Server， 此时的这个会话是失效的，会引发会话失效事件。该事件会触发客户端重新生成新的SesssionId 重新连接 Server。



## 会话连接超时管理--分桶策略

无论是会话状态还是会话连接事件，都与会话超时有着紧密的联系。zk 中对于会 话的超时管理，采用了一种特殊的方式——分桶策略。



## 基本概念

分桶策略是指，将超时时间相近的会话放到同一个桶中来进行管理，以减少管理的复杂 度。在检查超时时，只需要检查桶中剩下的会话即可，因为没有超时的会话已经被移出了桶，而桶中存在的会话就是超时的会话。



## 分桶依据

分桶的计算依据为：

ExpirationTime= CurrentTime + SessionTimeout 

BucketTime = (ExpirationTime/ExpirationInterval + 1) * ExpirationInterval

从以上公式可知，一个桶的大小为 ExpirationInterval 时间。只要 ExpirationTime落入到 同一个桶中，系统就会对其中的会话超时进行统一管理。

## ACL

ACL 全称为 Access Control List（访问控制列表），是一种细粒度的权限管理策略，可以针 对任意用户与组进行细粒度的权限控制。zk 利用 ACL 控制 znode 节点的访问权限，如节点数据读写、节点创建、节点删除、读取子节点列表、设置节点权限等。

UGO，粗粒度权限管理。



## zk 的 ACL 维度

Unix/Linux 系统的 ACL 分为两个维度：组与权限，且目录的子目录或文件能够继承父目 录的 ACL 的。而 Zookeeper 的 ACL 分为三个维度：授权策略 scheme、授权对象 id、用户权限 permission，子 znode 不会继承父 znode 的权限。

## 授权策略 scheme

授权策略用于确定权限验证过程中使用的检验策略（简单地说就是，通过什么来验证权 限，即一个用户要访问某个 znode，如何验证其身份），在 zk 中最常用的有四种策略。 

IP：根据 IP 进行验证。

digest：根据用户名与密码进行验证。 

world：对所有用户不做任何验证。

super：超级用户对任意节点具有任意操作权限。



## 授权对象 id

授权对象指的是权限赋予的用户。不同的授权策略具有不同类型的授权对象。下面是各 个授权模式对应的授权对象 id。



ip：将权限授予指定的 ip 

 digest：将权限授予具有指定用户名与密码的用户。

world：将权限授予一个用户 anyone

Super：将权限授予具有指定用户名与密码的用户。





权限 Permission

权限指的是通过验证的用户可以对 znode 执行的操作。共有五种权限，不过 zk 支持自 定义权限。

c：Create，允许授权对象在当前节点下创建子节点 

d：Delete，允许授权对象删除当前节点

r：Read，允许授权对象读取当前节点的数据内容及子节点列表

w：Write，允许授权对象修改当前节点数据内容及子节点列表（可以为当前节点增/删 除子节点）

a：Acl，允许授权对象对当前节点进行 ACL 设置





## Watcher 机制

zk 通过Watcher 机制实现了发布/订阅模式。

![1567650205876](d:\github\zookeeper\images\1567650205876.png)

## watcher 事件

对于同一个事件类型，在不同的通知状态中代表的含义是不同的。

![1567650241524](D:\github\zookeeper\images\1567650241524.png)

![1567650279449](C:\Users\roseonly\AppData\Roaming\Typora\typora-user-images\1567650279449.png)





## watcher 特性

zk 的watcher 机制具有以下几个特性。 

一次性：watcher 机制不适合监听变化非常频繁的场景。 

串行性：只有当当前的watcher 回调执行完毕了，才会向 server 注册新的watcher（注 意，是对同一个节点相同事件类型的监听）。

轻量级：Client 向 Server 发送的watcher 不是一个完整的，而是简易版的。另外，回调逻辑不是 Server 端的，而是 Client 的



