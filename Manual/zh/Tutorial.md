# 教程

在这篇教程中，你可以学到如何创建 Storm 的拓扑（topology），并将他们发布到 Storm 集群中。本文主要使用 Java，不过也会少量使用 Python 来说明 Storm 的多语言支持能力。

## 序言

本教程通过 [storm-starter](https://github.com/apache/storm/blob/master/examples/storm-starter) 项目中的几个例子来介绍 Storm 的用法。强烈建议读者将该项目 clone 到本地并实际动手运行一下。你可以先按照[配置开发环境](https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Setting-Up-A-Development-Environment.md)和[创建 Storm 项目](https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Creating-A-New-Storm-Project.md)两篇文档的说明配置好你的开发环境。

## Storm 集群的组件

