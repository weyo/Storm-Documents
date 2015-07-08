# Storm 内部技术实现

这部分的 wiki 是为了说明 Storm 是怎样实现的。在阅读本章之前你需要先了解怎样使用 Storm。

- 代码架构
- 拓扑的生命周期<sup>1</sup>
- 消息传递的实现<sup>1</sup>
- 应答框架的实现
- [Metrics][5]
- 事务型拓扑的工作机制<sup>1</sup>
- 单元测试<sup>2</sup>
	- 时间模拟
	- 完整的拓扑
	- 集群跟踪



---

>## 说明
<sup>1</sup> 该文内容已过期。  
<sup>2</sup> 该文官方文档暂未提供。  


[5]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Metrics.md