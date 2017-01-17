# 手写字符识别教程

## 背景介绍
当我们学习编程的时候，编写的第一个程序一般是实现打印"Hello World"。而机器学习（或深度学习）的入门教程，一般都是 [MNIST](http://yann.lecun.com/exdb/mnist/) 数据库上的手写识别问题。原因是手写识别属于典型的图像分类问题，比较简单，同时MNIST数据集也很完备。

MNIST数据集作为一个简单的计算机视觉数据集，包含一系列如图1所示的手写数字图片和对应的标签。图片是28x28的像素矩阵，而标签则对应着0~9的10个数字。每张图片都经过了大小归一化和居中处理。

<p align="center">
<img src="image/mnist_example_image.png" width="400"><br/>
图1. MNIST图片示例
</p>

MNIST数据集是从 [NIST](https://www.nist.gov/srd/nist-special-database-19) 的Special Database 3（SD-3）和Special Database 1（SD-1）构建而来。由于SD-3是由美国人口调查局的员工进行标注，SD-1是由美国高中生进行标注，因此SD-3比SD-1更干净也更容易识别。Yann
LeCun等人从SD-1和SD-3中各取一半作为MNIST的训练集（60000条数据）和测试集（10000条数据），其中训练集来自250位不同的标注员，此外还保证了训练集和测试集的标注员是不完全相同的。\[[1](#参考文献)\]


该数据库的提供者Yann  LeCun，早先在手写字符识别上做了很多研究，并在研究过程中提出了卷积神经网络（Convolutional Neural Network），大幅度地提高了手写字符的识别能力，也因此成为了深度学习领域的奠基人之一。如今的深度学习领域，卷积神经网络占据了至关重要的地位，从最早Yann
LeCun提出的简单LeNet，到如今ImageNet大赛上的优胜模型VGGNet、GoogLeNet、ResNet等（我们将在下一章中介绍这三个模型），人们在图像分类领域，利用卷积神经网络得到了一系列惊人的结果。
自从该数据集发布以来，有很多算法在MNIST上进行实验。1998年，LeCun分别用单层线性分类器、多层感知器（Multilayer Perceptron, MLP）和多层卷积神经网络LeNet进行实验，使得测试集上的误差不断下降（从12%下降到0.7%）。此后，科学家们分别基于K近邻（K-Nearest Neighbors）算法、支持向量机（SVM）、神经网络和Boosting方法在该数据集上做了大量实验，并采用多种预处理方法（如去除歪曲（deskewing），去噪（noise removal），模糊（blurring）等等）来提高识别的准确率。\[[2-13](#参考文献)\]

本章中，我们从简单的模型Softmax回归开始，带大家入门手写字符识别，并逐步进行模型优化。


## 模型概览

这是一个分类问题，基于MNIST数据，我们希望训练一个分类器 $f$。输入为MNIST数据库的图片， $28\times28$ 的二维图像，为了进行计算，我们一般将上将 $28\times28$ 的二维图像转化为 $n(n=784)$ 维的向量，因此我们采用$x_i(i=0,1,2,...,n-1)$(向量表示为$X$)来表示输入的图片数据。对于每张给定的图片数据 $X$，我们采用$ y_i (i=0,1,2,..9)$(向量表示为 $Y$来表示预测的输出，预测结果为 $ Y = f(X) $。然后，我们用 $label_i$ (向量表示为$L$)来代表标签，则预测结果 $Y$ 应该尽可能准确的接近真实标签 $L$。

$Y$和$L$具体含义为：比如说，$y_i$组成的向量$Y$为[0.2,0.3,0,1,0,0.1,0.1,0.1,0.1,0]，每一维分别代表图像数字预测为0~9的概率；而此时$label_i$组成的向量$L$可能为[0,1,0,0,0,0,0,0,0,0]，其代表标签为1，即输入$X$代表图片的数字为1。则$Y$和$L$尽可能接近的意思是$Y$中概率最大的一维为$L$中对应的标签，并且概率越大则代表越接近。

下面我们一一介绍本章中使用的三个分类器Softmax回归、多层感知器、卷积神经网络。


### Softmax回归(Softmax Regression)

神经网络中通常使用 $softmax$ 函数计算多分类问题中每一类的概率。为了熟悉 $softmax$ 函数，我们定义最简单的多分类网络：将输入层经过一个线性映射得到的特征，直接通过 $softmax$ 函数进行多分类。\[[14](#参考文献)\]

输入层的数据X传到 $softmax$ 层，在激活操作之前，会乘以相应的权重 $W$ ，并加上偏置变量 $b$ ，具体如下：

$$ y_i = softmax(\sum_j W_{i,j}x_j + b_i) $$

其中 $ softmax(x_i) = \frac{e^{x_i}}{\sum_j e^{x_j}} $

对于有 $N$ 个类别的多分类问题，指定 $N$ 个输出节点，$N$ 维输入特征经过 $softmax$ 将归一化为 $N$ 个[0,1]范围内的实数值，分别表示该样本属于这N个类别的概率。此处的 $y_i$ 即对应该图片为数字 $i$ 的预测概率。

图2为softmax回归的网络图，图中权重用黑线表示，偏置用红线表示，+1代表偏置参数的系数为1。
<p align="center">
<img src="image/softmax_regression.png"><br/>
图2. softmax回归网络结构图<br/>
</p>

神经网络的训练采用 `backpropagation` 的形式，其一般会定义一个损失函数（也称目标函数），训练的目的是为了减小目标函数的值。在分类问题中，我们一般采用交叉熵代价损失函数(cross entropy)，其形式如下：

$$  CE(label, y) = -\sum_i label_ilog(y_i) $$

上面公式为softmax输出层的交叉熵代价损失函数，CE为cross entropy的简称，y是预测每一类的概率，label是标签。



### 多层感知器(Multilayer Perceptron, MLP)

在softmax回归中，我们采用了最简单的两层神经网络，分别为输入的data层和输出的 $softmax$ 层。由于这个模型比较简单，其拟合能力有限。因此，为了达到更好的识别效果，我们可以考虑在输入层和输出层中间加上若干个隐藏层。\[[15-16](#参考文献)\]

在该网络层中，我们有输入X($x_i(i=0,1,2,...,n-1)$)，输出标签Y($y_i(i=0,1,2,..9)$)，为了表示方便，以下我们都直接用向量计算来表示。经过第一层网络，我们可以得到：

$$ H_1 = \phi(W_1X + b_1) $$

上面，$\phi$ 代表 [激活函数](#常见激活函数介绍)，其常见的为 $sigmoid$ ，$tanh$ 或 $ReLU$ 等函数。

经过第二层网络，可以得到：

$$ H_2 = \phi(W_2H_1 + b_2) $$

最后，再经过输出层：

$$ Y = softmax(W_3H_2 + b_3) $$

得到的$Y$即为最后的预测结果向量。


图3为多层感知器的网络结构图，图中权重用黑线表示，偏置用红线表示，+1代表偏置参数的系数为1
<p align="center">
<img src="image/MLP.png"><br/>
图3. 多层感知器网络结构图<br/>
</p>

### 卷积神经网络(Convolutional Neural Network, CNN)

#### 卷积层
<p align="center">
<img src="image/conv_layer.png"><br/>
图4. 卷积层图片<br/>
</p>
卷积层是卷积神经网络的核心基石。该层的参数由一组可学习的过滤器（也叫作卷积核）组成。在前向过程中，每个卷积核在输入层进行横向和纵向的扫描，与输入层对应扫描位置进行卷积，得到的结果加上偏置并用相应的激活函数进行激活，结果能够得到一个二维的激活图(activation map)。每个特定的卷积核都能得到特定的激活图(activation map)，如有的卷积核可能对识别边角，有的可能识别圆圈，那这些卷积核可能对于对应的特征响应要强。\[[17](#参考文献)\]

图4是卷积层的一个动态图。由于3D量难以表示，所有的3D量（输入的3D量（蓝色），权重3D量（红色），输出3D量（绿色））通过将深度在行上堆叠来表示。如图4，输入层是$W_1=5,H_1=5,D_1=3$，我们常见的彩色图片其实就是类似这样的输入层，彩色图片的宽和高对应这里的$W_1$和$H_1$，而彩色图片有RGB三个颜色通道，对应这里的$D_1$；卷积层的参数为$K=2,F=3,S=2,P=1$，这里的$K$是卷积核的数量，如图4中有$Filter W_0$和$Filter   W_1$两个卷积核，$F$对应卷积核的大小，图中$W0$和$W1$在每一层深度上都是$3\times3$的矩阵，$S$对应卷积核扫描的步长，从动态图中可以看到，方框每次左移或下移2个单位，$P$对应Padding扩展，是对输入层的扩展，图中输入层，原始数据为蓝色部分，可以看到灰色部分是进行了大小为1的扩展，用0来进行扩展；图4的动态可视化对输出层结果（绿色）进行迭代，显示每个输出元素是通过将突出显示的输入（蓝色）与滤波器（红色）进行元素相乘，将其相加，然后通过偏置抵消结果来计算的。\[[18](#参考文献)\]

#### 池化层
<p align="center">
<img src="image/max_pooling.png" width="400px"><br/>
图5. 池化层图片<br/>
</p>
池化是非线性下采样的一种形式，主要作用是通过减少网络的参数来减小计算量，并且能够在一定程度上控制过拟合。通常在卷积层的后面会加上一个池化层。

池化包括最大池化、平均池化等。其中最大池化是用不重叠的矩形框将输入层分成不同的区域，对于每个矩形框的数取最大值作为输出层，如图5所示。

#### LeNet网络
<p align="center">
<img src="image/cnn.png"><br/>
图6. 卷积神经网络结构<br/>
</p>
LeNet网络的典型结构是：输入的二维图像，首先经过两次卷积层到池化层，再经过全连接层，最后使用softmax分类作为输出层。

相比于全连接层的多层感知器来说，卷积神经网络有以下特性，这三个特性决定了卷积能够更好的识别图像。\[[17](#参考文献)\]

- 神经元的三维特性： 卷积层的神经元在宽度、高度和深度上进行了组织排列。每一层的神经元仅仅与前一层的一块小区域连接，这块小区域被称为感受野。
- 局部连接：CNN通过在相邻层的神经元之间实施局部连接模式来利用空间局部相关性。这样的结构保证了学习后的过滤器能够对于局部的输入特征有最强的响应。堆叠许多这样的层导致非线性“过滤器”变得越来越“全局”。这允许网络首先创建输入的小部分的良好表示，然后从它们组合较大区域的表示。
- 共享权重：在CNN中，每个滤波器在整个视野中重复扫描。 这些复制单元共享相同的参数化（权重向量和偏差）并形成特征图。 这意味着给定卷积层中的所有神经元检测完全相同的特征。 以这种方式的复制单元允许不管它们在视野中的位置都能检测到特征，从而构成平移不变性的性质。

更详细的关于卷积神经网络的具体知识可以参考斯坦福大学公开课 [cs231n]( http://cs231n.github.io/convolutional-networks/ )

###常见激活函数介绍 \[[19](#参考文献)\]
- sigmoid激活函数：

$$ f(x) = \frac{1}{1+e^{-x}} $$

- tanh激活函数：

$$ f(x) = tanh(x) = \frac{e^x-e^{-x}}{e^x+e^{-x}} $$

实际上，$tanh$ 函数只是规模变化的 $sigmoid$ 函数，将 $sigmoid$函数值放大2倍之后再向下平移1个单位：$ tanh(x) = 2sigmoid(2x) - 1 $

- ReLU激活函数：

$$ f(x) = max(0, x) $$


## 数据准备

### 数据介绍与下载

我们首先下载 [MNIST](http://yann.lecun.com/exdb/mnist/) 数据库,该数据库是手写字符识别常用的数据库。执行以下命令，进行下载并解压缩，并将训练集和测试集的地址分别写入train.list和test.list两个文件，供PaddlePaddle读取。

```bash
./data/get_mnist_data.sh
```

将下载下来的数据进行 `gzip` 解压，可以在文件夹 `data/raw_data` 中找到以下文件：

|    文件名称          |       说明              |
|----------------------|-------------------------|
|train-images-idx3-ubyte|  训练数据图片，60,000条数据 |
|train-labels-idx1-ubyte|  训练数据标签，60,000条数据 |
|t10k-images-idx3-ubyte |  测试数据图片，10,000条数据 |
|t10k-labels-idx1-ubyte |  测试数据标签，10,000条数据 |

训练和测试数据的图片格式类似如下：

| [偏移] | [类型] | [值] | [描述] |
|----------|--------|---------|---------------|
|   0000   | 32位整型 | 0x000000803(2051) | magic number (高字节放入低地址) |
|   0004   | 32位整型 | 60000             | 图片数量 |
|   0008   | 32位整型 | 28                | 行数     |
|   0012   | 32位整型 | 28                | 列数     |
|   0016   | 无符号字节 | ??                | 像素点 |
|   0017   | 无符号字节 | ??                | 像素点 |
|   ....   |   ....     |  ......           |  ......         |
|  xxxx    | 无符号字节 | ??                | 像素点 |


训练和测试数据的标签类似格式如下：

| [偏移] | [类型] | [值] | [描述] |
|----------|--------|---------|---------------|
|   0000   | 32位整型   | 0x000000801(2049) | magic number (高字节放入低地址) |
|   0004   | 32位整型   | 60000             | 标签数量 |
|   0008   | 无符号字节 | ??                | 标签   |
|   0009   | 无符号字节 | ??                | 标签   |
|   ....   |   ....     |  ......       |  ......|
|  xxxx    | 无符号字节 | ??                | 标签    |

表格格式说明：
开头的8个字节，前面四个字节为magic number，后面四个字节代表总共的数据量，训练数据为60,000，测试数据量为10,000，而对于图片数据中，第二个8个字节，前面四个字节为行数，后面四个字节为列数。之后的字节数据分别为图片数据的像素点和标签数据的标签。像素点用一个字节表示一个像素，可以得出有$28\times28$个字节表示一张图片，而标签中一个字节则可以代表一个标签。

对于magic number，开始的两个字节总是0，而第三个字节如下所示：

- 0x08: 无符号字节unsigned byte
- 0x09: 有符号字节signed byte
- 0x0B: 短整型short（两个字节）
- 0x0C: 整型int（四个字节）
- 0x0D: 单精度浮点float（四个字节）
- 0x0E: 双精度浮点double（八个字节）

MNIST数据对应的都是无符号字节的存储。

magic number的第四个字节则是代表储存向量或矩阵的维度：1代表向量，2代表矩阵......

数据是以C array的方式进行存储的，即最后一维的索引变量变化最快。\[[1](#参考文献)\]

用户可以通过以下脚本随机绘制10张图片

```bash
./data/load_data.py
```

### 提供数据给PaddlePaddle

我们使用python接口传递数据给系统，下面 `mnist_provider.py`针对MNIST数据给出了完整示例。

```python
# Define a py data provider
@provider(
    input_types={'pixel': dense_vector(28 * 28),
                 'label': integer_value(10)})
def process(settings, filename):  # settings is not used currently.
    with open( filename + "-images-idx3-ubyte", "rb") as f:             # 打开图片文件
        magic, n, rows, cols = struct.upack(">IIII", f.read(16))        # 读取开头的四个参数
        images = np.fromfile(                                           # 以无符号字节为单位一个一个的读取数据
            f, 'ubyte',
            count=n * rows * cols).reshape(n, rows, cols).astype('float32')

    with open( filename + "-labels-idx1-ubyte", "rb") as l:             # 打开标签文件
        magic, n = struct.upack(">II", l.read(8))                       # 读取开头的两个参数
        labels = np.fromfile(l, 'ubyte', count=n).astype("int")         # 以无符号字节为单位一个一个的读取数据

    for i in xrange(n):
        yield {"pixel": images[i, :], 'label': labels[i]}
```


## 模型配置说明

### 数据定义

在模型配置中，定义通过 `define_py_data_sources2` 函数从 `dataprovider` 中读入数据，如果该配置用于预测，则不需要数据定义部分。

```python
 if not is_predict:
     data_dir = './data/'
     define_py_data_sources2(
         train_list=data_dir + 'train.list',
         test_list=data_dir + 'test.list',
         module='mnist_provider',
         obj='process')
```

### 算法配置

然后指定训练相关的参数。

- batch_size： 表示神经网络每次训练使用的数据为128条。
- 训练速度(learning_rate)： 迭代的速度，影响着网络的训练收敛速度有关系。
- 训练方法(learning_method)： 代表训练过程在更新权重时采用动量优化器( `MomentumOptimizer` )，其中参数0.9代表动量优化每次保持前一次速度的0.9倍。
- 正则化(regularization)： 是防止网络过拟合的一种手段，此处采用L2正则化。

```python
settings(
    batch_size=128,
    learning_rate=0.1 / 128.0,
    learning_method=MomentumOptimizer(0.9),
    regularization=L2Regularization(0.0005 * 128))
```

### 模型结构

#### Softmax回归

定义好 `dataprovider` 之后，就可以通过 `data_layer` 调用来获取数据 `img`，然后通过一层简单的 `softmax`全连接层，得到预测的结果，然后指定训练的损失函数为分类损失( `classification_cost` )，一般分类问题的损失函数为交叉熵损失函数( `cross_entropy` )。通过控制变量 `is_predict` ，该配置脚本也可以在预测时候使用，将 `is_predict` 置为 `True` ，则最后直接输出预测结果，而不会经过损失函数来进行训练过程。

```python
data_size = 1 * 28 * 28
label_size = 10
img = data_layer(name='pixel', size=data_size)

def softmax_regression(img):
    predict = fc_layer(input=img, size=10, act=SoftmaxActivation())
    return predict
```

#### 多层感知器

以下是一个简单的带有两个隐藏层的多层感知器，也就是全连接网络，两个隐藏层的激活函数均采用 $ReLU$ 函数，最后的输出层用$softmax$ 激活函数。

```python
def multilayer_perceptron(img):
    # The first fully-connected layer
    hidden1 = fc_layer(input=img, size=128, act=ReluActivation())
    # The second fully-connected layer and the according activation function
    hidden2 = fc_layer(input=hidden1, size=64, act=ReluActivation())
    # The thrid fully-connected layer, note that the hidden size should be 10,
    # which is the number of unique digits
    predict = fc_layer(input=hidden2, size=10, act=SoftmaxActivation())
    return predict
```

#### 卷积神经网络LeNet-5

以下为LeNet-5的网络结构

```python
def convolutional_neural_network(img):
    # first conv layer
    conv_pool_1 = simple_img_conv_pool(
        input=img,
        filter_size=5,
        num_filters=20,
        num_channel=1,
        pool_size=2,
        pool_stride=2,
        act=TanhActivation())
    # second conv layer
    conv_pool_2 = simple_img_conv_pool(
        input=conv_pool_1,
        filter_size=5,
        num_filters=50,
        num_channel=20,
        pool_size=2,
        pool_stride=2,
        act=TanhActivation())
    # The first fully-connected layer
    fc1 = fc_layer(input=conv_pool_2, size=128, act=TanhActivation())
    # The softmax layer, note that the hidden size should be 10,
    # which is the number of unique digits
    predict = fc_layer(input=fc1, size=10, act=SoftmaxActivation())
    return predict
```

## 训练命令及日志

1.通过配置训练脚本 `train.sh` 来执行训练过程

```bash
config=mnist_model.py                   # 在mnist_model.py中可以选择网络
output=./softmax_mnist_model            # mlp: ./mlp_mnist_model        cnn: ./cnn_mnist_model 
log=softmax_train.log                   # mlp: mlp_train.log            cnn: cnn_train.log

paddle train \
--config=$config \
--dot_period=10 \
--log_period=100 \
--test_all_data_in_one_period=1 \
--use_gpu=0 \
--trainer_count=1 \
--num_passes=100 \
--save_dir=$output \
2>&1 | tee $log

python -m paddle.utils.plotcurve -i $log > plot.png
```

参数意义分别为：
- config:  网络配置的脚本。
- dot_period:  在每训练 `dot_period` 个批次后打印一个 `.`。
- log_period:  每隔多少batch打印一次日志。
- test_all_data_in_one_period:  每次测试是否用所有的数据。
- use_gpu:	是否使用GPU。
- trainer_count:  使用CPU或GPU的个数。
- num_passed:  训练进行的轮数（每次训练使用完所有数据为1轮）。
- save_dir:  模型存储的位置。
配置好参数之后，执行脚本 `./train.sh` 训练日志如下所示：

模型训练的日志类似如下：

```
I1227 02:58:08.519176   275 Util.cpp:229] copy mlp_mnist.py to ./mlp_mnist_model/pass-00087
.........
I1227 02:58:09.395433   275 TrainerInternal.cpp:165]  Batch=100 samples=12800 AvgCost=0.200571 CurrentCost=0.200571 Eval: classification_error_evaluator=0.0621875  CurrentEval: classification_error_evaluator=0.0621875 
.........
I1227 02:58:10.265552   275 TrainerInternal.cpp:165]  Batch=200 samples=25600 AvgCost=0.21248 CurrentCost=0.224389 Eval: classification_error_evaluator=0.065625  CurrentEval: classification_error_evaluator=0.0690625 
.........
I1227 02:58:11.120333   275 TrainerInternal.cpp:165]  Batch=300 samples=38400 AvgCost=0.209837 CurrentCost=0.204553 Eval: classification_error_evaluator=0.0649479  CurrentEval: classification_error_evaluator=0.0635938 
.........
I1227 02:58:11.964988   275 TrainerInternal.cpp:165]  Batch=400 samples=51200 AvgCost=0.215699 CurrentCost=0.233282 Eval: classification_error_evaluator=0.0667773  CurrentEval: classification_error_evaluator=0.0722656 
......I1227 02:58:12.554342   275 TrainerInternal.cpp:181]  Pass=88 Batch=469 samples=60000 AvgCost=0.218966 Eval: classification_error_evaluator=0.0676833 
I1227 02:58:12.871682   275 Tester.cpp:109]  Test samples=10000 cost=0.214518 Eval: classification_error_evaluator=0.064 
I1227 02:58:12.871961   275 GradientMachine.cpp:113] Saving parameters to ./mlp_mnist_model/pass-00088
```

2.用脚本 `plot_error.py` 可以画出训练过程中的误差变化曲线：

```bash
python plot_error.py softmax_train.log              # mlp: mlp_train.log        cnn: cnn_train.log
```

3.用脚本 `evaluate.py ` 可以选出最好的Pass训练出来的模型：

```bash
python evaluate.py softmax_train.log
```

#### softmax回归的训练结果

<p align="center">
<img src="image/softmax_train_log.png" width="400px"><br/>
图7. softmax回归的误差曲线图<br/>
</p>

评估模型结果如下：

```text
Best pass is 00047, testing Avgcost is 0.473053
The classification accuracy is 89.49%
```

从上面过程中可以看到，softmax回归模型分类效果最好的时候是pass-00047，分类准确率为89.49%，而最终的pass-00099的准确率为85.39%。从图中也可以看出，准确率最好的时候并以不定是最后一个pass的模型（这是因为迭代的中间Pass可能已经收敛达到局部最优值，后面的训练可能是在该局部最优值震荡或者达到另外的低于中间Pass局部最优值的其他局部最优。）

#### 多层感知器的训练结果

<p align="center">
<img src="image/mlp_train_log.png" width="400px"><br/>
图8. 多层感知器的误差曲线图
</p>

评估模型结果如下：

```text
Best pass is 00085, testing Avgcost is 0.164746
The classification accuracy is 94.95%
```

从训练日志中我们可以看出，最终训练的准确率为94.95%，相比于softmax回归模型有了显著的提升（这是因为softmax回归模型较为简单，无法拟合更为复杂的数据，而加入了隐藏层之后的多层感知器则具有更强的拟合能力）。

#### 卷积神经网络的训练结果

<p align="center">
<img src="image/cnn_train_log.png" width="400px"><br/>
图9. 卷积神经网络的误差曲线图
</p>

评估模型结果如下：

```text
Best pass is 00076, testing Avgcost is 0.0244684
The classification accuracy is 99.20%
```

从评估模型的结果可以看到，卷积神经网络的最好分类准确率达到惊人的99.20%。由此可以看到，对于图像问题而言，卷积神经网络能够比一般的全连接网络达到更好的识别效果，而这与卷积层具有局部连接和共享权重的特性是分不开的。同时，从图9中可以看到，卷积神经网络在很早的时候就能达到很好的效果，说明其收敛速度非常快。

## 应用模型

### 预测命令与结果
脚本  `predict.py` 可以对训练好的模型进行预测，例如softmax回归中：

- -c 指定模型的结构
- -d 指定需要预测的数据源，这里用测试数据集进行预测
- -m 指定模型的参数，这里用之前训练效果最好的模型进行预测

```bash
python predict.py -c softmax_mnist.py -d data/raw_data/ -m softmax_mnist_model/pass-00047
```

根据提示，输入需要预测的图片序号，则分类器能够给出各个数字的生成概率、预测的结果（取最大生成概率对应的数字）和实际的标签。

```
Input image_id [0~9999]: 3
Predicted probability of each digit:
[[  1.00000000e+00   1.60381094e-28   1.60381094e-28   1.60381094e-28
    1.60381094e-28   1.60381094e-28   1.60381094e-28   1.60381094e-28
    1.60381094e-28   1.60381094e-28]]
Predict Number: 0 
Actual Number: 0
```

从结果看出，该分类器接近100%地认为第3张图片上面的数字为0，而实际标签给出的类也确实如此。


## 总结

本章的softmax回归、多层感知器和卷积神经网络是最基础的深度学习模型，后续章节中复杂的神经网络都是从它们衍生出来的，因此这几个模型对之后的学习大有裨益。同时，我们也观察到从最简单的Softmax回归到稍复杂的卷积神经网络的时候，MNIST数据集上的识别准确率有了大幅度的提升，原因是卷积层局部连接和共享权重的特性。在之后学习新模型的时候，希望大家也要深入到新模型相比原模型带来效果提升的关键之处。此外，本章还介绍了PaddlePaddle模型搭建的基本流程，从dataprovider的编写、网络层的构建，到最后的训练和预测。对这个流程熟悉以后，大家就可以用自己的数据，定义自己的网络模型，并完成自己的训练和预测任务。

## 参考文献

1. Yann LeCun网站MNIST数据库介绍 http://yann.lecun.com/exdb/mnist/
2. LeCun, Yann, et al. "Gradient-based learning applied to document recognition." Proceedings of the IEEE 86.11 (1998): 2278-2324.
3. Simard, Patrice Y., David Steinkraus, and John C. Platt. "Best practices for convolutional neural networks applied to visual document analysis." ICDAR. Vol. 3. 2003.
4. Keysers, Daniel, et al. "Deformation models for image recognition." IEEE Transactions on Pattern Analysis and Machine Intelligence 29.8 (2007): 1422-1435.
5. Mori, Greg, and Jitendra Malik. "Estimating human body configurations using shape context matching." European conference on computer vision. Springer Berlin Heidelberg, 2002.
6. Kégl, Balázs, and Róbert Busa-Fekete. "Boosting products of base classifiers." Proceedings of the 26th Annual International Conference on Machine Learning. ACM, 2009.
7. Decoste, Dennis, and Bernhard Schölkopf. "Training invariant support vector machines." Machine learning 46.1-3 (2002): 161-190.
8. Simard, Patrice Y., David Steinkraus, and John C. Platt. "Best practices for convolutional neural networks applied to visual document analysis." ICDAR. Vol. 3. 2003.
9. Salakhutdinov, Ruslan, and Geoffrey E. Hinton. "Learning a Nonlinear Embedding by Preserving Class Neighbourhood Structure." AISTATS. 2007.
10. Ciresan, Dan Claudiu, et al. "Deep, big, simple neural nets for handwritten digit recognition." Neural computation 22.12 (2010): 3207-3220.
11. Ciresan, Dan C., et al. "Flexible, high performance convolutional neural networks for image classification." IJCAI Proceedings-International Joint Conference on Artificial Intelligence. Vol. 22. No. 1. 2011.
12. Deng, Li, et al. "Binary coding of speech spectrograms using a deep auto-encoder." Interspeech. 2010.
13. Lauer, Fabien, Ching Y. Suen, and Gérard Bloch. "A trainable feature extractor for handwritten digit recognition." Pattern Recognition 40.6 (2007): 1816-1824.
14. 维基百科Softmax函数介绍 https://en.wikipedia.org/wiki/Softmax_function
15. Rosenblatt F. The perceptron: a probabilistic model for information storage and organization in the brain[J]. Psychological review, 1958, 65(6): 386.
16. Bishop C M. Pattern recognition[J]. Machine Learning, 2006, 128.
17. 维基百科卷积神经网络介绍 https://en.wikipedia.org/wiki/Convolutional_neural_network
18. 斯坦福大学公开课cs231n http://cs231n.github.io/convolutional-networks/
19. 维基百科激活函数介绍 https://en.wikipedia.org/wiki/Activation_function
