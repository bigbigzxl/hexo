
---
title: CNN TensorFlow实现
categories: CNN  # 配置
---

说起CNN的模型图，要从经典CNN的相关paper开始：
[**CNN综述必须拜读啊！**](https://chenzomi12.github.io/2016/12/13/CNN-Architectures/)
[**学习资源在这**](https://jizhi.im/community/discuss/2017-03-13-10-9-2-pm)
[**本文搬移至这**](https://zhuanlan.zhihu.com/p/25754846)
# LeNet，1998年（开山鼻祖）；
+ 卷积
+ 非线性(ReLU)
+  池化或下采样
+ 分类（全连接层）

![简单的ConvNet](http://upload-images.jianshu.io/upload_images/4749583-b96b426c08c8d699.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

理解滤波器（卷积核）对于原输入图片来说，是个特征探测器。对于同一张照片，不同的滤波器将会产生不同的特征映射。

>**卷积神经网络与我们之前所学到的图像的卷积的区别：我们之前学图像处理遇到卷积，一般来说，这个卷积核是已知的，比如各种边缘检测算子、高斯模糊等这些，都是已经知道卷积核，然后再与图像进行卷积运算。然而深度学习中的卷积神经网络卷积核是未知的，我们训练一个神经网络，就是要训练得出这些卷积核，而这些卷积核就相当于我们学单层感知器的时候的那些参数W，因此你可以把这些待学习的卷积核看成是神经网络的训练参数W。**

>在CNN中，我们要训练的卷积核并不是仅仅只有一个，这些卷积核用于提取特征，卷积核个数越多，提取的特征越多，理论上来说精度也会更高，然而卷积核一堆，意味着我们要训练的参数的个数越多。在LeNet-5经典结构中，第一层卷积核选择了6个，而在AlexNet中，第一层卷积核就选择了96个，**具体多少个合适，还有待学习。**

>回到特征图(feature maps 也即卷积后的图片)概念，**CNN的每一个卷积层我们都要人为的选取合适的卷积核个数，及卷积核大小。**每个卷积核与图片进行卷积，就可以得到一张特征图了，比如LeNet-5经典结构中，第一层卷积核选择了6个，我们可以得到6个特征图，这些特征图也就是下一层网络的输入了。我们也可以把输入图片看成一张特征图，作为第一层网络的输入。

4、CNN的经典结构
![不同卷积核的效果](http://upload-images.jianshu.io/upload_images/4749583-49c8f504494473e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在实践当中，卷积神经网络在训练过程中学习滤波器的值，当然我们还是要在训练之前需要指定一些参数：
+ 滤波器的个数，滤波器尺寸、网络架构等等。
+ 滤波器越多，从图像中提取的特征就越多，模式识别能力就越强。

特征映射的尺寸由三个参数控制，我们需要在卷积步骤之前就设定好：
**深度(Depth)**： 深度就是卷积操作中用到的滤波器个数。如下图所示，我们对原始的船图用了三个不同的滤波器，从而产生了三个特征映射。你可以认为这三个特征映射也是堆叠的2d矩阵，所以这里特征映射的“深度”就是3。

![](http://upload-images.jianshu.io/upload_images/4749583-6053bdf6838e66ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**步幅(Stride)**：步幅是每次滑过的像素数。当Stride=1的时候就是逐个像素地滑动。当Stride=2的时候每次就会滑过2个像素。步幅越大，特征映射越小。

**补零(Zero-padding)**：有时候在输入矩阵的边缘填补一圈0会很方便，这样我们就可以对图像矩阵的边缘像素也施加滤波器。补零的好处是让我们可以控制特征映射的尺寸。补零也叫宽卷积，不补零就叫窄卷积。

### **非线性**：
ReLU是以像素为单位生效的，其将所有负值像素替换为0。ReLU的目的是向卷积网络中引入非线性，因为真实世界里大多数需要学习的问题都是非线性的（单纯的卷积操作时线性的——矩阵相乘、相加，所以才需要额外的计算引入非线性）。

![ReLU](http://upload-images.jianshu.io/upload_images/4749583-121c692f875600f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4749583-6421db9ebbc4bfa4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


其实可以这么理解，大于0的都是关心的数据，小于0的是不关心的数据，因此直接小于0的去掉，符合人的基本逻辑感知。
其他非线性方程比如tanh或sigmoid也可以替代ReLU，但大家一般都用ReLU。

### 池化
简单来说就是把图片降size的，降低处理量嘛；空间池化（也叫亚采样或下采样）降低了每个特征映射的维度，但是保留了最重要的信息。空间池化可以有很多种形式：最大(Max)，平均(Average)，求和(Sum)等等。


![](http://upload-images.jianshu.io/upload_images/4749583-4c54f1099461d5b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![池化效果](http://upload-images.jianshu.io/upload_images/4749583-beea7a88932f2f03.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


池化的功能逐步减少输入表征的空间尺寸。特别地，池化

+ 使输入表征（特征维度）更小而易操作
+  减少网络中的参数与计算数量，从而遏制过拟合
+ 增强网络对输入图像中的小变形、扭曲、平移的鲁棒性（输入里的微小扭曲不会改变池化输出——因为我们在局部邻域已经取了最大值/平均值）。
+  帮助我们获得不因尺寸而改变的等效图片表征。这非常有用，因为这样我们就可以探测到图片里的物体，不论那个物体在哪。

### 全连接层
全连接层(Fully Connected layer)就是使用了softmax激励函数作为输出层的多层感知机(Multi-Layer Perceptron)，其他很多分类器如支持向量机也使用了softmax。

“全连接”表示上一层的每一个神经元，都和下一层的每一个神经元是相互连接的。

到目前为止，前面的一些卷积、非线性、池化操作就已经把特征给选出来了，而且是**高级特征**哟，这个时候用感知机那一套就可以实现拟合了嘛，由于我们实际场景可能是多分类问题，因此我们用的是softmax。

除了分类以外，加入全连接层也是学习特征之间非线性组合的有效办法。卷积层和池化层提取出来的特征很好，但是如果考虑这些特征之间的组合，就更好了。

全连接层的输出概率之和为1，这是由激励函数Softmax保证的。Softmax函数把任意实值的向量转变成元素取之0-1且和为1的向量。


### 得开始训练啦！！！
从前面可知**卷积+池化是特征提取器，全连接层是分类器**。

![训练CNN](http://upload-images.jianshu.io/upload_images/4749583-ca46e92debab1a4a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如上因为输入的图片是船，因此最后的结果向量是[0,0,1,0]。

**训练过程如下：**

1. 用随机数初始化所有的**滤波器和参数/权重**。
2. 输入图片，前向执行**卷积、ReLU、池化及全连接的前向传播，并计算出每个类别对应的输出概率**。
    -  假设船图的输出概率是[0.2, 0.4, 0.1, 0.3]
    -  因为第一个训练样本的权重都是**随机**的，所以这个输出概率也跟随机的差不多.
3. 计算输出层的总误差（**4类别之和**）:这用的是均方误差！**MSE**！
Total Error = ∑  ½ (target probability – output probability) ²
4. 反向传播用总的误差，来计算每个权重的梯度并更新之，也就是我们眼光放到全局来做宏观调控，具体的“政策”细节参数则由上到下反过来一个一个调节。
   - 权重的调整程度与其**对总误差的贡献**成正比。
   - 当**同一图像**再次被输入，这次的输出概率可能是[0.1, 0.1, 0.7, 0.1]，与目标[0, 0, 1, 0]更接近了。
   - 这说明我们的神经网络已经学习着分类特定图片了，学习的方式是调整权重/滤波器以降低输出误差。
   - 如果滤波器个数、滤波器尺寸、网络架构这些参数，是在第一步之前就已经固定的，且不会在训练过程中改变（也就是这个网络结构可以随意改？）——只有卷积核跟全连接网络的权重会更新。

>上述步骤总结起来就是：**优化所有的权重和参数，使其能够正确地分类训练集里的图片。**

上例中，我们用了两组卷积+池化层，其实这些操作可以在一个卷积网络内重复无数次。**如今有些表现出众的卷积网络，都有数以十计的卷积+池化层！并且，不是每个卷积层后面都要跟个池化层。**由下图可见，我们可以有连续多组卷积+ReLU层，后面再加一个池化层。


![](http://upload-images.jianshu.io/upload_images/4749583-a6469df9b8c45340.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>小结：上面的卷积核有些是训练出来的，我们可以直接使用的呐！比如下面这位大神就用其中一个卷积核做了一件了不得的事，[**客官请移步!**](https://zhuanlan.zhihu.com/p/25774111)
---
# 可视化卷积神经网络

![Convolutional Deep Belief Network学习的特征](http://upload-images.jianshu.io/upload_images/4749583-53564fae9ccba3c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

一般来说，卷积层越多，能学会的特征也就越复杂。比如在图像分类中，一个卷积神经网络的第一层学会了探测像素中的边缘，然后第二层用这些边缘再去探测简单的形状，其他层再用形状去探测高级特征，比如脸型，如**图17**所示——这些特征是[Convolutional Deep Belief Network**](http://link.zhihu.com/?target=http%3A//web.eecs.umich.edu/%7Ehonglak/icml09-ConvolutionalDeepBeliefNetworks.pdf)学得的。这里只是一个简单的例子，实际上卷积滤波器可能会探测出一些**没有意义的特征**。

#其他卷积网络架构
卷积神经网络始自1990年代起，我们已经认识了最早的LeNet，其他一些很有影响力的架构列举如下：

>1990s至2012：从90年代到2010年代早期，卷积神经网络都处于孵化阶段。随着数据量增大和计算能力提高，卷积神经网络能搞定的问题也越来越有意思了。

+ AlexNet(2012)：2012年，Alex Krizhevsky发布了[AlexNet**](http://link.zhihu.com/?target=https%3A//papers.nips.cc/paper/4824-imagenet-classification-with-deep-convolutional-neural-networks.pdf)，是LeNet的更深、更宽版本，并且大比分赢得了当年的ImageNet大规模图像识别挑战赛(ILSVRC)。这是一次非常重要的大突破，现在普及的卷积神经网络应用都要感谢这一壮举。

+ ZF Net(2013)：2013年的ILSVRC赢家是Matthew Zeiler和Rob Fergus的卷积网络，被称作[ZF Net**](http://link.zhihu.com/?target=http%3A//arxiv.org/abs/1311.2901)，这是调整过架构超参数的AlexNet改进型。

+ GoogleNet(2014)：2014的ILSVRC胜者是来自Google的[Szegedy et al.**](http://link.zhihu.com/?target=http%3A//arxiv.org/abs/1409.4842)。其主要贡献是研发了*Inception Module*，它大幅减少了网络中的参数数量（四百万，相比AlexNet的六千万）。

+ VGGNet(2014)：当年的ILSVRC亚军是[VGGNet**](http://link.zhihu.com/?target=http%3A//www.robots.ox.ac.uk/%7Evgg/research/very_deep/)，突出贡献是展示了网络的深度（层次数量）是良好表现的关键因素。

+ ResNet(2015)： Kaiming He研发的[Residual Network**](http://link.zhihu.com/?target=http%3A//arxiv.org/abs/1512.03385)是2015年的ILSVRC冠军，也代表了卷积神经网络的最高水平，同时还是实践的默认选择（2016年5月）。

+ DenseNet（2016年8月）： 由Gao Huang发表，[Densely Connected Convolutional Network**](http://link.zhihu.com/?target=http%3A//arxiv.org/abs/1608.06993)的每一层都直接与其他各层前向连接。DenseNet已经在五个高难度的物体识别基础集上，显式出非凡的进步。
```
AlexNet，2012年 ；
ZF-net，2013年；
NIN，2013年；
GoogLeNet (InceptionV1)，2014年；
VGG，2014年；
Batch Norm (InceptionV2)，2015年；
InceptionV3 ，2015年；
ResNet，2015年；
InceptionV4，2016年。
```

---
# LeNet-5实现（TensorFlow）

LeNet-5是用于手写字体的识别的一个经典CNN：

![](http://upload-images.jianshu.io/upload_images/4749583-a8737898b4a0c8f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**输入：** 32*32的手写字体图片，这些手写字体包含0~9数字，我们可以从官网上下载标准的minist数据集（TF包里面自带下载函数的，是通过read_data_sets（）函数里面的下载函数下载的。）

```python
@retry(initial_delay=1.0, max_delay=16.0, is_retriable=_is_retriable)
def urlretrieve_with_retry(url, filename=None):
  return urllib.request.urlretrieve(url, filename)

def maybe_download(filename, work_directory, source_url):
  """Download the data from source url, unless it's already here.

  Args:
      filename: string, name of the file in the directory.
      work_directory: string, path to working directory.
      source_url: url to download from if file doesn't exist.

  Returns:
      Path to resulting file.
  """
  if not gfile.Exists(work_directory):
    gfile.MakeDirs(work_directory)
  filepath = os.path.join(work_directory, filename)
  if not gfile.Exists(filepath):
    temp_file_name, _ = urlretrieve_with_retry(source_url)
    gfile.Copy(temp_file_name, filepath)
    with gfile.GFile(filepath) as f:
      size = f.size()
    print('Successfully downloaded', filename, size, 'bytes.')
  return filepath
```
**输出**：分类结果，0~9之间的一个数

---

**LeNet-5结构是固定了的：**输入层为32*32的图片，也就是相当于1024个神经元。(我们直接调用TF包的数据集获取函数即可，实现见上述代码)

```python
#获取训练集
mnist = input_data.read_data_sets("MNIST_data/", one_hot=True)python
#同时创建一个session准备开始干活
#交互式session
sess = tf.InteractiveSession()
```

**C1层：**paper作者，选择6个特征卷积核，然后卷积核大小选择5\*5，这样我们可以得到6个特征图，然后每个特征图的大小为32-5+1=28，也就是神经元的个数由1024减小到了28*28=784。
```python
#已将28*28的图片展开成行向量，None代表该维度上数量不确定，
# 此处指的是取样的图像数量不确定
x = tf.placeholder(tf.float32, shape=[None,784])

#图片的标签
y_label = tf.placeholder(tf.float32, shape=[None,10])
```


