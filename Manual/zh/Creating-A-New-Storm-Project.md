# 创建 Storm 新项目

本文简单介绍了新建 Storm 开发项目的方法，包括以下步骤：

1. 将 Storm 的 jar 包添加到 classpath 中；
2. 如果使用多语言接口，同样需要将多语言接口目录添加到 classpath 中。

可以按照以下步骤来在 Eclipse 中设置 [storm-starter](https://github.com/apache/storm/blob/master/examples/storm-starter) 项目。

## 将 Storm 的 jar 包添加到 classpath 中

在开发 Storm 的拓扑之前需要先将 Storm 的 jar 包添加到开发环境的 classpath 中。这里我们特别推荐使用 [Maven](https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Building-Storm-With-Maven.md) 在构建项目。这里有一个设置 Storm 项目的 pom.xml 的[例子](https://github.com/apache/storm/blob/master/examples/storm-starter/pom.xml)。如果不想使用 Maven，你也可以将 Storm 程序包中的 jar 包导入到你的 classpath 中来实现相同的效果。

可以按照以下步骤完成 Eclipse 开发环境的设置：

1. 创建一个新的 Java 项目；
2. 将 `src/jvm/` 设置为 source path；
3. 将 `lib/` 和 `lib/dev` 中的所有 jar 包（前面通过 Maven 下载或者手动拷贝的 Storm jar 包）导入项目的 `Referenced Libraries` 模块。

## 如果使用多语言接口，将多语言目录添加到 classpath 中

如果需要使用 Java 以外的语言实现 spout 或者 bolt，这些实现应该放置在项目的 `multilang/resources/` 目录中。为了让 Storm 可以在本地模式下找到这些源文件，需要将 `resources/` 目录加入 classpath 中。这在 Eclipse 中，就是通过将 `multilang/` 目录设置为源目录（source folder）来实现。有时候同样也需要将 `multilang/resources` 目录添加到源目录中。

如果需要了解更多关于使用 Java 以外的语言实现拓扑的内容，请参阅[使用非 JVM 语言开发](https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Using-Non-JVM-Languages-With-Storm.md)一文。

可以通过运行 `WordCountTopology.java` 来测试 Storm 的 Eclipse 开发环境是否已经配置完好。如果一切正常，在运行该程序之后，应该可以在 Eclipse 的终端窗口中看到发射出来的消息。