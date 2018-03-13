---
title: 记一次从硬件到上层的socket问题定位分析
categories: python开源库分析
---

## 问题描述：
平板的串口连接超级网口(超级网口：理解为串口跟网口的映射)，python端通过socket库来读取数据，当我实例化两个socket client来读同一个超级网口的时候，其中一个Client概率性出现无法读取数据的现象。

**连接关系如下图所示：**平板将串口连到超级网口的串口端，超级网口在接收到串口数据之后立马通过TCP Server将数据发送给所有连接的客户端，我们当前的场景下是实例化了两个client，会出现其中某个串口偶尔读不到数据的现象。
![超级网口链接关系图](http://upload-images.jianshu.io/upload_images/4749583-629f7ca83687941a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**超级网口硬件连接图如下：**
![](http://upload-images.jianshu.io/upload_images/4749583-667e59795f9495e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**超级网口配置信息：**

![缓冲区配置信息](http://upload-images.jianshu.io/upload_images/4749583-a384cdf3f4d6880f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


>1) K2 在TCP Server 模式下也有KeepAlive 功能用于实时监测连接的完整。
>2) 通常用于局域网内与TCP 客户端的通信。适合于局域网内没有服务器并且有多台电脑或是手机向服务器请求数据的场景。同TCP Client 一样有连接和断开的区别，以保证数据的可靠交换。
>3) 本模式支持有人自主的同步波特率功能（RFC2217）功能
>4) 在TCP Server 模式下，K2 主动监听设置的本机端口，有连接请求时响应并创建连接，**当K2 的串口收到数据后，同时发送（也就是说是串口有数据就主动发送给连接的客户，）给所有与该K2 服务器建立连接的设备。**如果跨公网访问K2 的TCP Server，需要在路由器上做端口映射。
>5) K2 做TCP Server 的情况下，最多可以接受16 个Client 连接（连接数可自定义），本地端口号为固定值，不可设置为0。
>6) K2 做TCP Server，当连接Client 数量超过设定最大值时，默认新连接踢掉旧连接，可通过网页修改此功
能。
>7. TCP Server 模式下，连接Client 的数量可在1 到16 个之间任意设置，**---默认4 个---**，已连接Client的IP 可在内置网页状态界面显示，按连接计算发送/接收数据。
 >8. TCP Server 模式下，当连接数量达到最大值时，新连接是否踢掉旧连接可设置

## 原因：
由于我们的平台框架占用了一个client名额、虚拟串口占用一个名额、我的用例里面再占用一个名额，X区域（网页标签打开串口：websocket方式）占用了一个名额、因此四个名额（**就是上面说的默认四个，但是我实际上也将这个值修改为16后测试同样也是只有四个client是同步的**）被占用完了，我用例里面就**实时读取不到数据**，因为我的TCP Server允许接16个但并不代表我有同步发送16个client的能力，从实际效果（网页标签打开client方式）来看，仅有三个（虚拟串口占用一个）标签即client是同步收到数据的，其他client虽然偶尔能收到数据但是可以明显看到中间书有数据缺失的，因此要么改写框架的串口读写方式要么限制client数量。
即：由于超级网口厂商自身产品的缺陷导致的软件使用BUG，**结案**。

***一下是搜寻问题过程中的一些学习，留作记录。***

## 问题分解实时思路：
目前我还不知道socket的一些基础知识：

[在这可以看到基本使用方法](https://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001386832511628f1fe2c65534a46aa86b8e654b6d3567c000)

[在这可以看到更细参数及流程思想](https://gist.github.com/kevinkindom/108ffd675cb9253f8f71)

[在这可以看到详细且系统的TCP/UDP分析](https://www.gitbook.com/book/jerryc8080/understand-tcp-and-udp/details)

看完这三篇博客之后，有了个大概的认识，即我服务器端(超级网口内部实现)是接收到串口数据之后就把数据发送给当前所有连接的client，而我们client端只要不显示地关闭连接就不会断开连接的，~~因此基本可以确定是我在写代码的时候两个client的(IP,port)是一样的，所以一个client在后台一直轮训取数据，而我的程序里面新建同样的socket去读数据，正常情况下应该是报错才对~~(**这里我理解错了，socket中端口号存在的意义是实现不同电脑间进程之间的同步的，是一一对应的；每建立一个socket，client就会产生一个新的port编号，而我们的服务器端（超级网口）是同时最多发送给16个client的（根据当前连接的client来统一发送的），所以应该都会接收到数据才对**)，但是实际上却没报错，偶尔还能正常工作，因此我们去查看下**系统**的[源码](https://docs.python.org/3/library/socket.html)：
Socket Client的read实现方式如下：
```python
   def read(self, size=-1):
        # Use max, disallow tiny reads in a loop as they are very inefficient.
        # We never leave read() with any leftover data from a new recv() call
        # in our internal buffer.
        rbufsize = max(self._rbufsize, self.default_bufsize)
        # Our use of StringIO rather than lists of string objects returned by
        # recv() minimizes memory usage and fragmentation that occurs when
        # rbufsize is large compared to the typical return value of recv().
        buf = self._rbuf
        buf.seek(0, 2)  # seek end
        if size < 0:
            # Read until EOF
            self._rbuf = StringIO()  # reset _rbuf.  we consume it via buf.
            while True:
                try:
                    data = self._sock.recv(rbufsize)
                except error, e:
                    if e.args[0] == EINTR:
                        continue
                    raise
                if not data:
                    break
                buf.write(data)
            return buf.getvalue()
        else:
            # Read until size bytes or EOF seen, whichever comes first
            buf_len = buf.tell()
            if buf_len >= size:
                # Already have size bytes in our buffer?  Extract and return.
                buf.seek(0)
                rv = buf.read(size)
                self._rbuf = StringIO()
                self._rbuf.write(buf.read())
                return rv

            self._rbuf = StringIO()  # reset _rbuf.  we consume it via buf.
            while True:
                left = size - buf_len
                # recv() will malloc the amount of memory given as its
                # parameter even though it often returns much less data
                # than that.  The returned data string is short lived
                # as we copy it into a StringIO and free it.  This avoids
                # fragmentation issues on many platforms.
                try:
                    data = self._sock.recv(left)
                except error, e:
                    if e.args[0] == EINTR:
                        continue
                    raise
                if not data:
                    break
                n = len(data)
                if n == size and not buf_len:
                    # Shortcut.  Avoid buffer data copies when:
                    # - We have no data in our buffer.
                    # AND
                    # - Our call to recv returned exactly the
                    #   number of bytes we were asked to read.
                    return data
                if n == left:
                    buf.write(data)
                    del data  # explicit free
                    break
                assert n <= left, "recv(%d) returned %d bytes" % (left, n)
                buf.write(data)
                buf_len += n
                del data  # explicit free
                #assert buf_len == buf.tell()
            return buf.getvalue()
```
其中默认情况下的是把缓冲区所有数据都读出来：
```python
buf = self._rbuf
buf.seek(0, 2)  # seek end
if size < 0:
            # Read until EOF
            self._rbuf = StringIO()  # reset _rbuf.  we consume it via buf.
            while True:
                try:
                    data = self._sock.recv(rbufsize)
                except error, e:
                    if e.args[0] == EINTR:
                        continue
                    raise
                if not data:
                    break
                buf.write(data)
            return buf.getvalue()
```
self._rbuf在_init_()时候初始化成： `self._rbuf = StringIO()`

**因此首先得弄清楚StringIO()是什么：**可以粗略理解为就是一个存在于内存中的文件，操作她跟操作普通文件一样。

**StringIO：([可以参考这](http://blog.csdn.net/piaoyidage/article/details/42098911))**
>StringIO的行为与file对象非常像，但它不是磁盘上文件，而是一个内存里的“文件”。
我们可以像操作磁盘文件那样来操作StringIO。
就是生成一个StringIO对象，维护一个缓冲区，你可以像操作文件一样操作它。

**cStringIO：**
Python标准模块中还提供了一个cStringIO模块，它的行为与StringIO基本一致，但运行效率方面比StringIO更好。
```python
try:
    from cStringIO import StringIO
except ImportError:
    from StringIO import StringIO
```
***因为这个是python的一个内置库，所以我们可以去python官网看一下介绍：（[网址在这](https://docs.python.org/2/library/stringio.html)）***
**翻译：**这个模块以一种file-like的类来实现的，这个类读写字符串buffer或者叫**内存**文件。
` class StringIO.StringIO([buffer])`:
当一个[StringIO](https://docs.python.org/2/library/stringio.html#StringIO.StringIO)对象被创建的时候，能通过赋值一个字符串来初始化。如何没传字符串的话文件位置为0；
>**疑问：**这个内存文件是多长呢？还是说是可以变长的？那总得有个限制把？

现在我们回到上面：
当size<0,也就是默认情况下读，此时初始化的self._rbuf已经备份到buf了，我们将self._rbuf重新开一个**内存文件（StringIO()）**，相当于复位；然后开一个死循环，尝试去recv（rbufsize）数据，假如出现异常了分两种情况，一种是继续(操作被系统中断了，因此重新来)另一种是直接抛异常（其他异常不允许出现）
`其中EINTR源码中是这样定义的：EINTR = getattr(errno, 'EINTR', 4)，也就是说从errno文件中去看是否有EINTR属性，没有的话就返回默认值4，有则返回对应值。`
假如返回的数据是空，那么我直接跳出循环，把buf里面残留（假如上次我只读了1个字符，这次读全部，那么就有可能了~）的数据发出去就行了，假如有数据的话我将它拼接到buf中，然后一起发回去。

>那么什么是EINTR异常呢？简单来说就是：如果read（）读到数据为0，那么就表示文件读完了，如果在读的过程中遇到了中断则read()应该返回-1，同时**置errno为EINTR**。

>**慢系统调用(slow system call)：**此术语适用于那些可能永远阻塞的系统调用。永远阻塞的系统调用是指调用有可能永远无法返回，多数网络支持函数都属于这一类。如：若没有客户连接到服务器上，那么服务器的accept调用就没有返回的保证。

>**EINTR错误的产生：**当阻塞于某个慢系统调用的一个进程捕获某个信号且相应信号处理函数返回时，该系统调用可能返回一个EINTR错误。例如：在socket服务器端，设置了信号捕获机制，有子进程，当在父进程阻塞于慢系统调用时由父进程捕获到了一个有效信号时，内核会致使accept返回一个EINTR错误(被中断的系统调用)。当碰到EINTR错误的时候，可以采取有一些可以重启的系统调用要进行重启，而对于有一些系统调用是不能够重启的。例如：accept、read、write、select、和open之类的函数来说，是可以进行重启的。不过对于套接字编程中的connect函数我们是不能重启的，若connect函数返回一个EINTR错误的时候，我们不能再次调用它，否则将立即返回一个错误。针对connect不能重启的处理方法是，必须调用select来等待连接完成。


**总结：其实这里就是实现了一个上层协议，用来处理数据长度不可控时的buffersize定义问题。默认情况下我们先把上次可能没读完的（因为上次可能是先读一部分指定长度的数据呀，比如上次我就读了1个字符，那么极有可能这次我读的时候还在_rbuf中残留了一些数据，因此写代码的时候要考虑这个进去）数据先放到一个临时buf中，然后再直接recv（）去读一次数据，最多读rbufsize个数据，最少读0个，把数据读回来之后跟上次残留的一起拼接好然后返回给用户，这也就在read不指定长度的情况下把所有缓冲区的数据读出来的情况。而有指定长度的话基本原理还是差不多，只是检查一下长度再返回特定长度数据并做好数据维护。同时其他地方你也可以参考这个流程来做同样的上层协议。我认为官方的这个流程很是规范。**

**self._sock.recv(rbufsize)这个函数再啰嗦一句：它是马上返回的，假如是阻塞模式那么有个超时参数，非阻塞模式下假如一下没读到数据那么是会抛异常的(由于这个异常可控且已知所以可以选择性忽略掉的)，如下面的就是我们自己写的一个socket程序，仿照socket.read()写的，可以发现都是基于recv（）函数来的，然后再在其上做一些检测工作。**
```python
    def read(self, bufsize=2048, timeout=0, is_blocked=False, print_log=False):
        data = ''
        self.client.setblocking(is_blocked)
        if print_log:
            logger.info('++++++++read serail[%s] start++++++++' % str(self.address))
        if timeout:
            start_time = time.time()
            end_time = time.time() + timeout
            while time.time() <= end_time:
                try:
                    data = data + self.client.recv(bufsize)
                except socket.error, e:
                    if (str(e).find('10035') != -1 or str(e).find('11')) and is_blocked == False:
                        # if socket client is not blocked, then client recv nothing from server, it will raise 
                        # socker error, in windows, the error code is 10035, in linux, the error code is 11.
                        pass
                    else:
                        logger.error('dut read fail[%s]' % str(e))
                        traceback.print_exc()
                        self.connect()
                if time.time() > end_time:
                    break
        else:
            data = data + self.client.recv(bufsize)
        if print_log:
            if data:
                logger.debug(data)
            logger.info(
                '--------read serial[%s] finish--------' % str(self.address))
        self.client.setblocking(True)
        return data

```

stackoverflow上有关于recv（）的讨论，摘抄如下：
```python
socket.recv(*bufsize*[, *flags*])
Receive data from the socket. The return value is a string representing the data received.
 The maximum amount of data to be received at once is specifiedby *bufsize*. 
See the Unix manual page *recv(2)* for the meaning ofthe optional argument *flags*;
 it defaults to zero.
Note:
For best match with hardware and network realities, 
the value of *bufsize*should be a relatively small power of 2, for example, 4096.
```

+  *The bufsize param for the recv(bufsize) method is not optional. You'll get an error if you call recv() (without the param).*
+  *The bufferlen in recv(bufsize) is a maximum size. The recv will happily return fewer bytes if there are fewer available.*

这里可以知道两条信息：
+ **bufsize参数是必须的，不是可选的；**
+ **buffersize是一个最大值，如果没有足够的数据也是会及时返回哒~**
>**疑问：**那么当是阻塞模式的时候呢？

But now you have a new problem: how do you know when the sender has sent you a complete message? 
The answer is: you don't. You're going to have to make the length of the message an explicit part of your protocol. 
Here's the best way: prefix every message with a length, either as a fixed-size integer (converted to network byte order using socket.ntohs() or socket.ntohl() please!) or as a string followed by some delimiter (like '123:'). This second approach often less efficient, but it's easier in Python.

Once you've added that to** your protocol**, you need to change your code to handle recv() returning arbitrary amounts of data at any time. Here's an example of how to do this. I tried writing it as pseudo-code, or with comments to tell you what to do, but it wasn't very clear. So I've written it explicitly using the length prefix as a string of digits terminated by a colon. Here you go:
```python
length = None
buffer = ""
while True:
  data += self.request.recv()
  if not data:
    break
  buffer += data
  while True:
    if length is None:
      if ':' not in buffer:
        break
      # remove the length bytes from the front of buffer
      # leave any remaining bytes in the buffer!
      length_str, ignored, buffer = buffer.partition(':')
      length = int(length_str)

    if len(buffer) < length:
      break
    # split off the full message from the remaining bytes
    # leave any remaining bytes in the buffer!
    message = buffer[:length]
    buffer = buffer[length:]
    length = None
    # PROCESS MESSAGE HERE
```
上面的意思是，由于发过来的数据长度我们无法预估，因此需要我们上层自己定义协议（也就是方法）来处理数据的接收流程，上述代码就是一种处理方法。
```python
_socket.py:socket是导入了这个库的，这个库负责定义好了接口，并对接口进行描述，如下所示

    def recv(self, buffersize, flags=None): # real signature unknown; restored from __doc__
        """
        recv(buffersize[, flags]) -> data
        
        Receive up to buffersize bytes from the socket.  For the optional flags
        argument, see the Unix manual. 
        When no data is available, block until
        at least one byte is available or until the remote end is closed.  When
        the remote end is closed and all data is read, return the empty string.
        """
        pass
```
[关于socket阻塞问题](http://blog.csdn.net/huithe/article/details/5223785)：
**阻塞模式**下是缓冲区没数据就等，一有数据（尽管没有buffersize个数据）我就返回。
**非阻塞模式**下我来的时候缓冲区刚好有数据，拿了就返回，假如没有数据的话，我就报警了啊~不~我就报异常了噢~
```
作者：灵剑
链接：https://www.zhihu.com/question/51834325/answer/127694264
来源：知乎 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

socket分为阻塞和非阻塞两种，可以通过setsockopt，或者更简单的setblocking, settimeout设置。

阻塞式的socket的recv服从这样的规则：
当缓冲区内有数据时，立即返回所有的数据；当缓冲区内无数据时，阻塞直到缓冲区中有数据。

非阻塞式的socket的recv服从的规则则是：
当缓冲区内有数据时，立即返回所有的数据；当缓冲区内无数据时，产生EAGAIN的错误并返回（在Python中会抛出一个异常）。

两种情况都不会返回空字符串，返回空数据的结果是对方关闭了连接之后才会出现的。
由于TCP的socket是一个流，因此是不存在“读完了对方发送来的数据”这件事的。
你必须要每次读到数据之后，根据数据本身来判断当前需要等待的数据是否已经全部收到，来判断是否进行下一个recv。
可以看一下hiredis库的接口设计，hiredis中的Reader有两个接口，分别是feed和gets，feed每次送入一部分数据，不需要保证是正确分片的；
gets则返回已经得到的完整的结果，如果返回False，表示已经没有新的结果。
基本上所有的TCP的socket编程都是遵循这样的方法：读入新数据；判断有没有完整的新消息；处理新消息，或者等待更多数据。

题主：
一般的实现判断的方法有下面几种:
1.自定义协议的分界符，比如回车换行。
2.第一个字段给出长度，然后是数据,读的时候先拿到长度，然后读取那么多就好了
3.固定长度。
```

[socket网络编程](http://www.oschina.net/question/12_76126?sort=default&p=2#answers)：
这个博主讲到了socket编程的方方面面，讲了web通信的前后端知识以及基本实现代码，很好理解。

[关于socket.recv的精彩论述](https://www.zoulei.net/2016/06/17/socket_recv/)：
作者的一些理解，很有共鸣。