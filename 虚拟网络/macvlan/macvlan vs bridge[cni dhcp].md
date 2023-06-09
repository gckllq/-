# macvlan作用初识-与bridge设备对比发现

参考文章

1. [Linux虚拟网络设备之bridge(桥)](https://segmentfault.com/a/1190000009491002)
    - 介绍了bridge与veth在创建及配置过程中的网络架构, 十分清晰
    - bridge必须要配置IP吗?
    - bridge常用场景: 虚拟机, docker
    - 将物理网卡添加到bridge(有图示)
2. [Macvlan 网络方案实践](https://cloud.tencent.com/developer/article/1495218)
3. [Linux 虚拟网卡技术：Macvlan](https://cloud.tencent.com/developer/article/1495440)
    - macvlan设备与内核协议栈的图示
    - macvlan与bridge设备的对比(不过并不清晰, 还不如本文)
    - macvlan的4种工作模式

macvlan最多的是与bridge设备做对比, 在理解bridge的基础上, 说一说两者的不同.

在做kuber的cni插件实验时尝试使用bridge+dhcp.

```json
{
    "cniVersion": "0.3.1",
    "name": "mycninet",
    "type": "bridge",
    "bridge": "mybr0",
    "isGateway": false,
    "ipam": {
        "type": "dhcp"
    }
}
```

网络拓扑基本如下

```
+-----------------------------------------------------------+
|  +-----------------------------------------------------+  |
|  |                 Newwork Protocol Stack              |  |
|  +------↑-----------------------↑-------------------↑--+  |
|         |                       |                   |     |
|.........|.......................|...................|.....|
|         |                       |                   |     |
|         |         +--------+    |    +--------+     |     |
|         |         |  pod1  |    |    |  pod2  |     |     |
|         |         +----↑---+    |    +----↑---+     |     |
|         |              └─────┐  |  ┌──────┘         |     |
| +-------↓-------+         +--↓--↓--↓--+          ***↓**** |
| |      eth0     |         |   mybr0   |          * dhcp * |
| |192.168.0.10/24|         +-----------+          ***↓**** |
| +-------↑-------+                                         |
+---------|-------------------------------------------------+
          ↓
```

在新建Pod时, kubelet会调用`dhcp`插件作为客户端发请求广播包(UDP类型), 由于各Pod中的`eth0`接口都连接到了`mybr0`, 所以广播包是通过`mybr0`转发出来的(`tcpdump`抓包可以看到). 但是广播包并没有到达`eth0`, `tcpdump`上没有dhcp的请求.

按照参考文章1中所说, **`eth0`与`mybr0`是相同的, ta们两个与内核协议栈都是双向通信**, 所以通过`mybr0`发出的广播包最终由内核接收, 并寻找能处理`udp/53`的进程, 不再转发给`eth0`, 所以无法再通过`eth0`广播到物理网络中.

为了能实现广播到物理网络, 需要使用`ip link set eth0 master mybr0`, 将物理网卡`eth0`连接到`mybr0`. 参考文章1中有相关章节, 数据流向也有详细的图示, 此时`mybr0`将取代物理网卡的作用, 可以将广播包发送出去了.

```
+-----------------------------------------------------------+
|  +-----------------------------------------------------+  |
|  |                 Newwork Protocol Stack              |  |
|  +------------------------------↑-------------------↑--+  |
|                                 |                   |     |
|.................................|...................|.....|
|                                 |                   |     |
|                   +--------+    |    +--------+     |     |
|                   |  pod1  |    |    |  pod2  |     |     |
|                   +----↑---+    |    +---↑----+     |     |
|                        └─────┐  |  ┌─────┘          |     |
|                         +----↓--↓--↓----+           |     |
|   +-----------+         |     mybr0     |        ***↓**** |
|   |   eth0    |<------->|192.168.0.10/24|        * dhcp * |
|   +-----↑-----+         +---------------+        ******** |
+---------|-------------------------------------------------+
          ↓
```

但这种方式有明显的缺陷, 就是`eth0`的IP会由`mybr0`接管, 看上去怪怪的. 另外IP切换会导致路由变化, 宿主机会断网, 需要手动添加.

------

但是如果使用`macvlan+dhcp`就没有这么麻烦(主要还是`macvlan`的bridge模式).

```json
{
    "cniVersion": "0.3.1",

	"name": "mynet",
	"type": "macvlan",
	"master": "ens160",
	"ipam": {
		"type": "dhcp"
	}
}
```

假设在宿主机创建一个`macvlan`接口, 就相当于拥有了一个把`eth0`连接到`mybr0`的组合, 能直接实现上面示例中想要的功能.

```
+-------------------------------------------+
|   +--------+        +--------+            |
|   |  pod1  |        |  pod2  |            |
|   +----↑---+        +---↑----+            |
|        └─────┐    ┌─────┘                 |
|        +-----↓----↓-----+                 |
|        |     mymac0     |       ********  |
|        |      eth0      |       * dhcp *  |
|        | 192.168.0.10/24|       ********  |
|        +--------↑-------+                 |
+-----------------|-------------------------+
                  ↓                         
```

> 上图借鉴了参考文章3的**Macvlan vs Bridge**一节的macvlan示意图.

`mymac0`接口应该算是`eth0`的克隆, ta们可以共享数据, 宿主机上Pod的数据可以通过`mymac0`接口直接与`eth0`上的物理网络通信.

但是这种模式也有缺陷, 宿主机与其上的Pod之间不能相互通信. 按照参考文章2中的图示可以看到, `ns1`与`ns2`之间可以直接通过macvlan组成的网络进行通信, 掠过了物理网卡`ens160`. 而在`ns1/ns2`与物理网络通信时, 物理网卡`ens160`又只充当了网线的作用.

> 这里的`ens160`是指参考文章2图示中的网卡名称...

可以说, `macvlan`实现了将物理网卡与网桥设备结合的功能, 既拥有IP地址, 可以作为数据的收发的起点终点, 也可以作为交换机设备, 拥有交换功能转发二层数据包.
