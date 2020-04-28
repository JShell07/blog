---
title: Design_Pattern_II
date: 2019-01-28 18:03:48
tags: 
 - bookmarks
 - program
categories: bookmarks
---

## 1. 创建模式（Creational Patterns）
### 1.1. 工厂模式（Factory)
**目的**：定义一个创建对象的接口，让其子类自己决定实例化哪一个工厂类，工厂模式使其创建过程延迟到子类进行。
**优点**: 扩展性高，并且屏蔽具体的实现。
**缺点**: 每增加一个产品时，都需要实现具体类和对象实现工厂。

```c++

class Shape
{
    public:
        virtual void draw() {};
};

class Circle: public Shape
{
    public:
        void draw() { /* ... */ };
};

class Square: public Shape
{
    public:
        void draw() { /* ... */ };
};

class ShapeFactory
{
    public:
    class Shape * GetShape(string type) {
        if(type.compare("CIRCLE") == 0)
            return new Circle();
        else if(type.compare("SQUARE") == 0)
            return new Square();
        else 
            return null;
    }
}

void main()
{
    class ShapeFactory shapeFactory = new ShapeFactory();

    class Shape *shape1, shape2;
    
    shape1 = shapeFactory.GetShape("CIRCLE");
    shape1->draw();

    shape2 = shapeFactory.GetShape("SQUARE");
    shape2->draw();

    delete shape1;
    delete shape2;
}
```
![](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/CreationalPattern/Factory_Pattern.png)

<!--more-->

### 1.2. 抽象工厂模式（Abstract Factory)
**目的**：超级工厂又称为其他工厂的工厂，它提供了一种创建对象的最佳方式。

**优点**: 当一个产品族中的多个对象被设计成一起工作时，它能保证客户端始终只使用同一个产品族中的对象。
**缺点**: 产品族扩展困难。既要在抽象的creator 里修改，又要实现具体的代码。

虚基类的感觉。
![](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/CreationalPattern/AbstractFactory_Pattern.png)

AbstrctFactory 与Factory 区别在于：AbstractFactory 模式是为了创建**一组**（有多类）相关或依赖的对象，Factory模式是为**一类**对象提供接口。

### 1.3. 单例模式(Singleton)
**意图**：保证一个类仅有一个实例，并提供一个访问它的全局访问点。
**缺点**：没有接口，不能继承，与单一职责原则冲突，一个类应该只关心内部逻辑，而不关心外面怎么样来实例化。
创建唯一的变量（对象）

[Singleton mode image]

```c++
class Singleton
{
    public:
        static Singleton * Instance()
        {
            if(_instance == 0)
                _instance = new Singleton();

            return _instance;
        }
    protected:
        Singleton();
    private:
        static Singleton * _instance;
}
```

![](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/CreationalPattern/Singleton_Pattern.png)

注：Singleton 不可以被实例化（保证唯一一个），因此我们将它的构造函数声明为protected或private。

### 1.3. 建造者模式（Builder）
**意图**：将一个复杂的构建与其表示相分离，使得同样的构建过程可以创建不同的表示。

**优点**： 
    - 建造者独立，易扩展
    - 便于控制细节风险

**缺点**： 
    - 产品必须有共同点，范围有限制
    - 如内部变化复杂，会有很多的建造类

![Builder](https://raw.githubusercontent.com/JShell07/images/master/DesignPattern/CreationalPattern/Builder_Pattern.png)

注：
Builder 与AbstractFactory 在功能上相似，都用来创建复杂的对象。Builder 是通过的**相同的创建过程获取不同的结果**，而AbstractFactory 强调为多个相互依赖的对象提供一个统一的接口。另外，AbstractFactory 是立即返回，Builder是在一系列动作后返回对象。


### 1.4. 原型模式(Prototype)
原型模式是用于创建重复的对象，同时又能保证性能。这种模式是实现了一个原型接口，该接口用于创建当前对象的克隆。当直接创建对象的代价比较大时，则采用这种模式。
**优点**：  
    - 性能提高      
    - 逃避构造函数的约束    

**缺点**：  
    - 配备克隆方法需要对类的功能进行通盘考虑，这对于全新的类不是很难，但对于已有的类不一定很容易，特别当一个类引用不支持串行化的间接对象，或者引用含有循环结构的时候    
    - 必须实现 clone() 接口 

```c++
class BaseA
{
    public:
    BaseA(){};
    BaseA(const BaseA &A) //c++ 拷贝函数，深拷贝是指重新分配类里面的指针
    {
        member = A.member;
    }

    BaseA * Clone()
    {
        return new BaseA(*this);
    }

    private:
    int member;
}

void main()
{
    BaseA *p1 = new BaseA();
    BaseA *p2 = p1->Clone();
}
```
[C++ 拷贝构造函数(深拷贝，浅拷贝)](https://www.cnblogs.com/BlueTzar/articles/1223313.html)