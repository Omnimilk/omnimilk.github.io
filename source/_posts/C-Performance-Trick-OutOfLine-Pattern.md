---
layout: posts
title: 'C++ Performance Trick: OutOfLine Pattern'
date: 2020-08-14 10:54:58
tags: Performance, C++
---

# OutOfLine模式提升C++代码访存局部性

公司内网看到一个提升C++代码性能的模式（源头是[这篇blog](https://blog.headlandstech.com/2018/08/15/outofline-a-memory-locality-pattern-for-high-performance-c/)），能够以一种容易维护的方式很大程度提升访存的局部性，感觉挺有意思，记录一下。

## 从例子说起

假设你的系统需要打开大量的路径(文件、sockets，pipes)，对于这些路径你需要打开、处理、关闭对应的文件描述符并取消文件描述符和对应路径之间的联系。

对于这样的需求，我们可能写出下面的代码：

```c++
class UnlinkingFD {
  std::string path;
 public:
  int fd;

  UnlinkingFD(const std::string& p) : path(p) {
    fd = open(p.c_str(), O_RDWR, 0);
  }
  ~UnlinkingFD() {
      close(fd); 
      unlink(path.c_str());
  }
  UnlinkingFD(const UnlinkingFD&) = delete;
};
```

每个打开的路径对应一个上面的对象保存对应的路径和文件描述符，大量的打开的文件描述符可能以`UnlinkingFD`对象形式保存在数组中。上面的代码在功能和逻辑上是完好的，RAII也会在对象生命周期结束时关掉文件描述符并取消和对应路径之间的关联。

### 场景&性能分析

假设我们经常需要用到`fd`，但是`path`只在对象生命周期结束时使用。因为一个上面的`UnlinkingFD`对象需要占用40Bytes，而我们常用的`fd`只占用了其中的4Bytes，这意味着我们在访问`UnlinkingFD`数组时cache miss的概率大大提升。

### 方案1

将`array-of-structs`变为`struct-of-arrays`，这当然可以帮助提升性能，但是我们将无法使用RAII来帮我们做资源管理，单一的fd无法及时得到释放。

### 方案2

`UnlinkingFD`对象中不直接保存`string`对象，而是使用一个`unique_ptr`指针。这样一个`UnlinkingFD`对象就从40Bytes降低到了16Bytes，降低了cache miss的概率。

### 方案3：OutOfLine模式

OutOfLine模式可以帮我们将像`path`这样的冷数据完全移到对象之外，但是仍然保留RAII为我们管理资源。

#### OutOfLine用法

`OutOfLine`类是一个[静态多态](https://omnimilk.github.io/2020/07/01/expression-template/)的基类。 使用此基类时要传入两个模板参数，第一个模板参数为继承的子类，第二个模板参数为要挂在“热”对象上的冷数据的类型。

例如，对于上面的`UnlinkingFD`类，改写后的用法如下：

```c++
struct UnlinkingFD : private OutOfLine<UnlinkingFD, std::string> {
  int fd;

  UnlinkingFD(const std::string& p) : OutOfLine<UnlinkingFD, std::string>(p) {
    fd = open(p.c_str(), O_RDWR, 0);
  }
  ~UnlinkingFD();
  UnlinkingFD(const UnlinkingFD&) = delete;
};
```

#### 原理

OutOfLine模式将冷数据从对象中剥离出去的思路时通过一个全局对象将“热”对象的指针和冷数据关联起来。

```c++
template <class FastData, class ColdData>
class OutOfLine {
inline static std::map<OutOfLine const*, std::unique_ptr<ColdData>> global_map_;
 public:
    // 每次创建“热”数据对象时，将热数据对象的指针关联到对应的冷数据上
    template<class... TArgs>
    explicit OutOfLine(TArgs&&.. args) {
        global_map_[this] = std::make_unique<ColdData>(std::forward<TArgs>(args)...);
    }
    // “热”数据对象析构时，将全局映射中对应的冷数据对象也清理掉
    ~OutOfLine() {global_map_.erase(this);}
    // move “热”数据对象时，将对应的冷数据重新关联（此后不应该在通过旧的“热”数据对象来找冷数据）
    explicit OutOfLine(OutOfLine&& other) { *this = other;}
    OutOfLine& operator=(OutOfLine&& other) {
        global_map_[this] = std::move(global_map_[&other]);
        return *this;
    }
    // non-copyable for simplicity
    OutOfLine(OutOfLine const&) = delete;
    OutOfLine& operator=(OutOfLine const&) = delete;
    // 获取当前“热”对象对应的冷数据
    ColdData& cold() noexcept { return *global_map_[this]; }
    ColdData const& cold() const noexcept { return *global_map_[this]; }
}
```

#### 性能&资源管理

静态继承自`OutOfLine`类的`UnlinkingFD`现在只有4Bytes， 因此需要高频访问的`UnlinkingFD`对象的数组的cache miss被降到最低。同时RAII会同时替我们管理“热”数据对象及对应的冷数据。

#### NOTE: 构造函数初始化顺序

上面静态继承的过程实际上一定程度上限定了热数据和冷数据构造的顺序，因为C++构造函数初始化的顺序如下：

1. 构造虚拟基类，继承自多个虚拟基类时，按照被继承的顺序构造
2. 构造非虚拟基类，继承自多个非虚拟基类时，按照被继承的顺序构造
3. 构造成员对象，按照成员对象的声明顺序构造
4. 调用类自身的构造函数

因此，当我们构造一个“热”数据对象时，会首先构造`OutOfLine`基类（此时`OutOfLine`类的构造函数中会构造冷数据对象），然后构造“热”数据对象。这就意味着***冷数据对象的构造不能依赖于“热”数据对象***。

#### 控制初始化顺序

当冷数据对象的初始化依赖于热数据对象时，我们需要hack一下初始化的顺序。方法如下：为`OutOfLine`静态基类提供一个特别的构造函数，这个构造函数以tag类`TwoPhaseInit`为入参。当我们调用这个构造函数时，冷数据不会被初始化，此时处于一种半构造的状态。然后我们在“热”数据初始化之后显式的调用`init_cold_data`来初始化冷数据。释放时同理可以引入`release_cold_data`来hack析构的顺序。代码如下：

```c++
struct TwoPhaseInit {};
  OutOfLine(TwoPhaseInit){}
  template <class... TArgs>
  void init_cold_data(TArgs&&... args) {
    global_map_.find(this)->second = std::make_unique<ColdData>(std::forward<TArgs>(args)...);
  }
  void release_cold_data() { global_map_[this].reset(); }
```

### 性能测试

#### `OutOfLine`代码

```c++
#pragma once
#include <map>
#include <memory>

template <class FastData, class ColdData>
class OutOfLine {
  inline static std::map<OutOfLine const*, std::unique_ptr<ColdData>> global_map_;
 public:
  template <class... TArgs>
  OutOfLine(TArgs&&... args) : OutOfLine(TwoPhaseInit()) {
    init_cold_data(std::forward<TArgs>(args)...);
  }
  ~OutOfLine() { global_map_.erase(this); }
  explicit OutOfLine(OutOfLine&& other) : OutOfLine(OutOfLine::TwoPhaseInit()) { (*this) = std::move(other); }
  OutOfLine& operator=(OutOfLine&& other) {
    global_map_[this] = std::move(global_map_[&other]);
    return *this;
  }
  OutOfLine(OutOfLine const&) = delete;
  OutOfLine& operator=(OutOfLine const&) = delete;

  ColdData& cold() noexcept { return *global_map_[this]; }
  ColdData const& cold() const noexcept { return *global_map_[this]; }

  struct TwoPhaseInit {};
  OutOfLine(TwoPhaseInit) { global_map_.try_emplace(this); }
  template <class... TArgs>
  void init_cold_data(TArgs&&... args) {
    global_map_.find(this)->second = std::make_unique<ColdData>(std::forward<TArgs>(args)...);
  }

  void release_cold_data() { global_map_[this].reset(); }
};
```

#### benchmark代码

```c++
// g++ --std=c++1z -Wall -Wextra -O3 -o ool_benchmark ool_benchmark.cpp
#include "out_of_line.h"
#include <chrono>
#include <iostream>
#include <thread>
#include <vector>

// Deomonstrates keeping the cold data in the object with the fast data
struct Together {
  std::uint32_t value;
  std::string metadata;
};

// Demonstrates "giving up" and just not storing the associated cold-data
struct OnlyFastData {
  std::uint32_t value;
};

// Demonstrates using OutOfLine to store the cold data
struct WithOOL : public OutOfLine<WithOOL, std::string> {
  std::uint32_t value;
};

// The crux of the optimization is that there's no space-overhead
static_assert(sizeof(WithOOL) == sizeof(OnlyFastData));

// We synthesize the data up-front so the generation can't interfere with the measured behavior in any way
template <class Data>
auto make_data() {
  srand(20180101);
  std::vector<Data> data(10000000);
  for (Data& d : data) { d.value = rand(); }
  return data;
}

// We time to measure throughput of touching all the fast data once in sequence
template <class Data>
void trial(const char* const name, const Data& data, const bool print) {
  std::uint32_t running = 0U;
  const auto before = std::chrono::system_clock::now();
  for (const auto& d : data) { running += d.value; }
  const auto after = std::chrono::system_clock::now();

  if (print) {
    std::cout << name << " took " << (after - before).count() << "ns and " << running
              << " is a value I don't want optimized away" << std::endl;
  }
}

int main() {
  // We generate all the datasets up front, to keep the effects of the allocator distant
  const auto together  = make_data<Together>();
  const auto only_fast = make_data<OnlyFastData>();
  const auto with_ool  = make_data<WithOOL>();

  // We run each trial twice, recording the results only the second time. The idea is to give a fair comparison, where
  // all 3 options have their caches primed and nobody thus has data fresher from the allocation.
  trial("With cold data in-line (original)               ", together, false);
  trial("With cold data in-line (original)               ", together, true);

  trial("With cold data thrown away (best-case scenario) ", only_fast, false);
  trial("With cold data thrown away (best-case scenario) ", only_fast, true);

  trial("With OutOfLIne                                  ", with_ool, false);
  trial("With OutOfLIne                                  ", with_ool, true);
return 0;
}
```

#### 结果

注意map的`try_emplace`方法是C++17引入的新特性，编译命令如下：`g++ benchmark.cpp -o benchmark -std=c++17`。在我自己的笔记本上运行的结果如下：

```
With cold data in-line (original)                took 82936589ns and 3350498669 is a value I don't want optimized away
With cold data thrown away (best-case scenario)  took 74243139ns and 3350498669 is a value I don't want optimized away
With OutOfLIne                                   took 73445932ns and 3350498669 is a value I don't want optimized away
```

加速效果没有原始博客中的那么大，但付出的成本近乎免费，因此在性能优化时是很值得考虑的一个选项。



