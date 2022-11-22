# 前言

*——我们即将开始的旅行*

Flask源代码即使是最新版本代码量也不算大. 其凭借优秀的设计和高质量的代码
成为了一款主流的Python语言Web框架.
Flask源代码展现了杰出的软件工艺, 是我们学习Python进阶编程的一本非常好
的教科书.

本书既是作者自己的学习总结, 又特意从能为更多朋友
更高效深入地学习Flask源代码的视角做了出很多的努力.
本书中使用的是当前Flask的最新版本2.2.2
希望作者对源代码的解读, 能带你进行一场轻松又着实的Flask源代码之旅.

在开始这场旅行前, 我们需要假定读者已经熟悉Python基本的编程技术和Flask的基本使用.


## 旅途中的三座城市

在我们的旅途上, 有三座瑰丽的逻辑都市:

* Python

BaseServer BaseRequestHandler 

所有的知识点尽可能用源码进行讲解.

* Werkzeug

是的, 要透彻理解Flask就需要透彻了解Werkzeug.

* Flask

连接这三座城市的高速公路就是一个HTTP请求的处理流程.


## 旅程计划

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
