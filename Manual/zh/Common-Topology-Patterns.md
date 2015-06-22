# Storm 常用模式

本文列出了 Storm 拓扑中使用的一些常见模式，包括：

1. 数据流的 join
2. 批处理
3. BasicBolt
4. 内存缓存与域分组的结合
5. Top N 流式计算
6. TimeCacheMap
7. CoordinatedBolt 与 KeyedFairBolt

## Joins

数据流的 join 一般指的是通过共有的域来聚合两个或多个数据流的过程。与一般的数据库中 join 操作要求有限的输入与清晰的语义不同，数据流 join 的输入往往是无限的数据集，而且并不具备明确的语义。

join 的类型一般是由应用的需求决定的。有些应用需要将两个流在某个固定时间内的所有 tuple 进行 join，另外一些应用却可能要求对每个 join 域的 join 操作过程的两侧只保留一个 tuple，而其他的应用也许还有一些其他需求。不过这些 join 类型一般都会有一个基本的模式，那就是将多个输入流进行分区。Storm 可以很容易地使用域分组的方法将多个输入流聚集到一个联结 bolt 中，比如下面这样：

```java
builder.setBolt("join", new MyJoiner(), parallelism)
  .fieldsGrouping("1", new Fields("joinfield1", "joinfield2"))
  .fieldsGrouping("2", new Fields("joinfield1", "joinfield2"))
  .fieldsGrouping("3", new Fields("joinfield1", "joinfield2"));
```

当然，上面的代码只是个例子，实际上不同的流完全可以具有不同的输入域。

## 批处理

通常由于效率或者其他方面的原因，你需要使用将 tuple 们组合成 batch 来处理，而不是一个个分别处理它们。比如，在做数据库更新操作或者流聚合操作时，你就会需要这样的批处理形式。

要确保数据处理的可靠性，正确的方式是在 bolt 进行批处理之前将 tuple 们缓存在一个实例变量中。在完成批处理操作之后，你就可以一起 ack 所有的缓存的 tuple 了。

如果这个批处理 bolt 还需要继续向下游发送 tuple，你可能还需要使用多锚定（multi-anchoring）来确保可靠性。具体怎么做取决于应用的需求。想要了解更多关于可靠性的工作机制的内容请参考[消息的可靠性保障][1]一文。

## BasicBolt

Bolt 处理 tuple 的一种基本模式是在 `execute` 方法中读取输入 tuple、发送出基于输入 tuple 的新 tuple，然后在方法末尾对 tuple 进行应答（ack）。符合这种模式的 bolt 一般是一种函数或者过滤器。对于这种基本的处理模式，Storm 提供了 `IBasicBolt` 接口来自动实现这个过程。更多内容请参考[消息的可靠性保障][1]一文。

## 内存缓存与域分组的结合

在 Storm 的 bolt 中保存一定的缓存也是一种比较常见的方式。尤其是在于域分组结合的时候，缓存的作用特别显著。例如，假如你有一个用于将短链接（short URLs，例如 bit.ly, t.co，等等）转化成长链接（龙 URLs）的 bolt。你可以通过一个将短链接映射到长链接的 LRU 缓存来提高系统的性能，避免反复的 HTTP 请求操作。假如现在有一个名为 “urls” 的组件用于发送短链接，另外有一个 “expand” 组件用于将短链接扩展为长链接，并且在 “expand” 内部保留一个缓存。让我们来看看下面两段代码有什么不同：

```java
builder.setBolt("expand", new ExpandUrl(), parallelism)
  .shuffleGrouping(1);
```

```java
builder.setBolt("expand", new ExpandUrl(), parallelism)
  .fieldsGrouping("urls", new Fields("url"));
```

由于域分组可以使得相同的 URL 永远被发往同一个 task，第二段代码会比第一段代码高效得多。这样可以避免在不同的 task 的缓存中的复制动作，并且看上去短 URL 可以更好地在命中缓存。

## Top N

Storm 中一种常见的连续计算模式是计算数据流中某种形式的 Top N 结果。假如现在有一个可以以 ["value", "count"] 的形式发送 tuple 的 bolt，并且你需要一个可以根据 count 计算结果输出前 N 个 tuple 的 bolt。实现这个操作的最简单的方法就是使用一个对数据流进行全局分组的 bolt，并且在内存中维护一个包含 top N 结果的列表。

这种方法并不适用于大规模数据流，因为整个数据流都会发往同一个 task，会造成该 task 的内存负载过高。更好的做法是将数据流分区，同时对每个分区计算 top N 结果，然后将这些结果汇总来得到最终的全局 top N 结果。下面是这个模式的代码：

```java
builder.setBolt("rank", new RankObjects(), parallelism)
  .fieldsGrouping("objects", new Fields("value"));
builder.setBolt("merge", new MergeObjects())
  .globalGrouping("rank");
```

这个方法之所以可行是因为第一个 bolt 的域分组操作确保了每个小分区在语义上的正确性。你可以在 [storm-starter][2] 里看到使用这个模式的一个例子。

当然，如果待处理的数据集存在较严重的数据倾斜，那么还是应该使用 partialKeyGrouping 来代替 fieldsGrouping，因为 partialKeyGrouping 可以通过两个下游 bolt 分散每个 key 的负载。

```java
builder.setBolt("count", new CountObjects(), parallelism)
  .partialKeyGrouping("objects", new Fields("value"));
builder.setBolt("rank" new AggregateCountsAndRank(), parallelism)
  .fieldsGrouping("count", new Fields("key"))
builder.setBolt("merge", new MergeRanksObjects())
  .globalGrouping("rank");
```

这个拓扑中需要一个中间层来聚合来自上游 bolt 数据流的分区计数结果，但这一层仅仅会做一个简单的聚合处理，这样 bolt 就不会受到由于数据倾斜带来的负载压力。你可以在 [storm-starter][3] 中看到使用这个模式的一个例子。

## 支持 LRU 的 TimeCacheMap

有时候你可能会需要一个能够保留“活跃的”数据并且能够使得超时的“非活跃的”数据自动失效的缓存。[TimeCacheMap][4] 是一个可以高效地实现此功能的数据结构。它还提供了一个钩子用于实现在数据失效后的回调操作。

## 用于分布式 RPC 的 CoordinatedBolt 与 KeyedFairBolt

在构建 Storm 上层的分布式 RPC 应用时，通常会用到两种常用的模式。现在这两种模式已经被封装为 [CoordinatedBolt][5] 和 [KeyedFairBolt][6]，并且已经加入了 Storm 标准库中。

`CoordinatedBolt` 将你的处理逻辑 bolt 包装起来，并且在你的 bolt 收到了指定请求的所有 tuple 之后发出通知。`CoordinatedBolt` 中大量使用了直接数据流组来实现此功能。

`KeyedFairBolt` 同样包装了你的处理逻辑 bolt，并且可以让你的拓扑同时处理多个 DRPC 调用，而不是每次只执行一个。

如果需要了解更多内容请参考[分布式RPC][7]一文。


[1]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Guaranteeing-Message-Processing.md
[2]: https://github.com/apache/storm/blob/master/examples/storm-starter/src/jvm/storm/starter/RollingTopWords.java
[3]: https://github.com/apache/storm/blob/master/examples/storm-starter/src/jvm/storm/starter/SkewedRollingTopWords.java
[4]: http://storm.apache.org/javadoc/apidocs/backtype/storm/utils/TimeCacheMap.html
[5]: http://storm.apache.org/javadoc/apidocs/backtype/storm/task/CoordinatedBolt.html
[6]: http://storm.apache.org/javadoc/apidocs/backtype/storm/task/KeyedFairBolt.html
[7]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Distributed-RPC.md