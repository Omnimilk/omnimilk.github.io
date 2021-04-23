---
title: rvalue reference and emplace
date: 2020-07-03 11:37:11
categories: 编程
tags: c++
---

# 右值引用和emplace

这篇文章初衷是好奇`push_back`和`emplace_back`的区别，了解之后发现绕不开右值引用，在此一并记录一下。

## 右值引用

右值引用（`rvalue reference`）是C++11中引入的新特性，显然是与C++11之前普通左值引用相对的一个概念。下面的右值引用的介绍很多参考自[这篇文章](http://thbecker.net/articles/rvalue_references/section_02.html)。

### 左值与右值

一个粗略的定义如下：

左值是一个可以出现在赋值符号（`=`）左边或者右边的表达式, 可以理解为对一块内存的引用；

右值是一个只能出现在赋值符号右边的表达式， 注意右值不是对内存的引用，因此不能进行取地址操作。

如：

```c++
int a = 42;
int b = 43;
// a, b 均为左值
a = b;
b = a;
// a + b 为右值
int c = a  + b;
a + b = 42; // error!
// 不能对右值取地址
a  = foo(); // ok, foo() is a rvalue
int* p = &foo(); // invalid, 不能对右值取地址
```

### 为什么要右值引用？

#### move语义

假设类X中包括一个指向资源的指针`m_pResource`，我们想实现一个接收**临时对象**作为参数的拷贝赋值操作符，其实现可能如下：

```c++
X& X::operator=(const X &rhs) {
  //1. 拷贝rhs.m_pResource
  //2. 析构rhs.m_pResource指向的资源
  //3. 将拷贝的资源赋给self.m_pResource
}
```

不难看出，上面对资源`m_pResource`的拷贝和析构十分低效，我们可以直接和临时实例交换指针（此所谓move语义）。另外，对于非临时对象的拷贝，我们可能不想析构其资源。因此我们需要一个类型来标识这样的临时对象，对它进行单独的处理，例如：

```
X& X::operator=( <Desired type>rhs) {
  //1. swap this->m_pResource and rhs.m_pResource
}
```

其实右值引用就是我们想要的类型，上面代码中的`<Desired type>`我们可以用`X&&`替代，表示是X的右值引用。

## emplace_back or push_back?

### 一个小实验

猜猜看下面的代码会进行几次复制操作？

```c++
#include <iostream>
#include <vector>

struct Point {
  float x, y;
  Point(float x, float y):x(x), y(y){}

  Point(const Point& point):x(point.x), y(point.y) {
    std::cout << "copying!" << std::endl;
  }
};

int main() {
  std::vector<Point> points;
  points.push_back(Point(1, 2));
  points.push_back(Point(3, 4));
  points.push_back(Point(5, 6));
  return 0;
}
```

答案是六次，其中三次是将临时`Point`对象拷贝至容器，有一次是容器容量从1到2的拷贝，有两次是容器容量从2到4的拷贝。

#### 消除扩容拷贝

```c++
#include <iostream>
#include <vector>

struct Point {
  float x, y;
  Point(float x, float y):x(x), y(y){}

  Point(const Point& point):x(point.x), y(point.y) {
    std::cout << "copying!" << std::endl;
  }
};

int main() {
  std::vector<Point> points;
  // 预分配内存 NOTE: 和std::vector<Point> points(3)的区别，reserve只分配内存不调用构造函数
  points.reserve(3);
  points.push_back(Point(1, 2));
  points.push_back(Point(3, 4));
  points.push_back(Point(5, 6));
  return 0;
}
```

现在只用进行三次插入容器时的拷贝

#### move语义消除插入容器时的拷贝

直接将参数forward给容器，直接在容器中构造。

```c++
#include <iostream>
#include <vector>

struct Point {
  float x, y;
  Point(float x, float y):x(x), y(y){}

  Point(const Point& point):x(point.x), y(point.y) {
    std::cout << "copying!" << std::endl;
  }
};

int main() {
  std::vector<Point> points;
  points.emplace_back(1, 2);
  points.emplace_back(3, 4);
  points.emplace_back(5, 6);
  return 0;
}
```

现在拷贝的次数为0次！

### 接口

首先看看C++11中两者的接口：

```c++
template< class... Args >
void emplace_back( Args&&... args );

void std::vector<T, Allocator>::push_back( const T& value );
void std::vector<T, Allocator>::push_back( T&& value );
```

注意到接收右值的接口的差别，push_back只能接收Vector中存储类型的右值作为参数，而emplace可以接收变长模板作为参数，尝试为变长模板找到最合适的构造函数直接在容器中构建。

NOTE：实验中的例子，emplace_back的变长参数也可以接收`Point`对象，此时会先调用Point的默认构造函数然后调用复制构造函数。对应代码如下：

```c++
#include <iostream>
#include <vector>

struct Point {
  float x, y;
  Point(float x, float y):x(x), y(y){}

  Point(const Point& point):x(point.x), y(point.y) {
    std::cout << "copying!" << std::endl;
  }
};

int main() {
  std::vector<Point> points;
  points.reserve(3);
  points.emplace_back(Point(1, 2));
  points.emplace_back(Point(3, 4));
  points.emplace_back(Point(5, 6));
  return 0;
}

```



