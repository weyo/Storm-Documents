# 使用 Maven 构建 Storm 应用

在开发拓扑的时候，你需要在 classpath 中包含 Storm 的相关 jar 包。你可以将各个 jar 包直接包含到你的项目的 classpath 中，也可以使用 Maven 将 Storm 添加到依赖项中。Storm 已经集成到 Maven 的中心仓库中。你可以在项目的 pom.xml 中添加以下依赖来将 Storm 包含进项目中：

```xml
<dependency>
  <groupId>org.apache.storm</groupId>
  <artifactId>storm-core</artifactId>
  <version>0.9.5</version>
  <scope>provided</scope>
</dependency>
```

这是一个 Storm 项目的 pom.xml 文件的[例子][1]。

## Storm 开发

请参考 [DEVELOPER.md][2] 一文来了解更多相关信息。


[1]: https://github.com/apache/storm/blob/master/examples/storm-starter/pom.xml
[2]: https://github.com/apache/storm/blob/master/DEVELOPER.md