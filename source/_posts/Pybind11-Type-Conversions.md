---
title: Pybind11 Type Conversions
date: 2020-07-27 10:17:43
category: 编程
tags: pybind11, c++, python
---

# Pybind11类型转换

Pybind11帮助我们方便的实现C++和Python之间的调用，无论是相互的接口暴露还是数据传递都要求我们正确的在C++和Python类型之间进行转换（[官方文档](https://pybind11.readthedocs.io/en/stable/advanced/cast/overview.html)）。

## 三种场景

1. C++原生类型暴露给python
2. Python原生类型暴露给C++
3. C++和Python原生类型间相互转换

## C++原生类型暴露给Python

使用`py::class_`将自定义的C++类型暴露给Python（[参考这里](https://pybind11.readthedocs.io/en/stable/classes.html)）。C++类型传递给Python时会在原生C++类型外加一层wrapper，从Python取回时只用去掉wrapper即可。

## Python原生类型暴露给C++

在C++中取用Python的原生类型（例如：`tuple`、`list`），例如：

```c++
// C++中使用Python对象
void print_list(py::list my_list) {
  for (auto item : my_list) {
    std::cout << item << " ";
  }
}
```

Python中传入Python对象给C++:

```python
>>> print_list([1, 2, 3])
1 2 3
```

这个例子里Python中的`list`类型没有进行任何转换，只是被封装进了C++的`py::list`类中。目前Pybind11支持以下类型([参考这里](https://pybind11.readthedocs.io/en/stable/advanced/pycpp))：`handle`, `object`, `bool_`, `int_`, `float_`, `str`, `bytes`, `tuple`, `list`, `dict`, `slice`, `none`, `capsule`, `iterable`, `iterator`, `function`, `buffer`, `array`, `array_t`。

## C++和Python原生类型间相互转换

有些场景C++和Python都用的是各自的原生类型，Pybind11支持常见C++原生类型和Python原生类型间的相互转换，例如：

```c++
void print_vector(const std::vector<int> &v) {
  for (auto item : v) {
    std::cout << item << "\n";
  }
}
```

Python中调用：

```
>>> print_vector([1, 2, 3])
1 2 3
```

容易注意到Python中传入的为`list`类型，而C++中处理的类型为`const std::vector<int> &`。Pybind11这种默认类型转换会在原生类型间拷贝数据，例如上面Python中的调用会首先将list中的数据拷贝到一个`std::vector<int>`中，然后进行C++中的调用。

NOTE: 默认的拷贝可能开销很大，可以通过手写[opaque types](https://pybind11.readthedocs.io/en/stable/advanced/cast/stl.html#opaque)重载类型转换的wrapper来避免不必要的拷贝。opaque types还没有使用过，有机会单独挖坑写。

## STL库

Pybind11默认（`pybind11/pybind11.h`）支持`std::pair<>`，`std::tuple<>`和`list`, `set`, `dict`之间的自动转换。引入`pybind11/stl.h`可以新增`std::vector<>`, `std::deque<>`, `std::list<>`, `std::array<>`, `std::set<>`, `std::unordered_set<>`, `std::map<>`和`std::unordered_map<>`的与Python的自动转换。

### 绑定STL容器

`pybind11/stl_bind.h`可以帮助我们将STL容器作为原生对象暴露出来。例如：

```c++
#include <pybind11/stl_bind.h>

PYBIND11_MAKE_OPAQUE(std::vector<int>);
PYBIND11_MAKE_OPAQUE(std::map<std::string, double>);

// ...

// 绑定
py::bind_vector<std::vector<int>>(m, "VectorInt");
py::bind_map<std::map<std::string, double>>(m, "MapStringDouble");
// 或者同时设置绑定的作用域
py::bind_vector<std::vector<int>>(m, "VectorInt", py::module_local(false));
```



