# 命令行操作

本文介绍了 Storm 命令行客户端中的所有命令操作。如果想要了解怎样设置你的 Strom 客户端和远程集群的交互，请按照[配置开发环境][1]一文中的步骤操作。

Storm 中支持的命令包括：

1. jar
2. kill
3. activate
4. deactivate
5. rebalance
6. repl
7. classpath
8. localconfvalue
9. remoteconfvalue
10. nimbus
11. supervisor
12. ui
13. drpc

## jar

语法：`storm jar topology-jar-path class ...`

使用指定的参数运行 main 方法（也就是打包好的拓扑 jar 包中的 main 方法）。Storm 所需要的 jar 包和配置信息都在类路径（classpath）中。这个运行过程已经配置好了，这样 [StormSubmitter][2] 就可以在提交拓扑的时候将 `topology-jar-path` 中的 jar 包上传到集群中。

## kill

语法：`storm kill topology-name [-w wait-time-secs]`

杀死集群中正在运行的名为 `topology-name` 的拓扑。执行该操作后，Storm 首先会注销拓扑中的 spout，使得拓扑中的消息超时，这样当前的所有消息就会结束执行。随后，Storm 会将所有的 worker 关闭，并清除他们的状态。你可以使用 `-w` 参数来调整 Storm 在注销与关闭拓扑之间的间隔时间。

## activate

语法：`storm activate topology-name`

激活运行指定拓扑的所有 spout。

## deactivate

语法：`storm deactivate topology-name`

停止指定拓扑的所有 spout 的运行。

## rebalance

语法：`storm rebalance topology-name [-w wait-time-secs]`

有些场景下需要对正在运行的拓扑的工作进程（worker）进行弹性扩展。例如，加入你有 10 个节点，每个节点上运行有 4 个 worker，现在由于各种原因你需要为集群添加 10 个新节点。这时你就会希望通过扩展正在运行的拓扑的 worker 来使得每个节点只运行两个 worker，降低集群的负载。实现这个目的的一种直接的办法是 kill 掉正在运行的拓扑，然后重新向集群提交。不过 Storm 提供了再平衡命令可以以一种更简单的方法实现目的。

再平衡首先会在一个超时时间内（这个时间是可以通过 `-w` 参数配置的）注销掉拓扑，然后在整个集群中重新分配 worker。接着拓扑就会自动回到之前的状态（也就是说之前处于注销状态的拓扑仍然会保持注销状态，而处于激活状态的拓扑则会返回激活状态）。

## repl

语法：`storm repl`

打开一个带有类路径上的 jar 包和配置信息的 Clojure 的交互式解释器（REPL）。该命令主要用于调试。

## classpath

语法：`storm classpath`

打印客户端执行命令时使用的类路径环境变量。

## localconfvalue

语法：`storm localconfvalue conf-name`

打印出本地 Storm 配置中 `conf-name` 属性的值。这里的本地配置指的是 `~/.storm/storm.yaml` 和 `defaults.yaml` 两个配置文件综合后的配置信息。

## remoteconfvalue

语法：`storm remoteconfvalue conf-name`

打印出集群配置中 `conf-name` 属性的值。这里的集群配置指的是 `$STORM-PATH/conf/storm.yaml` 和 `defaults.yaml` 两个配置文件综合后的配置信息。该命令必须在一个集群机器上执行。

## nimbus

语法：`storm nimbus`

启动 nimbus 后台进程。该命令应该在 [daemontools][3] 或者 [monit][4] 这样的工具监控下执行。详细信息请参考[配置 Storm 集群][5]一文。

## supervisor

语法：`storm supervisor`

启动 supervisor 后台进程。该命令应该在 [daemontools][3] 或者 [monit][4] 这样的工具监控下执行。详细信息请参考[配置 Storm 集群][5]一文。

## ui

语法：`storm ui`

启动 UI 后台进程。UI 提供了一个访问 Storm 集群的 Web 接口，其中包含有运行中的拓扑的详细信息。该命令应该在 [daemontools][3] 或者 [monit][4] 这样的工具监控下执行。详细信息请参考[配置 Storm 集群][5]一文。

## drpc

语法：`storm drpc`

启动 DRPC 后台进程。该命令应该在 [daemontools][3] 或者 [monit][4] 这样的工具监控下执行。详细信息请参考[分布式 RPC][6]一文。


[1]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Setting-Up-A-Development-Environment.md
[2]: http://storm.apache.org/javadoc/apidocs/backtype/storm/StormSubmitter.html
[3]: http://cr.yp.to/daemontools.html
[4]: http://mmonit.com/monit/
[5]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Setting-Up-A-Storm-Cluster.md
[6]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Distributed-RPC.md
