---
    author: LuckyGong
    comments: true
    date: 2018-04-13 20:27
    layout: post
    title: [Papper]Distilling the Knowledge in a Neural Network
    categories:
    - modelcompression
    tags:
    - papper
    - modelcompression
    - knowledgedistillation
---

# 1.背景

## 1.1动机：

- 集成学习/神经网络模型笨重、计算耗费大、大量用户使用不便。
- 可以对它们的预测做平均，但是这种做法计算复杂度太高了。



- 一方面，在具有大量类别标签的训练任务中，一个复杂度很高的大模型地做法是为这些大量的标签分配概率分布。

  然而这种处理方式存在一个负作用：与正确标签相比，模型为所有的误标签都分配了很小的概率；然而实际上对于不同的错误标签，其被分配的概率仍然可能存在数个量级的悬殊差距。如：在识别一辆宝马汽车的图片时，分类器将该图片识别为垃圾车的概率是很小的，然而这种概率比起将其识别为胡萝卜的可能是会大出很多。然而由于在宏观上由于这些概率都很小，**这一部分的知识**很容易在训练过程中淹没。

- 另一方面，复杂的训练模型的训练过程中，所有的参数都经过了很强的正则化，通过这样的“蒸馏”方法，小模型的训练会模仿大模型的训练方式调整参数，效果要比从头开始训练的效果要好得多。

### 1.2前人的研究

 Rich Caruana可以将集成的模型压缩到一个模型中。

### 1.3本文研究

- 快速训练的模型。

###  1.4引入

### 1.4.1类比昆虫

- 昆虫作为幼虫时擅于从环境中汲取能量，但是成长为成虫后确是擅于其他方面，比如迁徙和繁殖等。
- ML中，训练和测试的目标不同：
  - 对训练数据拟合得好：可以利用大量的计算资源且不需要实时响应
  - 有很强的泛化能力：面临更加严格的要求包括计算资源限制，计算速度要求
- ”先训练复杂模型，再根据复杂模型训练简单模型“的效果要比直接训练简单模型的泛化效果要好。

### 1.4.2hard target与soft target

- hard target：比方说MNIST数据集是用来识别数字的，对于用来表示数字3的图片，它的label就是[0,0,1,0,0...0] ，从概率的角度上理解那就是说“图片显示的数字是3的概率是100%，是其他数字的概率是0%”，这就是所谓的hard target
- soft target：学习算法返回的可能就是[0.005 , 0.01 , 0.9 , ... , 0.005]这样，代表“图片是1的概率是0.5%，图片是2的概率是1%，图片是3的概率是90%...”，这种非100%的结果就是所谓的soft target。(3更像2而1没这么像)

# 2.算法

## 2.1soft target

- 随着T的增大，qi会越来越“soft”，就可以提供更大的信息熵，将已训练模型地知识更好地传递给新模型。T通常取1。
- 目的是使得小模型在预测实际目标的同时尽量匹配“软目标”。
- 通过参数T的调整，softmax层的映射曲线更加平缓，因而实例的概率映射将更为集中，便使得目标更加地"soft"。
- soft target包含了更多的信息，用这样的target作为label重新训练更小的模型，就是Distilling的中心思想。

## 2.2蒸馏

- 目的：用“蒸馏”把我们在应用端需要的配置的缩小模型从复杂模型中提取出来。
- 难点：如何缩减网络结构但是把网络中的知识保留下来
  - 知识就是一幅将输入向量导引至输出向量的地图。
  - 做复杂网络的训练时，目标是将正确答案的概率最大化，但这引入了一个副作用：这种网络为所有错误答案分配了概率，即使这些概率非常小。它定义了关于数据的丰富的相似性结构，但是对于交叉熵函数影响很小。
  - 不正确答案的相对概率告诉我们很多关于繁琐模型是如何趋于泛化的，也有一些信息。


- 思路：

  可以利用大模型的class probabilities作为小模型的soft targets，当softmax target有很高的entropy，那么它提供了比hard target更多的信息, 而且使得训练样本的梯度variance更小，小模型学习所需要的训练数据将会大量减少，所以小模型可以使用比大模型更少的数据和可以使用一个更大的学习率。

- 步骤：

  - 第一步，提升大模型final softmax中的T，使得复杂模型产生一个合适的、适当柔软的“软目标” 。
  - 第二步，采用同样的T来训练小模型，使得它产生相匹配的“软目标”。训练蒸馏模型时使用与大模型相同的T，但训练后小模型T为1。

- 注意：因为由soft target目标函数产生的梯度被scale到 1/T^2 在目标函数时需要乘以T^2


- 公式：

  - 模型：

    ![](https://images2018.cnblogs.com/blog/1230143/201803/1230143-20180312103448057-36633468.png)

  - 损失函数——正常：

    - 每个转换数据集中的案例都贡献了一个交叉熵梯度dC/dzi

    ![](https://upload-images.jianshu.io/upload_images/2517062-89859e5e2fd56a54.png)

    - zi是蒸馏后的模型的参数。
    - vi是复杂模型的参数。
    - pi是复杂模型提供的soft target，用logits（softmax层的输入）而不是softmax层的输出作为学习小模型的“软目标”。他们目标是是的复杂模型和小模型分别得到的logits的平方差最小。
    - qi是简单模型提供的soft target

    ​

    - 第一个目标函数是“软目标”的交叉熵，这个交叉熵用开始的那个比较大的T来计算。
    - 第二个目标函数是正确标签的交叉熵，这个交叉熵用小模型softmax层的logits来计算且T等于1。我们发现当第二个目标函数权重较低时可以得到最好的结果 


  - 损失函数——T很大：用正确标签来修正“软目标”，对两个目标函数设置权重系数。

    ![](https://upload-images.jianshu.io/upload_images/2517062-684b8a969360f1a4.png)



  - 损失函数——logits的均值为0：

$$
  要求\sum_jz_j=\sum_jv_j=0成立。
$$

  ![](https://upload-images.jianshu.io/upload_images/2517062-b9eda609e02cff21.png)

- 图示：



# 3.参考资料

- https://www.cnblogs.com/lainey/p/8531394.html
- http://blog.sina.com.cn/s/blog_76d02ce90102xnjf.html
- http://blog.csdn.net/cookie_234/article/details/72957100
- https://www.jianshu.com/p/4893122112fa
- https://zhuanlan.zhihu.com/p/24894102
- https://blog.csdn.net/zhongshaoyy/article/details/53582048
- https://www.zhihu.com/question/50519680
