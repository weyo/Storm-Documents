# 使用非 JVM 语言开发

- 两个步骤：创建拓扑，使用其他语言实现 spouts 与 bolts
- 由于 Storm 的拓扑都是基于 thrift 结构的，所以使用其他语言创建拓扑也是一件很容易的事情
- 使用其他语言实现的 spouts 与 bolts 称为“多语言组件”（multilang components）或者“脱壳”（shelling）
	- 这是具体的实现协议：[多语言接口协议](https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Multilang-Protocol.md)
	- thrift 结构允许你定义以一个程序和脚本的方式定义多语言组件（例如，可以使用 python 程序和文件实现 bolt）
	- 在 Java 中，需要覆写 ShellBolt 或者 ShellSpout 来创建多语言组件
		- 注意，输出域是在 thrift 结构中声明的，所以在 Java 中你需要这样创建多语言组件：
			- 在 Java 中声明域，并通过在 shellbolt 的构造器中指定输出域来处理其他语言的代码
	- 多语言组件在 STDIN/STDOUT 中使用 JSON 消息来和子进程通信
	- 已经实现了 Ruby，Python 等语言的相关协议，例如，python 支持 emit、anchor、ack 与 log等操作
- “storm shell” 命令简化了构造 jar 包与向 nimbus 上传文件的过程
	- 构建 jar 文件并将其上传
	- 使用 nimbus 的 host/port 以及 jar 文件的 id 来调用你的程序

## 以非 JVM 语言实现 DSL 的相关说明

实现非 JVM 语言 DSL（Domain Specific Language，领域专用语言）应该从 src/storm.thrift 文件开始。由于 Storm 拓扑是 Thrift 结构，而且 Nimbus 是一个 Thrift 后台进程，你可以以任意语言创建并提交拓扑。

当你创建 Thrift 结构的 spouts 与 bolts 的时候，spout 或者 bolt 的代码是以 ComponentObject 结构体的形式指定的：

```
union ComponentObject {
  1: binary serialized_java;
  2: ShellComponent shell;
  3: JavaObject java_object;
}
```

对于非 JVM 语言 DSL，你可能需要使用其中的 “2” 与 “3”。ShellComponent 负责指定运行该组件（例如，你的 python 代码）的脚本，而 JavaObject 则负责指定该组件的本地（native）Java spouts 与 bolts（而且 Storm 也会使用反射来创建 spout 或者 bolt）。

“storm shell” 命令可以用于提交拓扑。下面是一个示例：

```
storm shell resources/ python topology.py arg1 arg2
```

Storm shell 随后会将 `resources/` 打包到一个 jar 文件中，将该文件上传到 Nimbus，然后像这样调用你的 topology.py 脚本：

```
python topology.py arg1 arg2 {nimbus-host} {nimbus-port} {uploaded-jar-location}
```

接着你就可以使用 Thrift API 连接到 Nimbus 来提交拓扑，并将上传的 jar 文件地址作为参数传入 submitTopology 方法中。作为参考，下面给出了 submitTopology 的定义：

```
void submitTopology(1: string name, 2: string uploadedJarLocation, 3: string jsonConf, 4: StormTopology topology)
    throws (1: AlreadyAliveException e, 2: InvalidTopologyException ite);
```
