# 容错性

本文通过问答的形式解释了 Storm 的容错性原理。

## 工作进程（worker）死亡时会发生什么？

工作进程死亡的时候，supervisor 会重新启动这个进程。如果在启动过程中仍然一直失败，并且无法向 Nimbus 发送心跳，Nimbus 就会将这个 worker 重新分配到其他机器上去。

## 节点故障时会发生什么？

一个节点（集群中的工作节点，非 Nimbus 所在服务器）故障时，该节点上所有的任务（tasks）都会超时，然后 Nimbus 在检测到超时后悔将所有的这些任务重新分配到其他机器上去。

## Nimbus 或者 Supervisor 的后台进程挂掉时会发生什么

Nimbus 和 Supervisor 的后台进程本身是设计为快速失败（无论何时发生了异常情况之后都会启动自毁操作）和无状态（所有的状态由 ZooKeeper 负责管理）的。正如[配置 Storm 集群][1]这篇文章中所述，Nimbus 和 Supervisor 的后台进程实际上是在后台监控工具的监控之下运行的。所以，如果 Nimbus 或者 Supervisor 进程挂掉，他们就会静默地自动重启。

值得一提的是，Nimbus 和 Supervisor 的故障不会影响任何工作进程。这一点与 Hadoop 形成了鲜明对比，在 Hadoop 中 JobTracker 的故障会导致所有正在运行的 job 运行失败（Hadoop 2.x 中引入的 Yarn 架构已经提升了 Hadoop 系统的稳定性，目前的架构并不会如这里所说的那么不堪——译者注）。

## Nimbus 是系统的单故障点<sup>1</sup>吗？

如果你的 Nimbus 节点出现故障无法访问，集群中的 worker 仍然会继续保持运行。另外，此时 Supervisor 也仍然会正常工作，在 worker 挂掉时自动重启挂掉的进程。但是，由于缺少 Nimbus 的协调，worker 就不会在必要的时候重新分配到不同的机器中（看上去好像你丢失了一个 worker）。

所以这个问题的答案是，Nimbus 确实稍微有一点像 SPOF（单故障点，Single Point of Failure）。不过在实际应用中，Nimbus 的故障从来就不是什么问题。未来的开发计划中还会考虑让 Nimbus 具备更好的可用性。

## Storm 是如何保障消息的完全处理的？

对于节点故障或者消息丢失的情况，Storm 提供了一套完善的机制保障所有的消息都能够得到正确处理。[消息的可靠性保障][2]一文中解释了这方面更多的技术细节。

---

<sup>1</sup>单故障点是指在一个系统中的某个在失效或停止运转后会导致整个系统不能工作的部件，具体概念可以参考[维基百科][3]。

[1]: http://storm.apache.org/documentation/Setting-up-a-Storm-cluster.html
[2]: http://storm.apache.org/documentation/Guaranteeing-message-processing.html
[3]: https://en.wikipedia.org/wiki/Single_point_of_failure