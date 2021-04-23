---
layout: posts
title: LLVM0
date: 2020-09-07 10:29:44
tags: LLVM, compiler
---

# LLVM学习笔记0：安装

算是重拾LLVM，这次重头记录学习历程。

## 安装

安装比较简单，`apt-get`直接安装即可：

```shell
sudo apt-get install clang
sudo apt-get install llvm
```

## 测试安装

使用LLVM CookBook里的一个小例子验证安装是否成功。

demo code:

```
define i32 @test1(i32 %A) {
    %B = add i32 %A, 0
    ret i32 %B
}

define internal i32 @test(i32 %X, i32 %dead) {
    ret i32 %X
}

define i32 @caller() {
    %A = call i32 @test(i32 123, i32 456)
    ret i32 %A
}
```

1. 将上述LLVM代码输入到`testfile.ll`
2. 运行`opt`工具中指令合并的pass：`opt -S -instcombine testfile.ll -o output1.ll`
3. 运行死代码消除pass：`opt -S -deadargelim testfile.ll -o output2.ll`

我们容易通过查看上面运行pass后的输出代码确认pass已生效：

`output1.ll`：

```
; ModuleID = 'tesetfile.ll'

define i32 @test1(i32 %A) {
    ret i32 %A  ; eliminated adding 0
}

define internal i32 @test(i32 %X, i32 %dead) {
    return i32 %X
}

define i32 @caller() {
    %A = call i32 @test(i32 123, i32 456)
    ret i32 %A
}
```

`output2.ll`：

```
; ModuleID = 'testfile.ll'

define i32 @test1(i32 %A) {
    %B = add i32 %A, 0
    ret i32 %B
}

define internal i32 @test(i32 %X) { ; eliminated dead arg
    ret i32 %X
}

define i32 @caller() {
    %A = call i32 @test(i32 123) ; elminated dead arg
    ret i32 %A
}
```

