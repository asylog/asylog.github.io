---
title: quant
date: 2026-08-26
series: 股票
draft: true
---
## journey
## should I use coding agent for quant code?

yes, because

- 从中期看，这是个必须要学的技能；要学的是如何 guide AI，如何 manage multiple agents，等等。so 必须用；
- 对于小/复杂的 module，我可以不关心实现只关心实现是否正确地能 pass test，比如复杂的 regex 处理，so 部分写 yes；
- 对于整个项目，如果我全 agent 写且不关心实现，带来的问题就是，我会很难修改和记住。因为这是个长期项目，任何理解障碍都会积累和放大， so 全部写 no；
- 相比于手搓，agent 的效率高太多，产量非常大。但我需要一个 lean 的 quant 项目，so 全部写 no；
- 担心过程日志泄露给 ai 公司造成我的源码暴露。true but 基于 big-world heuristics，世界上那么多 code、那么多量化代码、那么多写的比我好的都会稀释这个风险，so no worry；
- 渐进式重写整个项目

## system
## 3 purposes

- to test ideas
- to generate buy/sell signal for trading
- to observe and understand market