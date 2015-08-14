# 使用非 JVM 语言开发

- 两个部分：创建拓扑，以及使用其他语言实现 spouts 与 bolts
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

>译者注：由于本文部分内容与另一篇文档[定义 Storm 的非 JVM 语言 DSL](https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Defining-A-Non-JVM-DSL-For-Storm.md)重复，这里不再罗列，详情请参阅该文档。
