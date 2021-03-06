# 多目标追踪
## 简介

基于检测的多目标追踪，主要使用mxnet框架搭建，重新实现了mxnet版
[AlignedReID](https://arxiv.org/abs/1711.08184)
## Demo

追踪流程为:目标检测-->特征提取-->相似度计算-->匹配，最终效果展示如下：
<div align=center><img src="images/demo.gif"/></div>

----
## 优化

### 特征平均


在实验过程中本文发现一些问题，下图中展示了检测器在相邻两帧中的检测效果，左侧为第*t-10*帧，中间为第*t*帧，右侧为第*t+1*帧。图中所示的是两个男子在擦肩而过时，由于位置重叠的原因，检测器在第*t-10*帧到第*t*帧中每帧仅给出一个检测框，而在第*t+1*帧中给出了两个检测框，此时就极易发生轨迹身份转换的问题，因为第*t*帧中的目标检测框同时涵盖了两个人的信息，所以在与下一帧中的两个框匹配时，到两个框的距离都及为相似，而我们的目标则是希望第*t*帧中的框可以匹配到处于前景中的人物，即图中背书包的男子。这是因为从第*t-10*帧到第*t*帧中每帧仅有一个检测框，相邻两帧中的框互相匹配最终构成一个轨迹，而该轨迹中的人物应该对应于前景中的人物，但在第*t+1*帧中却极易匹配到位于背景中的男子。其根本原因是在轨迹匹配时未使用到历史轨迹信息，单纯将第*t*帧和第*t+1*帧取出，第*t*帧中的检测框匹配到第*t+1*帧中的任何一个框都是合理，但考虑到历史轨迹信息，这样的匹配就是不合理的。
<div align=center><img width="600" height="300" src="images/7b791ccab45bec618e946469860cab50.png"/></div>

因此，我们在计算相似度时不再仅仅只考虑轨迹中上一帧的特征，而是考虑轨迹中若干个特征，新的特征相似为当前帧行人特征与轨迹中若干个人特征相似度的平均值。

### 向前K次搜索


基础算法高度依赖于检测器的准确性，当某一帧中某个目标丢失后，下一帧的相同编号目标就会出现匹配失败的问题，这样，连续轨迹就被分成了两段。除此之外，当发生行人之间的相互遮挡时，也会使得其中一个人的轨迹被打断。这是因为在做帧间目标匹配时，只考虑了相邻两帧中的目标。

为了解决上述问题，本文中提出一种向前K次搜索的算法，向前K次搜索算法的核心思想就是在当前帧中某个目标匹配失败时，继续在更靠前的中匹配，其伪代码描述如下
<div align=center><img width="600" height="500" src="images/pseudocode.png"/></div>

其中KMatch算法简述如下：

为了减少轨迹隔断问题的出现，出现匹配失败时，在更靠前的帧中搜索，所以，KMatch算法就是要将当前帧中未匹配到轨迹的目标与前*j\*S*帧中的轨迹匹配，即能够将某些目标分配到某些轨迹中去。接收两个参数，当前帧中未被匹配的目标UMO和前*j\*S*帧中未与当前帧匹配的轨迹UMT。同样返回两个值，第一个SMT是在前帧中成功匹配到的轨迹，第二个值SUMO是依然匹配失败的目标。
