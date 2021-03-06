---
    author: LuckyGong
    comments: true
    date: 2018-06-13 20:27
    layout: post
    title: Papper-Very deep convolutional networks for large-scale image rrcognition
    categories:
    - cv
    tags:
    - cv
    - papper
---



# Absract

- 工作：调查cnn深度对大规模图像识别精度的影响。
- 内容：增加深度到16-19层、3*3滤波器，将这种网络进行评估。
- 评价：深度加到16-19对结果有很大改进，适用性好。

# Introduction

- 现有工作：
  - Krizhevsky et al., 2012;
  - Zeiler & Fergus, 2013：减少步幅、较小窗口尺寸。
  - Sermanet et al., 2014;
  - Howard, 2014
  - Simonyan & Zisserman, 2014
  - Perronnin et al., 2010    
  - Ciresan et al. (2011); 

# 卷积配置

## 架构

- 卷积层配置同Ciresan et al. (2011)、Krizhevsky et al. (2012)
- 预处理：从每个像素中减去均值
- 初始输入尺寸：224 * 224 *3，若图像不是这个尺寸，则缩放后裁剪（每个sgd裁剪不同）
- 3 * 3卷积核、1 * 1卷积核（可以看作线性变换），步长为1。
- 卷积层输入的空间填充要满足卷积之后保留空间分辨率 ，即3×3卷积层的padding填充为1个像素。 
- 空间池化由五个最大池化层进行，这些层在一些卷积层之后。 在2 * 2像素窗口上进行最大池化，步长为2。 
- 之后是三个全连接（FC）层：前两个每个都有4096个通道，第三个执行1000维ILSVRC分类。
- 最后一层是soft-max层。所有网络中全连接层的配置是相同的。 
- 所有隐藏层都配备了修正（ReLU（Krizhevsky等，2012））非线性。 
- 除了一个，其他层都没有局部响应规范化LRN（这种规范化并不能提高在ILSVRC数据集上的性能，但增加了内存消耗和计算时间）。
- 通道由64到512

## 配置

![](http://upload-images.jianshu.io/upload_images/3232548-bcc2ded7859ee146.png)

- 格式：conv<卷积核大小>-<通道数>
- 没有画relu
- 参数量：113、113、134、138、144百万

## 讨论

- 两个3×3卷积层堆叠（没有空间池化）有5×5的有效感受野 ；三个3×3卷积层堆叠（没有空间池化）有7×7的有效感受野，但是可以减少参数。
  - 首先，我们结合了三个非线性修正层，而不是单一的，这使得决策函数更具判别性。
  - 其次，我们减少参数的数量 
    - 假设三层3×3卷积堆叠的输入和输出有C个通道，堆叠卷积层的参数为3(3^2C^2)=27C^2个权重；同时，单个7×7卷积层将需要7^2C^2=49C^2个参数，即参数多81％。这可以看作是对7×7卷积滤波器进行正则化，迫使它们通过3×3滤波器（在它们之间注入非线性）进行分解。 
- 结合1×1卷积层（配置C，表1）：增加决策函数非线性而不影响卷积层感受野。 

# 3.分类框架

## 3.1训练

- 训练方法参照Krizhevsky et al. (2012)，除了从多尺度训练图像中对输入裁剪图像进行采样外 
  - 通过使用小批量梯度下降（LeCun et al., 1989），利用动量来优化多项式逻辑回归目标来进行的。
  - batchsize：256，动量：0.9 
  - 训练通过权重衰减（L2惩罚乘子设定为5⋅10−45·10−4）进行正则化，前两个全连接层执行Dropout正则化（丢弃率设定为0.5）。
  - 学习率初始设定为10^−2，然后当验证集准确率停止改善时，减少10倍。学习率总共降低3次，学习在37万次迭代后停止（74个epochs）。 
  - 尽管与（Krizhevsky等，2012）相比我们的网络参数更多，网络的深度更大，但网络需要更小的epoch就可以收敛，这是由于：
    - 由更大的深度和更小的卷积滤波器尺寸引起的隐式正则化
    - 某些层的预初始化。 
- 网络权重的初始化很重要，因为由于深度网络中的梯度不稳定，初始化不好可能会导致学习停滞。
  - 为了避免这个问题，我们从训练配置A开始，这个配置足够浅，可以随机初始化进行训练，从具有零均值和10-2方差的正态分布中采样。
  - 然后，当训练更深的体系结构时，我们初始化了前四个卷积层和最后三个完全连接的层，中间层随机初始化。
  - 我们没有降低预初始化层的学习速率，允许它们在学习期间改变。
  - bias初始化为0
- 采用2种设置训练图像大小方法：
  - 固定训练集图片大小，如256 * 256和384 * 384；比这个大就剪裁
  - 多尺度训练，让训练集的大小在一个范围内随机变化，如S∈[Smin,Smax]=[256,512]，随机采样S来单独调整每个训练图像。
    - 由于图像中的物体可能具有不同的大小，因此在训练时考虑到这一点是有益的。
    - 可以看作是通过缩放抖动来增强训练集，单个模型被训练以识别范围广泛的物体。

## 3.2测试

- 对测试集做数据增强，采用水平翻转，最终取原始图像和翻转图像 的soft-max分类概率的平均值作为最终得分。 
- 首先，图像的最小边被各向同性的缩放到预定尺寸Q （Q不一定等于S）
- 然后将原先的全连接层改换成卷积层，在未裁剪的全图像上运用卷积网络， 输出是一个与输入图像尺寸相关的分类得分图，输出通道数与类别数相同 
  - 由于测试阶段采用全卷积网络，无需对输入图像进行裁剪，相对于多重裁剪效率会更高。但多重裁剪评估和运用全卷积的密集评估是互补的，有助于性能提升。 
- 对分类得分图进行空间平均化，得到固定尺寸的分类得分向量 

## 3.3实现细节

# 4.实验

