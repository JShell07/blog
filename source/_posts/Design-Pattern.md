---
title: Design_Pattern
date: 2018-07-12 19:45:14
tags: 
 - bookmarks
 - program
categories: bookmarks
---

## 背景
![](https://gss1.bdstatic.com/9vo3dSag_xI4khGkpoWK1HF6hhy/baike/c0%3Dbaike272%2C5%2C5%2C272%2C90/sign=970418febc003af359b7d4325443ad39/4a36acaf2edda3ccdd0e40d70be93901203f925b.jpg)
设计模式(design pattern)， 是在这1994年，由 Erich Gamma、Richard Helm、Ralph Johnson 和 John Vlissides 四人合著出版了一本名为 Design Patterns - Elements of Reusable Object-Oriented Software（中文译名：[设计模式 - 可复用的面向对象软件元素](https://baike.baidu.com/item/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F%EF%BC%9A%E5%8F%AF%E5%A4%8D%E7%94%A8%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E8%BD%AF%E4%BB%B6%E7%9A%84%E5%9F%BA%E7%A1%80/7600072?fr=aladdin)） 的书提出。主要基于：
- 对接口编程而不是对实现编程
- 优先使用对象组合而不是继承

面向对象系统的分析和设计实际上追求的就是两点,一是高内聚(Cohesion),而是低耦合(Coupling)。

<!--more-->

## 1. 设计模式鸟览
常见设计模式有23种。大体可以分为三大类：
- 创建模式（Creational Patterns）
- 结构型模式(Structural Patterns)
- 行为型模式（Behavioral Patterns)

当然，还有J2EE设计模式。

模式 | 描述
:-: | :-
创建模式 | 这些设计模式提供了一种在创建对象的同时**隐藏创建逻辑**的方式，而不是使用 new 运算符直接实例化对象。这使得程序在判断针对某个给定实例需要创建哪些对象时更加灵活。
结构型模式 | 这些设计模式**关注类和对象的组合**。继承的概念被用来组合接口和定义组合对象**获得新功能**的方式。
行为型模式 | 这些设计模式特别**关注对象之间的通信**。
J2EE 模式 | 这些设计模式特别**关注表示层**。这些模式是由 Sun Java Center 鉴定的。

**创建模式（Creational Patterns）**

![Creational Patterns](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/CreationalPattern.png)

**结构型模式(Structural Patterns)**

![Structural Patterns](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/StructuralPattern.png)

**行为型模式（Behavioral Patterns)**

![Behavioral Patterns](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/BehavioralPatternpng.png)

**J2EE设计模式**

![J2EE设计模式](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/J2EEPattern.png)

## Ref.
http://www.runoob.com/design-pattern/design-pattern-tutorial.html