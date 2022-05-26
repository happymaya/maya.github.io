---
title:  组合（combinatorial thinking）
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2019-03-15 23:33:00 +0800
categories:  [Architecture Design, Design Patterns]
tags:  [设计模式, 抽象工厂模式 , Design Patterns, Abstract Factory]
math: true
mermaid: true
---

## Unix 哲学的本质
## 对编程的启示

启示一：保持简单清晰性，能提升代码质量
软件复杂度一般有以下三个源：

- 代码库规模
- 技术复杂度
- 实现复杂度

启示二：借鉴组合理念，有效应对多变的需求

- 所有的命令都可以使用管道来交互
- 可以任意地替换程序
- 自定义环境变量

**把软件设计成独立组件并能随意地组合，才能真正应对更多变化的需求。**

要做到组合设计，Unix 哲学其实给我们提供了两个解决思路
第一个是解耦。
第二个是模块化。模块化更深层的含义：可替换的一致性
这两个解决思路就是高内聚、低耦合原则：模块内部尽量聚合以保持功能的一致性，模块外部尽量通过标准去耦合。

启示三：重拾数据思维，重构优化程序设计
