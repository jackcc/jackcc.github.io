---
layout:     post   				    # 使用的布局（不需要改）
title:      mmdetection-cascade-rcnn的tensorrt转换 				# 标题 
date:       2019-12-05 				# 时间
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - mmdetection（cascade-rcnn）
    - mmdetection (faster-rcnn)
    - tensorrt
    - 记录
---

---

自从十月份的博客更新了使用libtorch实现mmdetection之后，一直没有更新了博客，一方面是因为比较忙，另一方面也是犯了懒，今天在趁着等待百度网盘下载的间隙（顺便吐槽下：BD吃像真TM难看！！），更新一下博客

---

## 转换效率

由于在刚开始的时候，忘了测试libtorch的版本，所以暂时没有办法对于libtorch版本，不过根据上一个版本来看，应该是tensorrt有一定的优势，这个测试后续有条件的话补上

- 测试条件：

  - gpu: 1050ti 4GB
  - cpu: i7-7700 CPU @ 3.60GHz

- 测试模型：

  - cascade-rcnn : top_det 顶层检测器，14类，resnet50的backbone
  - faster-rcnn ： hat_smoke 安全帽吸烟检测器， 2类， resnet18的backbone

### tensorrt版本

| NET | GPU(MB) | CPU(GB) | input | time(ms) |
| :----: | :----: | :----: | :----: | :----: |
| cascade-rcnn | 1929 | 3 | 1080*1440 | 79400/200 |
| faster-rcnn | 1293 | 3 | 1920*1080 | 23315/200 |

### python版本

| NET | GPU(MB) | CPU(GB) | input | time(ms) |
| :----: | :----: | :----: | :----: | :----: |
| cascade-rcnn | 1200 | 2 | 1080*1440 | 65676/200 |
| faster-rcnn | 801 | 2 | 1920*1080 | 24034/200 |

## 转换过程

这部分的内容和libtorch的类似，具体可以参考上面一篇博客的内容。下面一些着重需要注意的点：

- 因为rcnn系列的输入都不是固定的，但是为了更好的进行优化，在使用tensorrt的时候，固定了输入的大小

  - 根据配置文件中的max_long_side和max_short_side,确定一个最近接的输入固定大小

  - 将输入的图像等比例缩放到最接近输入大小的尺寸

  - 贴图在输入的固定大小上

- backbone的输出接收，一定要注意顺序，取值时不要搞错，可以根据大小和通道数来进行区分

- 由于libtorch的输入设置为NCHW，tensorrt的输出也是NCHW，因此可以直接将tensorrt的输出送入libtorch中。也就是说：

  - 在tensorrt的inference完成后，不需要将数据结果拷贝至CPU，直接送给libtorch

  - libtorch的直接接收GPU的指针，初始化tensor

- libtorch中的tensor直接使用gpu的ptr相关代码如下：

```C++
    // 使用cpu的ptr创建的tensor
    // auto cls_score = torch::from_blob(m_cpu_head_buffer[i][0],{m_head_outshape[i][1].d[0],m_head_outshape[i][1].d[1]}, torch::kFloat);
    // auto bbox_pred = torch::from_blob(m_cpu_head_buffer[i][1],{m_head_outshape[i][2].d[0],m_head_outshape[i][2].d[1]}, torch::kFloat);
    // 把tensor从cpu转移到gpu
    // cls_score = cls_score.to(torch::kCUDA);
    // bbox_pred = bbox_pred.to(torch::kCUDA);
    // 直接采用gpu的ptr初始化tensor
    auto cls_score = torch::from_blob(m_gpu_head_buffer[i][1],{m_head_outshape[i][1].d[0],m_head_outshape[i][1].d[1]}, torch::kCUDA);
    auto bbox_pred = torch::from_blob(m_gpu_head_buffer[i][2],{m_head_outshape[i][2].d[0],m_head_outshape[i][2].d[1]}, torch::kCUDA);
```

- tensorrt接收libtorch的tensor指针进行inference代码如下：

```CPP
    roi = roi.contiguous(); //连续
    void *buffers[3];
    buffers[0] = roi.data<float>();
```

### tips

- tensorrt的输出可以使用opencv的cv::dnn::blobFromImage来整理，多batch的话，可以使用cv::dnn::blobFromImages

## 后续

- 由使用libtorch改用了tensorrt来进行inference，但是因为时间关系，中间的处理，还是采用了libtorch，有些复杂，后续如果有时间，会把libtorch从中剔除出去，使用cuda实现相关功能

- 代码中应该有部分内容可以梳理加速，这个有时间再搞吧
