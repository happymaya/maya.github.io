---
title:  使用支付功能演练 DDD
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2021-06-01 23:33:00 +0800
categories: [Architecture Design, DDD]
tags:  [Architecture Design, DDD]
math: true
mermaid: true
---
上一篇，总结了软件退化的根源，以及如何利用 DDD 解决软件退化的问题。本文结合工作中的经验，以支付功能为例，演练一下基于 DDD 的软件设计以及变更的过程。


当最开始收到关于用户付款功能的需求描述如图 1 所示：
![用户付款功能的流程.png](https://cdn.nlark.com/yuque/0/2021/png/12442250/1636992744071-9f937439-498e-4b14-a1aa-c3d76b044009.png#clientId=u8c1b7874-01bd-4&crop=0&crop=0&crop=1&crop=1&from=ui&id=u011e47f6&margin=%5Bobject%20Object%5D&name=%E7%94%A8%E6%88%B7%E4%BB%98%E6%AC%BE%E5%8A%9F%E8%83%BD%E7%9A%84%E6%B5%81%E7%A8%8B.png&originHeight=1163&originWidth=1194&originalType=binary&ratio=1&rotation=0&showTitle=false&size=172307&status=done&style=stroke&taskId=u98140c9b-4414-4493-911f-7d7224f2510&title=)
图 1 开发人员在最初阶段收到的关于用户付款功能的需求流程图 

以往当拿到这个需求的时候，就会草草的设计之后就开始编码，设计质量也就不怎么高。正确的做法应该是在拿到新需求以后，**采用领域驱动的方式，先进行需求分析，设计出领域模型**。

按照图 1 所示的业务场景，可以分析出：

- 该场景中有 “订单”，每个订单都对应一个用户；
- 一个用户可以有多个用户地址，但每个订单只能有一个用户地址；
- 此外，一个订单对应多个订单明细，每个订单明细对应一个商品，每个商品对应一个供应商。

最后，对订单进行的“下单”、“付款”、“查看订单状态”等操作，最终形成了以下领域模型图，如图 2 所示：
![订单的领域模型图.png](https://cdn.nlark.com/yuque/0/2021/png/12442250/1636994055382-b79f6aa7-06b5-45e7-8522-0a71195de44e.png#clientId=u8c1b7874-01bd-4&crop=0&crop=0&crop=1&crop=1&from=ui&height=358&id=u4edd83c6&margin=%5Bobject%20Object%5D&name=%E8%AE%A2%E5%8D%95%E7%9A%84%E9%A2%86%E5%9F%9F%E6%A8%A1%E5%9E%8B%E5%9B%BE.png&originHeight=1433&originWidth=2673&originalType=binary&ratio=1&rotation=0&showTitle=false&size=306742&status=done&style=none&taskId=u9506ae8c-5132-4be2-95b0-0b35f1ca426&title=&width=668)
图 2 用户付款功能中的订单领域模型图

有了图 2 所示的领域模型就可以进行以下的程序设计，如图 3 所示：
 ![订单的领域模型图以及实现.png](https://cdn.nlark.com/yuque/0/2021/png/12442250/1636994540957-2dac0e10-b4dc-4a92-a858-532ef6346675.png#clientId=u8c1b7874-01bd-4&crop=0&crop=0&crop=1&crop=1&from=ui&height=468&id=u15fe4cc5&margin=%5Bobject%20Object%5D&name=%E8%AE%A2%E5%8D%95%E7%9A%84%E9%A2%86%E5%9F%9F%E6%A8%A1%E5%9E%8B%E5%9B%BE%E4%BB%A5%E5%8F%8A%E5%AE%9E%E7%8E%B0.png&originHeight=1873&originWidth=2944&originalType=binary&ratio=1&rotation=0&showTitle=false&size=437452&status=done&style=none&taskId=u860c8aa4-e533-46f8-92f3-92cb7053b94&title=&width=736)
图 3 用户付款功能中的订单领域模型图

通过领域模型的指导，将 “订单”分为订单 Service 与 值对象，将“用户”分为用户 Service 与值对象，将“商品”分为商品 Service 与 值对象，在此基础上实现各自的方法。

# 商品折扣的需求变更
当付款功能按照领域模型完成了第一个版本的设计后，很快就迎来了**第一次需求变更，即增加折扣功能**，**并且该折扣功能分为限时折扣、限量折扣、某类商品的折扣、某个商品的折扣与不折扣。**

当我们拿到这个需求时应当怎样设计呢？很显然，在 payoff() 方法中去插入 if 语句是不 OK 的。这时，按照领域驱动设计的思想，应当将需求变更还原到领域模型中进行分析，进而根据领域模型背后的真实世界进行变更。
![1644779787242-004a2414-182f-4e1a-988a-03290bf6c8b6 (2).png](https://cdn.nlark.com/yuque/0/2022/png/12442250/1644780535183-721374d8-8d8b-4a82-b52b-71e92d52c595.png#clientId=udc8bb282-3dc2-4&crop=0&crop=0&crop=1&crop=1&from=ui&id=u388972c9&margin=%5Bobject%20Object%5D&name=1644779787242-004a2414-182f-4e1a-988a-03290bf6c8b6%20%282%29.png&originHeight=487&originWidth=922&originalType=binary&ratio=1&rotation=0&showTitle=false&size=57889&status=done&style=none&taskId=u8ad82f1d-56fd-4c23-aa1a-89dd1d6238c&title=)
这是上一个版本的领域模型，现在要在这个模型的基础上增加折扣功能，并且还要分为限时折扣、限量折扣、某类商品的折扣等不同类型。此时，首先分析付款与则扣的关系。

付款与折扣是什么关系呢？显然折扣是在付款的过程中进行的折扣，就认为应当将折扣写到付款中。这样对吗？显然不对，此时就因该基于一个重量级的设计原则 —— 单一职责原则。

什么是高质量的代码。你可能立即会想到“低耦合、高内聚”，以及各种设计原则，但这些评价标准都太“虚”。最直接、最落地的评价标准就是，当用户提出一个需求变更时，为了实现这个变更而修改软件的成本越低，那么软件的设计质量就越高。怎样才能在每次变更的时候都只修改一个模块就能实现新需求呢？这就需要在平时就不断地整理代码，将那些因同一个原因而变更的代码都放在一起，而将因不同原因而变更的代码分开放在不同的模块、不同的类中。这样，当因为这个原因而需要修改代码时，需要修改的代码都在这个模块、这个类中，修改范围就缩小了，维护成本降低了，自然设计质量就提高了。总之，单一职责原则要求我们在维护软件的过程中需要不断地进行整理，将软件变化同一个原因的代码放在一起，将软件变化不同原因的代码分开放

照这样的设计原则，分析“付款”与“折扣”之间的关系呢只需要回答以下两个问题：

- 当“付款”发生变更时，“折扣”是不是一定要变？
- 当“折扣”发生变更时，“付款”是不是一定要变？

当这两个问题的答案是否定时，就说明“付款”与“折扣”是软件变化的两个不同的原因，那么把它们放在一起，放在同一个类、同一个方法中，合适吗？不合适，就应当将“折扣”从“付款”中提取出来，单独放在一个类中。
同样的道理，举一反三。最后发现，不同类型的折扣也是软件变化不同的原因。将它们放在同一个类、同一个方法中，合适吗？通过以上分析，做出了如下设计：
![1644779787242-004a2414-182f-4e1a-988a-03290bf6c8b6 (3).png](https://cdn.nlark.com/yuque/0/2022/png/12442250/1644781209999-7b0b5859-f314-4417-862d-eed786166ea3.png#clientId=udc8bb282-3dc2-4&crop=0&crop=0&crop=1&crop=1&from=ui&id=u5f4ff13a&margin=%5Bobject%20Object%5D&name=1644779787242-004a2414-182f-4e1a-988a-03290bf6c8b6%20%283%29.png&originHeight=525&originWidth=748&originalType=binary&ratio=1&rotation=0&showTitle=false&size=49771&status=done&style=none&taskId=ucc6ba172-e645-4706-b871-bc9ea25c535&title=)

> 

# VIP 会员的需求变更
在第一次变更的基础上，很快迎来了第二次变更，这次是要增加 VIP 会员，业务需求如下。

1. 对不同类型的 VIP 会员（金卡会员、银卡会员）进行不同的折扣；
2. 在支付时，为 VIP 会员发放福利（积分、返券等）；
3. VIP 会员可以享受某些特权。

拿到这样的需求又应当怎样设计呢？同样，先回到领域模型，分析三个领域对象之间的关系

- 分析“用户”与“VIP 会员”的关系，“
- 付款”与“VIP 会员”的关系。在分析的时候，

还是回答那两个问题：

- “用户”发生变更时，“VIP 会员”是否要变？
- “VIP 会员”发生变更时，“用户”是否要变？

通过分析发现，“用户”与“VIP 会员”是两个完全不同的事物。

- “用户”要做的是用户的注册、变更、注销等操作；
- “VIP 会员”要做的是会员折扣、会员福利与会员特权；

而“付款”与“VIP 会员”的关系是在付款的过程中去调用会员折扣、会员福利与会员特权。
过以上的分析，做出了以下版本的领域模型：
![VIP 会员领域模型.png](https://cdn.nlark.com/yuque/0/2022/png/12442250/1644781510513-0ee938ca-c37a-4138-a3af-493af2365a06.png#clientId=udc8bb282-3dc2-4&crop=0&crop=0&crop=1&crop=1&from=ui&id=u89511952&margin=%5Bobject%20Object%5D&name=VIP%20%E4%BC%9A%E5%91%98%E9%A2%86%E5%9F%9F%E6%A8%A1%E5%9E%8B.png&originHeight=562&originWidth=608&originalType=binary&ratio=1&rotation=0&showTitle=false&size=40358&status=done&style=none&taskId=u63738c06-6ce7-4a64-858c-8f1ab4c94c8&title=)
有了这些领域模型的变更，然后就可以以此作为基础，指导后面程序代码的变更了。

# 支付方式的需求变更

同样，第三次变更是增加更多的支付方式，我们在领域模型中分析“付款”与“支付方式”之间的关系，发现它们也是软件变化不同的原因。因此，果断做出了这样的设计：
![支付方式的领域模型 (2).png](https://cdn.nlark.com/yuque/0/2022/png/12442250/1644781654584-f5e3894e-4a10-46d5-8a86-994bce9da125.png#clientId=udc8bb282-3dc2-4&crop=0&crop=0&crop=1&crop=1&from=ui&id=ued7da5c7&margin=%5Bobject%20Object%5D&name=%E6%94%AF%E4%BB%98%E6%96%B9%E5%BC%8F%E7%9A%84%E9%A2%86%E5%9F%9F%E6%A8%A1%E5%9E%8B%20%282%29.png&originHeight=440&originWidth=645&originalType=binary&ratio=1&rotation=0&showTitle=false&size=40249&status=done&style=none&taskId=u4255ce74-eece-415b-9509-eef4d5598d3&title=)
而在设计实现时，因为要与各个第三方的支付系统对接，也就是要与外部系统对接。为了使第三方的外部系统的变更对我们的影响最小化，在它们中间果断加入了“适配器模式”，设计如下：
![支付方式的领域模型-适配器.png](https://cdn.nlark.com/yuque/0/2022/png/12442250/1644781787749-7b52e62a-7f15-4e52-ae2b-4fee88dbf436.png#clientId=udc8bb282-3dc2-4&crop=0&crop=0&crop=1&crop=1&from=ui&id=ua1137ce8&margin=%5Bobject%20Object%5D&name=%E6%94%AF%E4%BB%98%E6%96%B9%E5%BC%8F%E7%9A%84%E9%A2%86%E5%9F%9F%E6%A8%A1%E5%9E%8B-%E9%80%82%E9%85%8D%E5%99%A8.png&originHeight=537&originWidth=641&originalType=binary&ratio=1&rotation=0&showTitle=false&size=31778&status=done&style=none&taskId=u51d983f1-7a2c-4de0-93e4-cf91ee2c936&title=)
通过加入适配器模式，订单 Service 在进行支付时调用的不再是外部的支付接口，而是“支付方式”接口，与外部系统解耦。只要保证“支付方式”接口是稳定的，那么订单 Service 就是稳定的。比如：

- 当支付宝支付接口发生变更时，影响的只限于支付宝 Adapter；
- 当微信支付接口发生变更时，影响的只限于微信支付 Adapter；
- 当要增加一个新的支付方式时，只需要再写一个新的 Adapter。

日后不论哪种变更，要修改的代码范围缩小了，维护成本自然降低了，代码质量就提高了。
