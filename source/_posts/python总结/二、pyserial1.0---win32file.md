---
title: 二、pyserial1.0---win32file
categories: python总结
---
我看最新的pyserial代码发现看不懂~Σ( ° △ °|||)︴，尝试用跑跑看一看效果，debug跟一跟流程的方式来熟悉理解，发现·······转太多弯啦！！！根本就hold不过来啊！脑容量跟基本功都不够啊！于是想着怎么从侧面来攻破这个堡垒............

然后在网上找学习经验，突然想到这个包当初肯定是有个起点(16years ago )的，起点是容易理解且不复杂的，解决一个单一问题的，就像我自己写框架一样（即先快速做出功能、原型，然后再反复迭代改进）。
然后就在github官网找到了它的各个版本：（如下图所示为各个发行版本）
![](http://upload-images.jianshu.io/upload_images/4749583-e6008fde95471ea2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中最底下三个貌似是有点问题的，比如release0_1和2是并口的文件，last-svn-state是无法运行的，因此从release1_0下手，（文件结构）

![](http://upload-images.jianshu.io/upload_images/4749583-ca79b1287178176c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*_init_.py*负责实现平台兼容，其实就是在导入这个包的时候根据平台来选择不同的处理类，我的是windows系统因此跑到*serialwin32.py*文件下去跟踪源码，第一行代码就是：

```import    win32file   #The base COM port and file IO functions.```

**win32file是个啥？包文件夹里面并没有这个文件，那么是在在哪导入的呢？**

在这之前还得知道pyd是啥文件：（[出处在这，点我](http://proupy.com/news/33)）
`
DLL文件即动态链接库文件，是一种可执行文件，它允许程序共享执行特殊任务所必需的代码和其他资源。打不开,不过可以使用反汇编;`

`PYD是一种PYTHON动态模块。实质上还是dll文件，只是改了后缀为PYD，pyd:
首先是我们最常见的.py文件。以.py扩展名的文件是源代码文件，由python.exe解释，可在控制台下运行。
当然也可以用文本编辑器进行修改。`

`.pyc文件：以.pyc为扩展名的是python的编译文件。.pyc文件是不能够用文本编辑器之类的进行编辑的，
但是同样它的优点在于.pyc文件的执行速度快于.py文件。
至于为什么要有.pyc文件，这个需求太明显了，因为py文件是可以直接看到源码的，如果你是开发商业软件的话，
不可能把源码也泄漏出去吧？所以就需要编译为pyc后，再发布出去。`

`.pyw文件。很多使用过.pyc文件的同学都知道，.pyc文件执行的时候桌面会出现黑糊糊的窗口，有的时候这是十分难看的。于是.pyw文件就应运而生了。.pyw文件与.pyc文件本质上没有什么区别，只是.pyw执行的时候不会出现黑窗口。.pyw 格式主要
是被设计来运行开发完成的纯图形界面程序的。 纯图形界面程序的用户不需要看到控制台窗口。值得一提的是，开发纯图形界面程序
的时候，你可以暂时把 .pyw 改成 .py ， 以便运行时能调出控制台窗口，看到所有错误信息，方便进行修改。`

`.pyo文件。pyo是优化编译后的程序。 python -O 源文件即可将源程序编译为pyo文件。同样.pyo文件也是不能用文本编辑器编辑的。`

`.pyd文件。.pyd文件并不是使用python编写而成，.pyd文件一般是其他语言编写的python扩展模块。（之前又在网上看到
过有关解释，.pyd文件是用D语言按照一定的格式编写，并处理成二进制文件。那么什么是**D语言**呢？？它是c/c++的综合进化版，不仅
具有二者的全部优点，而且整体性能更佳，但是其抽象程度高。）
扩展模块，一般用C或C++编写，其实可以说是一种更优秀的D语言编写的。`
`
**总的来说，pywin32就是在python跟windows系统API之间建立一个桥梁，中间件，可以在python下直接windows编程。而win32file是其中的一个组成部分，因此我们追根究底追到这基本上就可以打住了（知道提供哪些API就行了），因为再往下的话就是windows系统编程了。**
**假如硬是有人要缺根径，一定要追的话，那估计再往下得到系统驱动层的api，再往下到bsp层的api，再往下到汇编测api，因此其实最底下的思路还不是就这样---封装（例如ATC上层无论你整的多么复杂多么多的api，我再最底层就是一个单片机通过串口交互些数据，因此只要把单片机的功能划分好定义好数据结构及api，以后的高楼大厦都是基于此的，因此懂就好，要跳出来，不求甚解，抓住轮廓，当然前提是我从底层的硬件设计、驱动程序到上位的程序都写过才会有这个理解，假如那些一直写上位机代码的人估计会对底层的运行机制心虚吧~）**

[作者在此](http://blog.csdn.net/huiguixian/article/details/6968931)
*看到网上说的Pywin32可以像VC一样的形式来使用PYTHON开发win32应用，我就下载了个，但是不会使用，有基本的入门教程吗，或者谁给说说，比如说画界面什么的！*
-.`Python没有自带访问windows系统API的库的，需要下载第三方库。库的名称叫pywin32，可以从网上直接下载，下载链接：http://sourceforge.net/projects/pywin32/files%2Fpywin32/（下载适合的Python版本）`
-.`使用中如果出现ImportError: No module named win32api 或者出现 ImportError: No module named win32con，说明你的库没有安装好。`
-.`介绍**这个库里面最重要的两个模块：win32api和win32con*（也就是说在windows下安装win32all.exe之后会生成一个库，这个库里面就有win32api和win32con以及win32file，分别管不同的部分）***。`
-.`win32api顾名思义，就是用python对win32的本地api进行了封装；win32con个人理解为win32constant，即win32的常量定义。`

*这里是网上讨论API的一些言论：*
`
首先,API的意思是Application Program Interface,应用程序接口.
`
`
实际上,只要是程序,都可以对外提供API,比如你写一个网站.然后对外提供API,任何人都可以通过你提供的API获取到对应的信息.
`
`
例如你网站中的数据.win32 API是windows系统提供的API,.NET 也可以提供API虽然提供的作用可能会有重合,但是不影响说,其实这是两个不同程序提供的API.
不能因为说windows提供了API,那么.NET就不能提供API了.
而且.NET的API虽然很多是对WINDOWS的封装,但是这样可以避免一个人要学习.NET.还必须要去学习WINDOWS的API.
`
在读源码的时候可以看到使用了一个：win32file.OVERLAPPED()
[windows编程的API](http://blog.csdn.net/duanbeibei/article/details/49795425)**Overlapped I/O是win32的一项技术，你可以要求操作系统为你传送数据，并且在传送完毕时通知你。这项技术使你的程序在I/O进行中仍然能够继续处理事物。Overlapped I/O的基本形式是以ReadFile和WriteFile函数完成的。**

**WaitCommEvent(*handle**, overlapped***)
Waits for an event to occur for a specified communications device. The set of events that are monitored by this function is contained in the event mask associated with the device handle.

**Return Value**
The result is a tuple of (rc, mask_val), where rc is zero for success, or the result of calling GetLastError() otherwise.  The mask_val is the
new mask value once the function has returned, but if an Overlapped object is passed, this value will generally be meaningless.  See the
comments for more details.