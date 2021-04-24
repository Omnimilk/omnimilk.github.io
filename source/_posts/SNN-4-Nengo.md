---
title: 'SNN[4]: Nengo'
date: 2020-07-06 15:56:34
categories: AI
tags: SNN, Deep Learning
mathjax: true
---

# SNN学习笔记4： Nengo

[Nengo](https://www.nengo.ai/) 是一个用于神经建模框架，其扩展[NengoDL](https://nengo.ai/nengo-dl/)支持混用包含了生物细节的神经模型和现在流行的深度学习框架（例如：TensorFlow）。

## 安装

nengo安装命令：`pip install nengo nengo-gui`

测试是否安装成功可以尝试运行`nengo-gui`界面：`$: nengo`

nengo-dl安装命令：

1. 安装依赖的tensorflow：`conda install tensorflow`
2. 安装nengo-dl: `pip install nengo-dl`

## 使用

nengo有两种使用模式：GUI和Python解释器。Python解释器模式下，nengo的表现就是一个普通的Python库，因此下面仅介绍GUI模式的使用方式及限制。

### GUI模式

直接在命令行运行`$:nengo`即可在网页中运行图形界面。此时左侧会展示当前图的结构（方形表示`Nodes`，圆形表示`Ensembles`, 圆角矩形表示`Networks`），右侧展示对应的代码。

可以点击左上角的文件夹图标可以运行很多内建的例子，例如`/built-in examples/tutorial/15-lorenz.py`可以运行一个洛伦兹吸引子的例子，效果很酷。

需要注意的是，GUI模式下代码有如下限制：

1. 顶层网络必须叫`model`
2. 不能构建`Simulator对象`
3. 不能使用`Matplotlib`绘图

### 核心概念

![架构](/images/ecosystem.svg)

上图Nengo Core主要包含五个核心Nengo对象和一个基于Numpy的模拟器。五个对象如下：

1. nengo.Network： 一个网络可以包含ensembles、nodes、connections和其它网络
2. nengo.Ensemble：一组神经元，用于表征一个向量
   1. nengo.ensemble.Neurons: 用于连接ensemble中特定神经元的接口
3. Node：用于提供输入以及处理输出
4. Connection：连接两个对象 **NOTE: 和TensorFlow等不同，连接作为一个独立的对象，方便进行独立的设置**
   1. nengo.connection.LearningRule：为连接制定学习规则
5. Probe：用于将对象的数据在模拟器运行时取出

### Nengo-DL Demo: Relu VS Spiking neurons

下面的例子会展示Relu作为激活神经元和脉冲神经元的差异。

```python
import matplotlib.pyplot as plt
import nengo
from nengo.utils.matplotlib import rasterplot
import numpy as np
import nengo_dl

with nengo.Network() as net:
    # 输入节点，周期为1s的正弦波
    a = nengo.Node(lambda t: np.sin(2 * np.pi * t))

    # rate神经元，功能和Relu一致
    b_rate = nengo.Ensemble(10, 1, neuron_type=nengo.RectifiedLinear(), seed=2)
    nengo.Connection(a, b_rate)

    # spiking神经元
    b_spike = nengo.Ensemble(10, 1, neuron_type=nengo.SpikingRectifiedLinear(), seed=2)
    nengo.Connection(a, b_spike)

    # 模拟时取出输入输出数据
    p_a = nengo.Probe(a)
    p_rate = nengo.Probe(b_rate.neurons)
    p_spike = nengo.Probe(b_spike.neurons)

with nengo_dl.Simulator(net) as sim:
    # 运行模拟1S，上述Probe数据会被保存在sim.data字典中
    sim.run_steps(1000)

plt.figure()
plt.plot(sim.trange(), sim.data[p_a])
plt.xlabel("time")
plt.ylabel("input value")
plt.title("a")

plt.figure()
plt.plot(sim.trange(), sim.data[p_rate])
plt.xlabel("time")
plt.ylabel("firing rate")
plt.title("b_rate")

plt.figure()
# 时间栅格图，Spiking神经元的的脉冲事件出现与否以栅格形式展示
rasterplot(sim.trange(), sim.data[p_spike])
plt.xlabel("time")
plt.ylabel("neuron")
plt.title("b_spike")
plt.show()
```

上面代码运行的结果如下：

![a](/images/a.png)

![a](/images/relu.png)

![a](/images/raster.png)

可以看到每个神经元的初始连接权重及bias不同，因此对输入信号的相应略有不同；spiking neurons会在电压超过0时产生脉冲发射事件，注意图二和图三中的颜色对应，我们还可以观察到电压值越高，对应的脉冲发射频率越高。