# Trident State

Trident 中含有对状态化（stateful）的数据源进行读取和写入操作的一级抽象封装工具。这个所谓的状态（state）既可以保存在拓扑内部（保存在内存中并通过 HDFS 来实现备份），也可以存入像 Memcached 或者 Cassandra 这样的外部数据库中。而对于 Trident API 而言，这两种机制并没有任何区别。

Trident 使用一种容错性的方式实现对 state 的管理，这样，即使在发生操作失败或者重试的情况下状态的更新操作仍然是幂等的。基于这个机制，每条消息都可以看作被恰好处理了一次，然后你就可以很容易地推断出 Trident 拓扑的状态。

State 的更新过程支持多级容错性保证机制。在讨论这一点之前，我们先来看一个例子，这个例子展示了如何实现恰好一次的语义的技术。假定你正在对数据流进行一个计数聚合操作，并打算将计数结果存入数据库中。在这个例子里，你存入数据库的就是一个对应计数结果的值，每次处理新 tuple 的时候就会增加这个值。

考虑到可能存在的处理失败情况，tuple 有可能需要重新处理。这样就给 state 的更新操作带来了一个问题（或者其他的副作用）—— 你无法知道当前的这个 tuple 的更新操作是否已经处理过了。也许你之前没有处理过这个 tuple，那么你现在就需要增加计数结果；也许你之前已经处理过 tuple 了并且成功地增加了计数结果，但是在后续操作过程中 tuple 的处理失败了，并由此引发了 tuple 的重新处理操作，这时你就不能再增加计数结果了；还有可能你之前在使用这个 tuple 更新数据库的时候出错了，也就是说计数值的更新操作并未成功，此时在 tuple 的重新处理过程中你仍然需要更新数据库。

所以说，如果只是向数据库中简单地存入计数值，你确实无法知道 tuple 是否已经被处理过。因此，你需要一些更多的信息来做决定。Trident 提供了一种支持恰好一次处理的语义，如下所述：

1. 通过小数据块（batch）的方式来处理 tuple（可以参考[Trident 教程][1]一文）
2. 为每个 batch 提供一个唯一的 id，这个 id 称为 “事务 id”（transaction id，txid）。如果需要对 batch 重新处理，这个 batch 上仍然会赋上相同的 txid。
3. State 的更新操作是按照 batch 的顺序进行的。也就是说，在 batch 2 完成处理之前，batch 3 的状态更新操作不会进行。

基于这几个基本性质，你的 State 的实现就可以检测到 tuple 的 batch 是否已经被处理过，并根据检测结果选择合适的 state 更新操作。你具体采用的操作取决于你的输入 spout 提供的语义，这个语义对每个 batch 都是有效的。有三类支持容错性的 spout：“非事务型”（non-transactional）、“事务型”（transactional）以及“模糊事务型”（opaque transactional）。接下来我们来分析下每种 spout 类型的容错性语义。

## 事务型 spout（Transactional spouts）

记住一点，Trident 是通过小数据块（batch）的方式来处理 tuple 的，而且每个 batch 都会有一个唯一的 txid。spout 的特性是由他们所提供的容错性保证机制决定的，而且这种机制也会对每个 batch 发生作用。事务型 spout 包含以下特性：

1. 每个 batch 的 txid 永远不会改变。对于某个特定的 txid，batch 在执行重新处理操作时所处理的 tuple 集和它的第一次处理操作完全相同。
2. 不同 batch 中的 tuple 不会出现重复的情况（某个 tuple 只会出现在一个 batch 中，而不会同时出现在多个 batch 中）。
3. 每个 tuple 都会放入一个 batch 中（处理操作不会遗漏任何的 tuple）。

这是一种很容易理解的 spout，其中的数据流会被分解到固定的 batches 中。Storm-contrib 项目中提供了一种基于 Kafka 的[事务型 spout 实现][2]。

看到这里，你可能会有这样的疑问：为什么不在拓扑中完全使用事务型 spout 呢？这个原因很好理解。一方面，有些时候事务型 spout 并不能提供足够可靠的容错性保障，所以不需要使用事务型 spout。比如，`TransactionalTridentKafkaSpout` 的工作方式就是使得带有某个 txid 的 batch 中包含有来自一个 Kafka topic 的所有 partition 的 tuple。一旦一个 batch 被发送出去，在将来无论重新发送这个 batch 多少次，batch 中都会包含有完全相同的 tuple 集，这是由事务型 spout 的语义决定的。现在假设 `TransactionalTridentKafkaSpout` 发送出的某个 batch 处理失败了，而与此同时，Kafka 的某个节点因为故障下线了。这时你就无法重新处理之前的 batch 了（因为 Kafka 的节点故障，Kafka topic 必然有一部分 partition 无法获取到），这个处理过程也会因此终止。

这就是要有“模糊事务型” spout 的原因了 —— 模糊事务型 spout 支持在数据源节点丢失的情况下仍然可以实现恰好一次的处理语义。我们会在下一节讨论这类 spout。

顺便提一点，如果 Kafka 支持数据复制，那么就可以放心地使用事务型 spout 提供的容错性机制了，因为这种情况下某个节点的故障不会导致数据丢失，不过 Kafka 暂时还不支持该特性。（本文的写作时间应该较早，Kakfa 早就已经可以支持复制的机制了 —— 译者注）。

在讨论“模糊事务型” spout 之前，让我们先来看看如何为事务型 spout 设计一种支持恰好一次语义的 State。这个 State 就称为 “事务型 state”，它支持对于特定的 txid 永远只与同一组 tuple 相关联的特性。

假如你的拓扑需要计算单词数，而且你准备将计数结果存入一个 K-V 型数据库中。这里的 key 就是单词，value 对应于单词数。从上面的讨论中你应该已经明白了仅仅存储计数结果是无法确定某个 batch 中的tuple 是否已经被处理过的。所以，现在你应该将 txid 作为一种原子化的值与计数值一起存入数据库。随后，在更新计数值的时候，你就可以将数据库中的 txid 与当前处理的 batch 的 txid 进行比对。如果两者相同，你就可以跳过更新操作 —— 由于 Trident 的强有序性处理机制，可以确定数据库中的值是对应于当前的 batch 的。如果两者不同，你就可以放心地增加计数值。由于一个 batch 的 txid 永远不会改变，而且 Trident 能够保证 state 的更新操作完全是按照 batch 的顺序进行的，所以，这样的处理逻辑是完全可行的。

下面来看一个例子。假如你正在处理 txid 3，其中包含有以下几个 tuple：

```
["man"]
["man"]
["dog"]
```

假如数据库中有以下几个 key-value 对：

```
man => [count=3, txid=1]
dog => [count=4, txid=3]
apple => [count=10, txid=2]
```

其中与 “man” 相关联的 txid 为 1。由于当前处理的 txid 为 3，你就可以确定当前处理的 batch 与数据库中存储的值无关，这样你就可以放心地将 “man” 的计数值加上 2 并更新 txid 为 3。另一方面，由于 “dog” 的 txid 与当前的 txid 相同，所以，“dog” 的计数是之前已经处理过的，现在不能再对数据库中的计数值进行更新操作。这样，在结束 txid3 的更新操作之后，数据库中的结果就会变成这样：

```
man => [count=5, txid=3]
dog => [count=4, txid=3]
apple => [count=10, txid=2]
```

现在我们再来讨论一下“模糊事务型” spout。

## 模糊事务型 spout（Opaque transactional spouts）

前面已经提到过，模糊事务型 spout 不能保证一个 txid 对应的 batch 中包含的 tuple 完全一致。模糊事务型 spout 有以下的特性：

1. 每个 tuple 都会通过某个 batch 处理完成。不过，在 tuple 处理失败的时候，tuple 有可能继续在另一个 batch 中完成处理，而不一定是在原先的 batch 中完成处理。

[OpaqueTridentKafkaSpout][3] 就具有这样的特性，同时它对 Kafka 节点的丢失问题具有很好的容错性。`OpaqueTridentKafkaSpout` 在发送一个 batch 的时候总会总上一个 batch 结束的地方开始发送新 tuple。这一点可以保证 tuple 不会被遗漏，而且也不会被多个 batch 处理。

不过，模糊事务型 spout 的缺点就在于不能通过 txid 来识别数据库中的 state 是否是已经处理过的。这是因为在 state 的更新的过程中，batch 有可能会发生变化。

在这种情况下，你应该在数据库中存储更多的 state 信息。除了一个结果值和 txid 之外，你还应该存入前一个结果值。我们再以上面的计数值的例子来分析以下这个问题。假如你的 batch 的部分计数值是 “2”，现在你需要应用一个更新操作。假定现在数据库中的值是这样的：

```
{ value = 4,
  prevValue = 1,
  txid = 2
}
```

- 情形1：假如当前处理的 txid 为 3，这与数据库中的 txid 不同。这时可以将 “prevValue” 的值设为 “value” 的值，再为 “value” 的值加上部分计数的结果并更新 txid。执行完这一系列操作之后的数据库中的值就会变成这样：

```
{ value = 6,
  prevValue = 4,
  txid = 3
}
```

- 情形2：如果当前处理的 txid 为 2，也就是和数据库中存储的 txid 一致，这种情况下的处理逻辑与上面的 txid 不一致的情况又有所不同。因为此时你会知道数据库中的更新操作是由上一个拥有相同 txid 的batch 做出的。不过那个 batch 有可能与当前的 batch 并不相同，所以你需要忽略它的操作。这个时候，你应该将 “prevValue” 加上 batch 中的部分计数值来计算新的 “value”。在这个操作之后数据库中的值就会变成这样：

```
{ value = 3,
  prevValue = 1,
  txid = 2
}
```

这种方法之所以可行是因为 Trident 具有强顺序性处理的特性。一旦 Trident 开始处理一个新的 batch 的状态更新操作，它永远不会回到过去的 batch 的处理上。同时，由于模糊事务型 spout 会保证 batch 之间不会存在重复 —— 每个 tuple 只会被某一个 batch 完成处理 —— 所以你可以放心地使用 prevValue 来更新 value。

## 非事务型 spout（Non-transactional spouts）

非事务型 spout 不能为 batch 提供任何的安全性保证。非事务型 spout 有可能提供一种“至多一次”的处理模型，在这种情况下 batch 处理失败后 tuple 并不会重新处理；也有可能提供一种“至少一次”的处理模型，在这种情况下可能会有多个 batch 分别处理某个 tuple。总之，此类 spout 不能提供“恰好一次”的语义。

## 不同类型的 Spout 与 State 的总结

下图显示了不同的 spout/state 的组合是否支持恰好一次的消息处理语义：

![spout-state][4]

模糊事务型 state 具有最好的容错性特征，不过这是以在数据库中存储更多的内容为代价的（一个 txid 和两个 value）。事务型 state 要求的存储空间相对较小，但是它的缺点是只对事务型 spout 有效。相对的，非事务型要求的存储空间最少，但是它也不能提供任何的恰好一次的消息执行语义。

你选择 state 与 spout 的时候必须在容错性与存储空间占用之间权衡。可以根据你的应用的需求来确定哪种组合最适合你。

## State API

从上文的描述中你已经了解到了恰好一次的消息执行语义的原理是多么的复杂。不过作为用户你并不需要处理这些复杂的 txid 比对、多值存储等操作，Trident 已经在 State 中封装了所有的容错性处理逻辑，你只需要像下面这样写代码即可：

```java
TridentTopology topology = new TridentTopology();        
TridentState wordCounts =
      topology.newStream("spout1", spout)
        .each(new Fields("sentence"), new Split(), new Fields("word"))
        .groupBy(new Fields("word"))
        .persistentAggregate(MemcachedState.opaque(serverLocations), new Count(), new Fields("count"))                
        .parallelismHint(6);
```

所有处理模糊事务型 state 的逻辑已经封装在 `MemcachedState.opaque` 的调用中了。另外，状态更新都会自动调整为批处理操作，这样可以减小与数据库的反复交互的资源损耗。

基本的 `State` 接口只有两个方法：

```java
public interface State {
    void beginCommit(Long txid); // 对于类似于在 DRPC 流上进行 partitionPersist 的操作，此方法可以为空
    void commit(Long txid);
}
```

前面已经说过，state 更新操作的开始时和结束时都会获取一个 txid。对于你的 state 怎么工作，你在其中使用什么样的方法执行更新操作，或者使用什么样的方法从 state 中读取数据，Trident 并不关心。

假如你有一个包含有用户的地址信息的定制数据库，你需要使用 Trident 与该数据库交互。你的 State 的实现就会包含有用于获取与设置用户信息的方法，比如下面这样：

```java
public class LocationDB implements State {
    public void beginCommit(Long txid) {    
    }

    public void commit(Long txid) {    
    }

    public void setLocation(long userId, String location) {
      // code to access database and set location
    }

    public String getLocation(long userId) {
      // code to get location from database
    }
}
```

接着你就可以为 Trident 提供一个 StateFactory 来创建 Trident 任务内部的 State 对象的实例。对应于你的数据库（LocationDB）的 StateFactory 大概是这样的：

```java
public class LocationDBFactory implements StateFactory {
   public State makeState(Map conf, int partitionIndex, int numPartitions) {
      return new LocationDB();
   } 
}
```

Trident 提供了一个用于查询 state 数据源的 `QueryFunction` 接口，以及一个用于更新 state 数据源的 `StateUpdater` 接口。例如，我们可以写一个查询 LocationDB 中的用户地址信息的 “QueryLocation”。让我们从你在拓扑中使用这个操作的方式开始。假如在拓扑中需要读取输入流中的 userid 信息：

```java
TridentTopology topology = new TridentTopology();
TridentState locations = topology.newStaticState(new LocationDBFactory());
topology.newStream("myspout", spout)
        .stateQuery(locations, new Fields("userid"), new QueryLocation(), new Fields("location"))
```

这里的 `QueryLocation` 的实现可能是这样的：

```java
public class QueryLocation extends BaseQueryFunction<LocationDB, String> {
    public List<String> batchRetrieve(LocationDB state, List<TridentTuple> inputs) {
        List<String> ret = new ArrayList();
        for(TridentTuple input: inputs) {
            ret.add(state.getLocation(input.getLong(0)));
        }
        return ret;
    }

    public void execute(TridentTuple tuple, String location, TridentCollector collector) {
        collector.emit(new Values(location));
    }    
}
```

`QueryFunction` 的执行包含两个步骤。首先，Trident 会将读取的一些数据中汇总为一个 batch 传入 batchRetrieve 方法中。在这个例子中，batchRetrieve 方法会收到一些用户 id。然后 batchRetrieve 会返回一个与输入 tuple 列表大小相同的队列。结果队列的第一个元素与第一个输入 tuple 对应，第二个元素与第二个输入 tuple 相对应，以此类推。

你会发现这段代码并没有发挥出 Trident 批处理的优势，因为这段代码仅仅一次查询一下 LocationDB。所以，实现 LocationDB 的更好的方式应该是这样的：

```java
public class LocationDB implements State {
    public void beginCommit(Long txid) {    
    }

    public void commit(Long txid) {    
    }

    public void setLocationsBulk(List<Long> userIds, List<String> locations) {
      // set locations in bulk
    }

    public List<String> bulkGetLocations(List<Long> userIds) {
      // get locations in bulk
    }
}
```

然后，你可以这样实现 `QueryLocation` 方法：

```java
public class QueryLocation extends BaseQueryFunction<LocationDB, String> {
    public List<String> batchRetrieve(LocationDB state, List<TridentTuple> inputs) {
        List<Long> userIds = new ArrayList<Long>();
        for(TridentTuple input: inputs) {
            userIds.add(input.getLong(0));
        }
        return state.bulkGetLocations(userIds);
    }

    public void execute(TridentTuple tuple, String location, TridentCollector collector) {
        collector.emit(new Values(location));
    }    
}
```

这段代码大幅减少了域数据库的IO，具有更高的执行效率。

你需要使用 `StateUpdater` 接口来更新 state。下面是一个更新 LocationDB 的地址信息的 StateUpdater 实现：

```java
public class LocationUpdater extends BaseStateUpdater<LocationDB> {
    public void updateState(LocationDB state, List<TridentTuple> tuples, TridentCollector collector) {
        List<Long> ids = new ArrayList<Long>();
        List<String> locations = new ArrayList<String>();
        for(TridentTuple t: tuples) {
            ids.add(t.getLong(0));
            locations.add(t.getString(1));
        }
        state.setLocationsBulk(ids, locations);
    }
}
```

然后你就可以在 Trident 拓扑中这样使用这个操作：

```java
TridentTopology topology = new TridentTopology();
TridentState locations = 
    topology.newStream("locations", locationsSpout)
        .partitionPersist(new LocationDBFactory(), new Fields("userid", "location"), new LocationUpdater())
```

`partitionPersist` 操作会更新 state 数据源。`StateUpdater` 接收 State 和一批 tuple 作为输入，然后更新这个 State。上面的代码仅仅从输入 tuple 中抓取 userid 和 location 信息，然后对 State 执行一个批处理更新操作。

在 Trident 拓扑更新 LocationDB 之后，`partitionPersist` 会返回一个表示更新后状态的 `TridentState` 对象。随后你就可以在拓扑的其他地方使用 `stateQuery` 方法对这个 state 执行查询操作。

你也许注意到了 StateUpdater 中有一个 TridentCollector 参数。发送到这个 collector 的 tuple 会进入一个“新的数值流”中。在这个例子里向这个新的流发送 tuple 并没有意义，不过如果你需要处理类似于更新数据库中的计数值这样的操作，你可以考虑将更新后的技术结果发送到这个流中。可以通过 `TridentState.newValuesStream` 方法来获取新的流的数据。

## persistentAggregate

Trident 使用一个称为 `persistentAggregate` 的方法来更新 State。你已经在前面的数据流单词统计的例子里见过了这个方法，这里再写一遍：

```java
TridentTopology topology = new TridentTopology();        
TridentState wordCounts =
      topology.newStream("spout1", spout)
        .each(new Fields("sentence"), new Split(), new Fields("word"))
        .groupBy(new Fields("word"))
        .persistentAggregate(new MemoryMapState.Factory(), new Count(), new Fields("count"))
```

partitionPersist 是一个接收 Trident 聚合器作为参数并对 state  数据源进行更新的方法，persistentAggregate 就是构建于 partitionPersist 上层的一个编程抽象。在这个例子里，由于是一个分组数据流（grouped stream），Trident 需要你提供一个实现 `MapState` 接口的 state。被分组的域就是 state 中的 key，而聚合的结果就是 state 中的 value。`MapState` 接口是这样的：

```java
public interface MapState<T> extends State {
    List<T> multiGet(List<List<Object>> keys);
    List<T> multiUpdate(List<List<Object>> keys, List<ValueUpdater> updaters);
    void multiPut(List<List<Object>> keys, List<T> vals);
}
```

而当你在非分组数据流上执行聚合操作时（全局聚合操作），Trident 需要你提供一个实现了 `Snapshottable` 接口的对象：

```java
public interface Snapshottable<T> extends State {
    T get();
    T update(ValueUpdater updater);
    void set(T o);
}
```

[MemoryMapState][5] 与 [MemcachedState][6] 都实现了上面两个接口。

## 实现 Map State 接口

实现 `MapState` 接口非常简单，Trident 几乎已经为你做好了所有的准备工作。`OpaqueMap`、`TransactionalMap`、与 `NonTransactionalMap` 类都分别实现了各自的容错性语义。你只需要为这些类提供一个用于对不同的 key/value 进行 multiGets 与 multiPuts 处理的 IBackingMap 实现类。`IBackingMap` 接口是这样的：

```java
public interface IBackingMap<T> {
    List<T> multiGet(List<List<Object>> keys); 
    void multiPut(List<List<Object>> keys, List<T> vals); 
}
```

OpaqueMap 会使用 [OpaqueValue][7] 作为 vals 参数来调用 multiPut 方法，TransactionalMap 会使用 [TransactionalValue][8] 作为参数，而 NonTransactionalMap 则直接将拓扑中的对象传入。

Trident 也提供了一个 [CachedMap][9] 用于实现 K-V map 的自动 LRU 缓存功能。

最后，Trident 还提供了一个 [SnapshottableMap][10] 类，该类通过将全局聚合结果存入一个固定的 key 中的方法将 MapState 对象转化为一个 Snapshottable 对象。

可以参考 [MemcachedState][11] 的实现来了解如何将这些工具结合到一起来提供一个高性能的 MapState。`MemcachedState` 支持选择模糊事务型、事务型或者非事务型语义。


[1]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Trident-Tutorial.md
[2]: https://github.com/apache/storm/tree/master/external/storm-kafka/src/jvm/storm/kafka/trident/TransactionalTridentKafkaSpout.java
[3]: https://github.com/apache/storm/tree/master/external/storm-kafka/src/jvm/storm/kafka/trident/OpaqueTridentKafkaSpout.java
[4]: http://storm.apache.org/documentation/images/spout-vs-state.png
[5]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/storm/trident/testing/MemoryMapState.java
[6]: https://github.com/nathanmarz/trident-memcached/blob/master/src/jvm/trident/memcached/MemcachedState.java
[7]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/storm/trident/state/OpaqueValue.java
[8]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/storm/trident/state/TransactionalValue.java
[9]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/storm/trident/state/map/CachedMap.java
[10]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/storm/trident/state/map/SnapshottableMap.java
[11]: https://github.com/nathanmarz/trident-memcached/blob/master/src/jvm/trident/memcached/MemcachedState.java
