---
title: Tensorflow graph_matcher
date: 2020-07-20 09:34:12
categories: AI
tags: TensorFlow, Deep Learning
mathjax: true
---

# TF graph_matcher用法及源码探究

`graph_matcher`是`tensorflow.contrib`中量化模块（`quantize`）的一个子模块，用于在计算图中描述和匹配特定的模式。配合对模式匹配后的处理，可以在python层面实现计算图的pass。

## 用法: 以Conv + BN融合为例

Conv+BN是CNN网络中常见的组合。如果我们观察两者的计算公式，可以发现两者可以融合为一个算子达到运行加速的效果（实质上，Conv计算上等价于MatMul，所以融合也适用于FC+BN等组合）。

### 融合原理

首先，分别观察Conv和BN的计算公式：

​	Conv的计算公式：$z = w * x + b$

​	BN的计算公式：$y = \frac{(z - \mu_B) * \gamma}{ \sigma_B} + \beta$

实际上，我们可以通过更新Conv的`weight`和`bias`直接在Conv中完成Conv + BN所需完成的计算。略去推导，直接给出新的`weight`和`bias`的计算公式如下：

​	$w^{\prime} = \frac{w * \gamma}{\sigma_B}$, $b^{\prime} = \frac{(b - \mu_B)\gamma}{\sigma_B} + \beta$

代入新的$w^{'}$和$b^{'}$，容易验证新的Conv计算等价于Conv + BN:
$$
\begin{align}
z^{\prime} &= w^{\prime} * x + b^{\prime} \\
          &= \frac{w * \gamma * x}{\sigma_B} +\frac{(b - \mu_B) * \gamma} {\sigma_B} + \beta \\
          &=\frac{\gamma(w*x + b - \mu_B)}{\sigma_B} + \beta \\
          &=\frac{(z - \mu_B) * \gamma}{\sigma_B} + \beta = y
\end{align}
$$

### graph_matcher实现

`tensorflow.contrib.quantize.python`中包含了Conv + BN融合的[实现](https://github.com/tensorflow/tensorflow/blob/590d6eef7e91a6a7392c8ffffb7b58f2e0c8bc6b/tensorflow/contrib/quantize/python/fold_batch_norms.py#L151-L205)，完整的代码较长，我们重点关注其中对Conv+BN模式描述的部分。

```python
def _FindFusedBatchNorms(graph):
    """Finds all ops and tensors related to found FusedBatchNorms.
    	Args:
    		graph: Graph to inspect.
    	Returns:
    		_FusedBatchNormMatches.
    """
    input_pattern = graph_matcher.OpTypePattern('*')
    weight_pattern = graph_matcher.OpTypePattern('*')
    gamma_pattern = graph_matcher.OpTypePattern('*')
    beta_pattern = graph_matcher.OpTypePattern('*')
    mean_pattern = graph_matcher.OpTypePattern('*')
    variance_pattern = graph_matcher.OpTypePattern('*')
    
    moving_average_pattern = graph_matcher.OpTypePattern('*')
    bn_decay_pattern = graph_matcher.OpTypePattern('*')
    layer_pattern = graph_matcher.OpTypePattern('Conv2D|DepthwiseConv2dNative|MatMul', inputs=[input_pattern, weight_pattern])
    ...
    layer_output_pattern = graph_matcher.OneofPattern([layer_pattern_with_identity, layer_pattern, batch_to_space_pattern])
    ...
    bn_matcher = graph_matcher.GraphMatcher(graph_matcher.OneofPattern([matmul_bn_output_reshape_pattern, batch_norm_pattern]))
    ...
    def _GetLayerMatch(match_result):
        """Populates a layer match object containing ops/tensors for folding BNs.
        Args:
        	match_result: Matched result from graph matcher
        Returns:
        	layer_op: Matching conv/fc op prior to batch norm
        	BatchNormMatch: _BatchNormMatch containing all required batch norm parameters.
        """
        ...
    layer_matches = []
    matched_layer_set = set()
    for match_result in bn_identity_matcher.match_graph(graph):
        layer_op, layer_match = _GetLayerMatch(match_result)
        if layer_op is not None:
            if layer_op not in matched_layer_set:
                matched_layer_set.add(layer_op)
                layer_matches.append(layer_match)
    ...
    return layer_matches
def _FoldFusedBatchNorms(graph, is_training, freeze_batch_norm_delay):
    """Finds fused batch norm layers and folds them into preceding layers.
    Folding only affects the following layers: Conv2D, fully connected, depthwise convolution.
    Args:
    	graph: Graph to walk and modify.
    	is_training: Bool, true if training.
    	freeze_batch_norm_delay: How many steps to wait before freezing moving mean
      and variance and using them for batch normalization.
  Raises:
  	ValueError: When batch norm folding fails.
  	"""
    for match in _FindFusedBatchNorms(graph):
        scope, sep, _ = match.layer_op.name.rpartition('/')
        with graph.as_default(), graph.name_scope(scope + sep):
            with graph.name_scope(scope + sep + 'BatchNorm_Fold' + sep):
                # new weights = old weights * gamma / sqrt(variance + epsilon)
                # new biases = -mean * gamma / sqrt(variance + epsilon) + beta
                multiplier_tensor = match.gamma_tensor * math_ops.rsqrt(match.variance_tensor + match.bn_op.get_attr('epsilon'))
                bias_tensor = math_ops.subtract(match.beta_tensor, match.mean_tensor * multiplier_tensor, name='bias')
    ...
```

#### 流程

使用`GraphMatcher`进行pattern描述、匹配、替换主要分为以下几步：

1. 使用`OpTypePattern`及`OneofPattern`（语法糖）自底向上构建目标pattern
2. 用目标pattern构造`GraphMatcher`
3. 向`GraphMatcher`的`match_graph`方法传入要匹配的图，获得`match_result`
4. 从`match_result`中取出需要复用的pattern中的节点，构造新的节点替换pattern

## graph_matcher实现

graph_matcher的[实现](https://github.com/rockchip-linux/tensorflow/blob/master/tensorflow/contrib/quantize/python/graph_matcher.py)主要包括三个部分：`Pattern`, `GraphMatcher`, `MatchResult`。

### Pattern实现

Pattern作为一个抽象类，要求子类必须实现`match`方法。`match`方法接收两个参数：`op`；`tensor`。

Pattern类有两个子类：`OpTypePattern`类可以限定节点的类型、输入，可以描述一个类型树；`Oneof`

`Pattern`作为语法糖用于描述one-of关系，也就是匹配输入多个子模式之一即可。

NOTE: 当前实现了的模式中，`match`方法中的`tensor`只是占位用，没有实际使用到。

#### OpTypePattern

构造函数（`def __init__(self, op_type, name=None, inputs=None, ordered_inputs=True)`）通过限定节点的类型、输入来描述一个类型树；对应的，`match`中也会递归地对输入节点调用`match`函数。NOTE: `OpType`的匹配是使用字符串来完成的。

### MatchResult

保存match的结果，可以从`Pattern`实例映射到对应的匹配到的`op`和`tensor`。

### GraphMatcher

`GraphMatcher`中会保存一个pattern，提供方法来检验输入的`op`或者`graph`是否和`pattern`匹配，主要方法有：

1. match_op
2. match_ops
3. match_graph

## 拓展思考

`TensorFlow`中能够轻松的在python中操作图主要得益于图数据结构对python的暴露。当前MindSpore要在python中支持图pass(图中模式的匹配和替换)，可以对比两种思路：

1. python向C++注册pass，python中对模式和要替换的目标进行描述，C++中运行pass
   1. 优点：可以复用部分优化器部分的代码；执行效率较高
   2. 缺点：python pass中的pattern、target与C++通信较复杂
2. C++向python暴露图接口，直接在python中完成改图
   1. 优点：对图修改的逻辑全部包含在python中
   2. 缺点：效率较低，但此类任务通常较低频，性能要求不高

综合考虑，C++新增向python暴露图接口，直接在python中完成改图较合理。