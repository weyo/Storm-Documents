# Ack 框架的实现

Storm 的 [acker](https://github.com/apache/storm/blob/46c3ba7/storm-core/src/clj/backtype/storm/daemon/acker.clj#L28) 使用哈希校验和来跟踪每个 tuple 树的完成情况：每个 tuple 在被发送出的时候，它的值会与校验和进行异或运算，然后在 tuple 被 ack 的时候这个值又会再次与校验和进行异或运算。这样，一旦所有的 tuple 都被成功 ack，校验和就会变为 0（随机生成的校验和为 0 的概率极小，可以忽略不计）。

你可以在 [wiki](https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Guaranteeing-Message-Processing.md#storm-的可靠性-api) 中了解更多关于可靠性机制的信息。

## acker `execute()`

Acker 实际上也是一个 bolt，它的 [execute 方法](https://github.com/apache/storm/blob/46c3ba7/storm-core/src/clj/backtype/storm/daemon/acker.clj#L36) 是定义在 `mk-acker-bolt` 中的。在一个新的 tuple 树生成的时候，spout 为每个 tuple 发送一个用于异或的固有 id，acker 会将这些 id  记录在它的挂起队列中。每次 executor ack 一个 tuple 的时候，acker 会接收到一个部分校验和，这个校验和是 tuple 自身的 id（将其从挂起队列中清除）和 executor 发送的每个下游 tuple 的 id（放入挂起队列中）的异或值。

这个过程是这样的：

在接收到 tick tuple 信号的时候，将 tuple 树的校验值向超时方向移动并且返回。同时，在 tuple 树中更新或者创建一个记录。

- 初始化阶段：使用指定的校验和值进行初始化，并且记录 spout 的 id；
- ack 阶段：将部分校验和与当前的校验和进行异或运算；
- fail 阶段：仅仅将 tuple 标记为 failed 状态。

接下来，将记录[存入](https://github.com/apache/storm/blob/46c3ba7/storm-core/src/clj/backtype/storm/daemon/acker.clj#L50) RotatingMap（重新设置超时计数值）并且继续以下过程：

- 如果总校验和为 0， 表明 tuple 树已经完成：将记录从挂起队列中移除，并通知 spout 处理成功；
- 如果 tuple 树失败了，也会有一种完成状态：将记录从挂起队列中移除，并通知 spout 处理失败。

最后，发送一个我们自己的 ack 信号。

## 挂起 tuples 与 `RotatingMap`

Acker 将挂起树存放在一个 `RotatingMap` 中。`RotatingMap` 是一个在 Storm 中多处使用的简单工具，它主要用于高效地处理过程的超时。

RotatingMap 与 HashMap 类似，支持 O(1) 时间的 get 操作。

在 RotatingMap 内部有多个 HashMap（称为槽，buckets），每个 HashMap 都保存有一群会在同一时间超时的记录。我们称存在时间最长的 bucket 为死亡牢房（death row），而访问最多的 bucket 称为苗圃（nursery）。一个新的值在被 `.put()` 到 RotatingMap 中，它都会被重定位到 nursery 中，并且从其他的它之前可能在的 bucket 中移除（这是一种高效的重新设置延时时间的方法）。

在 RotatingMap 的所有者调用 `.rotate()` 方法的时候，RotatingMap 会将每个 bucket 向着超时的方向移动一步（一般 Storm 对象会在收到一个系统 tick 流 tuple 的时候调用 rotate 方法）。如果此时在前面所说的 death row bucket 中有 key-value 键值对，RotatingMap 会为每个 key-value 键值对触发一个回调函数（在构造器中定义的），让他们的所有者选择一个合适的操作（例如，将 tuple 标记为处理失败）。
