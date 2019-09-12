---
layout:     post   				    # 使用的布局（不需要改）
title:      CornerNet-Lite修改记录 				# 标题 
date:       2019-08-29 				# 时间
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - CornerNet-Lite
---

---
    CornerNet-Lite是一个anchor free的检测框架，在coco数据集上达到了比价好的效果，速度也很快。我们训练行人检测希望尝试这个框架，不过由于原始的是基于COCO的，所以需要进行一些修改，输入的目标只有行人。
---

## coco数据集只使用行人标注

1. 修改core/dbs/coco.py 的_load_coco_anns如下：

``` pyhton

    #class_ids = coco.getCatIds()
    #image_ids = coco.getImgIds()
    #需要检测其他类，修改person即可，也可以同时赋值多个；注意在进行测试时，修改将其修改回去，否则会报错
    class_ids = coco.getCatIds(catNms=['person'])
    image_ids = coco.getImgIds(catIds=class_ids)

```

1. 修改core/models/CornerNet_Squeeze.py(如果使用其他模型，请参考进行修改):

```python

    #tl_heats = nn.ModuleList([self._pred_mod(80) for _ in range(stacks)])
    #br_heats = nn.ModuleList([self._pred_mod(80) for _ in range(stacks)])
    tl_heats = nn.ModuleList([self._pred_mod(1) for _ in range(stacks)])
    br_heats = nn.ModuleList([self._pred_mod(1) for _ in range(stacks)])

```

1. 在测试coco数据集时，只测试行人一类，修改代码core/test/cornernet.py的cornernet函数：

```python 

    #cls_ids = list(range(1,categories + 1))
    cls_ids = list(range(1,2))

```

以上的修改均是只检测一类person时的修改，如果需要检测多类，按照同样的道理进行修改即可。
