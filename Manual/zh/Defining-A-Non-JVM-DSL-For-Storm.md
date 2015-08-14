# 定义 Storm 的非 JVM 语言 DSL

实现非 JVM 语言 DSL（Domain Specific Language，领域专用语言）应该从 [storm-core/src/storm.thrift](https://github.com/apache/storm/blob/master/storm-core/src/storm.thrift) 文件开始。由于 Storm 拓扑是 Thrift 结构，而且 Nimbus 是一个 Thrift 后台进程，你可以以任意语言创建并提交拓扑。

当你创建 Thrift 结构的 spouts 与 bolts 时，spout 或者 bolt 的代码是以 ComponentObject 结构体的形式定义的：

```
union ComponentObject {
  1: binary serialized_java;
  2: ShellComponent shell;
  3: JavaObject java_object;
}
```

对于非 JVM 语言 DSL（这里以 Python DSL 为例），你需要使用其中的 “2” 与 “3”。ShellComponent 负责指定运行该组件（例如你的 python 代码）的脚本，而 JavaObject 则负责指定该组件的本地（native）Java spouts 与 bolts（而且 Storm 也会使用反射来创建 spout 或者 bolt）。

“storm shell” 命令可以用于提交拓扑。下面是一个示例：

```
storm shell resources/ python topology.py arg1 arg2
```

Storm shell 随后会将 `resources/` 打包到一个 jar 文件中，将该文件上传到 Nimbus，然后像这样调用你的 topology.py 脚本：

```
python topology.py arg1 arg2 {nimbus-host} {nimbus-port} {uploaded-jar-location}
```

接着你就可以使用 Thrift API 连接到 Nimbus 来提交拓扑，并将上传的 jar 文件地址作为参数传入 submitTopology 方法中。作为参考，下面给出了 submitTopology 的定义：

```java
void submitTopology(1: string name, 2: string uploadedJarLocation, 3: string jsonConf, 4: StormTopology topology)
    throws (1: AlreadyAliveException e, 2: InvalidTopologyException ite);
```

最后，对于非 JVM DSL 还有一件非常重要的事就是要确保可以在一个文件中方便地定义出完整的拓扑（bolts，spouts，以及拓扑的其他部分定义）。
