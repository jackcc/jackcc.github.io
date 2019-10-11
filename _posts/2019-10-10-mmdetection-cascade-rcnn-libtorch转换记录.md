---
layout:     post   				    # 使用的布局（不需要改）
title:      mmdetection-cascade-rcnn-libtorch总结 				# 标题 
date:       2019-10-10 				# 时间
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - mmdetection（cascade-rcnn）
    - libtorch
    - 记录
---

---

放几个参考网址：  
[官方的cpp文档](https://pytorch.org/cppdocs/)  
[官方pytorch英文文档](https://pytorch.org/tutorials)  
[pytorch中文文档](https://pytorch.apachecn.org/docs/1.0/)  

公司内部使用mmdetection进行检测模型的训练，因此有需要将mmdetection从python代码转成libtorch的代码，后续的步骤是要转成tensorrt。这篇博客主要记录下在转换过程中的一些问题以及解决方法。

---

## 转换效率

目前刚刚转换完成第一版，没有进行优化，纯粹的进行代码转换，从inference到post，循环100次：

方法|耗时
-|-
libtorch|110
python|103
基本在可接受范围内

## 转换过程

cascade-rcnn可以拆分为两个部分，backbone和post处理部分

### backbone转换

backbone部分可以直接采用简单的torch::jit模块进行转换即可，其中需要注意的部分：

- 所有的backbone的inference代码均在base.py内

- 需要对代码进行网络代码进行一部分的屏蔽，***去除post部分的网络结构***

- 需要对input的部分进行一定的代码修改，使的inference的时候只需要输入image的数据即可

- backbone中出现的不属于pytorch原生支持的层，需要进行so的转换编译链接，具体可以参考文档

### post部分转换

post部分比较复杂，主要是因为不同的检测算法的post部分不同，此处只以cascade-rcnn进行说明，如果是其他的检测网络，需要进行修改。

- 根据不同的stage_number进行不同的bbox-head的转换，可能会产生多个bbox-head的pt文件，这些文件在libtorch使用时均需要进行加载

- 针对cascade-rcnn的post部分，主要包括bbox的roi-align以及后处理操作，roi-align同样不是官方的支持操作，可以直接把代码放入libtorch中，也可以对其进行库封装

- post中很多参数都没有明确指出，因为mmdetection作者强大的python能力，里面包含了很多的继承以及多态性，因此建议在拿不准代码中的某些参数时，采用简单粗暴的方式进行查找：***print***

### 一些libtorch函数的注意

libtorch中有一些变量或者函数的操作刚开始使用的时候会比较难搞，如list等，值会发生变动，而且变动很诡异，如果在转换过程中出现问题，建议优先检查一下libtorch一些操作是否是自己想要的

## 后续

- 进行代码梳理，提升速度

- 尝试进行多batch的操作
