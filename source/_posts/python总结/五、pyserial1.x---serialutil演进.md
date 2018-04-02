---
title: 五、pyserial1.x---serialutil演进
categories: python总结
---
>相比较于1.0版本，1.1版本开始就使用了serialutil有了win32file为什么还需要这个呢？这得从它是啥说起。
#serialutil.py是什么？
[源码](http://www.ast.cam.ac.uk/~rgm/dazle/software/dics/src/serialutil.html#SerialBase)的描述字段这这样说的：
`Python Serial Port Extension for Win32, Linux, BSD, Jython`
`#see __init__.py#(C) 2001-2003 Chris Liechti <cliechti@gmx.net>`
`# this is distributed under a free software license, see license.txt`
就是python 串口的一个扩展包，而且作者就是pyserial的作者自己啊！整个srialutil.py里面主要就一个FileLike类。
![继承关系](http://upload-images.jianshu.io/upload_images/4749583-c6b5a9f0916e7180.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**上图的来源是[(这儿)](http://www.ast.cam.ac.uk/~rgm/dazle/software/dics/src/serialutil.html#SerialBase),是pyserial2.1版本时的文件，里面详细讲解了类成员。**

从图中可以看出serial.serialwin32.Serial类是继承自serial.serialwin32.SerialBase类的，而这个类又是继承自serial.serialutil.FileLike类，这基本是成型形态了！也就是趋于稳定的版本了（**确实也是pyserial2.1系列的版本了**）

我们要**体会作者的设计历程**的话，还是得从**最初版本开始**，我们知道是在pyserial1.1系列开始后才有的serialutil.py（在之前是直接win32file操作），于是我们从这个时期的代码开始研究，分为三个阶段：
- pyserial1.* 阶段
-  pyserial2.* 阶段
-  pyserial3.* 阶段

 ##pyserial1.* 阶段
我diff了一下1.11、1.13、1.16、1.17四个版本，基本没改动，增加了个串口异常类，且还未实现的，也就是搭了个“棚子”，在1.18版本时将flush实现给去掉了（让子类serial去实现），同时在readlines里面加了点字符串结束判断的逻辑，直到1.19、1.21都是如此。接下来我们看下最初版本的源码，并尝试去体会下作者是怎么想的：

**serialutil.y    @pyserial1.11**
```python
class FileLike:
    """An abstract file like class.
    
    This class implements readline and readlines based on read and
    writelines based on write.
    This class is used to provide the above functions for to Serial
    port objects.
    
    Note that when the serial port was opened with _NO_ timeout that
    readline blocks until it sees a newline (or the specified size is
    reached) and that readlines would never return and therefore
    refuses to work (it raises an exception in this case)!
    """

    def read(self, size): raise NotImplementedError
    def write(self, s): raise NotImplementedError

    def readline(self, size=None):
        """read a line which is terminated with '\\n' or until timeout"""

        line = ''
        while 1:
            c = self.read(1)
            if c:
                line += c   #not very efficient but lines are usually not that long
                if c == '\n':
                    break
                if size is not None and len(line) >= size:
                    break
            else:
                break
        return line

    def readlines(self, sizehint=None):
        """read alist of lines, until timeout
        sizehint is ignored"""
        if self.timeout is None:
            raise ValueError, "Serial port MUST have enabled timeout for this function!"
        lines = []
        while 1:
            line = self.readline()
            if line:
                lines.append(line)
                if line[-1] != '\n':
                    break
            else:
                break
        return lines

    def xreadlines(self, sizehint=None):
        """just call readlines - just here for compatibility"""
        return self.readlines()

    def writelines(self, sequence):
        for line in sequence:
            self.write(line)

    def flush(self):
        """map flush to flushOutput, ignore  exception when that method is not available"""
        try:
            self.flushOutput()
        except NameError:
            pass
```

创建一个“祖先”基类 `class FIleLike`，这是一个抽象的类似于文件的类，也就是**把串口抽象为一个文件的类**。

他说是基于read和write来实现readlines和writelines，但是在这里面又没实现这两个方法，而是直接定义好接口，然后直接抛出异常，就是说**作者的意图是：定义好接口，子类去实现它，基类会做重构检测，也就是子类继承后没有重构这两个函数的话，在调用时就会报错，也算是一种检测机制。**

而在父类里面实现writelines和readlines，我估计是这样的：**把串口抽象成一个文件之后，底层的read/write函数肯定是需要你去实现的，但是一旦基础搭好之后，readlines这样的上层建筑就可以直接抽象出来，因为这是通用的流程啊！**

**还可以这么辅助理解：我们写linux驱动的时候，不也是把字符串设备抽离出统一的read/write接口嘛，这部分是直接跟硬件打交道的，不同的硬件写法当然不同啊！而一旦这个桥梁搭建好之后，想利用这接口实现什么逻辑这部分跟硬件就解耦了啊，因此我们就可以在基类里面实现这部分逻辑啊。**

**再比如我们的通信系统也是如此的思路：我们通过基站这个统一的接口连入网络，然后在网络里面我们可以玩很多网络游戏的啊，这些游戏是基于基站的，也就是说基站一垮，一切玩完，同时这些游戏在抽离出基站接口之后的逻辑是可以完全独立设计的。但作者为什么要把“基站接口”留给子类去实现呢？因为我接入网络的方式不只一种啊，还可以是wifi还可以是以太网啊，所以这个底层的read/write实现就不一样了呀，同时上层是不用改的啊！**

>***总结：我们设计抽象的基类时候，跟具体实现相关的我们直接定义出接口顺便做点边界检测就好，而脱离出具体场景的逻辑部分就可以直接实现了。***

```python
def readline(self, size=None):
        """read a line which is terminated with '\\n' or until timeout"""

        line = ''
        while 1:
            c = self.read(1)
            if c:
                line += c   #not very efficient but lines are usually not that long
                if c == '\n':
                    break
                if size is not None and len(line) >= size:
                    break
            else:
                break
        return line
```
从上面的代码可以看出，是无限轮训读完的数据，每次读一个字符，同时对读出的字符做存在判断（这里就限制了self.read需要返回什么样的数据，就是所谓的**“祖师爷定规矩”**），如果self.read返回True，我首先把字符串拴起来，接着判断是换行字符就跳出循环，字符数够了也跳出循环，假如没有获取到字符，跳出循环。

这里是不是可以这么理解：self.read(1)你**必须**去获取一个字符回来，只要返回的是None说明就是异常了，其中的timeout什么的，你就自己去搞定了！**我只看结果！！！！！！！！！！看到没有这就是典型企业中层领导者的做事风格啊！代码处处是哲学啊！于代码中体悟人类社会的框架！**

其次作者自己也吐了个槽：` #not very efficient but lines are usually not that long` 这种一次读一个字符的方式不是很有效率，但是通常字符串的长度也不是太长。**也就是说就当前的需求来看，能用就行了，不要一开始就想搞个大事情！饭要一口一口吃，路还得一步一步走！**

>***总结：逻辑很简单，我一次次读就行了，读到换行就结束并返回字符串，跟写单片机代码是一样一样的！***

\
\
**readlines()思路是：这里对timeout作了要求，但是实际上却仅仅做了个存在与否的断言，读取部分还是调用二级接口readline来实现，控制逻辑跟readline调用read是一样一样的啊！**
```python
 def readlines(self, sizehint=None):
        """read alist of lines, until timeout
        sizehint is ignored"""
        if self.timeout is None:
            raise ValueError, "Serial port MUST have enabled timeout for this function!"
        lines = []
        while 1:
            line = self.readline()
            if line:
                lines.append(line)
                if line[-1] != '\n':

                    break
            else:
                break
        return lines
```

\
**我们再来看下1.x系列最后一个版本跟最初版本之间有什么区别：**
```python
def readline(self, size=None, eol='\n'):
        """read a line which is terminated with end-of-line (eol) character
        ('\n' by default) or until timeout"""
        line = ''
        while 1:
            c = self.read(1)
            if c:
                line += c   #not very efficient but lines are usually not that long

                if c == eol:
                    break
                if size is not None and len(line) >= size:
                    break
            else:
                break
        return line
```
对比上面的readline可知，**这里将字符串结束的标志设计为可配置的，不再单一地限定为换行符了！(估计是某一个具体的实际场景中不再是以换行符为结束标志了)**其它部分什么都没变。

**同理，readlines做相应的重构。**
```python
 def readlines(self, sizehint=None, eol='\n'):
        """read a list of lines, until timeout
        sizehint is ignored"""
        if self.timeout is None:
            raise ValueError, "Serial port MUST have enabled timeout for this function!"
        lines = []
        while 1:
            line = self.readline(eol=eol)
            if line:
                lines.append(line)

                if line[-1] != eol:    #was the line received with a timeout?
                    break
            else:
                break
        return lines

```

**writelines则相当简陋！如下，直接调用接口就行，连写成功与否的判断都没有: "我反正写了，成不成不知道！"**Σ( ° △ °|||)︴ **且在1.x系列里面这都没改，看看2.x系列里面是否有改观。**
```python
def writelines(self, sequence):
        for line in sequence:
            self.write(line)
```
>1.x系列最后一个版本1.21里面把flush()给去掉了！也就是这部分也撒手不管了！为啥？