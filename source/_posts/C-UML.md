---
title: C++_UML
date: 2017-12-14 17:03:51
tags:
	- UML
categories: UML	
---

## 1. Basic 
### 1.1. Visibility
The C++ class has three visibility. 

- public, using "+"  
- private, using "-"  
- protected, using '#'  

See the base class examole:  
![Base Class](https://raw.githubusercontent.com/JShell07/images/master/C%2B%2B/C%2B%2B_BaseClass.png)

<!-- more -->
### 1.2. Class members
- static members, using <u>underline</u>, like <u>static int member</u>
- virtual functions, using **Italic** font, like ***virtual void functions()***

## 2. Class Relationship
The C++ has these relationship:

- Assocation 
- Dependency
- Aggregation
- Composition 
- Inheritance
- Class template  

![](https://raw.githubusercontent.com/JShell07/images/master/C%2B%2B/C%2B%2B_ClassRelationship.png)

### 2.1. Dependency
可以简单的理解，就是一个类A使用到了另一个类B，而这种使用关系是具有偶然性的、、临时性的、非常弱的，但是B类的变化会影响到A；比如某人要过河，需要借用一条船，此时人与船之间的关系就是依赖。
``` c++
class B {
	public:
		void display(class A* A) {A->display();}
}
```

### 2.2. Assocation 
他体现的是两个类、或者类与接口之间语义级别的一种<font color=red>强依赖关系</font>，比如我和我的朋友；这种关系比依赖更强、不存在依赖关系的偶然性、关系也不是临时性的，一般是长期性的
``` c++
class B
{
public:
    B(){}
    void setA(A *A) { ptrA_ = A;}

private/public:
    A* ptrA_; 
}
```
### 2.3. Aggregation
聚合是关联关系的一种特例，他体现的是整体与部分、拥有的关系，__即has-a的关系，此时整体与部分之间是可分离的，他们可以具有各自的生命周期。__
``` c++
class B {
	public:
	private:
		vector<class A*> ptrclassA;
}
```
### 2.4. Composition 
组合也是关联关系的一种特例，__他体现的是一种contains-a的关系，这种关系比聚合更强，也称为强聚合__，但此时整体与部分是不可分的，整体的生命周期结束也就意味着部分的生命周期结束。

``` c++
class B {
	private:
		class A a;
}
```

总的来说，上述关系所表现的强弱程度依次为：__组合>聚合>关联>依赖__

### 2.5. Inheritance

``` c++
class A {
	//..	
};

class B: public A {
	//..
}
```

### 2.6. Class template

``` c++
template<class T>
class A {
	//...
	private:
	T var;
};

class B {
	//..
};

A<B> a;
```

## 参看资料
[继承、实现、依赖、关联、聚合、组合的联系与区别](https://www.cnblogs.com/jiqing9006/p/5915023.html)