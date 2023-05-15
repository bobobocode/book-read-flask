# 前言

*——我们即将开始的旅行*

这是一本轻量级的小书. 目标是无负担地游览一遍Flask的源代码.
希望阅读它是轻松而又着实的.

Flask做为一款Python主流Web框架, 设计优秀, 工艺精良.
它的代码量即使是最新版本也不算大, 非常适合阅读和学习.

通过阅读Flask源代码, 不仅
能够学习到Web框架的内部原理, 而且能够
提高编写Python代码的能力和开发质量.

相信有兴趣翻看这本书的读者已经熟悉了Python基本的编程技术和Flask的基本使用,
所以我们将以此为前提启动我们的行程.

## 深度体验之旅

关于Flask的原理机制可以用一篇博客讲解
在此我们希望这场旅行能提供更深度的体验

本书中使用的是当前Flask的最新版本2.2.2

* Flask (2.2.2)

连接这三座城市的高速公路就是一个HTTP请求的处理流程.

* Werkzeug (2.2.2)

要透彻理解Flask就需要透彻了解Werkzeug.
虽然我们可以以Werkzeug提供的WSGI接口抽象为起点来了解Flask,
但是这会造成我们还是无法真正理解Flask到底怎样做的处理, 以及为什么要这
么去处理.

* Python (v3.10.6)

BaseServer BaseRequestHandler 

所有的知识点尽可能用源码进行讲解.

## 旅行计划?

然而源代码无论是编写过程还是逻辑关系都不是线性的, 所以我们的行程也是多维的.
为此我们对旅程的计划是这样的:

1. 自顶而下

可以形成对Flask源代码的总体把握

2. 自底向上

Flask构建核心设计概念的基础设施

只是总体把握源代码的脉络, 收获是框架性的, 必须填充饱满.

自底向上, 不用考虑上下文知识, 在一个较小的问题上，彻底分析清楚Flask是
怎么处理的.

3. 兜兜转转

按主题去探讨分析源代码, 将知识进行穿插关联, 形成对某个专题问题的认识

4. 回忆我们的旅行

总结思考
