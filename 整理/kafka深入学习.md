@[TOC](目录)

# MQ基础

**MQ的应用场景：**

- 异步处理 - 相比于传统的串行、并行方式，提高了系统吞吐量。
- 应用解耦 - 系统间通过消息通信，不用关心其他系统的处理。
- 流量削锋 - 可以通过消息队列长度控制请求量；可以缓解短时间内的高并发请求。
- 日志处理 - 解决大量日志传输。
- 消息通讯 - 消息队列一般都内置了高效的通信机制，因此也可以用在纯的消息通讯。比如实现点对点消息队列，或者聊天室等。

主要是：解耦、异步、削峰。

**解耦**：A 系统发送数据到 BCD 三个系统，通过接口调用发送。如果 E 系统也要这个数据呢？那如果 C 系统现在不需要了呢？A 系统负责人几乎崩溃…A 系统跟其它各种乱七八糟的系统严重耦合，A 系统产生一条比较关键的数据，很多系统都需要 A 系统将这个数据发送过来。如果使用 MQ，A 系统产生一条数据，发送到 MQ 里面去，哪个系统需要数据自己去 MQ 里面消费。如果新系统需要数据，直接从 MQ 里消费即可；如果某个系统不需要这条数据了，就取消对 MQ 消息的消费即可。这样下来，A 系统压根儿不需要去考虑要给谁发送数据，不需要维护这个代码，也不需要考虑人家是否调用成功、失败超时等情况。

就是一个系统或者一个模块，调用了多个系统或者模块，互相之间的调用很复杂，维护起来很麻烦。但是其实这个调用是不需要直接同步调用接口的，如果用 MQ 给它异步化解耦。

**异步**：A 系统接收一个请求，需要在自己本地写库，还需要在 BCD 三个系统写库，自己本地写库要 3ms，BCD 三个系统分别写库要 300ms、450ms、200ms。最终请求总延时是 3 + 300 + 450 + 200 = 953ms，接近 1s，用户感觉搞个什么东西，慢死了慢死了。用户通过浏览器发起请求。如果使用 MQ，那么 A 系统连续发送 3 条消息到 MQ 队列中，假如耗时 5ms，A 系统从接受一个请求到返回响应给用户，总时长是 3 + 5 = 8ms。

**削峰**：减少高峰时期对服务器压力。



**缺点：**

**系统可用性降低**

本来系统运行好好的，现在你非要加入个消息队列进去，那消息队列挂了，你的系统不是呵呵了。因此，系统可用性会降低；

**系统复杂度提高**

加入了消息队列，要多考虑很多方面的问题，比如：**一致性问题、如何保证消息不被重复消费、如何保证消息可靠性传输**等。因此，需要考虑的东西更多，复杂性增大。

**一致性问题**

A 系统处理完了直接返回成功了，人都以为你这个请求就成功了；但是问题是，要是 BCD 三个系统那里，BD 两个系统写库成功了，结果 C 系统写库失败了，咋整？你这数据就不一致了。

所以消息队列实际是一种非常复杂的架构，你引入它有很多好处，但是也得针对它带来的坏处做各种额外的技术方案和架构来规避掉，做好之后，你会发现，妈呀，系统复杂度提升了一个数量级，也许是复杂了 10 倍。

**MQ比较**

- RabbitMQ：Erlang语言开发，2.6w/s 单机吞吐量，Mozilla/Spring 维护，成熟
- RocketMQ：Java开发，11.6w/s 单机吞吐量，Alibaba维护
- Kafka：Apache 维护，应用大数据领域的实时计算、日志采集等场景 ，成熟，10w+单机吞吐量。
- 。。。

**设计MQ思路**

比如说这个消息队列系统，我们从以下几个角度来考虑一下：

首先这个 mq 得**支持可伸缩性吧，就是需要的时候快速扩容**，就可以增加吞吐量和容量，那怎么搞？设计个分布式的系统呗，参照一下 kafka 的设计理念，broker -> topic -> partition，每个 partition 放一个机器，就存一部分数据。如果现在资源不够了，简单啊，给 topic 增加 partition，然后做数据迁移，增加机器，不就可以存放更多数据，提供更高的吞吐量了？

其次你得考虑一下这个 mq 的数据要不要落地磁盘吧？那肯定要了，**消息落磁盘**才能保证别进程挂了数据就丢了。那落磁盘的时候怎么落啊？顺序写，这样就没有磁盘随机读写的寻址开销，**磁盘顺序读写**的性能是很高的，这就是 kafka 的思路。

其次你考虑一下你的 **mq 的可用性**啊？这个事儿，具体参考之前可用性那个环节讲解的 kafka 的高可用保障机制。多副本 -> leader & follower -> broker 挂了重新选举 leader 即可对外服务。

# 架构

http://www.topgoer.com/%E6%95%B0%E6%8D%AE%E5%BA%93%E6%93%8D%E4%BD%9C/go%E6%93%8D%E4%BD%9Ckafka/kafka%E6%B7%B1%E5%B1%82%E4%BB%8B%E7%BB%8D.html

![架构介绍](http://www.topgoer.com/static/9.2/1.jpg) 

- Producer：Producer即生产者，消息的产生者，是消息的⼊口。
- kafka cluster：kafka集群，一台或多台服务器组成
- Broker：Broker是指部署了Kafka实例的服务器节点。每个服务器上有一个或多个kafka的实 例，我们姑且认为每个broker对应一台服务器。每个kafka集群内的broker都有一个不重复的 编号，如图中的broker-0、broker-1等……
- Topic：消息的主题，可以理解为消息的分类，kafka的数据就保存在topic。在每个broker上 都可以创建多个topic。实际应用中通常是一个业务线建一个topic，类似于关系型数据库中表的概念。
- Partition：Topic的分区，每个topic可以有多个分区，分区的作用是做负载，提高kafka的吞 吐量。同一个topic在不同的分区的数据是不重复的，类似于一个队列，partition的表现形式就是一个一个的⽂件夹，文件夹中消息的数据文件*.log和索引文件。
- Replication:每一个分区都有多个副本，副本的作用是做备胎。当主分区（Leader）故障的 时候会选择一个备胎（Follower）上位，成为Leader。在kafka中默认副本的最大数量是10 个，且副本的数量不能大于Broker的数量，follower和leader绝对是在不同的机器，同一机 器对同一个分区也只可能存放一个副本（包括自己）。
- Consumer：消费者，即消息的消费方，是消息的出口。
- Consumer Group：我们可以将多个消费组组成一个消费者组，在kafka的设计中同一个分 区的数据只能被消费者组中的某一个消费者消费。**同一个消费者组的消费者可以消费同一个 topic的不同分区的数据**，这也是为了提高kafka的吞吐量。

**Kafka性能高的原因**

为什么要用kafka，不用其他的mq

**1、顺序写**

操作系统每次从磁盘读写数据的时候，需要先寻址，也就是先要找到数据在磁盘上的物理位置，然后再进行数据读写，如果是机械硬盘，寻址就需要较长的时间。

Kafka 的设计中，数据其实是存储在磁盘上面，一般来说，会把数据存储在内存上面性能才会好。

但是 Kafka 用的是顺序写，追加数据是追加到末尾，磁盘顺序写的性能极高，在磁盘个数一定，转数达到一定的情况下，基本和内存速度一致。

随机写的话是在文件的某个位置修改数据，性能会较低。

**2、零拷贝**

先来看看非零拷贝的情况：

可以看到数据的拷贝从内存拷贝到 Kafka 服务进程那块，又拷贝到 Socket 缓存那块，整个过程耗费的时间比较高。

Kafka 利用了 Linux 的 sendFile 技术（NIO），省去了进程切换和一次数据拷贝，让性能变得更好。 

# 生产消费

生产消息流程：

![⼯作流程](http://www.topgoer.com/static/9.2/2.jpg)

1. 生产者从Kafka集群获取分区leader信息
2. 生产者将消息发送给leader
3. leader将消息写入本地磁盘
4. 所有的follower从leader拉取消息数据
5. 所有的follower将消息写入本地磁盘后向leader发送ACK
6. leader收到所有的follower的ACK之后向生产者发送ACK

生产者的ACK应答机制：

producer在向kafka写入消息的时候，可以设置参数来确定是否确认kafka接收到数据，这个参数可设置 的值为 0,1,all。最后要注意的是，如果往不存在的topic写数据，kafka会⾃动创建topic，partition和replication的数量 默认配置都是1。

- 0代表producer往集群发送数据不需要等到集群的返回，不确保消息发送成功。安全性最低但是效 率最高。
- 1代表producer往集群发送数据只要leader应答就可以发送下一条，只确保leader发送成功。
- -1或者all代表producer往集群发送数据需要所有的follower都完成从leader的同步才会发送下一条，确保 leader发送成功和所有的副本都完成备份。安全性最⾼高，但是效率最低。

选择partition的原则：

- partition在写入的时候可以指定需要写入的partition，如果有指定，则写入对应的partition。
- 如果没有指定partition，但是设置了数据的key，则会根据key的值hash出一个partition。
- 如果既没指定partition，又没有设置key，则会采用轮询⽅式，即每次取一小段时间的数据写入某个partition，下一小段的时间写入下一个partition

生产者向Kafka 中发送1条消息的时候，可以指定(topic, partition, key) 3个参数。partiton 和 key 是可选的。如果你指定了 partition，那就是所有消息发往同1个 partition，就是有序的。并且在消费端，Kafka 保证，1个 partition 只能被1个 consumer 消费。或者你指定 key（比如 order id），具有同1个 key 的所有消息，会发往同1个 partition。也是有序的。

Topic和数据⽇志：

topic 是同⼀类别的消息记录（record）的集合。在Kafka中，⼀个主题通常有多个订阅者。对于每个 主题，Kafka集群维护了⼀个分区数据⽇志⽂件结构如下：

![Topic和数据⽇志](http://www.topgoer.com/static/9.2/3.jpg)

每个partition都是⼀个有序并且不可变的消息记录集合。当新的数据写⼊时，就被追加到partition的末尾。在每个partition中，每条消息都会被分配⼀个顺序的唯⼀标识，这个标识被称为offset，即偏移 量。注意，Kafka只保证在同⼀个partition内部消息是有序的，在不同partition之间，并不能保证消息 有序。

Partition在服务器上的表现形式就是⼀个⼀个的⽂件夹，每个partition的⽂件夹下⾯会有多组segment ⽂件，每组segment⽂件⼜包含 .index ⽂件、 .log ⽂件、 .timeindex ⽂件三个⽂件，其中 .log ⽂ 件就是实际存储message的地⽅，⽽ .index 和 .timeindex ⽂件为索引⽂件，⽤于检索消息。

消费消息：

多个消费者实例可以组成⼀个消费者组，并⽤⼀个标签来标识这个消费者组。⼀个消费者组中的不同消 费者实例可以运⾏在不同的进程甚⾄不同的服务器上。如果所有的消费者实例都在同⼀个消费者组中，那么消息记录会被很好的均衡的发送到每个消费者实 例。如果所有的消费者实例都在不同的消费者组，那么每⼀条消息记录会被⼴播到每⼀个消费者实例。

![消费数据](http://www.topgoer.com/static/9.2/4.png)

举个例⼦，如上图所示⼀个两个节点的Kafka集群上拥有⼀个四个partition（P0-P3）的topic。有两个 消费者组都在消费这个topic中的数据，消费者组A有两个消费者实例，消费者组B有四个消费者实例。 从图中我们可以看到，在同⼀个消费者组中，每个消费者实例可以消费多个分区，但是每个分区最多只 能被消费者组中的⼀个实例消费。也就是说，如果有⼀个4个分区的主题，那么消费者组中最多只能有4 个消费者实例去消费，多出来的都不会被分配到分区。其实这也很好理解，如果允许两个消费者实例同 时消费同⼀个分区，那么就⽆法记录这个分区被这个消费者组消费的offset了。如果在消费者组中动态 的上线或下线消费者，那么Kafka集群会⾃动调整分区与消费者实例间的对应关系。

**Zookeeper的应用**

https://blog.csdn.net/Peter_Changyb/article/details/81562855

Kafka使用zk的分布式协调服务，将生产者，消费者，消息储存（broker，用于存储信息，消息读写等）结合在一起。同时借助zk，kafka能够将生产者，消费者和broker在内的所有组件在无状态的条件下建立起生产者和消费者的订阅关系，实现生产者的负载均衡。

1. broker在zk中注册

kafka的每个broker（相当于一个节点，相当于一个机器）在启动时，都会在zk中注册，告诉zk其brokerid，在整个的集群中，broker.id/brokers/ids，当节点失效时，zk就会删除该节点，就很方便的监控整个集群broker的变化，及时调整负载均衡。

2. topic在zk中注册

在kafka中可以定义很多个topic，每个topic又被分为很多个分区。一般情况下，每个分区独立在存在一个broker上，所有的这些topic和broker的对应关系都有zk进行维护

3. consumer(消费者)在zk中注册

注册新的消费者，当有新的消费者注册到zk中，zk会创建专用的节点来保存相关信息，路径ls /consumers/{group_id}/  [ids,owners,offset]，Ids:记录该消费分组有几个正在消费的消费者，Owmners：记录该消费分组消费的topic信息，Offset：记录topic每个分区中的每个offset; 监听消费者分组中消费者的变化 

# 常见问题

https://zhuanlan.zhihu.com/p/341546586

1、消息重复

消费者的接口冥等即可；

用redis搞一个全局的消息id和消息value，消费前检验消息有没有被消费。

2、消息丢失

1）生产者丢消息

生产者发送消息的一般流程：

1. 生产者是与leader直接交互，所以先从集群获取topic对应分区的leader元数据；
2. 获取到leader分区元数据后直接将消息发给过去；
3. Kafka Broker对应的leader分区收到消息后写入文件持久化；
4. Follower拉取Leader消息与Leader的数据保持一致；
5. Follower消息拉取完毕需要给Leader回复ACK确认消息；
6. Kafka Leader和Follower分区同步完，Leader分区会给生产者回复ACK确认消息。

![img](https://pic1.zhimg.com/80/v2-ecd5faa23c910d19d1e6ad45428f9ad0_720w.jpg)

生产者采用push模式将数据发布到broker，每条消息追加到分区中，顺序写入磁盘。消息写入Leader后，Follower是主动与Leader进行同步。Kafka消息发送有两种方式：同步（sync）和异步（async），默认是同步方式，可通过producer.type属性进行配置。Kafka通过配置request.required.acks属性来确认消息的生产：

- 0表示不进行消息接收是否成功的确认；不能保证消息是否发送成功，基本不会用。
- 1表示当Leader接收成功时确认；只要Leader存活就可以保证不丢失，保证了吞吐量。
- -1或者all表示Leader和Follower都接收成功时确认；可以最大限度保证消息不丢失，但是吞吐量低。

kafka producer 的参数acks 的默认值为1，所以默认的producer级别是at least once，并不能exactly once。在生产环境中严格做到exactly once其实是难的，同时也会牺牲效率和吞吐量，最佳实践是业务侧做好补偿机制，万一出现消息丢失可以兜底 。

生产者丢消息的场景：

- 如果acks配置为0，发生网络抖动消息丢了，生产者不校验ACK自然就不知道丢了。
- 如果acks配置为1保证leader不丢，但是如果leader挂了，恰好选了一个没有ACK，没有存消息的follower，那也丢了。
- -1或all：保证leader和follower不丢，但是如果网络拥塞，没有收到ACK，会有重复发的问题。

2）kafka broker丢消息 

操作系统本身有一层缓冲区，叫做 Page Cache，当往磁盘文件写入的时候，系统会先将数据流写入缓冲区中，至于什么时候将缓冲区的数据写入文件中是由操作系统自行决定。Kafka提供了一个参数 producer.type 来控制是不是主动flush，如果Kafka写入到mmap之后就立即 flush 然后再返回 Producer 叫同步 (sync)；写入mmap之后立即返回 Producer 不调用 flush 叫异步 (async)。

![img](https://pic4.zhimg.com/80/v2-04dd7b312732a6d4f9d6deb49c2ed05f_720w.jpg) 

Kafka通过多分区多副本机制中已经能最大限度保证数据不会丢失，如果数据已经写入系统 cache 中但是还没来得及刷入磁盘，此时突然机器宕机或者掉电那就丢了，当然这种情况很极端。

3）消息者丢消息

消费者通过pull模式主动的去 kafka 集群拉取消息，与producer相同的是，消费者在拉取消息的时候也是找leader分区去拉取。多个消费者可以组成一个消费者组（consumer group），每个消费者组都有一个组id。**同一个消费组者的消费者可以消费同一topic下不同分区的数据**，但是不会出现多个消费者消费同一分区的数据。消费者消费的进度通过offset保存在kafka集群的__consumer_offsets这个topic中。

![img](https://pic1.zhimg.com/80/v2-92e5b7286b5299d2cc61bc2a972cfa70_720w.jpg)

消费者丢消息的场景：

场景一：先commit offset再处理消息。如果在处理消息的时候异常了，但是offset 已经提交了，这条消息对于该消费者来说就是丢失了，再也不会消费到了。

场景二：先处理消息再commit。如果在commit之前发生异常，下次还会消费到该消息，重复消费的问题可以通过业务保证消息幂等性来解决。

消息丢失的其他解决方法：

有些东西不是我们能控制的，比如网络抖动等不可抗拒的因素，这时候重试次数就很关键了，配置合适的retries**重试**次数，和 合适的retry.backoff.ms重试间隔时间，将为我们的数据发送提供更高的稳定性，当然如果实在发送不成功，怎么办呢？一般我们也可以**把发送不成功的数据保存在一个日志文件，如果数据很重要，那就发送警告信息**，人工干预一下。

3、消息的顺序性

保证全局顺序和局部顺序

1）局部顺序保证：

https://www.cnblogs.com/sddai/p/11340870.html

在大部分业务场景下，只需要保证消息局部有序即可，什么是局部有序？局部有序是指在某个业务功能场景下保证消息的发送和接收顺序是一致的。如：订单场景，要求订单的创建、付款、发货、收货、完成消息在同一订单下是有序发生的，即消费者在接收消息时需要保证在接收到订单发货前一定收到了订单创建和付款消息。

针对这种场景的处理思路是：针对部分消息有序（message.key相同的message要保证消费顺序）场景，可以在producer往kafka插入数据时控制，同一key（订单的id）分发到同一partition上面。

2）全局顺序保证

https://www.cnblogs.com/windpoplar/p/10747696.html

比如说我们建了一个 topic，有三个 partition。生产者在写的时候，其实可以指定一个 key，比如说我们指定了某个订单 id 作为 key，那么这个订单相关的数据，一定会被分发到同一个 partition 中去，而且这个 partition 中的数据一定是有顺序的。
消费者从 partition 中取出来数据的时候，也一定是有顺序的。到这里，顺序还是 ok 的，没有错乱。接着，**我们在消费者里可能会搞多个线程来并发处理消息**。因为如果消费者是单线程消费处理，而处理比较耗时的话，比如处理一条消息耗时几十 ms，那么 1 秒钟只能处理几十条消息，这吞吐量太低了。而多个线程并发跑的话，顺序可能就乱掉了。

![img](https://img2018.cnblogs.com/blog/1630503/201904/1630503-20190421231640388-1172315562.png)

解决方案

- 一个 topic，一个 partition，一个 consumer，内部单线程消费，单线程吞吐量太低，一般不会用这个。
- 写 N 个内存 queue，具有相同 key 的数据都到同一个内存 queue；然后对于 N 个线程，每个线程分别消费一个内存 queue 即可，这样就能保证顺序性 。不同的线程处理不同的订单，相同的key的数据在一个队列里面，然后使用多线程开worker按照key不同进行分别消费 。

![img](https://img2018.cnblogs.com/blog/1630503/201904/1630503-20190421231840924-2043607136.png)

4、队列满了处理

https://blog.csdn.net/qq_36236890/article/details/81174504

几千万条数据在MQ里积压了。这种情况只能操作临时紧急扩容了，具体操作步骤和思路如下：

- 先修复consumer的问题，确保其恢复消费速度，然后将现有consumer都停掉。
- 新建一个topic，partition是原来的10倍，临时建立好原先10倍或者20倍的queue数量。
- 然后写一个临时的分发数据的consumer程序，这个程序部署上去消费积压的数据，消费之后不做耗时的处理，直接均匀轮询写入临时建立好的10倍数量的queue。
- 接着临时征用10倍的机器来部署consumer，每一批consumer消费一个临时queue的数据。
- 这种做法相当于是临时将queue资源和consumer资源扩大10倍，以正常的10倍速度来消费数据。
- 等快速消费完积压数据之后，得恢复原先部署架构，重新用原先的consumer机器来消费消息。





















