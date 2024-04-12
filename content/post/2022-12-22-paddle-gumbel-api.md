---
showonlyimage: true
title:      "开发者经验分享！手把手教你为飞桨贡献Gumbel API"
subtitle:   开发经验分享
date:       2022-12-22
author:     "YuRonan"
image: "/img/paddle-header.png"
published: true 
tags:
    - Paddle 

categories: [ Tech ]
URL: "/2022/12/22/paddle-gumbel-api/"
---
## 任务介绍
耿贝尔（Gumbel）分布是一种极值型分布。Gumbel分布理论认为，最大值分布的潜在适用性与极值理论有关，如果基础样本数据的分布是正态或者指数类型，Gumbel分布就是有用的。Gumbel分布适用于海洋、水文、气象领域，用来计算不同重现期的极端高(低)潮位。而在概率论和统计学中，Gumbel分布常被用来模拟不同分布的样本的最大(或最小)分布。

![Gateway](/img/paddle/gumbel-prob.png)

飞桨框架目前还未集成Gumbel分布，本任务的目标是对飞桨框架中现有概率分布方案进行扩展，新增Gumbel API。增加此API能够扩大飞桨的应用处理范围，对飞桨来说是非常必要的。

## 设计思路
Gumbel API的设计需要做多个方面的知识储备：

* 详细了解Gumbel分布背后的数学原理以及应用场景；（Controller）也可以在模型指定。
* 深刻了解飞桨和业界概率分布设计实现的方法和技巧。

>注：“充分学习Gumbel背后的数学原理”这一点很重要，避免在开发时违背定理。印象最深的是在开发之初，我们没有了解到分布的mean如何计算，直接使用了一个不清楚的计算方式，导致在最开始的版本中，mean的计算方式错误。最后，通过查询资料得知标准Gumbel分布的mean为负欧拉常数，于是纠正。

接下来，我将详细描述Gumbel API的设计思路。
* 命名与参数设计

API的名称直接使用Gumbel分布的名称，参数保持Gumbel分布最原生的参数，包括“位置参数loc”以及“尺度参数scale”。预期Gumbel API的形式为：

paddle.distribution.gumbel.Gumbel ( loc, scale )

* Gumbel分布类的初始化方法

类初始化过程中，一方面要严格控制参数loc和scale的形状和数据类型。另一方面还要借助基础分布Uniform以及transforms初始化父类 TransformedDistribution。

* Gumbel API的功能

该API部分功能继承于TransformedDistribution，包括mean均值、variance方差、sample随机采样、rsample重参数化采样、prob概率密度、log_prob对数概率密度、entropy熵计算等。除了官方任务要求外，我们还添加了一些其他的方法，比如stddev标准差和cdf累积分布函数等。


## 类初始化方法

### 数据类型

首先我们需要判断loc和scale的数据类型是否是飞桨支持的标量数据类型。如果是飞桨支持的标量，需要将其转为飞桨支持的tensor类型。

* 上述判断实现如下：

```python
if     
      not isinstance(loc, (numbers.Real, framework.Variable)):    
      
    （抛出数据类型错误）    
      
    
      if     
      not isinstance(scale, (numbers.Real, framework.Variable)):    
      
    （抛出数据类型错误）    
      
    
      if isinstance(loc, numbers.Real):    
      
    （转为 paddle 类型的 tensor）    
      
    
      if isinstance(scale, numbers.Real):    
      
    （转为 paddle 类型的 tensor）   
```

此外，还要统一loc和scale的形状类型，我们选择使用paddle.broadcast_tensors() 广播机制来进行统一。

```python

if loc.shape != scale.shape:    
      
            
      self.loc,     
      self.scale = paddle.broadcast_tensors([loc, scale])    
      
    
      else:    
      
            
      self.loc,     
      self.scale = loc, scale   

```

### 父类调用

代码开发
本节介绍代码开发的过程，着重介绍在开发中遇到困难的两个部分，包括类初始化方法（__init__）和重参数化采样方法（rsample）。最后，再介绍开发过程中遇到的问题以及如何解决该问题。

## 代码开发
在经过以上准备，确定设计思路后，我们给出 paddle.distribution.gumbel.Gumbel中实现的属性、方法的伪代码。

### 类初始化方法：__init__

在进行此方法的开发中，我们着重关注两个方面：
* 参数（loc, scale）的形状，数据类型的判断；

* 使用基础分布Uniform以及transforms调用父类TransformedDistribution。

最终__init__方法如下 ：

```python
def __init__(self, loc, scale):    
      
        
      if     
      not isinstance(loc, (numbers.Real, framework.Variable)):    
      
        raise TypeError(    
      
            f    
      "Expected type of loc is Real|Variable, but got {type(loc)}")    
      
        
      if     
      not isinstance(scale, (numbers.Real, framework.Variable)):    
      
        raise TypeError(    
      
            f    
      "Expected type of scale is Real|Variable, but got {type(scale)}"
        )    
      
        
      if isinstance(loc, numbers.Real):    
      
        loc = paddle.full(shape=(), fill_value=loc)    
      
        
      if isinstance(scale, numbers.Real):    
      
        scale = paddle.full(shape=(), fill_value=scale)    
      
        
      if loc.shape != scale.    
      shape:
            
      self.loc,     
      self.scale = paddle.broadcast_tensors([loc, scale])    
      
        
      else:
            
      self.loc,     
      self.scale = loc, scale    
      
    finfo = np.finfo(dtype=    
      'float32')    
      
        
      self.base_dist = paddle.distribution.Uniform(    
      
        paddle.full_like(    
      self.loc, float(finfo.tiny)),    
      
        paddle.full_like(    
      self.loc, float(    
      1 - finfo.eps)))    
      
        
      self.transforms = ()    
      
        
      super(Gumbel,     
      self).__init_    
      _(    
      self.base_dist,     
      self.transforms)   
```

### 重参数化采样：rsample

由于我们对rsample的概念模糊，在开发此部分时非常困难。在查阅资料后得知rsample的真正用途是重参数化技巧，即从一个分布中进行采样。而该分布是带有参数的，如果直接进行采样（采样动作是离散的，其不可微），没有梯度信息，那么在反向传播的时候就不会对参数梯度进行更新。重参数化技巧可以保证我们从分布中进行采样，同时又能保留梯度信息。
在梳理清楚rsample的含义后，我们参考了业界的写法，使用了两个transform。
即paddle.distribution.AffineTransform和 paddle.distribution.AffineTransform，最终代码如下：

```python
def rsample(self, shape):    
      
    exp_trans = paddle.distribution.ExpTransform()    
      
    affine_trans_1 = paddle.distribution.AffineTransform(    
      
        paddle.full(shape=    
      self.scale.shape,    
      
                    fill_value=    
      0,    
      
                    dtype=    
      self.loc.dtype), -paddle.ones_like(    
      self.scale))    
      
    affine_trans_2 = paddle.distribution.AffineTransform(    
      
            
      self.loc, -    
      self.scale)    
      
        
      return affine_trans_2.forward(    
      
        exp_trans.inverse(    
      
            affine_trans_1.forward(    
      
                exp_trans.inverse(    
      self._base.sample(shape)))))   
     
```

## 其他属性、方法实现

详细实现可以参考 Paddle/python/paddle/distribution/gumbel.py。

## 问题以及解决方法

在开发Gumbel API时，遇到了一些问题。

* 多次使用仅支持动态图的tensor，最终使用paddle.full替代。
* 无法在初始化方法中将多个transforms作用到基础分布Uniform上。最终 选择在rsample方法中直接将多个transforms作用到Uniform上。
* 测试类中，使用Numpy实现方法的数据类型和飞桨数据类型没有统一。

最开始的解决办法是在各个测试方法里面将Numpy的数据类型转换成飞桨支持的数据类型，但是官方评审人员给出了一种更方便的解决办法：不增加类型转换逻辑，用xrand指定dtype。

## 成果展示

导入及初始化Gumbel类

```python
      import paddle    
      
       
      from paddle.distribution.gumbel     
      import Gumbel       
      
   loc = paddle.full([    
      1],     
      0.0)    
      
    scale = paddle.full([    
      1],     
      1.0)    
      
    dist = Gumbel(loc,  scale)   
```
使用rsample(shape) / sample(shape)进行随机采样/重参数化采样，需要指定采样的shape。

```python
             shape = [    
      2]    
      
    dist.sample(shape)    
      
    # Tensor(shape=[    
      2,     
      1], dtype=float32, place=Place(gpu:    
      0), stop_gradient=True,     
      [[-0.27544352], [-0.64499271]])    
      
   dist.rsample(shape)    
      
    # Tensor(shape=[    
      2,     
      1], dtype=float32, place=Place(gpu:    
      0), stop_gradient=True,     
      [[0.80463481], [0.91893655]])    
```

## 总结

本篇项目介绍了我们小组开发Gumbel API的历程。从学习飞桨以及业界相关内容、到设计提案，最后到API的开发、测试。这期间，我们学习到很多内容，比如：
* 如何使用GitHub进行代码协作开发；

* 更加熟悉国内顶尖的飞桨深度学习框架；

* 深刻掌握了包括Gumbel分布在内的多个数学分布；

* 锻炼了使用Python开发的能力。

当然，Gumbel API还存在着一些缺点和不足，后续需要不断改进：
* rsample方法中使用多个transforms，操作繁琐；

* 测试方法时，随机采样和重参数化采样的精度未能降低到1e-3以下。未来将尝试使用其他方式来实现属性方法，从而降低采样精度；

* API的__init__方法存在优化的余地，未来可以将rsample中的transforms整合到初始化方法中；

* 未能实现InverseTransform；

* 未能将多个参数的验证抽取出来，使用特定的参数验证方法进行验证，比如 _validate_value(value)；

* 未来可以尝试增加kl_divergence计算两个Gumbel分布的差异。

以上就是整个API开发的过程，虽然充满挑战，但是不断解决问题才是关键，很高兴能为飞桨框架做出一份贡献，也很高兴能有这个机会在这里给各位分享开发过程。


