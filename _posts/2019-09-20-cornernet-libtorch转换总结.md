---
layout:     post   				    # 使用的布局（不需要改）
title:      cornernet-libtorch总结 				# 标题 
date:       2019-09-20 				# 时间
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - CornerNet-Lite
    - libtorch
    - 总结
---

---

大概从月初开始进行cornernet-lite的框架使用，进行了行人的检测训练，考虑到速度的影响，主要是使用了cornernet-squeeze，后期如果有时间的话，会尝试对cornernet以及cornernet-saccade进行转换。

---

## 效果

### cornernet-lite的效果

- 效果还可以，主要是因为anchor free，会在检测图像中发现很奇怪的bbox

- 速度的话，在输入为960x540左右时，测试单张耗时为170ms左右，其中forward耗时大概为20ms左右，对于比其他的anchor free算法也同样能发现类似的问题，后处理耗时特别多，而且后处理不太能够进行很好的优化，所以优化也是比较困难

- 其次，看了代码之后发现，cornernet的train和test对于网络的使用时不一样的，可以根据inference实际使用的layer进行优化，不过按照上面一点的测试，这个前后处理的耗时比，优化这一些又不是很着急的事情了

- 最后一点，cornernet的训练速度是很慢，非常慢，要有足够的耐心和时间去训练

### libtorch的使用

- 为什么会考虑使用libtorch呢，首先是考虑转换的方便性，毕竟大部分的东西是一致的，所以会稍微简单些，后续会考虑转换到tensorrt上

- libtorch中的函数使用方法，多数是和pytorch中的一致

- libtorch的速度实现会比pytorch来的要更快一些，不过倒也不是相差特别多

- 在使用libtorch处理一些逻辑问题时，如果涉及到大量的本地运算或者频繁读取tensor，应该把tensor先放在cpu端为好

- libtorch依赖项还是比较干净的，官方提供的包已经能够完全满足

## 后续规划

- 实现batch的操作

- 实现多分类的模型转换以及调优

