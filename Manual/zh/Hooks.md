# Hooks

Storm 提供了一种 hooks 机制，可以实现在 Storm 的各种事件流中运行自定义代码的功能。可以通过继承 [BaseTaskHook][1] 类来创建 hook，还可以根据需要在继承的子类中覆写适当的方法来跟踪相关事件。注册 hook 有两种方法：

1. 在 spout 的 open 方法或者 bolt 的 prepare 方法中使用 [TopologyContext#addTaskHook][2] 方法；
2. 使用 Storm 配置表中的 [topology.auto.task.hooks][3] 配置项。之后这些 hook 会自动注册到每个 spout 和 bolt 中，这样就可以很方便地处理例如集成自定义的系统监控代码之类的事情了。


[1]: http://storm.apache.org/javadoc/apidocs/backtype/storm/hooks/BaseTaskHook.html
[2]: http://storm.apache.org/javadoc/apidocs/backtype/storm/task/TopologyContext.html
[3]: http://storm.apache.org/javadoc/apidocs/backtype/storm/Config.html#TOPOLOGY_AUTO_TASK_HOOKS