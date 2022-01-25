---
title: java Agentmain
tags: java,热部署
category: /java/热部署/2022-01
renderNumberedHeading: true
grammar_cjkRuby: true
---
2022-01-25 13:57

> 上一篇讲解了premain的使用, premain只会在JVM启动的时候main方法执行之前会进行替换. 
> 假如我们想要真正的实现不停机热部署的话, 需要agentmain
> 下面我们聊聊agentmain是如何使用的


# 继续沿用java-agent项目(懒)
* 查看java-agent项目
* pom.xml不动, 添加两个Agentmain类和Agent类转换器
* Agentmain