---
title: 'SNN[1]: LIF'
date: 2020-07-03 17:15:25
categories: AI
tags: SNN, Deep Learning
mathjax:  true
---

# SNN学习笔记1：LIF

## 什么是SNN？

[SNN](https://en.wikipedia.org/wiki/Spiking_neural_network)（Spiking neural network，脉冲神经网络）号称是第三代神经网络, 与当前流行的神经网络的主要区别是将神经脉冲传播的动态过程纳入学习和推理中。SNN中的神经元不会在每次传播中都激活，而是只有当神经元的膜电位超过阈值时才激活，在激活时发射电脉冲（spikes，神经科学中常称作动作电位），这些电脉冲通过轴突传递给其它的神经元。

因此， 神经元的膜电位的变化的描述变得十分重要。下面介绍几种描述膜电位变化的模型。

### Integrate-and-fire

Integrate-and-fire是最早用来描述膜电位变化的模型（1907年！），神经元膜电位的变化由下面的公式表示:
$$
I(t) = C_m \frac{dV_m(t)}{dt}
$$
$C_m$ 表示神经元的电容, 我们容易看出上式只是电容公式Q=CV两边同时对时间求导。

此模型有如下性质：当有电流输入时，膜电位将会升高。

#### 不应期

通过增加不应期（refractory period）`$t_{ref}$`可以使Integrate-and-fire模型更加精准。所谓不应期，就是在此期间神经元无法激活，这一现象在实际的神经元中已经被观察到，微观上可以由[钠钾离子通道的状态](https://www.youtube.com/watch?v=Gsf9IB-wQdU)来解释。

不应期限制了神经元激活的频率，激活频率与不应期`$t_{ref}$`的关系如下：
$$
f(I) = \frac{I}{C_mV_{th} + t_{ref}I}
$$

#### 缺点

一个显著的缺点是，上述模型没有实现时间相关记忆（time-dependent memory）,即如果这个模型收到一个远超阈值的信号时，会永远将此信号记在自己的膜电位中。

### LIF

[LIF](https://en.wikipedia.org/wiki/Biological_neuron_model#Leaky_integrate-and-fire)(Leaky integrate-and-fire)通过增加“漏电（Leaky）”项解决了Integrate-and-fire模型中缺少时间相关记忆的问题。模型的公式如下：
$$
I(t) - \frac{V_m(t)}{R_m} = C_m \frac{dV_m(t)}{dt}
$$
“漏电”项`$\frac{V_m(t)}{R_m}$`中的`$R_m$`表示膜电阻。

公式蕴含了下面这些有趣的性质：

1. 激活神经元的输入电流必须超过`$I_{th} = \frac{V_{th}}{R_m}$`, 否则漏电项会导致膜电位泄露
2. 大电流输入时，模型趋近于Integrate-and-fire+不应期

## LIF: Nengo实现

Nengo是一个SNN相关的库，利用Nengo可以很容易实现上述LIF模型。

```python
import matplotlib.pyplot as plt
import numpy as np
import nengo
from nengo.utils.matplotlib import rasterplot

from nengo.dists import Uniform

model = nengo.Network(label="A Single Neuron")
with model:
    neuron = nengo.Ensemble(1, dimensions=1, intercepts=Uniform(-.5, -.5), max_rates=Uniform(100, 100), encoders=[[1]])
    # input node
    cos = nengo.Node(lambda t: np.cos(8*t))
    nengo.Connection(cos, neuron)
    cos_probe = nengo.Probe(cos)
    # 神经元原始脉冲输出
    spikes = nengo.Probe(neuron.neurons)
    # 细胞体电压
    voltage = nengo.Probe(neuron.neurons, "voltage")
    # spikes filtered by a 10ms post-synaptic filter
    filtered = nengo.Probe(neuron, synapse=0.01)

with nengo.Simulator(model) as sim:
    sim.run(1)
    # plot the decoded output of the ensemble
    plt.figure()
    plt.plot(sim.trange(), sim.data[filtered])
    plt.plot(sim.trange(), sim.data[cos_probe])
    plt.xlim(0, 1)

    # plot the spiking output of the ensemble
    plt.figure(figsize=(10, 8))
    plt.subplot(221)
    rasterplot(sim.trange(), sim.data[spikes])
    plt.ylabel("Neuron")
    plt.xlim(0, 1)

    # plot the soma voltages of the neurons
    plt.subplot(222)
    plt.plot(sim.trange(), sim.data[voltage][:,0], 'r')
    plt.xlim(0, 1)
    plt.show()
```

