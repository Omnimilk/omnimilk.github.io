---
title: expression template and CRTP
date: 2020-07-01 08:52:02
category: 编程
tags: c++
---

# 表达式模板和静态多态

表达式模板（[expression template](https://www.en.wikipedia.org/wiki/Expression_templates)）是一种模板元编程技术，在编译期间延迟计算的求值，并构造表达计算的结构（expression tree）。利用expression tree的变换，可以实现运行前的自动循环融合（loop fusion）等功能。

静态多态（也叫[CRTP](https://www.en.wikipedia.org/wiki/Curiously_recurring_template_pattern)）用一个模板基类来在编译期完成多态的实际实现的分发(与运行时使用`virtual` `override`的多态机制相对, 因此叫静态多态)，不同类型的子类将自身类型作为基类的模板参数来继承，以此通知基类将对应类型的调用分发给自己。静态多态消除了动态多态的开销，可以提升运行时性能，经常和表达式模板一起使用。

## 静态多态demo

下面以一个简单demo展示上述静态多态的原理和用法：

```c++
#include <iostream>
#include <vector>

template <class T>
struct Animal {
  void Say() {
    static_cast<T*>(this)->Say_();
  }
};

struct Dog : Animal<Dog> {
  void Say_() {
    std::cout << "wang wang wang!" << std::endl;
  }
};

struct Cat : Animal<Cat> {
  void Say_() {
    std::cout << "meow meow meow~" << std::endl;
  }
};

class Zoo {
 public:
  template <class T>
  void AddAnimal(Animal<T> animal) {
    animal.Say();
  }
};

int main() {
  Dog dog{};
  Cat cat{};
  Zoo zoo{};
  zoo.AddAnimal(dog);
  zoo.AddAnimal(cat);
  return 0;
}
```

## 表达式模板demo

[wiki](https://www.en.wikipedia.org/wiki/Expression_templates)上给出了一个表达式模板很好的例子，对vector加法进行循环融合。原理大致如下：

1. 将+法操作符封装为表达式求和类型VecSum的构造函数，因此在+法时不进行实际的求值，而是进行表达式树的构建
2. 在对Vec进行赋值操作时进行表达式求值，此时对VecSum表达式中的`[]`操作符递归调用，直到完成实际的求和。

代码如下：

```c++
#include <iostream>
#include <vector>
#include <assert.h>

template <typename E>
class VecExpression {
  public:
    double operator[](size_t i) const {
      return static_cast<E const&>(*this)[i];
    }
    size_t size() const {return static_cast<E const&>(*this).size();}
};

class Vec : public VecExpression<Vec> {
  std::vector<double> elems;

 public:
  double operator[](size_t i) const { return elems[i];}
  double &operator[](size_t i) { return elems[i]; }
  size_t size() const { return elems.size();}

  Vec(size_t n) : elems(n) {}
  Vec(std::initializer_list<double> init) : elems(init) {};

  // 从VecExpression构建Vec, 此时对表达式求值
  template <typename E>
  Vec(VecExpression<E> const& expr) : elems(expr.size()) {
    for (size_t i = 0; i != expr.size(); i++) {
      elems[i] = expr[i];
    }
  }  

  void dump() {
    for (auto &elem : elems) {
      std::cout << elem << " ";
    }
    std::cout << std::endl;
  }
};

// 静态多态
template <typename E1, typename E2>
class VecSum : public VecExpression<VecSum<E1, E2>> {
  E1 const& _u;
  E2 const& _v;

 public:
  VecSum(E1 const &u, E2 const &v) : _u(u), _v(v) {
    assert(u.size() == v.size());
  }
 // Vec的构造函数中被调用，此时求值
  double operator[](size_t i) const { return _u[i] + _v[i]; }
  size_t size() const { return _v.size(); }
};

// 将VecSum的构造函数封装成+运算符
template <typename E1, typename E2>
VecSum<E1, E2> operator+(VecExpression<E1> const &u, VecExpression<E2> const &v) {
  std::cout << __PRETTY_FUNCTION__ << std::endl;
  return VecSum<E1, E2>(*static_cast<const E1*>(&u), *static_cast<const E2*>(&v));
}

int main() {
  Vec v0 = {1, 1, 1};
  Vec v1 = {2, 2, 2};
  Vec v2 = {3, 3, 3};
  Vec v3 = {4, 5, 6};

  Vec sum = v0 + v1 + v2 + v3;
  sum.dump();
  return 0;
}
```

输出如下：

```
VecSum<E1, E2> operator+(const VecExpression<E>&, const VecExpression<E2>&) [with E1 = Vec; E2 = Vec]
VecSum<E1, E2> operator+(const VecExpression<E>&, const VecExpression<E2>&) [with E1 = VecSum<Vec, Vec>; E2 = Vec]
VecSum<E1, E2> operator+(const VecExpression<E>&, const VecExpression<E2>&) [with E1 = VecSum<VecSum<Vec, Vec>, Vec>; E2 = Vec]
10 11 12 

```



