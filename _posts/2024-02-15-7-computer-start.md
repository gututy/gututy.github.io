---
title: 计算机系统启动过程分析
tags: 计算机体系结构 学习笔记
permalink: /notes/ca/computer-start
aside:
    toc: ture
sidebar:
    nav: notes
---

> 第七章：计算机启动过程分析
<!--more-->

# 一句话要点
```markdown
- 系统启动
  - 从复位到系统可用
- 初始化是什么
  - 将系统各种寄存器状态从不确定设置为确定，将一些模块状态从无序强制为有序的一个过程
- 什么东西需要初始化
  - CPU、内存、各类IO接口
- 怎么初始化
  - 按照从核内到核外，从片内到片外的次序进行
```

```
li a0,a1
```
{:.language-loongarchasm}