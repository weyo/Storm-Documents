# 多语言接口协议

本文描述了 Storm （0.7.1 版本以上）的多语言接口协议。

## Storm 多语言协议

### Shell 组件

Storm 的多语言支持主要通过 ShellBolt，ShellSpout 和 ShellProcess 类来实现。这些类实现了 IBolt 接口、ISpout 接口，并通过使用 Java 的 ProcessBuilder 类调用 shell 进程实现了执行脚本的接口协议。

### 输出域

输出域是拓扑的 Thrift 定义的一部分。也就是说，如果你在 Java 中使用了多语言接口，那么你就需要创建一个继承自 ShellBolt 并实现 IRichBolt 接口的 bolt，这个 bolt 还需要在 `declareOutputFields` 方法中声明输出域（ShellSpout 也有类似的问题）。

你可以在[基础概念](https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Concepts.md)一文中了解更多相关信息。

### 协议报头

最简单的协议是通过执行脚本或程序的标准输入输出（STDIN/STDOUT）来实现的。在这个过程中传输的数据都是以 JSON 格式编码的，这样可以支持很多种语言。

## 打包

为了在集群上运行壳组件，执行的外壳脚本必须和待提交的 jar 包一起置于 `resources/` 目录下。

但是，在本地开发测试时，resources 目录只需要保持在 classpath 中即可。

### 协议

注意：

- 输入输出协议的结尾都使用行读机制，所以，必须要修剪掉输入中的新行并将他们添加到输出中。
- 所有的 JSON 输入输出都由一个包含 “end” 的行结束标志。注意，这个定界符并不是 JSON 的一部分。
- 下面的几个标题就是从脚本作者的 STDIN 与 STDOUT 的角度出发的。

### 初始握手

两种类型壳组件的初始握手过程都是相同的：

- STDIN: 设置信息。这是一个包含 Storm 配置、PID 目录、拓扑上下文的 JSON 对象：

```
{
    "conf": {
        "topology.message.timeout.secs": 3,
        // etc
    },
    "pidDir": "...",
    "context": {
        "task->component": {
            "1": "example-spout",
            "2": "__acker",
            "3": "example-bolt1",
            "4": "example-bolt2"
        },
        "taskid": 3,
        // 以下内容仅支持 Storm 0.10.0 以上版本
        "componentid": "example-bolt"
        "stream->target->grouping": {
            "default": {
                "example-bolt2": {
                    "type": "SHUFFLE"}}},
        "streams": ["default"],
        "stream->outputfields": {"default": ["word"]},
        "source->stream->grouping": {
            "example-spout": {
                "default": {
                    "type": "FIELDS",
                    "fields": ["word"]
                }
            }
        }
        "source->stream->fields": {
            "example-spout": {
                "default": ["word"]
            }
        }
    }
}
```

你的脚本应该在这个目录下创建一个以 PID 命名的空文件。比如，PID 是 1234 的时候，在目录中创建一个名为 1234 的空文件。这个文件可以让 supervisor 了解到进程的 PID，这样，supervisor 在需要的时候就可以关闭该进程。

Storm 0.10.0 加强了发送到壳组件的上下文的功能，现在的上下文中包含了兼容 JVM 组件的拓扑上下文中的所有内容。新增的一个关键因素是确定拓扑中某个壳组件的源与目标（也就是输入与输出）的功能，这是通过 `stream->target->grouping` 和 `source->stream->grouping` 字典实现的。在这些关联字典的底层，分组是以字典的形式表示的，至少包含有一个 `type` 键，并且也可以含有一个 `fields` 键，该键可以用于指定在 `FIELDS` 分组中所涉及的域。

- STDOUT: 你的 PID，以 JSON 对象的形式展现，比如 `{"pid": 1234}`。这个壳组件将会把 PID 记录到它自己的日志中。

接下来怎么做就要取决于组件的具体类型了。

### Spouts

Shell Spouts 都是同步的。以下内容是在一个 while(true) 循环中实现的：

- STDIN: 一个 next、ack 或者 fail 命令。

“next” 与 ISpout 的 `nextTuple` 等价，可以这样定义 “next”：

```
{"command": "next"}
```

可以这样定义 “ack”：

```
{"command": "ack", "id": "1231231"}
```

可以这样定义 “fail”：

```
{"command": "fail", "id": "1231231"}
```

- STDOUT: 前面的命令对你的 spout 作用产生的结果。这个结果可以是一组 emits 和 logs。

emit 大概是这样的：

```
{
    "command": "emit",
    // tuple 的 id，如果是不可靠 emit 可以省略此值，该 id 可以为字符串或者数字
    "id": "1231231",
    // tuple 将要发送到的流 id，如果发送到默认流，将该值留空
    "stream": "1",
    // 如果是一个直接型 emit，需要定义 tuple 将要发送到的任务 id
    "task": 9,
    // 这个 tuple 中的所有值
    "tuple": ["field1", 2, 3]
}
```

如果不是直接型 emit，你会立即在 STDIN 上收到一条表示 tuple 发送到的任务的 id 的消息，这个消息是以 JSON 数组形式展现的。

“log” 会将消息记录到 worker log 中，“log” 大概是这样的：

```
{
    "command": "log",
    // 待记录的消息
    "msg": "hello world!"
}
```

- STDOUT: “sync” 命令会结束 emits 与 logs 的队列，“sync” 是这样使用的：

```
{"command": "sync"}
```

在 sync 之后， ShellSpout 不会继续读取你的输出，直到它发送出新的 next，ack 或者 fail。

注意，与 ISpout 类似，worker 中的所有 spouts 都会在调用 next，ack 或者 fail 之后锁定，直到你调用 sync。同样，如果没有需要发送的 tuple，你也应该在 sync 之前 sleep 一小段时间。ShellSpout 不会自动 sleep。

### Bolts

Shell Bolts 的协议是异步的。你会在有 tuple 可用时立即从 STDIN 中获取到 tuple，同时你需要像下面的示例这样调用 emit，ack，fail，log 等操作写入 STDOUT：

- STDIN: 就是一个 tuple！这是一个 JSON 编码的结构：

```
{
    // tuple 的 id，为了兼容缺少 64 位数据类型的语言，这里使用了字符串
    "id": "-6955786537413359385",
    // 创建该 tuple 的 id
    "comp": "1",
    // tuple 将要发往的流 id
    "stream": "1",
    // 创建该 tuple 的任务
    "task": 9,
    // tuple 中的所有值
    "tuple": ["snow white and the seven dwarfs", "field2", 3]
}
```

- STDOUT: 一个 ack，fail，emit 或者 log。例如，emit 是这样的：

```
{
    "command": "emit",
    // 标记这个输出 tuple 的 tuples 的 ids
    "anchors": ["1231231", "-234234234"],
    // tuple 将要发送到的流 id，如果发送到默认流，将该值留空
    "stream": "1",
    // 如果是一个直接型 emit，需要定义 tuple 将要发送到的任务 id
    "task": 9,
    // 这个 tuple 中的所有值
    "tuple": ["field1", 2, 3]
}
```

如果不是直接型 emit，你会立即在 STDIN 上收到一条表示 tuple 发送到的任务的 id 的消息，这个消息是以 JSON 数组形式展现的。注意，由于 shell bolt 协议的异步特性，如果你在 emit 之后立即接收数据，有可能不会收到对应的任务 id，而是收到上一个 emit 的任务 id，或者是一个待处理的新 tuple。然而，最终接收到的任务 id 序列仍然是和 emit 的顺序完全一致的。

ack 是这样的：

```
{
    "command": "ack",
    // 待 ack 的 tuple
    "id": "123123"
}
```

fail 是这样的：

```
{
    "command": "fail",
    // 待 fail 的 tuple
    "id": "123123"
}
```

“log” 会将消息记录到 worker log 中，“log” 是这样的：

```
{
    "command": "log",
    // 待记录的消息
    "msg": "hello world!"
}
```

- 注意：对于 0.7.1 版本，shell bolt 不再需要进行“同步”。

### 处理心跳（0.9.3 及以上版本适用）

Storm 0.9.3 通过在 ShellSpout/ShellBolt 与他们的多语言子进程之间使用心跳来检测子进程是否处于挂起或僵死状态。所有通过多语言接口与 Storm 交互的库都必须使用以下步骤来……

#### Spout

Shell Spouts 是同步的，所有子进程会在 `next()` 的结尾发送 `sync` 命令。因此，你不需要为 spouts 做过多的处理。也就是说，在 `next()` 过程中不能够让子进程的 sleep 时间超过 worker 的延时时间。

#### Bolt

Shell Bolts 是异步的，所以 ShellBolt 会定期向它的子进程发送心跳 tuple。心跳 tuple 是这样的：

```
{
    "id": "-6955786537413359385",
    "comp": "1",
    "stream": "__heartbeat",
    // 这个 shell bolt 的系统任务 id
    "task": -1,
    "tuple": []
}
```

在子进程收到心跳 tuple 之后，它必须向 ShellBolt 发送一个 `sync` 命令。
