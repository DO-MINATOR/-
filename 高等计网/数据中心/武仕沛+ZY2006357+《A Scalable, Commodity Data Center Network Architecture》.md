### 摘要

传统的数据中心网络架构采用树形拓扑，即使是采用高端的网络设备，顶层的设备都难以支撑网络边缘的数据总流量。面临着流量的非均匀化，软件协议设计的复杂性，使得成本大大提高。

作者展示了利用传统的网络交换设备并设计良好的架构可以换来高性能的服务，并且完全兼容现有网络协议和硬件接口。

### 传统认知

- 数据中心有成千上万的计算机构成，且带宽需求量巨大。
- 网络架构由路由和交换机组成树形结构，越来越多的高端设备的加入使得网络层次结构越发的明晰。
- 即使采用最高端的网络设备也只能支撑整个网络50%的带宽
- 流量大小的不一致性导致传统的算法很难适配数据中心，或者说需要设计很复杂的算法

### 新的数据中心应当预备的特点

- 大规模使用传统的网络设备就可以支撑大量的网络流量
- 采取新型架构可以以较低的成本换取高性能
- 完全向后兼容，不必更改现有网络设备和协议

### 关键字

网络设计架构、等值路由

### 问题分析

商用集群的出现使得任何一个组织都可以轻松实现大量的计算和PB级的存储。关键瓶颈在于节点之间的通讯，web查询、云计算、并行计算。

解决方案，利用新型互联硬件和软件，如infiniBand，Myrinet，扩展性较好，但实现成本高，且无法兼容现有网络协议。二是采用传统网络设别和协议，但随着网络节点数量的增加，管理成本、利用率、可扩展性效果都变得极差。大部分都采用第二种设计模式，并升级设备性能，但网络整体性能仍受限于中心节点的处理能力。

### 设计目标

- 可伸缩性，且任意加入主机都能够以本地连接处的全部带宽与其它主机进行通信。
- 廉价性，能够利用传统设备集群实现大规模部署，
- 兼容性：新的架构能够向后兼容网络接口和软件协议。

作者展望了利用传统网络，结合胖树架构，可以轻松实现大规模网络互联，即使出现更高端的网络设备，他们设计的架构也具有同等性能，且更加廉价。

### 背景

#### 拓扑结构

![image-20201117095629548](https://imagebag.oss-cn-chengdu.aliyuncs.com/img/image-20201117095629548.png)

三层网络架构包括核心层、汇聚层、边缘层，相比二层架构，可以提供更多的主机互联。

- 边缘层：汇聚边缘流量，向上传输。
- 汇聚层：同上
- 

#### 超额订阅

1：1代表所有主机都能够以本链路的全部带宽与其他主机进行通信。5：1，代表只有20%的主机可以实现。

#### 等价路由

ECMP支持任意主机间的通信可以采用不同的路径进行传输，采用ECMP可以使得有更多的节点实现全速率通信。然而，ECMP基于静态负载划分， 没有动态的支持，使得超额订阅增加，此外，ECMP提供的等价路由数较少，无法实现数据中心网络所需要的数量，随着路由数的增多，路由表项的个数也将成倍的增加。

#### 花费

不光考虑部署架构时带来的花费，还应当考虑采用不同超额订阅比时，网络所需要的额外支撑。可见，当超额订阅比为1：1时，网络总体成本最高。采用3：1时效益成本比最高，但意味着平均每个主机只能利用1/3的链路带宽。

![image-20201117101955345](https://imagebag.oss-cn-chengdu.aliyuncs.com/img/image-20201117101955345.png)

另外，使用传统的路由算法无法充分的利用胖树架构的特性，导致单位传输成本上升。

基于胖树架构，需要考虑传输瓶颈，布线复杂性，通过打包和放置技术改善这种开销

### 架构设计

包含：路由表结构的设计、ip地址的划分。两级路由查找，路由算法的实现，同时提出了优化等价路由的流分类和流调度技术。实现目的是达到多路径等价路由的充分利用，以及超额订阅比，

难点：如何实现路由算法能够将网络流量均匀的分配到不同路径上，传统的OSPF基于跳数选择“最短路径”，然而基于K-ary的胖树结构本来拥有k/2个最短路径，但却只选择了一个，网络流被集中到一个端口上，无法利用等价路由和改善超额订阅。而OSPF-ECMP改进版则会导致路由表项数量的增加。最终提出基于两级路由查找的算法来充分利用网络带宽。

#### 地址分配

汇聚层中第二个byte代表pod数，第三个byte代表哪一个交换机，边缘层，根据所连接的交换机，其第四个byte依次+1。

![image-20201117105604200](https://imagebag.oss-cn-chengdu.aliyuncs.com/img/image-20201117105604200.png)

#### 两级路由查找

根据改进的路由表结构和地址划分，使得路由转发的路径更均匀。两级路由表的设计需要在每一个路由表项中插入一个额外的指针指向更小层次的路由表项。

使用CAM内容寻址内存，进行并行搜索，查找引擎通过三元凸轮TCAM，但CAM存储密度极低，且能耗较高。不过该体系结构中，内容表可以在规模相对适中的TCAM中实现。路由表包含两层结构，一层是后缀表，决定北向路由，一层是前缀表，决定南向路由。路由表项的建立可以采用动态、静态结合，我们假设一个中心实体完全了解集群互连拓扑。这个中央路由控制负责静态生成所有路由表，并在网络设置阶段将这些表加载到交换机中。动态路由协议也将负责检测单个交换机的故障并执行路径故障转移。

#### 路由算法

胖树结构中边缘层和汇聚层都含有指向该机架中的子网前缀列表，所以一个子网中的数据包如果被发送到另一个机架中但位于不同子网的主机上，则该机架中所有上层的交换机都将有一个指向目标子网的前缀表项。对于所有其他往上传输的pod的间通信，pod交换机有一个默认的/0前缀，带有一个匹配主机id的辅助表(目标IP地址的最低有效字节)。这将导致流量均匀地向上分布到核心交换机。这还将导致到同一主机的后续数据包遵循相同的路径，从而避免数据包重新排序。

- 汇聚层转发：在每个pod交换机中，我们为包含在同一pod中的子网分配相同的终止前缀。对于pod间通信，我们添加一个/0前缀和一个匹配主机id的辅助表。对于较低层的pod交换机，省略了/24子网前缀步骤，因为子网本身的流量在内部被交换，而pod间的流量应该在较高的交换机中平均分配。
- 核心层转发：由于每个核心交换机都连接到每个pod交换机，因此核心交换机只包含指向目的地pod的/16前缀表项。

#### 流分类

但是基于上述方案设计的转发存在一个问题，即是如果两个流的源ip和目的ip相同，则会将其分配到通过一端口进行输出，这会导致端口竞争，从而无法发挥全部的胖树优势。基于此，提出了一些商业路由器中可用的方案。这种方案存在潜在的好处，但会带来额外的包开销。不过这种技术一旦失效或者带来的收益不及它所消耗的资源时，系统可终止该方案的实施。与此同时，保证同一个流的数据总是经过相同的交换机进行转发，从而避免后续的包进行重排序，

#### 流调度

一些研究表明，互联网流量的分布具有少数大的长生命流和许多小的短生命流的特征。我们认为，大流量在确定网络可实现的对均分带宽中起着最重要的作用，因此值得特殊处理。在这种流管理的替代方法中，我们对大型流进行调度，以尽量减少彼此之间的重叠。中心调度器利用网络中所有活动的全局感知做出相应调度。

边缘开关在本地分配一个新的流到初始占用最少的端口。同时，边缘开关还会检测大小超过预定义阈值的任何传出的流，中心调度器发送通知，以表示边缘开关请求将该流放置在非竞争路径中。注意，边缘交换机只能请求分配路径，但无法私自决断使用哪条路径。

中心调度器会跟踪所有的请求，并尽可能为其分非冲突路径，同时维护一个布尔变量，以指示该路径是一条大流量传输路径。

维护流调度需要两个主要的数据结构，一个是所有连接链路的二进制数组，一共4 ∗ k ∗ (k/2)^2^ 条链路，一个是哈希表，包含以往已分配的流及其路径，对于新的流的线性搜索需要2 ∗ (k/2)^2^次的访问次数，使得调度器的时间复杂度为k^2^，空间复杂度为k^3^。 

#### 容错

胖树结构所具备的冗余链路特性使其天然的就具备容错的特性，该方案中，网路中的每一个交换机维护双向的链路状态，分别是上行链路状态和下行链路状态。

- 下层链路故障会影响到三类流量
  1. 如果是pod内部的流量，则链路故障影响不大，通常做法是为其分配另一个pod内的交换机。
  2. 来自其他pod的流量，此时链路故障状态被广播到其他pod的交换机上，因此当有外部流量想进入时，会避免经过出现故障的交换机。
  3. pod内流量，并且想要传输到其他pod内部，此时分配一个新的pod内部交换机，使得流量可以正常转发出去。
- 上层链路故障影响到两类流量
  1. 传输出去的流量，此时选择pod内部另一个未收影响的交换机。
  2. 传入到该pod内部的流量，同样将链路故障状态进行广播，其他pod的流量就会避免使用该故障的交换机。

如果交换机链路状态恢复正常，将取消上述的操作，

### 能耗问题

数据中心的能耗通常是上千瓦，因此需要专用的散热设备。本节中将分析来自数据表中报告的相关数据，来分析数据中心的需求和散热。由于采用胖树架构的数据中心拥有更好的传输效率，因此可以不必使用高端的10Gbps的交换机，自然功耗和热能产生就下降了。实际的测算得出，功耗和散热都优于现有的数据中心设计，功耗降低56.6%，散热降低56.5%。

### 测验

测量目标包括交换机、流分类、流调度器，同时基于3.6：1的超额订阅与现有的数据中心进行比较。实现是层次化的胖树结构，有四台机器，每台机器下连四个主机，四台机器运行四个pod交换机，每个交换机有一个上行链路。四个pod开关连接到一个在专用机器上运行的4端口核心开关。为了在从pod交换机到核心交换机的上行链路上实现3.6:1的超额订阅，这些链路的带宽限制为106.67Mbit/s，而所有其他链路限制为96Mbit/s。

测试策略是通过随机生成通信对，和指定传输对和数量，并测量该种情况下所对应的胖树结构、二级路由表、流分类、流调度所产生的贡献百分比，一共提出了如下5中测试对象：

1. 随机通信对，pod内任意主机发送数据到其他pod内的任意主机
2. 固定通信对，pod内x编号的主机发送数据到其他pod内编号为(x+i)%16的主机
3. 概率通信 (SubnetP, PodP)，即以podP的概率向SubnetP内的主机发送数据，以1-podP的概率向其他子网中发送数据
4. 多个pod发送数据到同一个子网中的不同主机，该情况下，最坏的超额订阅比为(k-1)：1
5. 同一子网不同主机发送数据到其他pod的主机，该情况下静态路由使得网络流竞争到同一个端口上，这是，流调度可以最大限度的改善着这种情况

![image-20201117170724207](https://imagebag.oss-cn-chengdu.aliyuncs.com/img/image-20201117170724207.png)

上图显示了实验测试结果，可见当同一子网主机向外发送数据产生链路竞争时，流调度总能很好的完成工作，而二级路由表提供的是多条等价路由，因此在相同源ip和目的ip的情况下，起到了链路负载的作用，流分类更多提供的是一种避免重排序的作用，所以其提供的收益是始终存在的。在多个pod向同一子网发送数据的情况下，最终达到了28%的链路利用率，这也验证了3.6：1的超额订阅情况。

### 打包技术

由于并没有采用高端设备，如10Gbps的交换机，因此为实现同样的链路传输效率，需要多部署10倍的线路，为了不影响胖树结构的可扩展性和降低管理难度，因此提出了线路打包技术。