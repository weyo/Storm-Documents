# 配置开发环境

本文详细讲解了配置 Storm 开发环境的相关信息。简单地说，配置过程包含以下几个步骤：

1. 下载 [Storm 发行版](http://storm.apache.org/downloads.html)，将其解压缩并复制到你的 `PATH` 环境变量的 `bin` 目录中（也可以根据需要自定义安装目录 —— 译者注）；
2. 如果需要在远程集群中运行拓扑，则需要在 `~/.storm/storm.yaml` 文件中配置好集群的相关信息。

上述几步的详细内容如下。

## 什么是开发环境？

Storm 包含两种操作模式：本地模式与远程模式（即集群模式 —— 译者注）。在本地模式下，你可以在本地机器上的一个进程中完成所有的开发、测试拓扑的工作。而在远程模式下，为了运行拓扑，你需要先向服务器集群提交该拓扑。

Storm 的开发环境已经为你准备好了一切，因此，你可以在本地模式下完成开发、测试拓扑的工作，将拓扑打包并提交到远程服务器，并在远程服务器集群上运行或者终止拓扑。

我们再来回顾一下本地机器与远程集群之间的关系。Storm 集群是由一个称为 “Nimbus” 的主节点管理的。本地机器通过与 Nimbus 通信来提交代码（代码已经打包为 jar 格式），这样代码文件中包含的拓扑就可以在集群中运行。Nimbus 会小心地维护着代码在集群中的分布式结构，并为待运行的拓扑分配 worker。本地机器可以使用一个称为 `storm` 的命令行客户端来与 Nimbus 进行通信。不过，`storm` 客户端仅用于远程模式，不能用于本地模式下开发、测试拓扑。

## 在本地机器上安装 Storm

如果要从本地机器上直接向远程集群提交拓扑，你需要在本地机器上安装 Storm 程序。本地的 Storm 程序可以提供与远程集群交互的 `storm` 客户端。在安装本地 Storm 之前，你需要从[这里](http://storm.apache.org/downloads.html)下载一个 Storm 安装程序并将其解压到你的电脑的某个位置。然后将 Storm 的 `bin/` 目录添加到你的 `PATH` 环境变量中，确保 `bin/storm` 脚本可以直接运行。

在本地机器上安装的 Storm 仅能用于与远程集群的交互。对于本地模式下的开发、测试拓扑，推荐使用 Maven 来将 Storm 添加到你的项目的开发依赖中。关于 Maven 的使用请参考[此文](https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Building-Storm-With-Maven.md)。

## 在远程集群上开始/终止拓扑的运行

在上一步中我们已经安装好了本地的 `storm` 客户端。接下来就需要告诉客户端需要连接哪一个 Storm 集群。这可以通过在 `~/.storm/storm.yaml` 文件中填写 Storm 集群的主节点的 host 地址来实现：

```
nimbus.host: "123.45.678.890"
```

另外，如果你在 AWS 上应用 [storm-deploy](https://github.com/nathanmarz/storm-deploy) 项目来配置 Storm 集群，它会自动配置好你的 `~/.storm/storm.yaml` 文件。你也可以使用 `attach` 命令手动配置附属的 Storm 集群（或者在多个集群之间切换）：

```
lein run :deploy --attach --name mystormcluster
```

更多内容请参考 storm-deploy 项目的 [wiki](https://github.com/nathanmarz/storm-deploy/wiki)。