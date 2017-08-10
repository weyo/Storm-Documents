# Trident 教程

Trident 是 Storm 的一种高度抽象的实时计算模型，它可以将高吞吐量（每秒百万级）数据输入、带状态的流式处理与低延时的分布式查询无缝结合起来。如果你了解 Pig 或者 Cascading 这样的高级批处理工具，你就会发现 Trident 的概念非常相似。Trident 同样有联结（join）、聚合（aggregation）、分组（grouping）、函数（function）以及过滤器（filter）这些功能。Trident 为数据库或者其他持久化存储上层的状态化、增量式处理提供了基础原语。由于 Trident 有着一致的、恰好一次的语义，因此推断出 Trident 拓扑的状态也是一件很容易的事。

## 使用范例

让我们先从一个 Trident 使用的例子开始。这个例子中做了两件事情：

1. 从一个句子的输入数据流中计算出单词流的数量
2. 实现对一个单词列表中每个单词总数的查询

为了实现这个目的，这个例子将会从下面的数据源中无限循环地读取语句数据流：

```java
FixedBatchSpout spout = new FixedBatchSpout(new Fields("sentence"), 3,
               new Values("the cow jumped over the moon"),
               new Values("the man went to the store and bought some candy"),
               new Values("four score and seven years ago"),
               new Values("how many apples can you eat"));
spout.setCycle(true);
```

这个 Spout 会循环地访问语句集来生成语句数据流。下面的代码就是用来实现计算过程中的单词数据流统计部分：

```java
TridentTopology topology = new TridentTopology();        
TridentState wordCounts =
     topology.newStream("spout1", spout)
       .each(new Fields("sentence"), new Split(), new Fields("word"))
       .groupBy(new Fields("word"))
       .persistentAggregate(new MemoryMapState.Factory(), new Count(), new Fields("count"))                
       .parallelismHint(6);
```

让我们一行行地来分析上面的代码。首先我们创建了一个 `TridentTopology` 对象，这个对象提供了构造 Trident 计算过程的接口。`TridentTopology` 有一个叫做 `newStream` 的方法，这个方法可以从一个输入数据源中读取数据创建一个新的数据流。在这个例子中，输入的数据源就是前面定义的 `FixedBatchSpout`。输入数据源也可以是像 Kestrel 和 Kafka 这样的消息系统。Trident 会通过 ZooKeeper 一直跟踪每个输入数据源的一小部分状态（Trident 具体消费对象的相关元数据）。例如这里的 “spout1” 就对应着 ZooKeeper 中的一个节点，而 Trident 就会在该节点中存放数据源的元数据（metadata）。

Trident 会将数据流处理为很多个小块 tuple 的集合，例如，输入的句子流就会像下面这样被分割成很多个小块：

![batches][1]

这些小块的大小主要取决于你的输入吞吐量，一般可能会在数万甚至数百万元组的级别。

Trident 为这些小块提供了一个完全成熟的批处理 API。这个 API 和你见到过的 Pig 或者 Cascading 这样的 Hadoop 的高级抽象语言很相似：你可以处理分组（group by）、联结（join）、聚合（aggregation）、函数（function）、过滤器（filter）等各种操作。当然，分别处理每个小块并不是件好事，所以，Trident 提供了适用于处理各个小块之间的聚合操作的函数，并且可以在聚合后将结果保存到持久化存储中，而且无论是内存、Memcached、Cassandra 还是其他类型的存储都可以支持。最后，Trident 还提供了用于查询实时状态结果的一级接口。而这个结果状态既可以像这个例子中演示的那样由 Trident 负责更新，也可以作为一个独立的状态数据源而存在。

再回到这个例子中，输入数据源 spout 发送出了一个名为 “sentence” 的数据流。接下来拓扑中定义了一个 `Split` 方法用于处理流中的每个 tuple，这个方法接收 “sentence” 域并将其分割成若干个单词。每个 sentence tuple 都会创建很多个单词 tuple —— 例如 “the cow jumped over the moon” 这个句子就会创建 6 个 “word” tuple，下面是 `Split` 的定义：

```java
public class Split extends BaseFunction {
   public void execute(TridentTuple tuple, TridentCollector collector) {
       String sentence = tuple.getString(0);
       for(String word: sentence.split(" ")) {
           collector.emit(new Values(word));                
       }
   }
}
```

从上面的代码中你会发现这个过程真的很简单。这个方法中的所有操作仅仅是抓取句子、以空格分隔句子并且为每个单词发射一个 tuple。

拓扑的剩余部分负责统计单词的数量并将结果保存到持久化存储中。首先，数据流根据 “word” 域分组，然后使用 `Count` 聚合器持续聚合每个小组。`persistentAggregate` 方法用于存储并更新 state 源中的聚合结果。在这个例子中，单词的数量结果是保存在内存中的，不过可以根据需要切换到 Memcached、Cassandra 或者其他持久化存储中。切换存储模型也非常简单，只需要像下面这样（使用 [trident-memcached][2] 修改 `persistentAggregate` 行中的一个参数（其中，“serverLocations” 是 Memcached 集群的地址/端口列表）即可：

```java
.persistentAggregate(MemcachedState.transactional(serverLocations), new Count(), new Fields("count"))
```

`persistentAggregate` 方法所存储的值就表示所有从数据流中发送出来的块的聚合结果。

Trident 的另一个很酷的特性就是它支持完全容错性和恰好一次处理的语义。如果处理过程中出现错误需要重新执行处理操作，Trident 不会向数据库中提交多次来自相同的源数据的更新操作，这就是 Trident 持久化 state 的方式。

`persistentAggregate` 方法也可以将数据流结果传入一个 `TridentState` 对象中。这种情况下，这个 `TridentState` 就表示所有的单词统计信息。这样我们就可以使用 `TridentState` 对象来实现整个计算过程中的分布式查询部分。

接下来我们就可以在拓扑中实现 word count 的一个低延时分布式查询。这个查询接收一个由空格分隔的单词列表作为参数，然后返回这些单词的数量统计结果。这个查询看上去与普通的 RPC 调用并没有什么分别，不过在后台他们是并发执行的。下面是一个实现这种查询的例子：

```java
DRPCClient client = new DRPCClient("drpc.server.location", 3772);
System.out.println(client.execute("words", "cat dog the man");
// prints the JSON-encoded result, e.g.: "[[5078]]"
```

如你所见，这个查询看上去只是一个普通的远程过程调用（RPC），不过在后台他是在一个 Storm 集群中并发执行的。这种查询的端到端延时一般在 10 ms 左右。当然，更大量的查询会花费更长的时间，尽管这些查询还是取决于你为这个计算过程分配了多少时间。

拓扑中的分布式查询的实现是这样的：

```java
topology.newDRPCStream("words")
       .each(new Fields("args"), new Split(), new Fields("word"))
       .groupBy(new Fields("word"))
       .stateQuery(wordCounts, new Fields("word"), new MapGet(), new Fields("count"))
       .each(new Fields("count"), new FilterNull())
       .aggregate(new Fields("count"), new Sum(), new Fields("sum"));
```

这里还需要使用前面的 `TridentTopology` 对象来创建一个 DRPC 数据流，这个创建数据流的方法叫做 “words”。前面使用 `DRPCClient` 进行 RPC 调用的第一个参数必须与这个方法名完全相同。

在这段代码里，首先是使用 `Split` 方法来将请求的参数分割成若干个单词。这些单词构成的单词流是通过 “word” 域来分组的，而 `stateQuery` 运算符就是用来查询拓扑中第一个部分中生成的 `TridentState` 对象的。`stateQuery` 接收一个 state（在这个例子中就是拓扑前面计算得到的单词数结果）和查询这个 state 的方法作为参数。在这个例子里，`stateQuery` 调用了 `MapGet` 方法，用于获取每个单词的个数。由于 DRPC 数据流是和 TridentState 采用的完全相同的方式进行分组的（通过 “word” 域），每个单词查询都可以精确地定位到 TridentState 对象中的指定部分，同时 TridentState 对象中维护着对应的单词的更新状态。

接下来，个数为 0 的单词会被 `FilterNull` 过滤器过滤掉，然后就可以使用 `Sum` 聚合器来获取其他的单词统计个数。接着 Trident 就会自动将结果返回给等待的客户端。

Trident 很聪明，它知道怎么以最好的性能运行拓扑。在这个拓扑中还有两个会自动发生的有趣的事：

1. 从 state 中读取或写入的操作（例如 persistentAggregate 和 stateQuery）会自动批处理化。因此，如果当前的批处理过程需要对数据库执行 20 个更新操作，Trident 就会自动将读取或写入操作当作批处理过程，仅仅会对数据库发送一次读请求和一次写请求，而不是发送 20 次读请求和 20 次写请求（而且一般你还可以在你的 state 里使用缓存来消除读请求）。这样做就有两个方面的好处：可以按照你指定的方式来执行你的计算过程，同时还可以维持较好的性能。

2. Trident 的聚合器是高度优化的。在向网络中发送 tuple 之前，Trident 有时候会做部分聚合操作，而不是将一个分组的所有的 tuple 一股脑地发送到同一台机器中来执行聚合。例如，`Count` 聚合器就是这样先计算每个小块的个数，然后向网络中发送很多个部分计数的结果，接着再将所有的部分计数结果汇总来得到最终的统计结果。这个技术与 MapReduce 的 combiner 模型很相似。

我们再来看看 Trident 的另一个例子。

## Reach

这个例子是一个纯粹的 DRPC 拓扑，计算了一个指定 URL 的 Reach 数。Reach 指的是 Twitter 上能够看到一个指定的 URL 的独立用户数。要想计算 Reach，你需要先提取所有转发了该 URL 的用户，提取这些用户的关注者，将关注者放入一个 set 集合中来去除重复的关注者，然后再统计这个 set 中的数量。对于单一的一台机器来说，计算 reach 太耗时了，这个过程大概需要数千次数据库调用并生成数千万 tuple。而使用 Storm 和 Trident 就可以通过一个集群来将计算过程的每个步骤进行并行化处理。

这个拓扑会从两个 state 源中读取数据。其中一个数据库建立了 URL 和转发了该 URL 的用户列表的关联表。另一个数据库中建立了用户和用户的关注者列表的关联表。拓扑的定义是这样的：

```java
TridentState urlToTweeters =
       topology.newStaticState(getUrlToTweetersState());
TridentState tweetersToFollowers =
       topology.newStaticState(getTweeterToFollowersState());

topology.newDRPCStream("reach")
       .stateQuery(urlToTweeters, new Fields("args"), new MapGet(), new Fields("tweeters"))
       .each(new Fields("tweeters"), new ExpandList(), new Fields("tweeter"))
       .shuffle()
       .stateQuery(tweetersToFollowers, new Fields("tweeter"), new MapGet(), new Fields("followers"))
       .parallelismHint(200)
       .each(new Fields("followers"), new ExpandList(), new Fields("follower"))
       .groupBy(new Fields("follower"))
       .aggregate(new One(), new Fields("one"))
       .parallelismHint(20)
       .aggregate(new Count(), new Fields("reach"));
```

这个拓扑使用 `newStaticState` 方法创建了两个分别对应外部于两个外部数据库的 `TridentState` 对象。在拓扑的后续部分就可以对这两个 `TridentState` 对象执行查询操作。和 state 的所有数据源一样，为了最大程度地提升效率，对这些数据库的查询将会自动地批处理化。

拓扑的定义很直接 —— 就是一个简单的批处理 job。首先，会通过查询 urlToTweeters 数据库来获取转发了 URL 的用户列表，然后就可以调用 `ExpandList` 方法来为每个 tweeter 创建一个 tuple。

接下来必须要获取每个 tweeter 的关注者。由于需要调用 shuffle 方法将所有的 tweeter 均衡分配到拓扑的所有 worker 中，所以这个步骤必须并发进行，这一点非常重要。然后就可以查询关注者数据库来获取每个 tweeter 的关注者列表。你可能注意到了这个过程的并行度非常高，因为这是整个计算过程中复杂度最高的部分。

再接下来，关注者就会被放入一个单独的 set 集合中用于计数。这里包含两个步骤。首先，会根据 “follower” 域来执行 “group by” 分组操作，并在每个组上运行 `One` 聚合器。“One”聚合器的作用仅仅是为每个组发送一个包含数字 1 的 tuple。然后，就可以通过统计这些 one 结果来得到关注者 set 的大小，也就是真正的关注者数量。下面是 “One” 聚合器的定义：

```java
public class One implements CombinerAggregator<Integer> {
   public Integer init(TridentTuple tuple) {
       return 1;
   }

   public Integer combine(Integer val1, Integer val2) {
       return 1;
   }

   public Integer zero() {
       return 1;
   }        
}
```

这是一个“组合聚合器”，它知道怎样在向网络中发送 tuple 之前以最好的效率进行部分聚合操作。同样，Sum 也是一个组合聚合器，所以在拓扑结尾的全局统计操作也会有很高的效率。

下面让我们再来看看 Trident 中的一些细节。

## 域（Fields）与元组（tuples）

Trident 的数据模型 TridentTuple 是一个指定的值列表。在一个拓扑中，tuple 是在一系列操作中不断生成的。这些操作一般会输入一个“输入域”（input fields）集合，然后发送出一个“方法域”（function fields）的集合。输入域主要用于选取一个 tuple 的子集作为操作的输入，而“方法域”主要用于为该操作的输出结果域命名。

我们来看看这样一个场景。假设你有一个名为 “stream” 的数据流，其中包含域 “x”、“y” 和 “z”。如果要运行一个接收 “y” 作为输入的过滤器 MyFilter，你可以这样写：

```java
stream.each(new Fields("y"), new MyFilter())
```

再假设 MyFilter 的实现是这样的：

```java
public class MyFilter extends BaseFilter {
   public boolean isKeep(TridentTuple tuple) {
       return tuple.getInteger(0) < 10;
   }
}
```

这样就会保留所有 “y” 域的值小于 10 的 tuple。MyFilter 输入的 TridentTuple 将会仅包含有 “y” 域。值得注意的是，Trident 可以在选取输入域时以一种非常高效的方式来投射 tuple 的子集：这个投射过程非常灵活。

我们再来看看 “function fields” 是怎么工作的。假设你有这样一个函数：

```java
public class AddAndMultiply extends BaseFunction {
   public void execute(TridentTuple tuple, TridentCollector collector) {
       int i1 = tuple.getInteger(0);
       int i2 = tuple.getInteger(1);
       collector.emit(new Values(i1 + i2, i1 * i2));
   }
}
```

这个函数接收两个数字作为输入，然后发送出两个新值：分别是两个数字的和和乘积。再假定你有一个包含 “x”、“y” 和 “z” 域的数据流，你可以这样使用这个函数：

```java
stream.each(new Fields("x", "y"), new AddAndMultiply(), new Fields("added", "multiplied"));
```

这个函数的输出增加了两个新的域。因此，这个 each 调用的输出 tuple 会包含 5 个域：“x”、“y” 、“z”、“added” 和 “multiplied”。其中 “added” 与 AddAndMultiply 的第一个输出值相对应，“multiplied” 和 AddAndMultiply 的第二个输出值相对应。

另一方面，通过聚合器，函数域也可以替换输入 tuple 的域。假如你有一个包含域 “val1” 和域 “val2” 的数据流，通过这样的操作：

```java
stream.aggregate(new Fields("val2"), new Sum(), new Fields("sum"))
```

就会使得输出数据流中只包含一个只带有 “sum” 的域的 tuple，这个 “sum” 域就代表了在哪个批处理块中所有的 “val2” 域的总和值。

通过数据流分组，输出就可以同时包含用于分组的域以及由聚合器发送的域。举个例子：

```java
stream.groupBy(new Fields("val1"))
     .aggregate(new Fields("val2"), new Sum(), new Fields("sum"))
```

这个操作就会使得输出同时包含域 “val1” 以及域 “sum”。

---

## State

实时计算的一个关键问题就在于如何管理状态（state），使得在失败与重试操作之后的更新过程仍然是幂等的。错误是不可消除的，所以在出现节点故障或者其他问题发生时批处理操作还需要进行重试。不过这里最大的问题就在于怎样执行一种合适的状态更新操作（不管是针对外部数据库还是拓扑内部的状态），来使得每个消息都能够被执行且仅仅被执行一次。

这个问题很麻烦，接下来的例子里面就有这样的问题。假如你正在对你的数据流做一个计数聚合操作，并且打算将计数结果存储到一个数据库中。如果你仅仅把计数结果存到数据库里就完事了的话，那么在你继续准备更新某个块的状态的时候，你没法知道到底这个状态有没有被更新过。这个数据块有可能在更新数据库的步骤上成功了，但在后续的步骤中失败了，也有可能先失败了，没有进行更新数据库的操作。你完全不知道到底发生了什么。

Trident 通过下面两件事情解决了这个问题：

1. 在 Trident 中为每个数据块标记了一个唯一的 id，这个 id 就叫做“事务 id”（transaction id）。如果数据块由于失败回滚了，那么它持有的事务 id 不会改变。
2. State 的更新操作是按照数据块的顺序进行的。也就是说，在成功执行完块 2 的更新操作之前，不会执行块 3 的更新操作。

基于这两个基础特性，你的 state 更新就可以实现恰好一次（exactly-once）的语义。与仅仅向数据库中存储计数不同，这里你可以以一个原子操作的形式把事务 id 和计数值一起存入数据库。在后续更新这个计数值的时候你就可以先比对这个数据块的事务 id。如果比对结果是相同的，那么就可以跳过更新操作 —— 由于 state 的强有序性，可以确定数据库中已经包含有当前数据块的值。而如果比对结果不同，就可以放心地更新计数值了。

当然，你不需要在拓扑中手动进行这个操作，操作逻辑已经在 State 中封装好了，这个过程会自动进行。同样的，你的 State 对象也不一定要实现事务 id 标记：如果你不想在数据库里耗费空间存储事务 id，你就不用那么做。在这样的情况下，State 会在出现失败的情形下保持“至少处理一次”的操作语义（这样对你的应用也是一件好事）。在[这篇文章][3]里你可以了解到更多关于如何实现 State 以及各种容错性权衡技术。

你可以使用任何一种你想要的方法来实现 state 的存储操作。你可以把 state 存入外部数据库，也可以保存在内存中然后在存入 HDFS 中（有点像 HBase 的工作机制）。State 也并不需要一直保存某个状态值。比如，你可以实现一个只保存过去几个小时数据并将其余的数据删除的 State。这是一个实现 State 的例子：[Memcached integration][4]。

## Trident 拓扑的运行

Trident 拓扑会被编译成一种尽可能和普通拓扑有着同样的运行效率的形式。只有在请求数据的重新分配（比如 groupBy 或者 shuffle 操作）时 tuple 才会被发送到网络中。因此，像下面这样的 Trident 拓扑：

![trident-topology][5]

就会被编译成若干个 spout/bolt：

![trident-to-spout-and-bolt][6]

## 总结

Trident 让实时计算变得非常简单。你已经看到了高吞吐量的数据流处理、状态操作以及低延时查询处理是怎样通过 Trident 的 API 来实现无缝结合的。总而言之，Trident 可以让你以一种更加自然，同时仍然保持着很好的性能的方式实现实时计算。


[1]: http://storm.apache.org/releases/0.9.6/images/batched-stream.png
[2]: https://github.com/nathanmarz/trident-memcached
[3]: http://storm.apache.org/documentation/Trident-state
[4]: https://github.com/nathanmarz/trident-memcached/blob/master/src/jvm/trident/memcached/MemcachedState.java
[5]: http://storm.apache.org/releases/0.9.6/images/trident-to-storm1.png
[6]: http://storm.apache.org/releases/0.9.6/images/trident-to-storm2.png
