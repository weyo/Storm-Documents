# Trident API 概述

Trident 的核心数据模型是“流”（Stream），不过与普通的拓扑不同的是，这里的流是作为一系列数据块来处理的。数据流是分布在集群中的不同节点上运行的，并且对数据流的操作也是在数据流的每个小分区上并行运行的。

Trident 中有 5 类操作：

1. 针对每个小分区的本地操作，这类操作不会产生网络数据传输；
2. 针对一个数据流的重新分区操作，这类操作不会改变数据流中的内容，但是会产生一定的网络传输；
3. 通过网络数据传输进行的聚合操作；
4. 针对数据流的分组操作；
5. 融合与联结操作。

## 本地分区操作

本地分区操作是在每个分区块上独立运行的操作，其中不涉及网络数据传输。

### 函数

函数负责接收一个输入域的集合并选择输出或者不输出 tuple。输出 tuple 的域会被添加到原始数据流的输入域中。如果一个函数不输出 tuple，那么原始的输入 tuple 就会被直接过滤掉。否则，每个输出 tuple 都会复制一份输入 tuple 。假设你有下面这样的函数：

```java
public class MyFunction extends BaseFunction {
    public void execute(TridentTuple tuple, TridentCollector collector) {
        for(int i=0; i < tuple.getInteger(0); i++) {
            collector.emit(new Values(i));
        }
    }
}
```

再假设你有一个名为 “mystream” 的数据流，这个流中包含下面几个 tuple，每个 tuple 中包含有 “a”、“b”、“c” 三个域：

```
[1, 2, 3]
[4, 1, 6]
[3, 0, 8]
```

如果你运行这段代码：

```
mystream.each(new Fields("b"), new MyFunction(), new Fields("d")))
```

那么最终输出的结果 tuple 就会包含有 “a”、“b”、“c”、“d” 4 个域，就像下面这样：

```
[1, 2, 3, 0]
[1, 2, 3, 1]
[4, 1, 6, 0]
```

### 过滤器

过滤器负责判断输入的 tuple 是否需要保留。以下面的过滤器为例：

```java
public class MyFilter extends BaseFilter {
    public boolean isKeep(TridentTuple tuple) {
        return tuple.getInteger(0) == 1 && tuple.getInteger(1) == 2;
    }
}
```

通过使用这段代码：

```java
mystream.each(new Fields("b", "a"), new MyFilter())
```

就可以将下面这样带有 “a”、“b”、“c” 三个域的 tuple

```
[1, 2, 3]
[2, 1, 1]
[2, 3, 4]
```

最终转化成这样的结果 tuple：

```
[2, 1, 1]
```

### partitionAggregate

`partitionAggregate` 会在一批 tuple 的每个分区上执行一个指定的功能操作。与上面的函数不同，由 `partitionAggregate` 发送出的 tuple 会将输入 tuple 的域替换。以下面这段代码为例：

```
mystream.partitionAggregate(new Fields("b"), new Sum(), new Fields("sum"))
```

假如输入流中包含有 “a”、“b” 两个域并且有以下几个 tuple 块：

```
Partition 0:
["a", 1]
["b", 2]

Partition 1:
["a", 3]
["c", 8]

Partition 2:
["e", 1]
["d", 9]
["d", 10]
```

经过上面的代码之后，输出就会变成带有一个名为 “sum” 的域的数据流，其中的 tuple 就是这样的：

```
Partition 0:
[3]

Partition 1:
[11]

Partition 2:
[20]
```

Storm 有三个用于定义聚合器的接口：`CombinerAggregator`，`ReducerAggregator` 以及 `Aggregator`。

这是 `CombinerAggregator` 接口：

```java
public interface CombinerAggregator<T> extends Serializable {
    T init(TridentTuple tuple);
    T combine(T val1, T val2);
    T zero();
}
```

`CombinerAggregator` 会将带有一个域的一个单独的 tuple 返回作为输出。`CombinerAggregator` 会在每个输入 tuple 上运行初始化函数，然后使用组合函数来组合所有输入的值。如果在某个分区中没有 tuple， `CombinerAggregator` 就会输出 `zero` 方法的结果。例如，下面是 `Count` 的实现代码：

```java
public class Count implements CombinerAggregator<Long> {
    public Long init(TridentTuple tuple) {
        return 1L;
    }

    public Long combine(Long val1, Long val2) {
        return val1 + val2;
    }

    public Long zero() {
        return 0L;
    }
}
```

如果你使用 aggregate 方法来代替 partitionAggregate 方法，你就会发现 `CombinerAggregator` 的好处了。在这种情况下，Trident 会在发送 tuple 之前通过分区聚合操作来优化计算过程。

`ReducerAggregator` 的接口实现是这样的：

```java
public interface ReducerAggregator<T> extends Serializable {
    T init();
    T reduce(T curr, TridentTuple tuple);
}
```

`ReducerAggregator` 会使用 `init` 方法来产生一个初始化的值，然后使用该值对每个输入 tuple 进行遍历，并最终生成并输出一个单独的 tuple，这个 tuple 中就包含有我们需要的计算结果值。例如，下面是将 Count 定义为 `ReducerAggregator` 的代码：

```java
public class Count implements ReducerAggregator<Long> {
    public Long init() {
        return 0L;
    }

    public Long reduce(Long curr, TridentTuple tuple) {
        return curr + 1;
    }
}
```

`ReducerAggregator` 同样可以用于 persistentAggregate，你会在后面看到这一点。

最常用的聚合器接口还是下面的 `Aggregator` 接口：

```java
public interface Aggregator<T> extends Operation {
    T init(Object batchId, TridentCollector collector);
    void aggregate(T state, TridentTuple tuple, TridentCollector collector);
    void complete(T state, TridentCollector collector);
}
```

`Aggregator` 聚合器可以生成任意数量的 tuple，这些 tuple 也可以带有任意数量的域。聚合器可以在执行过程中的任意一点输出tuple，他们的执行过程是这样的：

1. 在处理一批数据之前先调用 init 方法。init 方法的返回值是一个代表着聚合状态的对象，这个对象接下来会被传入 aggregate 方法和 complete 方法中。
2. 对于一个区块中的每个 tuple 都会调用 aggregate 方法。这个方法能够更新状态并且有选择地输出 tuple。
3. 在区块中的所有 tuple 都被 aggregate 方法处理之后就会调用 complete 方法。

下面是使用 Count 作为聚合器的代码：

```java
public class CountAgg extends BaseAggregator<CountState> {
    static class CountState {
        long count = 0;
    }

    public CountState init(Object batchId, TridentCollector collector) {
        return new CountState();
    }

    public void aggregate(CountState state, TridentTuple tuple, TridentCollector collector) {
        state.count+=1;
    }

    public void complete(CountState state, TridentCollector collector) {
        collector.emit(new Values(state.count));
    }
}
```

有时你可能会需要同时执行多个聚合操作。这个过程叫做链式处理，可以使用下面这样的代码来实现：

```java
mystream.chainedAgg()
        .partitionAggregate(new Count(), new Fields("count"))
        .partitionAggregate(new Fields("b"), new Sum(), new Fields("sum"))
        .chainEnd()
```

这段代码会在每个分区上分别执行 Count 和 Sum 聚合器，而输出中只会包含一个带有 ["count", "sum"] 域的单独的 tuple。

### stateQuery 与 partitionPersist

stateQuery 与 partitionPersist 会分别查询、更新 state 数据源。你可以参考 [Trident state doc][1] 来了解如何使用它们。

### projection

`projection` 方法只会保留操作中指定的域。如果你有一个带有 ["a", "b", "c", "d"] 域的数据流，通过执行这段代码：

```java
mystream.project(new Fields("b", "d"))
```

就会使得输出数据流中只包含有 ["b", "d"] 域。

## 重分区操作

重分区操作会执行一个用来改变在不同的任务间分配 tuple 的方式的函数。在重分区的过程中分区的数量也可能会发生变化（例如，重分区之后的并行度就有可能会增大）。重分区会产生一定的网络数据传输。下面是重分区操作的几个函数：

1. shuffle：通过随机轮询算法来重新分配目标区块的所有 tuple。
2. broadcast：每个 tuple 都会被复制到所有的目标区块中。这个函数在 DRPC 中很有用 —— 比如，你可以使用这个函数来获取每个区块数据的查询结果。
3. partitionBy：该函数会接收一组域作为参数，并根据这些域来进行分区操作。可以通过对这些域进行哈希化，并对目标分区的数量取模的方法来选取目标区块。partitionBy 函数能够保证来自同一组域的结果总会被发送到相同的目标区间。
4. global：这种方式下所有的 tuple 都会被发送到同一个目标分区中，而且数据流中的所有的块都会由这个分区处理。
5. batchGlobal：同一个 batch 块中的所有 tuple 会被发送到同一个区块中。当然，在数据流中的不同区块仍然会分配到不同的区块中。
6. partition：这个函数使用自定义的分区方法，该方法会实现 `backtype.storm.grouping.CustomStreamGrouping` 接口。

## 聚类操作

Trident 使用 aggregate 方法和 persistentAggregate 方法来对数据流进行聚类操作。其中，aggregate 方法会分别对数据流中的每个 batch 进行处理，而 persistentAggregate 方法则会对数据流中的所有 batch 执行聚类处理，并将结果存入某个 state 中。

在数据流上执行 aggregate 方法会执行一个全局的聚类操作。在你使用 `ReducerAggregator` 或者 `Aggregator` 时，数据流首先会被重新分区成一个单独的分区，然后聚类函数就会在该分区上执行操作。而在你使用 `CombinerAggregator` 时，Trident 首先会计算每个分区的部分聚类结果，然后将这些结果重分区到一个单独的分区中，最后在网络数据传输完成之后结束这个聚类过程。`CombinerAggregator` 比其他的聚合器的运行效率更高，在聚类时应该尽可能使用 `CombinerAggregator`。

下面是一个使用 aggregate 来获取一个 batch 的全局计数值的例子：

```java
mystream.aggregate(new Count(), new Fields("count"))
```

与 partitionAggregate 一样，aggregate 的聚合器也可以进行链式处理。然而，如果你在一个处理链中同时使用了 `CombinerAggregator` 和非 `CombinerAggregator`，Trident 就不能对部分聚类操作进行优化了。

想要了解更多使用 persistentAggregate 的方法，可以参考 [Trident state doc][2] 一文。

## 对分组数据流的操作

通过对指定的域执行 partitionBy 操作，groupBy 操作可以将数据流进行重分区，使得相同的域的 tuple 分组可以聚集在一起。例如，下面是一个 groupBy 操作的示例：

![groupBy][3]

如果你在分组数据流上执行聚合操作，聚合器会在每个分组（而不是整个区块）上运行。persistentAggregate 同样可以在一个分组数据里上运行，这种情况下聚合结果会存储在 [MapState][4] 中，其中的 key 就是分组的域名。

和其他操作一样，对分组数据流的聚合操作也可以以链式的方式执行。

## 融合（Merge）与联结（join）

Trident API 的最后一部分是联结不同的数据流的操作。联结数据流最简单的方式就是将所有的数据流融合到一个流中。你可以使用 TridentTopology 的 merge 方法实现该操作，比如这样：

```java
topology.merge(stream1, stream2, stream3);
```

Trident 会将融合后的新数据流的域命名为为第一个数据流的输出域。

联结数据流的另外一种方法是使用 join。像 SQL 那样的标准 join 操作只能用于有限的输入数据集，对于无限的数据集就没有用武之地了。Trident 中的 join 只会应用于每个从 spout 中输出的小 batch。

下面是两个流的 join 操作的示例，其中一个流含有 ["key", "val1", "val2"] 域，另外一个流含有 ["x", "val1"] 域：

```java
topology.join(stream1, new Fields("key"), stream2, new Fields("x"), new Fields("key", "a", "b", "c"));
```

上面的例子会使用 “key” 和 “x” 作为 join 的域来联结 stream1 和 stream2。Trident 要求先定义好新流的输出域，因为输入流的域可能会覆盖新流的域名。从 join 中输出的 tuple 中会包含：

1. join 域的列表。在这个例子里，输出的 “key” 域与 stream1 的 “key” 域以及 stream2 的 “x” 域对应。
2. 来自所有流的非 join 域的列表。这个列表是按照传入 join 方法的流的顺序排列的。在这个例子里，“
a” 和 “b” 域与 stream1 的 “val1” 和 “val2” 域对应；而 “c” 域则与 stream2 的 “val1” 域相对应。

在对不同的 spout 发送出的流进行 join 时，这些 spout 上会按照他们发送 batch 的方式进行同步处理。也就是说，一个处理中的 batch 中含有每个 spout 发送出的 tuple。

到这里你大概仍然会对如何进行窗口 join 操作感到困惑。窗口操作（包括平滑窗口、滚动窗口等 —— 译者注）主要是指将当前的 tuple 与过去若干小时时间段内的 tuple 联结起来的过程。

你可以使用 partitionPersist 和 stateQuery 来实现这个过程。过去一段时间内的 tuple 会以 join 域为关键字被保存到一个 state 源中。然后就可以使用 stateQuery 查询 join 域来实现这个“联结”（join）的过程。


[1]: http://storm.apache.org/documentation/Trident-state.html
[2]: http://storm.apache.org/documentation/Trident-state.html
[3]: http://storm.apache.org/documentation/images/grouping.png
[4]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/storm/trident/state/map/MapState.java
