---
title: Teach Yourself Programming in Ten Years
date: 2019-08-15 17:20:55
category: 工作感悟
tags:
- 持续学习
---

如果是昨天的我，肯定没有勇气开这篇帖子。在极客时间耗子叔的左耳听风专栏《程序员练级攻略：开篇词》中，看到这篇推荐文章[《Teach Yourself Programming in Ten Years》](http://norvig.com/21-days.html)，然后读下去，突然有了沉下心的感觉。我想我可以花上一年，两年，十年甚至更多时间，去当一个自己心目中的程序员。

<!--more-->

文章的作者是Peter Norvig，AAAI fellow，人工智能协会创始会员，ACM院士。这篇文章被耗子叔誉为“传世之文”，从满大街的“21天教你学会***”这一现象说起，讲述了如何耐心地花上更多，10年/10000个小时的时间来真正成为一个资深的程序员。文章不长，初读起来没有太大的感觉，因为其中的很多想法都或多或少地通过其他渠道阅读过。但读后回味，竟然真的能劝说我沉下心来打磨自己，一时感到有些诧异。

文章中阐述的如何成为一个程序员的方法罗列如下：

+ Get interested in programming
+ Program
+ Talk with other programmers
+ Go to college (or a graduate school)
+ Work on projects with other programmers
+ Work on projects after other programmers
+ Learn at least a half dozen programming languages
+ Remember that there is a "computer" in "computer science"
+ Get involved in a language standardization effort
+ Have the good sense to get off the language standardization effort as quickly as possible

细节不再详述，收获最大的是程序员这份职业，更像是一个手工匠人，需要更多的实践。即使需要阅读学习，但如果不做出自己的作品，就永远无法证明自己的价值，甚至无法证实自己是否有价值。

与其总是幻想着马上成功，不如定一个小目标：

*** 踏踏实实地学习24个月（两年）  ***

有路线，有时间，只差执行下去的毅力。

另外文章中还罗列了一张表格，第一步是把它背下来：了解我的计算机。

Execution                           | Time                                   |
-                                   | -                                      |
execute typical instruction         | 1/1,000,000,000 sec = 1 nanosec        |
fetch from L1 cache memory          | 0.5 nanosec                            |
branch misprediction                | 5 nanosec                              |
fetch from L2 cache memory          | 7 nanosec                              |
Mutex lock/unlock                   | 25 nanosec                             |
fetch from main memory              | 100 nanosec                            |
send 2K bytes over 1Gbps network    | 20,000 nanosec                         |
read 1MB sequentially from memory   | 250,000 nanosec                        |
fetch from new disk location (seek) | 8,000,000 nanosec                      |
read 1MB sequentially from disk     | 20,000,000 nanosec                     |
send packet US to Europe and back   | 150 milliseconds = 150,000,000 nanosec |