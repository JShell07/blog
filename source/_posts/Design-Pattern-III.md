---
title: Design_Pattern_III
date: 2019-01-28 19:15:09
tags: 
 - bookmarks
 - program
categories: bookmarks
---

## 1.结构型模式
### 1.1. 桥接模式(Bridge)
桥接（Bridge）是用于把抽象化与实现化解耦，使得二者可以独立变化。这种类型的设计模式属于结构型模式，它通过提供抽象化和实现化之间的桥接结构，来实现二者的解耦。
其实围绕的本质还是面向对象的原则：松耦合(Compling)， 高内聚（Cohesion）。

![FixMe image of Bridge](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/StructuralPattern/Bridge_Pattern.png)

<!--more-->

### 1.2. 适配器模式（Adapter）
适配器模式（Adapter Pattern）是作为两个不兼容的接口之间的桥梁。
**通过继承或依赖（推荐）可以实现适配器模式**
**优点**： 
    - 可以让任何两个没有关联的类一起运行
    - 提高了类的复用

**缺点**： 过多地使用适配器，会让系统非常零乱，不易整体进行把握

![FixMe image of Adapters](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/StructuralPattern/Adapter_Pattern_1.png)

![FixMe image of Adapters](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/StructuralPattern/Adapter_Pattern_2.png)

### 1.3. 装饰器模式(Decorator)
主要思想使用装饰器类中含有基类指针，我们就可以使用装饰类去修饰多个子类（面向对象的多态特性）。Decorator采用组合方式取得毕采用继承方式更好的效果。

![FixMe Decorator image](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/StructuralPattern/Decorator_Pattern.png)

### 1.4. 组合模式（Composite)
Composite 模式于Decorate 模式有着类似的结构图。**但是Composite模式旨在构造类（重在表示），而Decorate模式重在不生成子类即可给对象添加职责（重在修饰）。** 

### 1.5. 享元模式（Flyweight）
享元模式（Flyweight Pattern）主要用于减少创建对象的数量，以减少内存占用和提高性能。

在C++ 中使用STL Vector容器，提供共享对象的仓库。
**关键代码**：用 HashMap 存储这些对象（JAVA）, C++ 用STL Vector。

**优点**：大大减少对象的创建，降低系统的内存，使效率提高。

**缺点**：提高了系统的复杂度，需要分离出外部状态和内部状态，而且外部状态具有固有化的性质，不应该随着内部状态的变化而变化，否则会造成系统的混乱。

### 1.6. 外观模式（Facade）
外观模式（Facade Pattern）隐藏系统的复杂性，并向客户端提供了一个客户端可以访问系统的接口。

**关键代码**：在客户端和复杂系统之间再加一层，这一层将调用顺序、依赖关系等处理好。

**优点**：  
    - 减少系统相互依赖  
    - 提高灵活性    
    - 提高了安全性  
    - 
**缺点**：  
    - 不符合开闭原则，如果要改东西很麻烦，继承重写都不合适  

![FixMe Facade Pattern Image](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/StructuralPattern/Facade_Pattern.png)

### 1.7.代理模式(Proxy)
在代理模式中，一个类代表另一个类的功能。为其他对象提供一种代理以控制对这个对象的访问。比如说：要访问的对象在远程的机器上。在面向对象系统中，有些对象由于某些原因如对象创建开销很大，或者某些操作需要安全控制，或者需要进程外的访问, 我们可以在访问此对象时加上一个对此对象的访问层。
使用到了C++的多态的特性。

![FixMe Proxy Image](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/StructuralPattern/Proxy_Pattern.png)
