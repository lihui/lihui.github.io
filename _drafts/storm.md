---
layout: post
title: 实时分布式流数据计算框架述要
author: 灵元
---


## 1 介绍
实时流数据处理的应用场景非常广泛，如实时统计分析，在线机器学习等。支撑实时流处理数据的系统通常有如下功能需求：（1）可伸缩性，运营团队可以容易地在系统中地添加、删除计算节点，而不影响正在运行的数据流处理作业（2）可恢复性，又称作“弹性”，在大规模集成系统中，容灾是一个必备的特性（3）可扩展性，也即在该系统中，要能够容易地实现新的处理逻辑（4）易于管理。

与hadoop mapreduce,spark 等批处理系统不同，流处理框架所处理的是连续数据流，这意味着作业将一直运行，所有的task同时启动着。虽然两者的计算逻辑都可以用map，group，reduce等基本操作构造而成，但流计算通常会引入状态管理和消息传递的可靠性问题，这引入了额外的复杂性。

流计算框架一般包含如下几个关键要素：

1. 数据流模型，即流如何定义的
2. 计算任务节点的编程模型
3. 数据交互，任务节点之间的数据流是如何关联起来的
4. 并行机制，为了支持大规模数据计算，一个数据流不大可能仅在一台机器上处理，因此流计算框架要能够对数据流进行切分，并且分配到几个并发的计算实例上去
5. 消息传送保障机制
6. 窗口
6. 失效恢复

本文将从这几个角度介绍几个流行的分布式流计算框架。

为了叙述方便，我们设定一个典型流计算应用场景。在线商店中需要统计商品查看、购买、收藏、关键字查询等操作数量，独立访问量等。这里涉及计数、基数统计等问题。下面我们将以这些操作的统计作为示例来演示流计算的基本编程方法。

## 2 storm

###2.1 storm流计算概览

  画图，展示各组成结构
 考虑叙述整体运行过程，从TopologySubbmiter开始。

###2.2 数据流模型
在storm中数据流由$&lt;ComponentID,StreamID，TupleSchema &gt;$唯一定义，其中$ComponentID$对应着流计算拓扑图（以后称$topology$）中的一个计算节点。是用户编写的计算组件（以后称为$component$）类在$topology$定义时的一次出现（类比于程序中的静态实体）。 在$topology$中通常一个组件类对应一个 $ComponentID$，但如果该$topology$比较复杂的话也可以对应着多个$ComponentID$，一个$ComponentID$在运行时会产生一个或者多个并发实例（称为$task$），并发度在构造$topology$的时候指定。

$component$可发送一条或者多条数据流，每条数据流由$StreamID$标示。大多数$component$只发送一条数据流，这时在发送数据流中的元素时就不必显式地指明$StreamID$，系统会使用一个默认的名字“default”。

数据流中的每一个元素叫做$Tuple$，从概念上是一个字段列表。为了简化编程接口的复杂性，这些字段都是动态类型的，只有字段名称。在每一个$component$类的定义中，都会通过一个叫declareOutputFields的方法向storm申明它所产生的数据流中的元素包含那些字段，这就是$TupleSchema$。

然而在storm系统中，为了支撑可靠性传输等服务，在网络传输中的$Tuple$对象可能包含有其他的信息，如MessageId，但这些信息对开发者是不可见的。



### 2.3 编程模型

使用storm框架进行流计算涉及三个基本概念，spout，bolt，topology。spout定义了流计算的消息来源，系统外的数据自此而进入，bolt则将来自于spout的消息进行处理，并且对下游发出新的消息，或者将数据处理结果存储在系统外的某个地方，spout和bolt统称task，是流计算的基本单元。topology则定义spout、bolt之间的消息依赖和并发度，使之构成完整的计算逻辑。


####2.3.1 spout任务
定义spout一般需要继承自基类BaseRichSpout，它为IRichSpout，IComponent接口提供空方法默认实现。下面列出常用方法。
<pre><code class="java">public class CustomSpout extends BaseRichSpout {
    void open(Map conf, TopologyContext context, 
              SpoutOutputCollector collector);
    void close();
    void nextTuple();
    void ack(Object msgId);
    void fail(Object msgId);
    void declareOutputFields(OutputFieldsDeclarer declarer);
}
</code></pre>
    
1. open方法：在spout实例启动的时候调用，一般在此初始化访问外部资源的链接，将其存储在对象的成员变量中，供后续使用。如果spout需要处理消息失效问题（通过ack，fail方法得到通知），那么还需要建立缓存，保存已经发送过的消息，在得到消息失效的通知时，需要在此发送，在消息得到确认后，则可从缓存中删除。为了降低网络开销提高效率，有些实现会从外部数据源成批地读取数据，因此也会在此建立待发送的数据队列（pending quene）
2. close方法：负责释放资源
3. nextTuple方法：被storm方法调用以获取新的消息，是接口要实现的主体方法。其逻辑实现一般是：若待发送队列还有数据，就取出一条发送，若没有则从外部批量取出一条，再发送。如果要提供“确保一次”传输保证，在发送消息时就应该构造一个唯一msgId，与消息一起在collector.emit方法中发送出去
4. ack方法：在消息确认时被storm调用，实现时一般应将消息从待确认队列中删除
5. fail方法：在消息失效时被storm调用，实现时一般应重发msgId所对应的消息

当然对于某些知名的外部数据源一般都有现成的，无需我们自己实现，如对于Kafka数据源网上就有很多个版本的KafkaSpout实现。

假设网店系统使用Kafka记录用户操作日志，发送的消息为: &lt;userId,productId,actionType,timestamp &gt;。

#### 2.3.2 bolt任务
定义bolt一般需要继承自基类BaseRichBolt，它为接口IRichBolt,IComponent提供空方法默认实现。下面列出常用方法：
<pre><code class="java">public CustomBolt extends BaseRichSpout{
    public void prepare(Map stormConf, TopologyContext context);
    public void execute(Tuple input, BasicOutputCollector collector);
    public void cleanup();
    void declareOutputFields(OutputFieldsDeclarer declarer);
}
</code></pre>

1. prepare方法：在bolt实例启用时调用，此初始化状态，如若要将数据写入到系统外部，还需要初始化访问外部资源的连接等
2. execute方法：计算的主体方法，若要支持“确保一次”的传送保证，在发送的时候，需要建立anchor,以将本消息与上游消息关联起来，通过这种方式，构建整个确认树。方法是BasicOutputCollector.emit(Tuple anchor,List<Object> values)。如果本条消息是由上游的多条消息构成，那么作为anchor的Tuple就是个Tuple数组：emit(List<Tuple> anchors,List<Object>values)
3. cleanup方法：storm结束时调用，进行资源清理

 
注意在上述task nextTuple、execute的实现中，不应该使用阻塞操作，将当前线程block掉的话会影响storm的整体调度性能。如果当前没有立即可读的数据，nextTuple在本次调用中可以不发射数据，系统会停顿一些时间如1ms后再次调用。

####2.3.3 Toplogy

Topology使用TopologyBuilder进行构造，定义了计算流图的整体拓扑结构，描述task之间数据流关系以及并发度，是一种纯数据的实体。使用代码样例能更好地表达清楚topology构造的要点：
<pre><code class="java">TopologyBuilder builder=new TopologyBuilder();
builder.setSpout("kafka",new KafkaSpout());
builder.setBolt("")

</code></pre>


###2.4 并行机制
并行度由数据切分和并行计算单元的数目共同确定。
####2.4.1 数据流切分

如同批处理系统一样，流计算的并行性也主要来源于数据并行。因此一个数据流被切分为多个子流分别发送给下游$task$处理。数据流划分方法由group策略指定：
<table width="100%" height="100%" class="table table-bordered table-striped table-condensed">
   <tr >
      <td valign="middle">shuffleGrouping</td><td>数据流中的数据元组均匀随机分发给下游组件的各个$task$,这种分组策略不会产生负载失衡的情况。但这种方案不能单独用于依某字段值进行分组统计的功能 </td>
   </tr>
   <tr>
      <td>fieldsGrouping</td><td> 数据流中的数据元组依据一个或者多个字段（组成key）的hash值映射到下游组件的某个$task$，每一个Key值相关的元组只会被路由到下游组件某一个$task$。适合实现分组统计的功能 </td>
   </tr>
   <tr>
      <td>parialKeyGrouping</td><td> 这种分组策略是上述两种策略的折中，一个key值可以被分发到两个$task$</td>
   </tr>
      <tr>
      <td>All grouping</td><td>数据流元组发送给下游所有task，每个task都收到相同的一份数据</td>
   </tr>
      <tr>
      <td>Global grouping</td><td>整个数据流只发给下游的一个task通常是taskId值最低的那个</td>
   </tr>
      <tr>
      <td>None grouping</td><td>与shuffleGrouping一样，未来可能会在调度上有优化</td>
   </tr>
      </tr>
      <tr>
      <td>Direct grouping</td><td>由数据流的源端指定目标端的$TaskId$，在源端使用collector的emitDirect方法将数据发送至目标$task$</td>
   </tr>
      </tr>
      <tr>
      <td>localOrShuffleGrouping</td><td>语义类同shuffleGrouping，但利用了$task$间的局部性进行优化，优先发送给与源task在同一个进程的那些目标$task$</td>
   </tr>
</table>
有必要更进一步叙述parialKeyGrouping的意义。相比shuffleGrouping会将元组分发到任意$task$,fieldsGroup只会将相同的key分发到一个$task$，parialKeyGrouping则会将相同的key发送到两个$task$，该策略的提出是因为有些key值的分布可能极度不均匀，会导致下游有些$task$过载，有些则很空闲。

假设要作基于某个key的分组统计，key有K种不同的值，且有T个$task$。如果使用shuffleGrouping进行部分统计，那么统计所需的内存开销是O(K*T)，fieldsGrouping是O(K)。而使用parialKeyGrouping替代shuffleGrouping做部分统计的话，内存开销仅是fieldsGrouping的两倍，但大大地降低了负载失衡的情况。

####2.4.2 task数量配置
TopologyBuilder的setBolt方法有个参数parallelism_hint用来设置组件的并发度,注意这不等于生成的task的数目，而是运行该组件的线程数目，只不过默认线程数目等于task数目罢了。setBolt方法返回的BoltDeclarer对象可以用setNumTasks来设置task数目。

####2.5 消息传送保障机制

（记着调研并行执行单元增加降低的情况，或者storm不允许运行时动态伸缩？）

##3 samza

##4 流计算常用模式
