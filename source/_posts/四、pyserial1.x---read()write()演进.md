---
title: 四、pyserial1.x---read()/write()演进
categories: python开源库分析
---

![](https://www.etlhive.com/wp-content/uploads/2016/10/19-PYTHON-The-Most-Popular-Coding-Language.jpg)

**摘要**：基于最开始的pyserial1.0讲解read/write基本原理，之后就三个阶段（pyserial1.x，pyserial2.x，pyserial3.x）讲解实现方式的演进，本篇围绕pyserial1.0讲解原理并尝试分析pyserial1.x版本期间作者所做的改进措施。


# write（）实现机理

**放在最开始:** 发现一本书，讲python在win32下编程的，叫[《Python Programming on Win32》](https://books.google.com.hk/books?id=fzUCGtyg0MMC&printsec=frontcover&hl=zh-TW#v=onepage&q&f=false)

通过*notepad++*里面的*compare*插件进行比较各个版本（目前看了**1.0、1.1、1.11、1.13**版本），发现在各版本中*write（）*函数基本上就没变。

代码如下，接下来我们**按行来解析代码**：
```python
    def write(self, s):
    "write string to serial port"
	
    if not self.hComPort: raise portNotOpenError
    #print repr(s),
	
    overlapped = win32file.OVERLAPPED()
    overlapped.hEvent = win32event.CreateEvent(None, 0, 0, None)
	
    win32file.WriteFile(self.hComPort, s, overlapped)
    # Wait for the write to complete.
    win32event.WaitForSingleObject(overlapped.hEvent, win32event.INFINITE)
    #old: win32file.WriteFile(self.hComPort, s) #, 1,  NULL)
```
*self.hcomPort*是来自**serialwin32.Serial类初始化**中（注意1.0版本是没有继承**serialutil.FileLike类**的）：
```python
 self.hComPort = win32file.CreateFile(self.portstr,# self.portstr = 'COM%d' % (port+1) 
               win32con.GENERIC_READ | win32con.GENERIC_WRITE,
               0, # exclusive access
               None, # no security
               win32con.OPEN_EXISTING,
               win32con.FILE_ATTRIBUTE_NORMAL | win32con.FILE_FLAG_OVERLAPPED,None)

#这里有个疑问：这个函数是怎么将串口映射为一个文件的？
#FileName：The name of the file, pipe, or “other resource” to open.
#这里暂且推断是通过filename来分发、多态实现的。
```
好了，那么*win32file.CreateFile()*这个函数又是**来自哪儿**呢？

这么想吧：我们不是最终要控制串口硬件嘛，但是呐，python是一个寄居在windows操作系统上的一个软体而已，而我们的串口是一个独立的硬件载体，就是天仙跟牛郎的关系嘛，两者之间相互独立，硬是要通信的话，怎么办?孔雀搭桥嘛！windows系统在其中就承担了搭桥的角色，作为一个中间件，信息传递层。

好了，我们知道串口硬件在windows下有对应的底层驱动程序（linux同样如此），因此windows直接系统编程可以很方便的控制串口，但是我们的python这仙女呢，是**基于**操作系统用c语言写的一个抽象的软体，没有自己的底层基础，即“飞的起来”这个意识形态上层建筑全靠操作系统一整套的底层基础来支撑；

`这就好比你在别人的地盘上谋生，同时你又需要利用一些别人的“矿产资源”，你不通过他们的官方渠道来获取，而是选择自己建立一全套采矿产业链，这行的通吗？**首先**你精力、资源、时间够建立这套产业链吗？**其次**人家会允许你在他家门口抽取资源野蛮发展吗？`

因此，于效率于规范我们都得通过windows的一些统一API接口来控制串口硬件，**那这些统一API怎么调用呢？**

他山之石可以攻玉，我们可以借鉴一下windows自己是怎么做的！比如windows下的.net框架是将底层系统API封装成抽象程度更高的API，于是我们编写C#代码的时候用起来就很简单了！捋一下：C#是“仙女1号”，python是“仙女7号”，仙女1号要控制硬件可以直接去调用系统级API，但是这样太繁琐了，会导致很大的代码量，就跟你用汇编写单片机代码一样，于是.net将通用的整理成一个标准框架，以供天下豪杰使用！bingo~那我们也可以给仙女7号也整个中间层框架啊！好的，这部分工作已经有人给做了！

**就是pywin32**（[链接在这](https://pypi.python.org/pypi/pywin32)）：官方描述就是`Python extensions for Windows`，即对windows的一个python拓展包，已收录在python的官方扩展库里面了。
下载下来是一个exe文件，安装后python就可以直接调用它里面的库文件里面的API来进行windows系统编程了！嗯，我们理解到这一层就行了！再往下挖的话就到了windows编程、驱动的范畴了，我们关注于API的使用即可，点到即止。

好了，回到之前的问题，*win32file.CreateFile()*里面的win32file就是pywin32安装后的其中一个库文件里的API，[(官网docs文件在这)](http://docs.activestate.com/activepython/2.5/pywin32/win32file__CreateFile_meth.html)。

## 1.1 win32file.CreateFile()
```python
win32file.CreateFile
PyHANDLE = CreateFile(fileName,desiredAccess,shareMode,attributes,CreationDisposition,flagsAndAttributes,hTemplateFile)
//Creates or opens the a file or other object and returns a handle that can be used to access the object.
```
从代码可知是创建或者打开一个文件或者对象，并返回一个指针。

**这是实际调用：**
```python
 self.hComPort = win32file.CreateFile(self.portstr,# self.portstr = 'COM%d' % (port+1) 
               win32con.GENERIC_READ | win32con.GENERIC_WRITE,
               0, # exclusive access
               None, # no security
               win32con.OPEN_EXISTING,
               win32con.FILE_ATTRIBUTE_NORMAL | win32con.FILE_FLAG_OVERLAPPED,None)
```
**这是API参数描述：**
```python
fileName : PyUnicode #The name of the file
desiredAccess : int #access (read-write) mode Specifies the type of access to the object. An application can obtain read access, write access, read-write access, or device query access. This parameter can be any combination of the following values.
shareMode : int #Set of bit flags that specifies how the object can be shared. If dwShareMode is 0, the object cannot be shared. Subsequent open operations on the object will fail, until the handle is closed. To share the object, use a combination of one or more of the following values:
attributes : PySECURITY_ATTRIBUTES #The security attributes, or None
CreationDisposition : int   # Specifies which action to take on files that exist, and which action to take when files do not exist. For more information about this parameter, see the Remarks section. This parameter must be one of the following values:
flagsAndAttributes : int  #file attributes
hTemplateFile: PyHANDLE  #Specifies a handle with GENERIC_READ access to a template file. The template file supplies file attributes and extended attributes for the file being created. Under Win95, this must be 0, else an exception will be raised.
```
```
The following objects can be opened:
files
pipes
mailslots
communications resources
disk devices (Windows NT only)
consoles\directories (open only)
*问题：串口属于哪个类型？*
```
也即：

`self.portstr = 'COM%d' % (port+1) `:文件名就是串口号
`win32con.GENERIC_READ | win32con.GENERIC_WRITE`:打开串口对象的读写权限（win32con里面是全部的常量）
`  0, # exclusive access`:关闭共享模式，假如对串口进行连续打开操作将会fail
`None, # no security`:如描述，非安全模式。
` win32con.OPEN_EXISTING,`当文件在或者不存在的时候进行的操作，文件不存在时，打开失败。
ps.在他们的讨论区说了，为什么在对设备使用createfile（）函数时要使用OPEN_EXISTING标志位。

` win32con.FILE_ATTRIBUTE_NORMAL | win32con.FILE_FLAG_OVERLAPPED`：创建的文件属性，
`None`不使用属性模版

## 1.2 win32file.OVERLAPPED()
关于`win32con.FILE_FLAG_OVERLAPPED`上一篇已有相关探寻：
`win32file.OVERLAPPED()：Overlapped I/O是win32的一项技术，你可以要求操作系统为你传送数据，并且在传送完毕时通知你。这项技术使你的程序在I/O进行中仍然能够继续处理事物。Overlapped I/O的基本形式是以ReadFile和WriteFile函数完成的。`

**小结：异步文件处理机制，用来提高并发及其他性能的。**


## 1.3 win32event.CreateEvent()
`win32event.CreateEvent(None, 0, 0, None)`:
Creates or opens a named or unnamed event object.
The **result** is a PyHANDLE object referencing the requested object.
**小结：也就是创建一个事件，这个时间可以跟对象挂钩。**


至此基本可以完整分析作者写write函数的思路了：
```python
 def write(self, s):
    "write string to serial port"
    if not self.hComPort: raise portNotOpenError
    #print repr(s),
    overlapped = win32file.OVERLAPPED()
    overlapped.hEvent = win32event.CreateEvent(None, 0, 0, None)
    win32file.WriteFile(self.hComPort, s, overlapped)
    # Wait for the write to complete.
    win32event.WaitForSingleObject(overlapped.hEvent, win32event.INFINITE)
    #old: win32file.WriteFile(self.hComPort, s) #, 1,  NULL)
```
***首先检测self.hComPort（文件指针）是否存在，假如不存在说明文件创建失败了，就raise一个异常；
然后为了提高文件操作性能（毕竟是底层I/O处理啊！），这里开一个异步I/O: overlapped；
创建一个不带名字的事件，并将其注册给异步I/O的的事件部分（应该是windows常规代码流程）；
然后就以overlapped方式将s字符串写到self.hComPort指向的文件中去，并立马返回（异步处理的核心）。
然后事件处理器将等待overlapped.hEvent对象被触发，并且是无限等,也就是死等（有误，见下面分析）到文件被写完。***

**这里一个问题：假如我在程序里死等，那么跟同步就没区别了啊！？**
```python
WaitForSingleObject()
As the name implies, this function allows you to wait for a single object to become signaled.
 It takes two parameters: the handle to the object you wish to wait for, and a timeout in milliseconds
 (or win32event.INFINITE for no timeout). 
The return value from the function is win32event.WAIT_OBJECT_0 if the object becomes signaled, 
win32event.WAIT_TIMEOUT if the timeout interval expired, or win32event.WAIT_ABANDONED in certain situations
 for mutexes (see the Win32 documentation).
```
**这里也并没有讲清楚！**
还好在微软的官网[(网址在这)](https://msdn.microsoft.com/en-us/library/windows/desktop/ms687032(v=vs.85).aspx)找到了下面的描述：
注意the calling thread，既然是线程那么进入等待状态就是休眠挂起咯~这就是真的异步了啊！让出了CPU！
```python
Remarks
The WaitForSingleObject function checks the current state of the specified object.
 If the object's state is nonsignaled, |||the calling thread |||enters the wait state until the object is signaled or
 the time-out interval elapses.

The function modifies the state of some types of synchronization objects. 
Modification occurs only for the object whose signaled state caused the function to return. 
For example, the count of a semaphore object is decreased by one.

The WaitForSingleObject function can wait for the following objects:
    Change notification
    Console input
    Event
    Memory resource notification
    Mutex
    Process
    Semaphore
    Thread
    Waitable timer

```
# read() 实现原理剖析
[read() 官方sample](https://msdn.microsoft.com/en-us/library/windows/desktop/bb540534(v=vs.85).aspx)
```python
    def read(self, size=1):
        "read num bytes from serial port"
        if not self.hComPort: raise portNotOpenError
        read = ''
        if size > 0:
            while len(read) < size:
                flags, comstat = win32file.ClearCommError( self.hComPort )
                rc, buf = win32file.ReadFile(self.hComPort, size-len(read), self.overlapped)
                if self.timeout:
                    rc = win32event.WaitForSingleObject(self.overlapped.hEvent, self.timeout*1000)
                    if rc == win32event.WAIT_TIMEOUT: break
                    #TODO: if reading more than 1 byte data can be lost when a timeout occours!!!!
                else:
                    win32event.WaitForSingleObject(self.overlapped.hEvent, win32event.INFINITE)
                read = read + str(buf)
        #print "read %r" % str(read)
        return str(read)
```

**win32file.ClearCommError( self.hComPort ):**

```Retrieves information about a communications error and reports the current status of a communications device. The function is called when a communications error occurs, and it clears the device's error flag to enable additional input and output (I/O) operations.```

总结上面这段话的意思就是，你给我个设备，我来检索设备的通信异常并且报告设备的当前状态。在检索到异常的时候这个函数就会被调用，用来清楚异常标志位好使得“不堵车”，也就是说这个函数就是一名“交警”。


**注意：这两个是在初始化的时候就创建了的，也就是说read是直接使用它的！而在write里面使用的本地变量，重新生成了事件，为何这般处理？也许后续代码跟进的时候会有不同理解吧！留后解决。** `（在pyserial1.6里面就把read函数里面的self.overlapped，直接本地化了，初始化里面定义那两行去掉了，也即这是当时设计的一个小缺陷@2017.8.8）`
```python
__init__():
self.overlapped = win32file.OVERLAPPED()
self.overlapped.hEvent = win32event.CreateEvent(None, 0, 0, None)

write():
overlapped = win32file.OVERLAPPED()
overlapped.hEvent = win32event.CreateEvent(None, 0, 0, None)
```
`今天先写到这，后续的慢慢跟进代码继续分析。2017.08.07@AW`

**win32file.ReadFile(self.hComPort, size-len(read), self.overlapped)[ (详细链接在这)](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365467(v=vs.85).aspx)**
```python
BOOL WINAPI ReadFile(
  _In_        HANDLE       hFile,
  _Out_       LPVOID       lpBuffer,
  _In_        DWORD        nNumberOfBytesToRead,
  _Out_opt_   LPDWORD      lpNumberOfBytesRead,
  _Inout_opt_ LPOVERLAPPED lpOverlapped
);
```
**hFile [in]**
`A handle to the device (for example, a file, file stream, physical disk, volume, console buffer, tape drive, socket, communications resource, mailslot, or pipe).`

**lpBuffer [out]**
`A pointer to the buffer that receives the data read from a file or device.
    This buffer must remain valid for the duration of the read operation. The caller must not use this buffer until the read operation is completed.`


**nNumberOfBytesToRead [in]**
    `The maximum number of bytes to be read.`

**lpNumberOfBytesRead [out, optional]**
   ` A pointer to the variable that receives the number of bytes read when using a synchronous hFile parameter. ReadFile sets this value to zero before doing any work or error checking. Use NULL for this parameter if this is an asynchronous operation to avoid potentially erroneous results.
    This parameter can be NULL only when the lpOverlapped parameter is not NULL.
    For more information, see the Remarks section.`

**lpOverlapped [in, out, optional]**
`A pointer to an *OVERLAPPED* structure is required if the *hFile* parameter was opened with *FILE_FLAG_OVERLAPPED*, otherwise it can be *NULL*.`

**Return value**
`If the function succeeds, the return value is nonzero (**TRUE**).
If the function fails, or is completing asynchronously, the return value is zero (**FALSE**). To get extended error information, call the **GetLastError** function.
**Note**  The **GetLastError** code **ERROR_IO_PENDING** is not a failure; it designates the read operation is pending completion asynchronously. For more information, see Remarks.`

 **remark**
`When reading from a communications device, the behavior of *ReadFile is* determined by the current communication time-out as set and retrieved by using the *SetCommTimeouts* and *GetCommTimeouts* functions. Unpredictable results can occur if you fail to set the time-out values. For more information about communication time-outs, see *COMMTIMEOUTS*.`


**Considerations for working with asynchronous file handles:**
`**ReadFile** may return before the read operation is complete. In this scenario, **ReadFile** returns **FALSE** and the **GetLastError** function returns **ERROR_IO_PENDING**, which allows the calling process to continue while the system completes the read operation.
The *lpOverlapped* parameter must not be **NULL** and should be used with the following facts in mind:Although the event specified in the **OVERLAPPED** structure is set and reset automatically by the system, the offset that is specified in the **OVERLAPPED** structure is not automatically updated.
**ReadFile** resets the event to a nonsignaled state when it begins the I/O operation.
The event specified in the **OVERLAPPED** structure is set to a signaled state when the read operation is complete; until that time, the read operation is considered pending.
Because the read operation starts at the offset that is specified in the **OVERLAPPED** structure, and **ReadFile** may return before the system-level read operation is complete (read pending), neither the offset nor any other part of the structure should be modified, freed, or reused by the application until the event is signaled (that is, the read completes).
If end-of-file (EOF) is detected during asynchronous operations, the call to **GetOverlappedResult** for that operation returns **FALSE** and **GetLastError** returns **ERROR_HANDLE_EOF**.`
**上小段总结**就是：异步文件读写的时候，I/O操作可以挂起等待，主进程能够继续运行，当完成的时候就会触发事件来处理（触发事件、文件读写的相关操作都定义在OVERLAPPED这个结构体里面了）


## pyserial1.1    @read（）
```python
 def read(self, size=1):
        "read num bytes from serial port"
        if not self.hComPort: raise portNotOpenError
        #print "read %d" %size           ####debug
        read = ''
        if size > 0:
            while len(read) < size:
                flags, comstat = win32file.ClearCommError( self.hComPort )
                rc, buf = win32file.ReadFile(self.hComPort, size-len(read), self.overlapped)
                if self.timeout:
                    rc = win32event.WaitForSingleObject(self.overlapped.hEvent, self.timeout*1000)
                    if rc == win32event.WAIT_TIMEOUT: break
                    #TODO: if reading more than 1 byte data can be lost when a timeout occours!!!!
                else:
                    win32event.WaitForSingleObject(self.overlapped.hEvent, win32event.INFINITE)
                read = read + str(buf)
        #print "read %r" % str(read)
        return str(read)
```
## pyserial1.21    @read（）
```python
def read(self, size=1):
        """read num bytes from serial port"""
        if not self.hComPort: raise portNotOpenError
        if size > 0:
            win32event.ResetEvent(self._overlappedRead.hEvent)
            flags, comstat = win32file.ClearCommError(self.hComPort)
            if self.timeout == 0:
                n = min(comstat.cbInQue, size)
                if n > 0:
                    rc, buf = win32file.ReadFile(self.hComPort, win32file.AllocateReadBuffer(n), self._overlappedRead)
                    win32event.WaitForSingleObject(self._overlappedRead.hEvent, win32event.INFINITE)
                    read = str(buf)
                else:
                    read = ''
            else:
                rc, buf = win32file.ReadFile(self.hComPort, win32file.AllocateReadBuffer(size), self._overlappedRead)
                n = win32file.GetOverlappedResult(self.hComPort, self._overlappedRead, 1)
                read = str(buf[:n])
        else:
            read = ''
        return read
```
总结起来是：最开始是采用每次读一个字符，轮训读来获取整串字符，典型的单片机式写法，1.21版本就主要围绕参数的检测开始写，首先保证你输入的参数是对的，以输入参数为中心来划分我函数内部的逻辑块，尤其注意:
**pyserial1.1   :**  `win32file.ReadFile(self.hComPort, size-len(read), self.overlapped)`
**pyserial1.21:** 
	`win32file.ReadFile(self.hComPort,`
	` win32file.AllocateReadBuffer(n), `
	`self._overlappedRead)`

不同之处在于win32file.AllocateReadBuffer(n)：这是python下的一个标准化的分配缓冲的接口,在同步读文件的时候直接传一个int值就好了，但是异步读取模式就必须得用这个分配一块临时内存，具体原因请见我写的一篇解析感悟：[《Python Programming on Win32》中关于文件部分摘要及思考](http://www.jianshu.com/p/9468803ae8b8)。

```python
PyOVERLAPPEDReadBuffer = AllocateReadBuffer(bufSize)
Allocates a buffer which can be used with an overlapped Read operation using win32file::ReadFile
Parameters:
       bufSize : int
      The size of the buffer to allocate.
```

`PyOVERLAPPEDReadBuffer Object`
 ` An alias for a standard Python buffer object. Previous versions of the Windows extensions had a custom object for holding a read buffer. This has been replaced with the standard Python buffer object.Python does not provide a method for creating a read-write buffer of arbitary size, so currently this can only be created by win32file::AllocateReadBuffer.`

网上讨论当频繁调用.AllocateReadBuffer时会不会出现内存泄漏的情况：由于python的内存管理机制问题，当你的内存引用计数器为零时，内存会被自动回收，所以不存在内存泄漏问题。

```
Spencer Ernest Doidge wrote:
> If I call win32file.AllocateReadBuffer(..) in a method that gets called
> repeatedly and terminates each time it is done, will I have a memory leak
> if I don't somehow free the allocated memory before concluding the run of
> the method? Or does the memory get de-allocated automagically when the
> method is finished running?

Normal Python reference counting semantics apply - ie, when there are no 
references to the buffer it will be deleted.  Thus, there should be no 
leaks.
Mark.
```