---
title: Design_Pattern_IV
date: 2019-01-28 19:21:28
tags: 
 - bookmarks
 - program
categories: bookmarks
---

## 1. 行为模式
### 1.1. 模板模式（Template）
在模板模式（Template Pattern）中，一个抽象类公开定义了执行它的方法的方式/模板。它的子类可以按需要重写方法实现，但调用将以抽象类中定义的方式进行。
**将通用算法（逻辑）封装起来，而将算法细节让子类实现（多态）**。

```c++
class AbstractClass
{
    public:
        virtual ~AbstractClass();
        void TemplatMethod();
    protected:
        virtual void Operation1() = 0;
        virtual void Operation2() = 0;
}

class ConcreteClass1: public AbstractClass
{
    public:
        ConcreteClass1();
        ~ConcreteClass1();
    protected:
        void Operation1() {cout<<"ConcreteClass1 Operation1"<<endl;}
        void Operation2(){cout<<"ConcreteClass1 Operation1"<<endl;}
}

class ConcreteClass2: public AbstractClass
{
    public:
        ConcreteClass2();
        ~ConcreteClass2();
    protected:
        void Operation1() {cout<<"ConcreteClass2 Operation1"<<endl;}
        void Operation2(){cout<<"ConcreteClass2 Operation1"<<endl;}
}

void main()
{
    AbstractClass * c1 = new ConcreteClass1();
    AbstractClass * c2 = new ConcreteClass2();

    c1->TemplateMethod();
    c2->TemplateMethod();

    delete c1;
    delete c2;
}

```

<!--more-->

### 1.2.策略模式（Strategy)
在策略模式（Strategy Pattern）中，一个类的行为或其算法可以在运行时更改。

Strategy 与Template 模式类似，但是Strategy 是将逻辑（算法）封装到一个类中，并采取组合的方式解决。

![](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/BehavioralPattern/Strategy_Pattern.png)

**面向对象的设计中有一条很重要的原则就是：优先使用（对象）组合，而非类继承（favorComposition Over Inheritance)。**

### 1.3.状态模式（state）
将状态与逻辑实现分离，当一个操作中要维护大量的switch case分支语句，并且这些分支依赖于对象的状态。

state 与 strategy 比较类似，但是两者的关注点不同， state 在于状态的改变， strategy 在于算法逻辑的解藕。

![FixMe state pattern image](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/BehavioralPattern/State_Pattern.png)

### 1.4.观察者模式（Observer)
**观察者模式可以说是应用最多。影响最广的模式之一**。当一个对象被修改时，则会自动通知它的依赖对象。 

**解决问题**： 建立一个一（Subject）对多（observer）的依赖关系，当“一”变化时，“多”也能同步改变。

**优点**：  
    - 观察者和被观察者是抽象耦合的      
    - 建立一套触发机制  

**缺点**：  
    - 如果一个被观察者对象有很多的直接和间接的观察者的话，将所有的观察者都通知到会花费很多时间   
    - 如果在观察者和观察目标之间有循环依赖的话，观察目标会触发它们之间进行循环调用，可能导致系统崩溃。  

我们通过在Context 中的list 或者Vector 维持一组观察者（observer），在Context 更新时，调用Notify()函数，再使用C++的多态，调用到子类的Update（）函数。

![FixMe Add Observer Image](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/BehavioralPattern/Observer_Pattern.png)

### 1.5.备忘录模式（Memento）
备忘录模式（Memento Pattern）保存一个对象的某个状态，以便在适当的时候恢复对象。 

**应用实例**： 
    - 后悔药  
    - 打游戏时的存档  
    - Windows 里的 ctri + z  
    - IE 中的后退  
    - 数据库的事务管理  

**缺点**：消耗资源。如果类的成员变量过多，势必会占用比较大的资源，而且每一次保存都会消耗一定的内存。

注： 申明Originator 为Memento类的友元类，以便可以访问Memento 的私有成员。Originator 通过Memento 类来备份还原。

![FixMe Add Memento Pattern Image](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/BehavioralPattern/Memento_Pattern.png)

### 1.6.中介者模式（Mediator）
中介者模式是一种**很有用并且很常用的类**，用来降低多个对象和类之间的通信复杂性。将多对多的通信转化为一对多的通信。

Mediator 可以有解藕特性，通过Mediator，各个Colleague 就不必维护各自通信的对象和协议。

![](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/BehavioralPattern/Mediator_Pattern.png)

### 1.7. 命令模式（Command）
请求以命令的形式包裹在对象中，并传给调用对象。调用对象寻找可以处理该命令的合适的对象，并把该命令传给相应的对象，该对象执行命令。 

![FixMe Add Command Image](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/BehavioralPattern/Command_Pattern.png)

### 1.8. 访问者模式(Visitor)
我们使用了一个访问者类，它改变了元素类的执行算法。通过这种方式，元素的执行算法可以随着访问者改变而改变。

![FixMe Add Visitor Image](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/BehavioralPattern/Visitor_Pattern.png)


### 1.9. 责任链模式(Chain of Responsibility)
在这种模式中，通常每个接收者都包含对另一个接收者的引用。如果一个对象不能处理该请求，那么它会把相同的请求传给下一个接收者，依此类推。
职责链上的处理者负责处理请求，客户只需要将请求发送到职责链上即可，无须关心请求的处理细节和请求的传递，所以职责链将请求的发送者和请求的处理者解耦了。

**优点**：  
    - 降低耦合度,它将请求的发送者和接收者解耦   
    - 简化了对象。使得对象不需要知道链的结构  
    - 增强给对象指派职责的灵活性。通过改变链内的成员或者调动它们的次序，允许动态地新增或者删除责任  

**缺点**：  
    - 不能保证请求一定被接收    
    - 系统性能将受到一定影响，而且在进行代码调试时不太方便，可能会造成循环调用  

![FixMe add Chain of resbonsibility Image](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/BehavioralPattern/Chain_of_Responsibility_Pattern.png)

每个记录器消息的级别是否属于自己的级别，如果是则相应地打印出来，否则将不打印并把消息传给下一个记录器。通过SetNextLogger（）设定下一个记录器。

### 1.10.迭代器模式（Iterator）
迭代器模式用于顺序访问集合对象的元素，不需要知道集合对象的底层表示。把在元素之间游走的责任交给迭代器，而不是聚合对象。我们可以使用STL中预定义好的Iterator（Vector,set）等。


### 1.11.解释器模式（Interpreter）
解释器模式提供了评估语言的语法或表达式的方式,对于一些固定文法构建一个解释句子的解释器。

**优点**：  
    - 可扩展性比较好，灵活  
    - 增加了新的解释表达式的方式    

**缺点**：  
    - 可利用场景比较少  
    - 对于复杂的文法比较难维护  
    - 解释器模式会引起类膨胀    
    - 解释器模式采用递归调用方法。


![FixMe add Interpreter Pattern Image](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/BehavioralPattern/Interpreter_Pattern.png)

