---
title: cpp variadic template
date: 2020-06-30 16:49:10
categories: 编程
tags:
- c++
---

# C++变长模板使用

C++11开始支持支持变长模板（variadic template），模板参数可以为任意多个, 变长模板参数用省略号来标识(...)。

变长模板参数通常以递归的形式打开（因此需要定义一个没有变长参数的base case模板），下面借用一个[例子](https://www.raymii.org/s/snippets/Cpp_variadic_template_recursive_example.html)来说明:

```c++
#include <iostream>

template <typename Arg>
void Foo(Arg arg) {
  std::cout << __PRETTY_FUNCTION__ << "\n";
}

template <typename First, typename... Args>
void Foo(First first, Args... args) {
  std::cout << __PRETTY_FUNCTION__ << "\n";
  Foo(first);
  Foo(args...);
}

int main() {
  std::string one = "one";
  const char* two = "two";
  Foo(one);
  std::cout << std::endl;
  Foo(one, two);
  return 0;
}
```

上面代码的输出如下：

```
void Foo(Arg) [with Arg = std::__cxx11::basic_string<char>]

void Foo(First, Args ...) [with First = std::__cxx11::basic_string<char>; Args = {const char*}]
void Foo(Arg) [with Arg = std::__cxx11::basic_string<char>]
void Foo(Arg) [with Arg = const char*]
```

## 求和

下面尝试使用变长模板来实现一个求和的函数。

```c++
#include <iostream>

template <typename Arg>
Arg Sum(Arg arg) {
  std::cout << __PRETTY_FUNCTION__ << "\n";
  return arg;
}

template <typename First, typename... Args>
First Sum(First first, Args... args) {
  std::cout << __PRETTY_FUNCTION__ << "\n";
  return first + Sum(args...);
}

int main() {
  std::cout << Sum(1, 2, 3, 4, 5) << std::endl;
  return 0;
}
```

得到的结果如下：

```c++
First Sum(First, Args ...) [with First = int; Args = {int, int, int, int}]
First Sum(First, Args ...) [with First = int; Args = {int, int, int}]
First Sum(First, Args ...) [with First = int; Args = {int, int}]
First Sum(First, Args ...) [with First = int; Args = {int}]
Arg Sum(Arg) [with Arg = int]
15
```

