---
title: Pybind11 classes
date: 2020-07-27 14:10:51
categories: 编程
tags: pybind11, c++, python
---

# Pybind11 class绑定

最近工作需要用到Pybind11 class绑定中一些较为高级的特性（虚函数、重载、继承），在此整理记录一下。

## 基本用法

class绑定对应[上一篇](https://omnimilk.github.io/2020/07/27/Pybind11-Type-Conversions/)中的第一种场景，即将C++原生类型通过`class_`函数向Python暴露。下面用一个小例子展示class绑定最基本的用法：

```c++
#include <pybind11/pybind11.h>
namespace py = pybind11;

struct Pet {
    Pet(const std::string &name) : name(name) { }
    void setName(const std::string &name_) { name = name_; }
    const std::string &getName() const { return name; }

    std::string name;
};

PYBIND11_MODULE(example, m) {
    py::class_<Pet>(m, "Pet")
        .def(py::init<const std::string &>())
        .def("setName", &Pet::setName)
        .def("getName", &Pet::getName);
}
```

上面的代码向Python暴露了`Pet`类，其中有三个方法：`__init__`, `setName`, `getName`。

### 设置关键字参数和默认参数

在def函数时可以添加参数的名字及设置默认值。

```c++
#include <pybind11/pybind11.h>

int add(int i, int j) {
  return i + j;
}

PYBIND11_MODULE(example, m) {
  //m.def("add", &add, "A function which adds two numbers");
  // 变为关键字参数
  m.def("add", &add, "A function which adds two numbers", py::arg("i"), py::arg("j"));
}
// python调用：example.add(i=1,  j=2)
```

设置默认值类似（NOTE: C++接口中定义的默认值不会自动捕获，需要bind时设置）：

```c++
PYBIND11_MODULE(example, m) {
  // 设置参数默认值
  m.def("add", &add, "A function which adds two numbers", py::arg("i") = 1, py::arg("j") = 2);
}
```

### 绑定Lambda函数

上面的方法绑定是将Python方法绑定到方法指针上，实际上可以用Lambda函数替换。例如，给Pet类增加一个`__repr__`方法：

```c++
py::class_<Pet>(m, "Pet")
    .def(py::init<const std::string &>())
    .def("setName", &Pet::setName)
    .def("getName", &Pet::getName)
    .def("__repr__",
        [](const Pet &a) {
            return "<example.Pet named '" + a.name + "'>";
        }
    );
```

### 绑定成员变量

C++中公有成员变量可以绑定为Python中可以读写的attributes（const成员绑定为只读attributes），例如：

```c++
PYBIND11_MODULE(example, m) {
    py::class_<Pet>(m, "Pet")
        .def(py::init<const std::string &>())
        .def("setName", &Pet::setName)
        .def("getName", &Pet::getName)
        .def_readwrite("name", &Pet::name); //公有非const成员变量
}
// Python中调用：
// >>> p = example.Pet("Molly")
// >>> p.name
// >>> p.name = "Charly"
```

私有成员变量可以通过绑定C++中的getter和setter成为Python中的property：

```c++
py::class_<Pet>(m, "Pet")
  .def(py::init<const std::string &>())
  .def_property("name", &Pet::getName, &Pet::setName);
  ...
```

类似的方法还有：`def_readwrite_static()`, `def_readonly_static()`, `def_property_static()`, `def_property_readonly_static()`

NOTE: Python可以动态的添加属性，绑定时也可以使能C++暴露的类在Python中支持动态属性：

```c++
// py::dynamic_attr()支持动态属性
py::class_<Pet>(m, "Pet", py::dynamic_attr())
    .def(py::init<>())
    .def_readwrite("name", &Pet::name);

// Python中动态向Pet添加属性
// >>> p = example.Pet()
// >>> p.name = "charly"
// >>> p.age = 2 # 动态添加age属性
```

### 保持继承关系

C++中的继承关系有两种方式保留到暴露到Python的类中，下面以一个例子说明。

```c++
struct Pet {
    Pet(const std::string &name) : name(name) { }
    std::string name;
};

struct Dog : Pet {
    Dog(const std::string &name) : Pet(name) { }
    std::string bark() const { return "woof!"; }
};
```

第一种方法是子类暴露时在`class_`的模板中指定基类

```c++
py::class_<Pet>(m, "Pet")
   .def(py::init<const std::string &>())
   .def_readwrite("name", &Pet::name);

// Method 1: template parameter:
py::class_<Dog, Pet /* <- specify C++ parent type */>(m, "Dog")
    .def(py::init<const std::string &>())
    .def("bark", &Dog::bark);
```

第二种方法是在子类暴露时在`class_`的参数中传入基类的`class_`闭包

```c++
py::class_<Pet> pet(m, "Pet");
pet.def(py::init<const std::string &>())
   .def_readwrite("name", &Pet::name);

// Method 2: pass parent class_ object:
py::class_<Dog>(m, "Dog", pet /* <- specify Python parent type */)
    .def(py::init<const std::string &>())
    .def("bark", &Dog::bark);
```

### 自动向下类型转换（downcasting）

自动向下类型转换指的是**多态类型**的基类指针形式返回子类对象，此指针被自动识别并转换为子类的指针。需要注意的是，上面的`Pet`和`Dog`并不是多态类型，因为`Pet`中没有定义虚函数。

下面以`Pet`和`Dog`为例展示**没有**自动downcasting时的行为：

```c++
// 返回指向子类对象的基类指针
m.def("pet_store", []() { return std::unique_ptr<Pet>(new Dog("Molly")); });

// 没有自动downcasting时的行为
// >>> p = example.pet_stor()
// >>> type(p)
Pet                                          # 没有被识别为Dog
// p.bark()
AttributeError: 'Pet' object has no attribute 'bark'
```

我们可以向`Pet`中添加一个虚函数来通知Pybind11这种多态关系：

```c++
struct PolymorphicPet {
    virtual ~PolymorphicPet() = default;
};

struct PolymorphicDog : PolymorphicPet {
    std::string bark() const { return "woof!"; }
};

// Same binding code
py::class_<PolymorphicPet>(m, "PolymorphicPet");
py::class_<PolymorphicDog, PolymorphicPet>(m, "PolymorphicDog")
    .def(py::init<>())
    .def("bark", &PolymorphicDog::bark);

// Again, return a base pointer to a derived instance
m.def("pet_store2", []() { return std::unique_ptr<PolymorphicPet>(new PolymorphicDog); });

// Python中调用
// >>> p = example.pet_store2()
// >>> type(p)
// PolymorphicDog  # automatically downcast
// >>> p.bark()
// u'woof!'
```

### 方法重载

C++中重载的方法直接通过方法名取指针会有歧义，Pybind11有两种方法消除这种歧义。

```c++
struct Pet {
    Pet(const std::string &name, int age) : name(name), age(age) { }

    void set(int age_) { age = age_; }
    void set(const std::string &name_) { name = name_; }

    std::string name;
    int age;
};

// 方法1：指定方法指针类型，C++11+支持
py::class_<Pet>(m, "Pet")
   .def(py::init<const std::string &, int>())
   .def("set", (void (Pet::*)(int)) &Pet::set, "Set the pet's age")
   .def("set", (void (Pet::*)(const std::string &)) &Pet::set, "Set the pet's name");
// 方法2：py::overload_cast自动推导返回值类型， C++14+支持
py::class_<Pet>(m, "Pet")
    .def("set", py::overload_cast<int>(&Pet::set), "Set the pet's age")
    .def("set", py::overload_cast<const std::string &>(&Pet::set), "Set the pet's name");
```

### 枚举类型

```c++
struct Pet {
    enum Kind {
        Dog = 0,
        Cat
    };

    Pet(const std::string &name, Kind type) : name(name), type(type) { }

    std::string name;
    Kind type;
};

py::class_<Pet> pet(m, "Pet");

pet.def(py::init<const std::string &, Pet::Kind>())
    .def_readwrite("name", &Pet::name)
    .def_readwrite("type", &Pet::type);

py::enum_<Pet::Kind>(pet, "Kind")
    .value("Dog", Pet::Kind::Dog)
    .value("Cat", Pet::Kind::Cat)
    .export_values();

// Python 调用：
// >>> p = Pet('Lucy', Pet.Cat)
// >>> p.type
// Kind.Cat
// >>> int(p.type)
// 1L
```

## 高级特性

### 覆写虚函数

直接以例子来说明：

```c++
class Animal {
public:
    virtual ~Animal() { }
    // 纯虚函数， 有纯虚函数的类无法实例化， 因此无法定义构造函数
    virtual std::string go(int n_times) = 0;
};

class Dog : public Animal {
public:
    std::string go(int n_times) override {
        std::string result;
        for (int i=0; i<n_times; ++i)
            result += "woof! ";
        return result;
    }
};

// 传入基类指针，调用实际对象的go方法
std::string call_go(Animal *animal) {
    return animal->go(3);
}

// Bad binding! 无法扩展，因为此处Animal类没有构造函数
PYBIND11_MODULE(example, m) {
    py::class_<Animal>(m, "Animal")
        .def("go", &Animal::go);

    py::class_<Dog, Animal>(m, "Dog")
        .def(py::init<>());

    m.def("call_go", &call_go);
}
```

我们可以在C++中新增一个跳转类来解决上述问题

```c++
class PyAnimal : public Animal {
public:
    /* Inherit the constructors */
    using Animal::Animal;

    /* Trampoline (need one for each virtual function) */
    std::string go(int n_times) override {
        /* 对纯虚函数使用此宏 */
        /* 对有默认实现的虚函数，使用PYBIND11_OVERLOAD*/
        PYBIND11_OVERLOAD_PURE(
            std::string, /* Return type */
            Animal,      /* Parent class */
            go,          /* Name of function in C++ (must match Python name) */
            n_times      /* Argument(s) */
        );
    }
};

PYBIND11_MODULE(example, m) {
    py::class_<Animal, PyAnimal /* <--- trampoline*/>(m, "Animal")
        .def(py::init<>())
        .def("go", &Animal::go);

    py::class_<Dog, Animal>(m, "Dog")
        .def(py::init<>());

    m.def("call_go", &call_go);
}
/*
Python中进行扩展和调用：
>>> from example import *
>>> d = Dog()
>>> call_go(d)
u'woof! woof! woof! '
>>> class Cat(Animal):
...     def go(self, n_times):
...             return "meow! " * n_times
...
>>> class Dachshund(Dog):
...        def __init__(self, name):
...            Dog.__init__(self) # init c++ part
...            self.name = name
...        def bark(self):
...            return "yap!"
>>> c = Cat()
>>> call_go(c)
u'meow! meow! meow! '
*/
```

NOTE: 用Python类扩展C++暴露出来的类时，不要使用`super()`, 不要使用`super()`, 不要使用`super()`! 应该直接使用对应类的`__init__`方法， 因为Python的方法解析顺序（MRO）和C++不一致。

### 绑定虚函数和继承

继续直接从例子开始

```c++
class Animal {
public:
    virtual std::string go(int n_times) = 0;
    virtual std::string name() { return "unknown"; }
};
class Dog : public Animal {
public:
    std::string go(int n_times) override {
        std::string result;
        for (int i=0; i<n_times; ++i)
            result += bark() + " ";
        return result;
    }
    virtual std::string bark() { return "woof!"; }
};

// 定义两个跳转类， 覆写虚函数
class PyAnimal : public Animal {
public:
    using Animal::Animal; // Inherit constructors
    std::string go(int n_times) override { PYBIND11_OVERLOAD_PURE(std::string, Animal, go, n_times); }
    std::string name() override { PYBIND11_OVERLOAD(std::string, Animal, name, ); }
};
class PyDog : public Dog {
public:
    using Dog::Dog; // Inherit constructors
    std::string go(int n_times) override { PYBIND11_OVERLOAD(std::string, Dog, go, n_times); }
    std::string name() override { PYBIND11_OVERLOAD(std::string, Dog, name, ); }
    std::string bark() override { PYBIND11_OVERLOAD(std::string, Dog, bark, ); }
};
```

NOTE: 上面通过pybind11注册过的类的子类都需要定义跳转类来覆写父类中的虚函数，当子类较多或者虚函数较多时，可以使用模板类来避免大量重复代码（参考[这里](https://pybind11.readthedocs.io/en/stable/advanced/classes.html#combining-virtual-functions-and-inheritance)）。

### 绑定自定义构造函数

使用`py::init<Args, ...>()`或者`py::init_alias<Args, ...>()`绑定构造函数较方便，但有时我们需要绑定自定义的方法作为构造函数（例如：工厂方法，单例获取静态方法）。

下面的代码展示多种将C++方法绑定为Python构造函数的途径：

```c++
class Example {
private:
    Example(int); // private constructor
public:
    // Factory function - returned by value:
    static Example create(int a) { return Example(a); }

    // These constructors are publicly callable:
    Example(double);
    Example(int, int);
    Example(std::string);
};

py::class_<Example>(m, "Example")
    // Bind the factory function as a constructor:
    .def(py::init(&Example::create))
    // Bind a lambda function returning a pointer wrapped in a holder:
    .def(py::init([](std::string arg) {
        return std::unique_ptr<Example>(new Example(arg));
    }))
    // Return a raw pointer:
    .def(py::init([](int a, int b) { return new Example(a, b); }))
    // You can mix the above with regular C++ constructor bindings as well:
    .def(py::init<double>())
    ;
```

#### 有虚函数跳转类的构造函数

两种方法：

1. 以右值引用的方式将基类值传给子类构造函数
2. 向`py::init<>`提供两个工厂函数，第一个在不需要子类时调（即暴露的类只在Python中被使用而没有被继承），第二个在需要子类时被调用

两种方式的demo如下：

```c++
#include <pybind11/factory.h>
class Example {
public:
    // ...
    virtual ~Example() = default;
};
class PyExample : public Example {
public:
    using Example::Example;
    // 第一种方式：跳转类以右值引用方式接收基类
    PyExample(Example &&base) : Example(std::move(base)) {}
};
py::class_<Example, PyExample>(m, "Example")
    // Returns an Example pointer.  If a PyExample is needed, the Example instance will be moved via the extra constructor in PyExample, above.
    .def(py::init([]() { return new Example(); }))
    // 第二种方式：提供两个工厂函数
    .def(py::init([]() { return new Example(); } /* no alias needed */,
                  []() { return new PyExample(); } /* alias needed */))
    // *Always* returns an alias instance (like py::init_alias<>())
    .def(py::init([]() { return new PyExample(); }))
    ;
```

### 多重继承

在`class_`的模板参数中指定所有的基类即可：

```c++
py::class_ <MyType, BaseType1, BaseType2, BaseType3>(m, "MyType")
...
```



