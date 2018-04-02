---
title: CNN网络架构变迁史
categories: CNN  # 配置
---

+ [**《AN ANALYSIS OF DEEP NEURAL NETWORK MODELSFOR PRACTICAL APPLICATIONS》论文来源**](https://arxiv.org/abs/1605.07678)
+ [**转载出处**](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/)

深度学习算法最近变得越来越流行和越来越有用的算法，然而深度学习或者深度神经网络的成功得益于层出不穷的神经网络模型架构。这篇文章当中作者回顾了从1998年开始，近18年来深度神经网络的架构发展情况。
[![](http://upload-images.jianshu.io/upload_images/4749583-5179ad748965d918.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/acc_vs_net_vs_ops.png)
图中的坐标轴我们可以看出横坐标是操作的复杂度，纵坐标是精度。模型设计一开始的时候模型权重越多模型越大，其精度越高，后来出现了resNet、GoogleNet、Inception等网络架构之后，在取得相同或者更高精度之下，其权重参数不断下降。值得注意的是，并不是意味着横坐标越往右，它的运算时间越大。在这里并没有对时间进行统计，而是对模型参数和网络的精度进行了纵横对比。

其中有几个网络作者觉得是必学非常值得学习和经典的：AlexNet、LeNet、GoogLeNet、VGG-16、NiN。
如果你想了解更多关于深度神经网络的架构及其对应的应用，不妨看一下这篇综述 [An Analysis of Deep Neural Network Models for Practical Applications](https://arxiv.org/abs/1605.07678)。

>转载自ZOMI的博客 [chenzomi12.github.io]

## LeNet5（1998）
作者Yann LeCun经过了很多次反复的试验之后发现5个卷积层效果最好，因此以lenet5命名！

![LeNet5（1998）](http://upload-images.jianshu.io/upload_images/4749583-8ebea41879250e64.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**LeNet5小结：**
- 卷积神经网络使用3层架构：卷积、下采样、非线性激活函数
- 使用卷积提取图像空间特征
- 下采样使用了图像平均稀疏性
- 激活函数采用了tanh或者sigmoid函数
- 多层神经网络（MLP）作为最后的分类器
- 层之间使用稀疏连接矩阵，以避免大的计算成本

总的来说LeNet5架构把人们带入深度学习领域，值得致敬。从2010年开始近几年的神经网络架构大多数都是基于LeNet的三大特性。

---

然后就是十几年的空白期啊！直到10年Dan Claudiu Ciresan和Jurgen Schmidhuber发表了一个[GPU神经网络](http://arxiv.org/abs/1003.0358)，能处理9层了，同时由于使用的是GPU训练的，因此Nvidia成为宠儿。

---
## AlexNet（2012）
2012年，Alex Krizhevsky发表了[AlexNet](https://papers.nips.cc/paper/4824-imagenet-classification-with-deep-convolutional-neural-networks.pdf)，相对比LeNet它的网络层次更加深，从LeNet的5层到AlexNet的7层，更重要的是AlexNet还赢得了2012年的ImageNet竞赛的第一。AlexNet不仅比LeNet的神经网络层数更多更深，并且可以学习更复杂的图像高维特征。

![ AlexNet（2012）](http://upload-images.jianshu.io/upload_images/4749583-255b8148b2533706.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

经过10多年的冷冻期之后，突然取得很好的出镜效果（获奖了啊），重回大众视野！这应归功于研究人员的不断坚持和付出以及硬件的快速发展。

**AlexNet小结：**
+ 使用ReLU函数作为激活函数，降低了Sigmoid类函数的计算量
+ 利用dropout技术在训练期间选择性地剪掉某些神经元，避免模型过度拟合
+ 引入max-pooling技术
+ 利用双GPU NVIDIA GTX 580显著减少训练时间

## Network-in-network（2013年底）
2013年年尾，Min Lin提出了在卷积后面再跟一个1x1的卷积核对图像进行卷积，这就是[Network-in-network](https://arxiv.org/abs/1312.4400)的核心思想了。

NiN在每次卷积完之后使用，目的是为了在进入下一层的时候合并更多的特征参数。同样NiN层也是违背LeNet的设计原则（浅层网络使用大的卷积核），但却有效地合并卷积特征，减少网络参数、同样的内存可以存储更大的网络。

根据Min Lin的NiN论文，他们说这个“网络的网络”（NIN）能够提高CNN的局部感知区域。
例如没有NiN的当前卷积是这样的：3x3 256 [conv] -> [maxpooling]；
当增加了NiN之后的卷积是这样的：3x3 256 [conv] -> 1x1 256 [conv] -> [maxpooling]。

[![Network-in-network（2013）](http://upload-images.jianshu.io/upload_images/4749583-739951b93bbf0bea.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/nin.jpg)

MLP多层感知的厉害之处就在于它把卷积特征结合起来成为一个更复杂的组合，这个思想将会在后面ResNet和Inception中用到。

## VGG（2014）
2014年是个绽放年，出了两篇重要的论文：VGG、GoogLeNet。

来自牛津大学的[VGG](http://arxiv.org/abs/1409.1556)网络是第一个在每个卷积层使用更小的3×3卷积核对图像进行卷积，并把这些小的卷积核排列起来作为一个卷积序列。通俗点来讲就是对原始图像进行3×3卷积，然后再进行3×3卷积，**连续使用小的卷积核对图像进行多次卷积**。

或许很多人看到这里也很困惑为什么使用那么小的卷积核对图像进行卷积，并且还是使用连续的小卷积核？

VGG一开始提出的时候刚好与LeNet的设计原则相违背，因为LeNet相信大的卷积核能够捕获图像当中相似的特征（权值共享）。AlexNet在浅层网络开始的时候也是使用9×9、11×11卷积核，并且尽量在浅层网络的时候避免使用1×1的卷积核。但是VGG的神奇之处就是在于使用多个3×3卷积核可以模仿较大卷积核那样对图像进行局部感知。

后来多个小的卷积核串联这一思想被GoogleNet和ResNet等吸收。

从下图我们可以看出来，VGG使用多个3x3卷积来对高维特征进行提取。因为如果使用较大的卷积核，参数就会大量地增加、运算时间也会成倍的提升。例如3x3的卷积核只有9个权值参数，使用7*7的卷积核权值参数就会增加到49个。因为缺乏一个模型去对大量的参数进行归一化、约减，或者说是限制大规模的参数出现，因此训练核数更大的卷积网络就变得非常困难了。

VGG相信如果使用大的卷积核将会造成很大的时间浪费，减少的卷积核能够减少参数，节省运算开销。虽然训练的时间变长了，但是总体来说预测的时间和参数都是减少的了。

[![](http://upload-images.jianshu.io/upload_images/4749583-8035cd26df60f970.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/vgg.jpg)

## GoogLeNet（2014）
2014年，在google工作的Christian Szegedy为了找到一个深度神经网络结构能够有效地减少计算资源，于是有了这个[GoogleNet](https://arxiv.org/abs/1409.4842)了（也叫做Inception V1）。

从2014年尾到现在，深度学习模型在图像内容分类方面和视频分类方面有了极大的应用。在这之前，很多一开始对深度学习和神经网络都保持怀疑态度的人，现在都涌入深度学习的这个领域，理由很简单，因为深度学习不再是海市蜃楼，而是变得越来越接地气。就连google等互联网巨头都已经在深度学习领域布局，成立了各种各样的额人工智能实验室。

Christian在思考如何才能够减少深度神经网络的计算量，同时获得比较好的性能的框架。即使不能两全其美，退而求其次能够保持在相同的计算成本下，能够有更好的性能提升这样的框架也行。于是后面Christian和他的team在google想出了这个模型：

[![](http://upload-images.jianshu.io/upload_images/4749583-19417159ee882cf8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/inceptionv1.jpg)

其乍一看基本上是1×1,3×3和5×5卷积核的并行合并。但是，最重要的是使用了1×1卷积核（NiN）来减少后续并行操作的特征数量。这个思想现在叫做“bottleneck layer”。

[](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/#Bottleneck-layer)


## Bottleneck layer

受NiN的启发，googleNet的Bottleneck layer减少了特征的数量，从而减少了每一层的操作复杂度，因此可以加快推理时间。在将数据传递到下一层卷积之前，特征的数量减少了4左右。因此这种设计架构因为大量节省计算成本而名声大噪。

让我们详细研究一下这个Bottleneck layer。假设输入时256个feature map进来，256个feature map输出，假设Inception层只执行3x3的卷积，那么这就需要这行 (256x256) x (3x3) 次卷积左右（大约589,000次计算操作）。再假设这次589,000次计算操作在google的服务器上面用了0.5ms的时间，计算开销还是很大的。现在Bottleneck layer的思想是先来减少特征的数量，我们首先执行256 -> 64 1×1卷积，然后在所有Bottleneck layer的分支上对64大小的feature map进行卷积，最后再64 -> 256 1x1卷积。 操作量是：
256×64 × 1×1 = 16,000s
64×64 × 3×3 = 36,000s
64×256 × 1×1 = 16,000s

总共约70,000，而我们以前有近600,000。几乎减少10倍的操作！
虽然我们做的操作较少，但我们并没有失去这一层特征。实际上，Bottleneck layer已经在ImageNet数据集上表现非常出色，并且也将在稍后的架构例如ResNet中使用到。

**成功的原因是输入特征是相关的，因此可以通过适当地与1x1卷积组合来去除冗余**。然后，在卷积具有较少数目的特征之后，它们可以再次扩展并作用于下一层输入。
[](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/#Inception-V3)

## Inception V3（2015.12）
Christian的团队确实很厉害，2015年2月他们又发表了新的文章关于在googleNet中加入一个[Batch-normalized](http://arxiv.org/abs/1502.03167)层。

Batch-normalized层归一化计算图层输出处所有特征图的平均值和标准差，并使用这些值对其响应进行归一化。这对应于“白化”数据非常有效，并且使得所有神经层具有相同范围并且具有零均值的响应。这有助于训练，因为下一层不必学习输入数据中的偏移，并且可以专注于如何最好地组合特征。

2015年12月，他们发布了一个新版本的GoogLeNet([Inception V3](http://arxiv.org/abs/1512.00567))模块和相应的架构，并且更好地解释了原来的GoogLeNet架构，GoogLeNet原始思想：
+ 通过构建平衡深度和宽度的网络，最大化网络的信息流。在进入pooling层之前增加feature maps
+ 当网络层数深度增加时，特征的数量或层的宽度也相对应地增加
+ 在每一层使用宽度增加以增加下一层之前的特征的组合
+ 只使用3x3卷积

因此最后的模型就变成这样了：
[![](http://upload-images.jianshu.io/upload_images/4749583-940828e42b1d3728.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/inceptionv3.jpg)
网络架构最后还是跟GoogleNet一样使用pooling层+softmax层作为最后的分类器。
[](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/#ResNet)

## ResNet（2015.12）
2015年12月[ResNet](https://arxiv.org/pdf/1512.03385v1.pdf)发表了，时间上大概与Inception v3网络一起发表的。其中ResNet的一个重要的思想是：输出的是两个连续的卷积层，并且输入时绕到下一层去。这句话不太好理解可以看下图。
[![](http://upload-images.jianshu.io/upload_images/4749583-2c2199f82c290b7d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/resnetb.jpg)
但在这里，他们绕过两层，并且大规模地在网络中应用这中模型。在2层之后绕过是一个关键，因为绕过单层的话实践上表明并没有太多的帮助，然而绕过2层可以看做是在网络中的一个小分类器！看上去好像没什么感觉，但这是很致命的一种架构，因为通过这种架构最后实现了神经网络超过1000层。傻了吧，之前我们使用LeNet只是5层，AlexNet也最多不过7层。

[](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/resnetbottleneck.jpg) 

该层首先使用1x1卷积然后输出原来特征数的1/4，然后使用3×3的卷积核，然后再次使用1x1的卷积核但是这次输出的特征数为原来输入的大小。如果原来输入的是256个特征，输出的也是256个特征，但是这样就像Bottleneck Layer那样说的大量地减少了计算量，但是却保留了丰富的高维特征信息。

ResNet一开始的时候是使用一个7x7大小的卷积核，然后跟一个pooling层。当然啦，最后的分类器跟GoogleNet一样是一个pooling层加上一个softmax作为分类器。下图左边是VGG19拥有190万个参数，右图是34层的ResNet只需要36万个参数：
[![](http://upload-images.jianshu.io/upload_images/4749583-97e6407ba625abcc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/ResNet.jpg) 
**ResNet网络特征**
+ ResNet可以被看作并行和串行多个模块的结合
+ ResNet上部分的输入和输出一样，所以看上去有点像RNN，因此可以看做是一个更好的生物神经网络的模型


## SqueezeNet（2016.11）
2016年11月才发表的文章，一看论文的标题可以被镇住：[SqueezeNet: AlexNet-level accuracy with 50x fewer parameters and < 0.5MB model size](https://arxiv.org/pdf/1602.07360v4.pdf)。文章直接说SqueezeNet有着跟AlexNet一样的精度，但是参数却比Alex少了接近50倍并且参数只需要占用很小的内存空间。这里的设计就没有SegNet或者GoogleNet那样的设计架构惊艳了，但SqueezeNet却是能够保证同样的精度下使用更少的参数。
[![](http://upload-images.jianshu.io/upload_images/4749583-c03b835b98a7c9ab.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/SqueezeNet.jpg)
[](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/#Xception)
## Xception
[Xception](https://arxiv.org/abs/1610.02357)模型使用与ResNet和Inception V4一样简单且优雅的架构，并且改进了Inception模型。
从Xception模型我们可以看出来Xception模型的架构具有36个卷积层，与ResNet-34非常相似。但是模型的代码与ResNet一样简单，并且比Inception V4更容易理解。
[![](http://upload-images.jianshu.io/upload_images/4749583-0a64dd8d668ee56c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/xception-net.jpg)
从 [here](https://gist.github.com/culurciello/554c8e56d3bbaf7c66bf66c6089dc221) 或者 [here](https://keras.io/applications/#xception) 可以找到Xception的实现代码。
[](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/#总结)

## 总结
我们再来回顾开篇的对比图。从图中我们可以看出来，AlexNet一类的模型没有考虑太多权重参数的问题，因此圆圈比较大。一开始AlexNet只是希望用深度网络能够找到更多图像当中的高维特征，后来发现当网络越深的时候需要的参数越多，硬件总是跟不上软件的发展，这个时候如果加深网络的话GPU的显存塞不下更多的参数，因此硬件限制了深度网络的发展。为了能够提高网络的深度和精度，于是大神们不断地研究，尝试使用小的卷积核代替大的卷积核能够带来精度上的提升，并且大面积地减少参数，于是网络的深度不再受硬件而制约。
可是另外一方面，随着网络的深度越深，运算的时间就越慢，这也是个很蛋疼的事情，不能两全其美。作者相信在未来2-3年我们能够亲眼目睹比现有网络更深、精度更高、运算时间更少的网络出现。因为硬件也在快速地发展，特斯拉使用的NVIDIA Driver PX 2的运算速率已经比现在Titan X要快7倍。
[![](http://upload-images.jianshu.io/upload_images/4749583-df5bde894944b516.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/acc_vs_net_vs_ops.png)
[](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/#后话)

##后话
其实作者觉得神经网络的架构体系对于了解“深度学习”和对于了解深度学习的发展是非常重要的，因此强烈推荐大家去深入研读一下上面提到的网络架构对应的文章。
总是有一些人会问为什么我们需要那么多时间去了解这些深度网络的架构体系呢，而不是去研究数据然后了解数据背后的意义和如何对数据进行预处理呢？对于如何研究数据可以搜一下另外一篇文章《人工智能的特征工程问题》。对，数据很重要，同时模型也很重要。简单的举一个例子，如果你对某种图像数据很了解，但是不懂CNN如何对这些图像进行提取高维特征，那么最后可能还是会使用HOG或者传统的SIFT特征检测算法。
还要注意的是，在这里我们主要谈论计算机视觉的深度学习架构。类似的神经网络架构在其他领域还在不断地发展，如果你有精力和时间，那么可以去研究更多不一样的架构进化历。