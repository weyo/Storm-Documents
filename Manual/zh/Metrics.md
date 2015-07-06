# Metrics

Storm 提供了一个可以获取整个拓扑中所有的统计信息的度量接口。Storm 内部通过该接口可以跟踪各类统计数字：executor 和 acker 的数量、每个 bolt 的平均处理时延、worker 使用的最大堆容量等等，这些信息都可以在 Nimbus 的 UI 界面中看到。

## Metric 类型

使用 Metrics 只需要实现一个接口方法：`getValueAndReset`，在方法中可以查找汇总值、并将该值复位为初始值。例如，在 MeanReducer 中就实现了通过运行总数除以对应的运行计数的方式来求取均值，然后将两个值都重新设置为 0。

Storm 提供了以下几种 metric 类型：

- [AssignableMetric][1] -- 将 metric 设置为指定值。此类型在两种情况下有用：1. metric 本身为外部设置的值；2. 你已经另外计算出了汇总的统计值。
- [CombinedMetric][2] -- 可以对 metric 进行关联更新的通用接口。
- [CountMetric][3] -- 返回 metric 的汇总结果。可以调用 `incr()` 方法来将结果加一；调用 `incrBy(n)` 方法来将结果加上给定值。
	- [MultiCountMetric][4] -- 返回包含一组 CountMetric 的 HashMap
- [ReducedMetric][5]
	- [MeanReducer][6] -- 跟踪由它的 `reduce()` 方法提供的运行状态均值结果（可以接受 `Double`、`Integer`、`Long` 等类型，内置的均值结果是 `Double` 类型）。MeanReducer 确实是一个相当棒的家伙。
	- [MultiReducedMetric][7] -- 返回包含一组 ReducedMetric 的 HashMap

## Metric Consumer

## 构建自定义 metric

## 内建的 Metric

[builtin_metrics.clj][8] 为内建的 metrics 设置了数据结构，以及其他框架组件可以用于更新的虚拟方法。metrics 本身是在回调代码中实现计算的 -- 请参考 `clj/b/s/daemon/daemon/executor.clj` 中的 `ack-spout-msg` 的例子。


[1]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/backtype/storm/metric/api/AssignableMetric.java
[2]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/backtype/storm/metric/api/CombinedMetric.java
[3]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/backtype/storm/metric/api/CountMetric.java
[4]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/backtype/storm/metric/api/MultiCountMetric.java
[5]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/backtype/storm/metric/api/ReducedMetric.java
[6]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/backtype/storm/metric/api/MeanReducer.java
[7]: https://github.com/apache/storm/blob/master/storm-core/src/jvm/backtype/storm/metric/api/MultiReducedMetric.java
[8]: https://github.com/apache/storm/blob/46c3ba7/storm-core/src/clj/backtype/storm/daemon/builtin_metrics.clj