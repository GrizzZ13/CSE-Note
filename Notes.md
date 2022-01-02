$$
CSE\ Notes\ for\ the\ second\ half\ semester \\
Wang\ Haotian
$$

[TOC]



## 11.09 DB

kv-store不足以解决复杂应用场景：



consistency problem：

1. Quorum（写4读2）

2. 先做所有的备份，全部完成再说写完了



早期的DB是实现了ACID TX的kv-store

解耦意味着生产力提高，因此程序员希望把**逻辑层和物理层**解耦



DBMS中：

- data model
- query language

软件和硬件的解耦



OS也是这样诞生的，在OS诞生之前，软件和逻辑是紧密相连的



DB可以对比kv-store优点：

- DB的SQL可以便捷地提供检查，检查constraint，检查外键等

- 如果用kv-store查找，需要遍历全部
- kv-store如果想要加速，就需要自行维护一个索引
- kv-store编程复杂度与SQL语句相比很高
- 用sql，join语句复杂性远远比kv-store的低
- SQL语句支持嵌套



数据模型：

- document比关系模型更加灵活
- OneToMany问题得到了很好的解决，但是ManyToOne的问题很棘手
- 当要修改所有document中的某一个value，很复杂。



## 11.11 DBMS Storage

relational model是逻辑模型，最终还是要存储在磁盘上

数据存储在什么位置：

层次越高，越快，但是persistency不能保证

因为要确保power failure之后还能保证persistency，需要放在非易失性存储设备上边



tuple怎么保存：

- tuple和schema都要保存下来
- schema是metadata
- 上层什么语言访问？数据类型？多大？
- 字符集也很重要，linux和windows的字符集会有差异



提到字符集，想想RPC：

- 数据格式不太一样，marshall
- 读磁盘时也要考虑相似的问题（little-endian  big-endian）



### 物理存储

说完字符集、编码等问题，想想tuple怎么存？

- 关系型数据库怎么保存tuple？可以在每个tuple前边加一个元数据，到底有多长
- table有两部分，metadata和data
- metadata中记录attr的长度等
- tuple的header是一个bitmap，记录了null还是not null的信息
- 再来想想metadata，即schema
- schema在DB中也被认为是一张table，数据库内部的表存了每个schema的信息
- 存schema的schema，每个DB只有一份



说完tuple，想想实际磁盘存储：

- log structure kv-store
- 文件系统



先来看kv-store式的存储：

- 对k和v没有特殊要求，v甚至可以式json
- MySQL+RocksDB=MyRocks
- primary key是key，整个tuple就是value
- 结构可以实file sysyem上层有kv-store，kv-store上层有关系模型relational model



LSMT：

- 优点，优化顺序写操作，充分利用磁盘顺序写的优点
- 内部排序、层间顺序，搜索时减少搜索次数

- 缺点，读放大，写放大
- range query麻烦



因此用B Tree：

- B Tree下边的每一个Node都是fixed size
- B Tree是排序的
- 所有value都在叶子节点上的
- range query很简单，内部顺序访问
- 大量读写的支撑很好
- DBMS的page就是B Tree的node，是DBMS的最小单位



primary key主键索引被DBMS自动创建

把tuple放在page中，便于range query



### DBMS Page

page的基础概念

- fixed size
- 一个page包括多个tuple以及metadata
- 最基本的存储数据的抽象
- 每一个page有唯一的identifier
- DBMS把page id映射到物理id



给DBMS一个4T的硬盘，它可以怎么存储文件呢？

1. 创建一个巨大的4T的文件，在其中分块
2. 或者创建很多小文件，比如说创建100个40G的文件，每个文件放很多个Block，当找某一个block时候，可以直接到一个特定的文件去取page



Page Size如何选择：

- 常见的有 512byte-16K



page size变大：

- 好处是B Tree会变矮，找的次数会变少，
- 坏处是wasted space可能变多
- all or nothing无法用硬件保证，需要额外的atomicity



page 的操作

创建、读、写、删、遍历



### How to manage page



DBMS使用heap file

- 如果直接用文件系统的抽象，每个page对应一个block，元数据太多

- heap file是保存page的文件，每一个heap file可以放很多page
- heap file是磁盘的抽象，DB完全看不到磁盘



如何找到 free page：

1. linked list

- 有两个链表，一个是free page list，另一个是data page list
- data不可能把4K撑满，data不连续

2. page directory

- 用bitmap，哪些free，哪些not free
- 必须consistency，解决方法是log



### How to store tuple

page头部放有number of tuples

tuple按行存储在page中



问题：删除tuple？变长attr？

- 顺序不重要，因此直接删除是没问题的
- 变长attr可以直接分最大的，但是为了提高page 利用率，用slotted pages
- 直觉上，前边放绝对offset，tuple从后边开始放，删除时做compact
- 如果删除了头，一下子改了好多offset，因此可以记录相对offset
- 这些时间现在不是瓶颈，因为从磁盘加载到内存时间更长



tuple比page大？不行！有限制，不能超过page

如果真的特别大，可以用blob存储



join操作locality差？Pre-join优化！



磁盘太慢了，要在内存做cache

为什么不希望OS管理这个move呢？

通过mmap映射到VMA，访问到某一块会触发page fault。当物理内存不够，会发生swap，但是DBMS希望自己管理swap要换走的page，OS可能会把index给踢掉



## 11.16 Buffer Pool

### 数据库分层、物理存储

数据库的分层结构

1. 基于file system，一个大文件或者许多个小文件，但是对上层提供的抽象仍然是一个整体
2. 数据库而言最底层heap file
3. heap file被划分成许多个page，每一个heap file头部存有bitmap记录page的占有情况
4. 在heap file的上边，提供抽象，由page number拿到一个4k的page（page size可配置）



数据是怎么存放的：

- page中存放data
- 紧接着要创建索引，B+ Tree有许多节点，每个节点对应一个page
- 对于存放数据的page，采用头部存放slot，tuple从尾部堆叠的方式，slot和tuple meet in the middle，存放数据，4k内部进一步拆分
- 再上层就是一些逻辑概念比如schema，table



找到free page：

- linked list
- directory（bitmap）



### Buffer Pool解决太慢的问题

需要在内存和硬盘之间来回挪动数据，但是DBMS不想让OS管理这样的Move，需要自己实现**Buffer Pool**

为什么不用mmap而是自己实现Buffer Pool？



Buffer Pool：

- 在DRAM里边
- 使用O_DIRECT的机制
- 当交给OS管理时，读磁盘时，磁盘的数据被缓存在内存里边，内存中有一个磁盘的缓存page cache，这是由kernel维护的。当用户态访问磁盘，先读page cache，如果hit；从page cache取，**如果miss，则要把数据load到page cache中，下一次用户态再从page cache读取数据**
- DBMS不用page cache，自己维护一个buffer pool，这个buffer pool如果命中，从buffer pool读，如果miss，再取磁盘（也要先加载到内存哦）
- DBMS可以在open文件时加入O_DIRECT选项，bypass掉kernel的page cache
- buffer pool仍然采用大数组方式管理，内存中每一个元素叫做frame
- 访问一个page，会先copy到frame中，再访问frame



如何做buffer pool到frame之间的映射？

- page table映射
- 访问page先查page table，hit就返回frame，如果miss就更新page table和buffer pool
- page table是由软件管理的，可以使用Hash，可以比OS要复杂，因为OS的是由MMU读，不能太复杂
- page table如果crash，无所谓的
- page directory和pagetable不一样
- page directory要持久化，记录的是page的占用情况（free or not）
- page table不用持久化，丢了就丢了
- 如果写了一个page到frame，应该write back到磁盘上边，但是每次写串会很慢，因此会记录dirty bit，空闲时批量操作
- frame不够用了怎么搞，evit一个，如果dirty page就写回



如果buffer pool和query同时访问一个frame怎么办？

- 使用**latch**
- lock保护的是用户看得见的data，叫做logical content
- latch保护数据库看得见的internal data structure，用户看不见，比如B+ Tree Index，buffer pool
- 对于数据来说，需要保证TX，采用2PL
- 如果不区分latch和lock而统一采用2PL，任何操作都会再internal data structure上边产conflict，而且造成性能的损失，耗费太多时间
- 但是如果区分开lock和latch，当我load完数据之后，就可以把latch放掉，这样子就不会破坏掉2PL的语义，假设A放掉latch之后B拿到了latch并且立刻evict A，造成的后果也仅仅是把A从内存中evict，下次访问重新load就好，用户不会体会到任何语义上的差异
- latch不需要persist



### Buffer Pool优化

#### 1. multiple buffer pool

1. 不同page有不同access patterns，index page和data page的访问模式是不同的
2. 单个buffer pool可能增大竞争，因为全局只有一个page table



两种模式

- 每个DB有一个buffer pool
- 每个page type有一个buffer pool



当有多个buffer pool，如何知道去哪一个buffer pool找到cached page？

- 这是一个partition的问题
- 两种经典的划分方式：hash partition & range partition
- hash数据划分更加均匀
- range的locality更好，但是不知道每一个page里边是什么东西，因此需要上层做协调



#### 2. page prefetch

OS也可以prefetch，但是DBMS不用，为什么？

- OS是读什么fetch什么，最多是加载后边连着的几页
- OS看的太简单，看的是文件系统，不知道DB的page size，更看不到上层的语义
- OS可能产生错误的语义，当我们访问B+ Tree结构时，树的结构可能和磁盘上的顺序是不同的，导致取到非预想的index、page



#### 3. Scan sharing

看这两个query

- select * from Items
- select AVG(Price) from Items



优化：

1. DBMS会把Q1和Q2优化，防止Q1刚刚访问完的东西在Q1继续访问时被evict，导致Q2必然miss
2. DBMS会让Q1先停住等等Q2，等Q2到了之后一起做query，这样增加性能



### Buffer Replacement Policies

goal：

1. High Accuracy
2. Quick Speed
3. Low metadata overhead



#### 基本的policy：LRU

LRU的实现：

- 首先加timestamp是很难的，对timestamp排序又需要额外的数据结构，overhead很高
- 解决方案是clock算法，容易实现并且性能近似LRU



LRU & Clock are not perfect：

- 针对locality很低的应用场景（听音乐、看电影……）

- Problem of **sequential flooding**
- select * from Items
- 会把buffer pool全部踢掉
- 这种情况下需要MRU（most recently used）



解决sequential flooding的三种方案：

1. 多个buffer pool，针对select *的用一个小的buffer pool并且用MRU
2. 使用别的策略LRU-k（在k范围内做LRU）
3. 不用buffer pool，使用light scan，类似于网络中的sliding window滑动窗口



#### 针对query execution来优化policy

假设把一家商店的商品全部加到另一家，因为是一直往下加的，可以让DBMS一直保存这个路径



如果是where ID=xxx，点查询，应该尽可能放上层的index，因为ID是随机的，越放上边，命中的概率就越高



当发生evict时候，如果page没有被改过，可以直接t掉，但是如果dirty page就需要write back

可以采用background writing，在系统空闲的时候把dirty frame写到page，把dirty frame清零



## 11.23 DBMS Query execution

Query plan

拿到一个SQL语句，首先变成一个tree（火山模型）

- Each node is and operator in relational algebra
- 数据自下向上流动
- root node的输出就是SQL的结果



### Processing model

#### Iterator Model

实现next接口

自上向下实现，每次调用next返回一个

缺点是函数调用的开销，而且如果用C++实现，虚函数访问，更加overhead



为什么还要Iterator Model呢？

传统数据库访问磁盘，访问磁盘慢的更多



#### Materialization Model

既然每次返回一个的next太慢，我们可以一次返回很多个

问题：当一次返回的太多了怎么办：加LIMIT

OLTP的场景适合materialization model，访问的tuple很少，牵扯到数据量很少

OLAP不适合，如果analysis一次，要大量扫描，很不合适



#### Vectorizaed/Batch Model

这种model在上述两种model之间取平衡

每次返回一组tuple，比如说每次next返回20个

这样，拿到的数据就不是一个tuple，而是一个tuple vector



- 适合OLAP比较合适
- 可以针对性地设置N的大小，在特定的硬件架构非常合适，一条指令多个操作
- 而且这个N的大小和用户行为是无关的，在用户不知道的情况下自动找一个合适的N



### operator的实现以及优化

问题：Index存的是什么？

heap file中的offset



挑一个select语句讲优化

1. IndexScan
   - 假设where语句中的predicate是存在index的
   - select * from Item where Item.Y > 2020
   - 找到index为2020，然后返回后边的tuple即可

2. 当predicates比较复杂：
   - select * from Item where Item.A > 100 and Item.B < 200
   - bitmap scan
   - 第一个bitmap找A>100
   - 第二个bitmap找B<200
   - 把两个bitmap做交集

3. 还可以把query语句缓存下来，下次有同样的搜索，直接返回



### SQL演变

OldSQL = Relational Mode + SQL + ACID

传统sql不能适应现在高速发展互联网的要求

早期：

- transaction
- write-mostly（银行、机票等）

现在的互联网：

- read-mostly读操作变多
- scalability
- availablity
- transactions
- large dataset



应对大规模应用

- 需要水平拓展，加一层middle ware
- 可以应对单个record的读和写
- 但是middle ware无法实现acid



NoSQL：Build for scratch

- 转用简单的data model，减少复杂性（比如kv-store，document，wide-column）
- 弱化transaction（互联网应用对tx要求低），局部tx可以保证，但是全局tx被牺牲掉了，牺牲consistency，得到scalability和availability
  - 担心的异常情况在100万read只会发生一次
  - 如果加上tx，导致真正起作用的useful work只占用4%
- 异步复制，可能导致primary和backup的inconsistency



因为NoSQL又支持了tx，不妨称作NewSQL：

- 支持SQL，而不是简单的kv-store
- ACID
- non-locking concurrency control
- 单个节点性能更高
- 多机，不是单机



如何实现NewSQL？

TiDB：

- 搭积木，使用很多现成的开源组件
- partition algorithms，分成很多region，每一个region的最底层使用kv-store，TiKV （TiDB's storage engine）
  - Raft - consensus
  - RocksDB - kv-store
  - MVCC + 2PC - transaction
  - Ext4 - File System
- TiKV在最外层加上RPC（用的gRPC），通过kv-API接口给外部提供服务
- 最外层发SQL，解析给多个TiDB server，TiDB通过kv-API操作TiKV



## 11.25 Intro to network & Link

复杂系统的MAHL：

- modularity
- abstraction
- hierarchy
- layer



### Layers in network

- Application layer（我们的课程中。application layer和transport layer合并成为end-to-end layer）
- transport layer：强调从一个点怎么找到另一个点
- Network layer：把link layer的一段一段连接起来，找到从一个点到另一个点的最短路径
- Link layer：强调点到点的传输，比如光纤、电缆……



ICMP是ping的协议，优先级很高，跑在kernel里边，会回复ping。是一个非常危险的操作。



IP协议是best effort，可靠性只有95%。因为并不是所有需求都要求100%的可靠。为了低时延，可以牺牲掉一部分的正确性。



packet有点像栈

数据切分成64~1500bytes的包发出去，包小了，包头overhead占比大；但是错了只用重发包。



### Transport layer

TCP：考虑识别不同的session，有seq和ack

1. client - seq=x
2. server - ack=x+1, seq=y
3. client - ack=y+1, seq=x+1,附带data过去
4. server - ack=x+2,seq=y+1

UDP：无所谓，可靠性不高，也不需要sequence number



### Network layer

需要关注网关、网桥和路由器，需要关注路由表



### Link layer

1. 物理上传输
2. 一条线复用
3. 编码
4. 找到错误，避免重传
5. 提供接口



所以没有全局时钟的情况下，我们需要 ready 和 ack 的协议。我们先把数据（要传输的 内容）放在这根线上，等我们把数据放好了，我们设置一根线叫做 ready（我们已经把数据 放置好了）。B 一看到 ready 设置成了 1，那么就知道 data 就可以读了，如果 ready 没有设置成 1，那么我们的数据可能就处于 all-or-nothing 的中间状态。 

ready为1，这样 B 就可以把数据读走，然后 ACK 设为 1，那么 A 就知道可传下一个数据了，它先把 Ready 设置成 0，B 就知道要传下一个了，此时就把 ACK 设置为 0。然后继续传下一组数据

通过 ready 和 ack，a 和 b 之间就可以去做交互，这就比较适合在两台计算机去做。

#### 数据并行传输

假设传输时间是 $\Delta t$，那么我们每传一个 bit，就需要 $2 \Delta t$ 时间，最快传输速率就是$1/2\Delta t$，为 了加快速度，我们可以把数据线加粗，比如从 1bit->32bit->64bit，这就是加粗总线，也就是 Parallel Transmission。

但是数据在并发传输的过程中，很难做到大家一样快，并且线和线之间会存在一些干扰 导致传输速度上不去

#### 数据线性传输

曼彻斯特编码+压控振荡器，变化率何以体现在数据传输过程中，但是利用率只有原来的一半了

曼彻斯特编码：

- 0 => 01
- 1 => 10

这样连续的数据也是会发生变化的

11变为1010，10变为1001，01变为0110，00变为0101



一根线怎么共享？

同步模式，建立connection，一部分时间片分给你。如果达到上限，没有时间片可用，就不能建立连接

异步模式：包交换网络，数据发包，但是可能排队很后边，不能保证低时延。好处是，不发包就不占用资源。至于包的开始和结束的判断，可以设置magic number

#### Error Handling

海明距离：一个number变成另一个number，需要翻转多少个bit

海明编码：

- 2bit 变成 3bit，编码空间是8种，但是只有四种是合法的
- 4bit 变成 7bit

**1 1 0 1** => *1 0* **1** *0* **1 0 1**，其中3，5，6，7是原来的bit

P1 = P7 xor P5 xor P3

P2 = P7 xor P6 xor P3

P4 = P7 xor P6 xor P5

这样在检查xor的三个等式之后，可以纠正一个bit的翻转错误

| Not Match        | Error |
| ---------------- | ----- |
| None             | None  |
| P1               | P1    |
| P2               | P2    |
| P4               | P4    |
| P1 and P2        | P3    |
| P1 and P4        | P5    |
| P2 and P4        | P6    |
| P1 and P2 and P4 | P7    |

为什么要这样编排呢，这样子index sum就是出错的bit



## 11.30 Network Layer

### Routing & IP

核心数据结构Route table，记录哪一个口对应哪一个网络

Control plane & Data plane

- Control plane：路由表长什么样子，路由算法
- Data plane：查表以及转发，如何发包发的快，并不关心route table长什么样子

每一个包过来到发包只要几千个cycle，性能要求很高



Goal of routing protocol

- 每个节点找到到其他节点的路
- 找cost最小的路
- build routig table（不断变化的，随时随地都会发生crash）



构建路由表：

1. my neighbors
2. advertisement - my neighbors' neighbors
3. minimum cost



两种常见协议

- link state
  - flooding advertisement知道每个node的neighbor
  - dijkstra找到最短路径
  - 缺点是flooding的overhead太大
- distance vector
  - advertisement发送的不仅仅是自己的neighbor，包括所有自己知道的node
  - 需要迭代
  - For each (dst, cost) in the advertisement, X needs to check two things
    1. 如果X通过Y到dst的，更新cost
    2. 如果X不是通过Y到达dst的，看看Y能不能提供一个更短的path



去中心化，加入社区只需要向周围发送给adv就可以让neighbor更新route table

但是去中心化的容错不太行，比如路由劫持



distance vector会有点问题，可能导致infinity问题

- 因为得到的信息不仅仅是自己的neighbor，而是包括了全局的信息
- 如果是link state，如果断了，会直接丢失孤立节点的信息
- 但是distance vector在某一时刻，network断掉之后，别的node可能没有更新这个消息
- 导致infinity
- 解决方案是split horizon，不把来自于B的信息再发回给B



总结：

1. link state：

	- 优点：收敛很快
	- 缺点：flooding is costly

2. distance vector：

	- 优点：low overhead
	- 缺点：收敛时间正比于最长的路径infinity问题



如何解决-三种方式：

1. path-vector routing（可以检查loop，path信息更全，收敛速度更快）
2. hierarchy of routing（用region取代具体节点，region forwarding和local forwarding）
3. topological addressing



BGP is the path-vector protocol used across regions.

18.0.0.0……18.0.0.255，想表示18.0.0.xxx

子网掩码是255.255.255.0

一个IP来了之后，先和子网掩码做一个and，如果是18.0.0.0匹配则说明是这个region

18.0.0.0/24，说明子网掩码是255.255.255.0，前边有24个1



Forwarding an IP packet:

- loopup packet's destination in forwarding table

- TTL:time to live，每次被传递就会-1，防止loop

- update header checksum

- forward packet to outgoing port

- transmit packet onto link



### NAT

需要做ip地址转化

NAT通过端口映射、端口号进行转换

但是！

1. 端口有限，有最大限制
2. 破坏了层级，使用了port，port是上层end-to-end layer的概念
3. 如果不是TCP/UDP，就没有port的概念，就没有办法使用NAT（但是TCP/UDP已经成为标准）

如果设备过多，路由器中回记录很多连接，导致路由器内存不足，会卡



### Ethernet（Link Layer）

一般说以太网，都是link layer，局域网

不同局域网的设备，他们的以太网地址可以相同

listening-before-sending，所以有最小包的限制

以太网使用48bit，IP地址32bit

以太网连接有两种模式：

1. Hub：一条线，在线上边发，每个人都能听到，但是只会判断之后接受自己的包（broadcast每个人都收，有一个特定的地址）
2. Switch：交换机模式



OS维护一个IP映射到MAC地址的表，当向指定IP发包时，会查MAC地址

真正在以太网上流动的数据不是通过IP地址传输的，而是以太网地址（MAC地址）

当A给在另一个局域网的E-15发送数据，地址不能设置为15，而是设置为router的地址，router在向外网发送数据时，在更改以太网地址

ARP协议（Address Resolution Protocol）：

- 发送一个不知道MAC地址、只有IP地址的包，会通过broadcast询问MAC地址
- 当发送给局域网之外的IP地址或者是一个没人知道对应MAC地址的IP地址，就会发送给router



防止ARP的中间人攻击：

- 人工管理ARP表，static ARP entries
- ARP watch



## 12.02 End-to-end Layer

### Assurance of End-to-end Protocol

1. at least once delivery：重发

   - fix timer：会对整个网络造成很大问题，如果整个网络的所有机器都是fix timer进行某一项行为，可能会导致整个网络的崩溃。导致fix timer出现问题的另一个原因是，client没有实现完整协议。当server reply GO-AWAY，client应该停止请求。但是如果不实现这个协议，就不行。

   - adaptive timer：可以采用指数递增的方式进行重发

   - NAK（Negative Acknowledgment）：ack告诉你的是哪一个包没有收到，而不是告诉你收到了什么包

2. at most once

   - 只要记录任何东西，就会导致存在不想删掉的东西
   - 因此需要容忍犯错的可能

3. data integrity（数据完整性）

   - 基本的做法是checksum
   - link layer已经做了checksum，为什么end-to-end还要checksum？底层的可靠不能保证上层的可靠，减少对底层安全性的依赖

4. 分片Sements and Reassembly of Long Message

   - buffer：必须保存所有的message。buffer满了还有几个包没收到，只能丢掉重传
   - order：3种solution
     1. ack in order packets, discards others
     2. hold early packets in buffer
     3. combine the two above
   - message length

5. jitter control（jitter：variablity in delivery time）

   - strategy：delay all arriving segments
   - $Number\ of \ segment\ buffers\ = (D_{long}-D_{short})/D_{headway}$
   - $D_{headway}$ is the average delay between arriving segments.
   - 可以通过缓存机制，问题在于缓存的大小

6. Authenticity and Privacy

   - 非对称算法加密：核心是公钥和私钥
   - 但是非对称加密很慢
   - 大量数据传输，用对称算法加密，这样比较快

7. End-to-end Performance

   下一节课讲

## 12.07 End-to-end Performance & DNS

#### 对于发包Performance

Multi-segment message

1. tarde-off between complexity and performance
2. lock-step protocol



直观的想法：Pipeline

packets or ack may be lost



Fixed window：

连发N个包，收N个ack，再接着发N个包，收N个ack

fixed window可能会有idle time可以利用



Sliding window：

发N个包，收到第1个ack时，接着发送N+1个包，有一个sliding window

发生丢包时，假如丢了second packet，那么就会仍然返回ack 1（duplicate ack）



RTT（round-trip time）

确定window size：

window size >= RTT * bottleneck data rate



#### 对于Congestion

TCP and Congestion Control

网络存在超时重发，如果太多，会导致还在排队包就已经超时了

这样网络会发生拥堵



Network Layer体会到拥塞会向上传递

End-to-end Layer控制传输速率



解决congestion，发包速率应该慢慢变大，而且发生丢包时立刻下降

- Solution：AIMD

- AIMD算法：线性增长，指数下降
- 但是AIMD在开始时非常慢，可以开始时以乘法速率增长



AIMD保证fairness，但如果是additive decrease，就不能保证fairness



### Weakness of TCP

在有线场景下，丢包可能是因为congestion

但是wireless的场景下，丢包可能是因为WIFI信号不好

TCP在数据中心表现不好，应用场景不同



### DNS域名服务器

DNS也是end-to-end layer，但是扮演很重要的role

DNS：look-ip algorithm

域名存在超时机制



一个域名可以有多个ip，一个ip也可以有多个域名



DNS必须是去中心化的，必然流量太大，撑不住

如何组织域名服务器：Hierarchy结构，分工防止部分责任过重



Fault Tolerance：每一个zone都会有多个server



优化：

1. initial DNS request可以问任何一个name server，不一定要问root server，最先问最近的DNS，如果找不到，找上一级，直到问到root server
2. 递归访问查询是client做还是name server做？
   - server做的话，是有状态的，最多服务的数量会比较少，但是server可以cache到域名，而且server之间连接速度很快
   - client可能会稍微慢一些
3. caching



DNS服务器要有多个replcia

如果是organization的name server，至少有一台放在organization外边，否则外边访问不到内网



### DNS design

优点：

- performance
- global names，无需指定context

- 层次结构
- fault tolerance
- replica

缺点

- 政策相关
- root server负载仍然很大
- 安全性：僵尸网络攻击



## 12.09 CDN & P2P

### CDN

Content Distribution Network

互联网刚开始不是为了视频设计的，所以CDN很重要

访问网站时，如何知道取哪一个服务器呢？

1. HTTP redirect

   优点是控制比较细粒度，服务器根据client 的IP找一个近的server，坏处是client有两次请求，而且esrver要进行运算确定重定向到哪一个server

2. Naming-DNS based server selection

   问DNS服务器找server，好处是避免TCP setup delay（HTTP基于TCP），DNS有cache，相对细粒度的控制；缺点是基于local DNS server的IP，还会有hidden-load effect

cient请求CNN的图片，会经过很多次域名解析后，向CDN请求资源，CDN向CNN server拿到资源之后返回给client，CDN取资源比client自己取要快。client缓存了CDN的IP，下次访问就会很快。

中心化和去中心化：

中心化的性能是很高的，但是去中心化可以提高容错能力

中心化的管理成本很高



### P2P

No central servers



BitTorrent：

3 Roles：

1. tracker
2. seeder
3. peer



工作原理

- Publish  a .torrent file on a Web server

- Tracker organizes a swarm of peers (who has the block)

- Seeder posts the URL for .torrent with tracker (Seeder must have complete copy of file)

- Peer asks tracker for a list of peers to download from and tracker returns a list with random selection of peers

- Peers contact peers to learn what parts of the file they have (Download from other peers)



下载顺序：

- Strict?
- Rarest first?
- Random?
- Parallel?



### DHT & 一致性哈希

如果tracker不够用怎么办？DHT（Distributed hash table）

P2P implementation of DHT

Overlay Network：

- partition hash table over n nodes
- not every node knows about all other n nodes
- Route to find right hash table



对key和ip都做hash，这样他们的域就相同了

如果key的hash和ip的hash正好相同，就好了

可惜很难找，不能做点到点的匹配，就做点到range的匹配



一致性哈希！

每一个key和ip同时做hash

ip的hash将环形空间分为许多个range，每一个key做相应的hash后，嵌入这个环中，交给对应range的下家对应的节点保存

1. 每一个node记录自己的下一个node，如果自己找不到，就去问下一个node
   - 时间复杂度O(N)
2. 记录finger table，这样把时间复杂度变为O(log(N))



如果节点failed，可能会出错

- 因此还需要保存successor list，保存后边几个就可以
- 当finger list找不到，就去找successor list



新加入节点，变化很小，适合分布式场景



为了实现load balance，还可以使用virtual node，根据virtual node实现key的平均分配，将节点和物理机解耦



### BitCoin

比特币采用的 SHA256 生成的Hash值都为256位，每一位有62种可能性 (大小写字母与数字的组合)。Block data为上一笔交易资料，比特币的演算即是结合上一笔交易资料与一个随机值nonce并hash出一个位数并不长的值，当此值符合难度值Difficulty时即为成功挖到矿，若不符合难度值时便不断更改随机值直到符合为止。所谓挖到矿就是猜到一个 Nonce 值让该区块的摘要值小于一个会根据难度而调整的难度值。

为了减慢block增加的速度，也会规定算出来的hash的前多少位为0

比特币的区块由区块头及该区块所包含的交易列表组成。区块头的大小为80字节，由4字节的版本号、32字节的上一个区块的散列值、32字节的Merkle Root Hash、4字节的时间缀（当前时间）、4字节的当前难度值、4字节的随机数组成。



### Smart Contract

比特币在上传的时候可以加上长度为 100 字节的注释。

可以让它变成一个脚本，并且要求矿工必须要执行这个脚本。

比如说：“我承诺给B个比特币，但是这个条件还没有满足，假如后面100个block的最后一位是单数，那么我们就给。”

这其实就是一个赌约，但是没有人可以更改这个赌约，到那个时候所有计算机都会执行这个脚本，只是延迟了。

可以把它一个通用的脚本，一个图灵完备的脚本，让全世界挖矿的计 算机都执行这个脚本。

就是一个非常大的分布式系统。一下子把比特币从分布式记账变成了分布式计算。



## 12.14 Security Intro

### 零元购

Shopper A

Shop B

Amazon

A向B买东西，B给A一个单子需要付钱，A向Amazon付钱之后，Amazon告诉B说A已经付过了钱



A现在拿信用卡开了一家店，然后向B买东西，拿到单子之后，把单子上边的B的签名换成自己开的店的签名，自己给自己签名，转账给自己，然后Amazon告诉B说A已经付钱了，直接完成零元购



security is a negative goal and negative goals are hard to achieve



### Policy : Goals

CIA：

- Confidentiality：who can read 隐私性
- Integrity : who can write 完整性
- Availability : ensure service keeps operating 可用性



### Threat Model: What are the Assumptions ?

What does a threat model look like

Threat model的假设是说，你预想攻击者有什么，没有什么，能做什么，不能做什么

控制了一部分电脑，但不是全部？

攻击者知道程序的bug？

攻击者能控制电脑，但不能完全控制？

物理攻击？

社交场景下的攻击（社会工程学）？



### Guard Model

complete mediation：完全中介，谁都不能绕过

- client-server modularity，不能直接访问resource
- must ensure server properly invokes the guard in all the right places，所有的地方都有guard

确认身份和权限



## 12.16 Authentication & Password

Authentication：Who you are

- Something you know
- Something you have
- Something you are



unix文件系统，用fd，无法绕过fd

每一个fork出的紫禁城有相同的user ID

权限认证xwr



Password的验证

数据库可以存储密码的hash

彩虹表：常用密码的hash反推password

加盐：用己方极小的代价极大地提升攻击者的难度，导致攻击者需要为每一个用户单独计算彩虹表，很慢很慢



session cookie：strawman

{username, expiration, Hash(server_key, username, expiration)}

攻击者没有server_key，无法伪造cookie



### Key Problem of Password

- challenge-response scheme
  - 给server发送一个随机数R
  - 在本地算一个Hash(R+pwd)发送过去
  - 服务器需要保存密码重新算一遍
- use password to authemticate the server
  - client选择一个Q发送给server，server计算Hash(Q+pwd)发送给client
  - 只有真正的服务器知道密码
- turn offline into online attack
  - 请你上传一张图片，登陆时server给你显示这一张图片，你就知道这个确实是真的server（应该是真的server）
  - 没有解决安全性问题，但是中国银行发现有人输入用户名，拿到图片之后不输入密码，会知道存在安全风险
- 每一个网站使用不同的密码
- One-time password
  - server嵌套算100次hash
  - 本地算99次hash，server拿到这个value之后再算一次hash，发现和原来一样，存下来
  - 下一次登录，本地算98次hash
  - 可以控制password的使用次数
- bind authentication and request authorization
  - 为每一个请求都加一个authentication
  - {username, "do something", Hash(password, "do something")}
  - 这样token只能用于"do something"
- FIDO：replace the password
  - 使用一下小设备保存private key，登陆时检验是否有这样的小设备
  - server保存有public key
  - Google现在在使用这种方式
  - 指纹数据的保存？指纹数据是不能存放在iOS、Android中的，手机里有一个小的区域，运行额外的OS叫做TrustZone，专门负责指纹校验

注意上述的前两种方法不能同时使用！



Bootstrapping：

更改密码的url包含timestamp、username等信息，同时还包含一个随机数

这个随机数是server生成的，在/dev/random中



## 12.21 Secure Data Flow

Flow：数据被加减乘除

twio usages of data flow tracking

- protect critical data
- defend against suspiciois input data



Ways an attacker cant steal your secrets:

KeyLogger：输入法记录你的信息

Phishing：整一个一模一样的界面，启动应用就转到前台

MemScan：权限高，对内存做扫描

曾经三星有一款手机有一个漏洞：

/dev/pmem权限777

pmem是physica memory，这么重要，权限竟然是777（所有用户可读可写可执行）



进程间隔离依靠的是虚拟内存，但是如果直接access到了物理内存，就跳过了OS



### Taint Tracker

首先思考需要保护的是什么数据

敏感数据的生命周期应该尽可能最小化，数据在内存中时间越久，越容易被暴露：

- 可能发生内存交换，一部分数据swap到磁盘，就偷到了数据
- hibernation，休眠时一部分数据会写到硬盘上



给密码加上一些标记颜色，如果检测到标记颜色的数据出现在奇怪的地方，就说明应用程序可能出现问题

data flow tracking，如果对数据进行打乱、压缩，产生的数据都会被标记，不是简单的数据操作就可以去掉taint的，会监测data flow



- 代码层面的taint需要依靠编译器实现，在编译时会产生大量关于taint维护的代码，导致效率降低10到50倍，涉及许多taint table的访存操作
- 既然这么麻烦，能不能做一个特殊的Android系统，对特定数据、特定API等加上taint，这样就不用再去观察应用app的代码了
- 引入**Taint Droid**
- 安卓应用使用Java编写的，跑在JVM上边，发布的应用是ByteCode，通过JVM变成binary，可以在ByteCode到binary的阶段加上taint
- 粒度也进行了放大，一旦消息中有一个byte是taint、一个文件有一部分taint，那就给整体加上taint。虽然损失精确度，但是也能用而且开销没有那么大
- 用上述方式说的taint降低到了30%以内，因此是可用的



### Defending Malicious Input



如何找到一个bug？

- choose a library(open-source)
  - try ffmpeg
- list the demuxers of ffmpeg
- identifu the input data
- trace the data



越小众的东西，bug越多

具体例子看ppt吧



### TaintCheck Detection Modules

一些taint tracking技术找内存的bug

- 写入一个内存地址，这个地址不能和输入相关，否则认为非法
- jump的地址不能是taint，否则可能根据输入进行控制流修改
- ……



TaintCheck Detection 执行：

- taint seed：标记utrusted data，视为tainted
- taint tracker：追踪每一条指令，看是否是tainted
- taint assert：查看taint数据是否用在危险的地方

性能会较差，大约37倍



### No Data to Protect

garbage collector识别敏感数据（sensitive data），休眠状态时，会在后台进行数据加密，解锁之后再解密

另一种方法时把密码存在trusted server，手机上不放sensitive data



### 安全信道

加密算法，有key才能加密解密

MAC，可以说是加密的Hash，给一个key和一个message，生成一个token，token不能逆向。生成一个token，证明这个token一定包含了key和message的信息



| Alice:         | Bob            |
| -------------- | -------------- |
| c=encrypt(k,m) | m=decrypt(k,c) |
| h=MAC(k,m)     | MAC(k,m)==h?   |

Alice发给Bob的是h和c



reply attack：原因是h和c可以重发好多次

解决方法：seq



但是可能有reflection attack

解决方法：发送和接收采取不同的密钥



### 密钥交换的Diffie-Hellman

1. Alice 和 Bob 可以先通过公开渠道确立质数 p 和数 g
2. 然后 Alice 随机生成一个数 a，计算$g^a\ mod\ p$发送给 Bob，Bob 随机生成一个 b 计算$g^b\ mod\ p$
3. 这样双方都可以计算出$key\ = \ g^{ab}\ mod p$
4. 对于旁人来说，单单从 p 和 g 是很难逆向推出 a 和 b 的。

这个算法是有数论基础的：
$$
(g^a\ mod\ p)^b\ mod\ p=g^{ab}\ mod p=(g^b\ mod\ p)^a\ mod\ p
$$


但是DH没办法防止中间人攻击

防止中间人攻击目前只能用RSA加密算法



## 12.23 ROP & CFI

### Principle of least trust

通过合理的架构，对软件、硬件的信任降低到最低



信任原则：

- 相信OS，不相信应用程序
- 相信OS的核心，不相信OS的其他部分，这是微内核的特点
- 相信CPU（CPU有没有可能被恶意修改？有，但是很难）



TCB：Trusted Computing Base

TCB is the parts that are trusted:

- 进程不相信其他进程，但是信任自己的所有线程
- OS不相信任何进程，但是相信硬件
- VMM不相信VM，但是相信硬件（VMM：虚拟机监控器）
- ……



Root of Trust：

信任BIOS，BIOS启动VMM，VMM验证kernel loader，验证kernel签名，一旦对应就启动OS kernel

硬件层面TPM：Trusted Platform Module

TPM是硬件绑定的，是计算机的Root of Trust，存在物理保护机制

TPM可以验证VMM是否可信，从而构成一条信任链



### Buffer overflow

int 0x80触发系统调用，int全称是interrupt，模拟发生了一次中断，但是其实是exception（由软件触发，必然的确定的）

防止Buffer Overflow Attack的方法是 data execution prevention，数据区是**可写不可执行**



### Return Oriented Programming

假设我们想实现：Mem[v2] = v1

正常情况下

```assembly
movl $v1, %eax
movl $v2, %ebx
movl %eax, (%ebx)
```

但是攻击者需要围绕栈来写

```assembly
pop eax
ret
pop ebx
ret
mov (ebx), eax
```



为了防止攻击

- 把binary藏起来，no way to get any gadget
- ASLR（Address Space Layout Randomization，随机地址）
- Canary金丝雀机制



### CFI：Control-flow Integrity

监控执行流，再程序执行时检查Control-flow Graph



程序什么时候会跳转？

- 直接跳转
  - direct call
  - direct jump
- 间接跳转
  - return（由栈上的返回地址决定）
  - indirect call（call后边的参数是eax，……）
  - indirect jump

binary中direct jump占大多数

运行时indirect jump占大多数



间接跳转的规律：

- 94.7%跳到一个地方
- 99.3%跳转到的地方的数量小于2
- 只有0.1%的程序可能跳转的地方大于10



做跳转、返回检查：

分析完控制流之后，可以为所有的跳转和返回进行编号，只有特定的 label 可以跳转和返回

但是CFI定义的控制流是不精确的，当可能跳转的位置很多时，仍然有可能修改控制流



当一个没有打过CFI-patched的库调用打过patch的库怎么办：

prefetchnta：prefetch memory to cache，如果memory available，就会变成nop指令



CFI 精度不高

- 如果A调用C，B调用C和D，那么C、D的tag是相同的，这种情况下CFI认为A调用D是合法的
- 如果A和B都调用F，那么F返回时可以返回A和B，这也是无法精确检测的

解决CFI精度问题需要shadow stack：

- shadow stack 只存放 return address
- 存在ROP的原因是数据和控制混为一体了，导致overflow会更改返回地址
- 只需要把控制流的return address单独放在shadow stack中，返回时从shadow stack取或者对比返回地址是否和shadow stack相同



### 一个攻击的例子

由于子进程fork出来的，布局和父进程完全相同，则ASLR失效了

当这一个栈帧被回收时，`addq $0xx, %rsp`，就会之心return addr位置的代码

通过buffer overflow更改return addr

看下边的栈

```
stop addr
crash gadget
crash gadget
crash gadget
crash gadget
crash gadget
crash gadget
return address (change this address by buffer overflow)
buf[1024]
```

通过不断更改return addr 尝试找到gadget

```
pop %rbx
pop %rbp
pop %r12
pop %r13
pop %r14
pop %r15
ret
```

### 摆烂了，记不住了，下次一定



## 12.28 Data Privacy

Interatice Zero-Knowledge Proof

- A to B : promise
- B to A : challenge
- A to B : response



### PIR - Private Information Retrieval

Naiive Method：Alice可以把所有数据都查到本地。可以，但是很蠢



2-Server PIR

1. 向server1查询Q1
2. 向server2查询Q1和Q2
3. 将两次的结果做差集就是Q2的结果
4. 但是要求server之间不能互相串通



### OT - Oblivious Transfer

RSA加密

场景：Alice有m0和m1，Bob想要其中一个，但是不想让Alice知道是哪一个



Sol-1：

1. Alice生成两对公私钥，并且把两个公钥发给Bob
2. Bob选取一个随机数，用其中一个公钥加密这个随机数发给Alice
3. Alice分别用两把私钥解密message之后得到k0和k1
4. 分别用k0和k1与m0和m1做xor，得到e0和e1，发送给Bob
5. Bob用自己的随机数做xor得到有用的消息



Sol-2：

1. Alice生成pub_key和两个随机数r0和r1，把pub_key、r0、r1发送给Bob
2. Bob生成一个key，用pub_key对key做加密之后与r0或r1做xor得到消息V
3. Bob把V发送给Alice，Alice不知道哪一个是Bob选择的随机数r
4. Alice把V分别用r0和r1做xor之后，使用private_key解密得到k0和k1，再用k0和k1分别对m0和m1加密之后发送给Bob
5. Bob得到两个消息之后用自己的key解密，得到其中一条是有用的消息



限制：真实场景的Server（也就是Alice）有海量数据，对于系统是无法接受的



### Differential Privacy

总的统计量和去掉某一个同学的统计量

对抗方法是增加noise，加一些混淆



### Secret Sharing

把秘密数据分成若干份，至少找到多少份才能得到完整数据
$$
f(x)=S+a_1 x + a_2 x^2 + \dots + a_{k-1} x^{k-1}
$$
S是要保存的数据

把这条线上边的 n 个点告诉不同的人，只要知道 k 个点的值，就可以还原出多项式的系数

取x=0就可以得到 S



n个人只要有k个人就可以把数据恢复出来，每个人手里的仅仅是一个part，而不是replica

在空间上用少于k-replica的空间得到k-replica的效果



### Secure MPC（安全的多方计算）

百万富翁问题



sMPC的前提是Alice和Bob要遵守protocol



### 同态加密（Homomorphic Encryption）

加密搜索：

公司将数据加密之后发送给Google，Google进行对加密数据的索引，厂商将关键词加密之后查询，就可以拿到索引



加密运算：

加密之后发送给云服务厂商进行运算，运算后得到模型可以解密



目前没有全同态的加密算法



全同态加密算法：支持所有类型的运算，而且保序



在某些非常注重安全的场景，如果做range query，将数据库做保序加密之后运算，然后删掉整个数据库



### 可信执行环境（Hardware Enclave硬件飞地）

两个特点：

1. 隔离执行，任何人不能偷到数据
2. 远程证明（Remote Attestation），证明自己是飞地



Isolation Execution：

在 user process 里，OS 在地址空间的头部

地址空间内部存在 enclave data 和 enclave code，当 OS 要访问 enclave 的时候，就会被 CPU 禁止

不妨设置 CPU reserved memory，这样就可以在 MMU 的时候就禁止住。

CPU外部是密文就可以，CPU内部很难盗取



Remote Attestation：

CPU的私钥在CPU内部



EnclaveDB
