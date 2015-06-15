# About
本项目是 Apache Storm 官方文档的中文翻译版，致力于为有实时计算项目需求和对 Apache Storm 感兴趣的同学提供有价值的中文资料，希望能够对大家的工作和学习有所帮助。由于本人水平有限，翻译中仍然存在不少问题，还请大家不吝斧正。

>说明：如果没有特殊声明，本项目文档中所述 Storm 版本均为 0.9.x 版本。

---

# Storm 官方文档索引

原版资料来源（官方网站）：[http://storm.apache.org/documentation/Documentation.html](http://storm.apache.org/documentation/Documentation.html)

---

## Storm 基础篇
- [Javadoc][1]<sup>1</sup>
- [基础概念][2]
- [配置][3]
- 消息的可靠性保证
- 容错性
- 命令行操作
- 理解 Storm 拓扑的并行度(parallelism)概念
- FAQ

---

## Trident

> _`Trident` 是 Storm 的一种高级操作接口，它能够提供可靠的数据流一次性处理模式、“事务型”数据持久化存储功能以及一系列数据流分析操作通用组件。_

- Trident 学习指南 —— 基本概念与走查
- Trident API 概述 —— 数据的转换与整合操作
- Trident State —— 精确的数据一次性处理与快速、持久化的聚合方式
- Trident Spouts —— 事务性与非事务性的数据入口

---

## 配置与部署

- 配置 Storm 集群
- 本地模式
- 故障排除
- 在生产环境下运行 topology
- 使用 Maven 构建 Storm 应用

---

## Storm 中级篇

- 序列化
- 通用模式
- Clojure DSL
- 使用非 JVM 语言开发
- 分布式 RPC
- 事务型拓扑
- Storm 与 Kestrel
- Hooks
- 软件度量
- 一个 trident 元组的生命周期

---

## Storm 高级篇

- 为 Storm 定义非 JVM 语言 DSL
- 多语言接口协议（如何为其他语言定义接口）
- 技术实现相关文档

---

<sup>1</sup> 说明：JavaDoc 暂时不在翻译计划之中。

[1]: http://storm.apache.org/javadoc/apidocs/index.html
[2]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Concepts.md
[3]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Configuration.md