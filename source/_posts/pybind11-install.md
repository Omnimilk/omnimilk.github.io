---
title: pybind11 install
date: 2020-07-02 19:26:50
categories: 编程
tags: pybind11, c++, python
---

# Pybind11 安装及简单使用

之前项目里用到了pybind11，效果强大&很好用。但是由于那个项目整个工程直接包含了pybind11的头文件和构建脚本，因此无需自己动手折腾pybind11的环境和构建。最近自己想用pybind11做些POC的小实验，在此记录一些搭建环境的过程。

## 安装

1. 依赖安装：`sudo apt-get install python-dev cmake`
2. pybind11 python: `conda install pybind11`
3. pybind11 安装：
   1. 下载repo：`git clone https://github.com/pybind/pybind11.git`
   2. `cd pybind11`
   3. `mkdir build`
   4. `cd build`
   5. `cmake ..`
   6. `make check -j8`

## 使用

尝试编译官网的玩具样例，代码如下（保存到文件`toy.cc`）:

```c++
#include <pybind11/pybind11.h>

int add(int i, int j) {
  return i + j;
}

PYBIND11_MODULE(example, m) {
  m.doc() = "pybind11 example plugin";
  m.def("add", &add, "A function which adds two numbers");
}
```

编译命令：

```
c++ -O3 -Wall -shared -std=c++11 -fPIC `python3 -m pybind11 --includes` toy.cc -o shared`python3-config --extension-suffix`
```

NOTE: 上述命令初看很唬人，我们尝试运行一下`python3 -m pybind11 --includes`和`python3-config --extension-suffix`, 得到的结果如下：`-I/home/xxx/anaconda3/envs/mindspore/include/python3.7m -I/home/xxx/anaconda3/envs/mindspore/include`, `.cpython-37m-x86_64-linux-gnu.so`。