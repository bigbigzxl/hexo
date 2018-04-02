---
title: 三、《Python Programming on Win32》关于Win32file摘要及思考
categories: python总结
---
# Win32file Module概述
**Native File Manipulation。**
There are times when the standard Python file objects can't meet your requirements, and you need to use the Windows API to manipulate files.
当通用的文件操作方法(就是python里面的open、read、write)无法满足特殊需求（安全性啊啥的~）时，这个时候windows操作文件的API就是时候发挥效果了！

This can happen in a number of situations, such as:
`• You need to read or write data to or from a Windows pipe.`
你需要从一个windows管道读出或者写入数据时。
`• You need to set custom Windows security on a file you are creating.`
需要设定特殊文件的安全性时。
`• You need to perform advanced techniques for performance reasons, such as "Overlapped" operations or using completion ports.`
Completion ports利用一些线程，帮助平衡由I/O请求所引起的负载
OVERLAPPED I/O（[异步I/O](http://blog.csdn.net/xiaoding133/article/details/7775139)）是一个包含了用于异步输入输出的信息的结构体
**总之：就是你要提高文件操作性能的时候用这些。**
```
win32file.WriteFile() 
returns the error code from the operation. 
This is either zero or win32error.ERROR_IO_PENDING if overlapped I/O is used. 
All other error codes are converted to a Python exception.
```
```
The win32file.CreateFile() function opens or creates standard files, 
returning a handle to the file. 
Standard files come in many flavors, including synchronous files 
(where read or write operations don't return until the 
operation has completed); asynchronous (or overlapped I/O) files, 
where read and write operations return immediately; 
and temporary files that are automatically deleted when the handle is closed. 
Files may also be opened requesting that
 Windows not cache any file operations, that no buffering is performed, etc.

This function returns a PyHANDLE object. 
PyHANDLEs are simply objects that wrap standard Win32 HANDLEs. 
When a PyHANDLE object goes out of scope, it's automatically closed;
 thus, it's generally not necessary to close
 these HANDLEs as it is necessary when using these from C or C++.
```
```
Overlapped I/O
Windows provides a number of techniques for high-performance file I/O. 
The most common is overlapped I/O. Using overlapped I/O, 
the win32file.ReadFile() and win32file.WriteFile() operations are asynchronous 
and return before the actual I/O operation has completed. 
When the I/O operation finally completes, a Windows event is signaled.

Overlapped I/O does have some requirements normal I/O operations don't:
• The operating system doesn't automatically advance the file pointer.
 When not using overlapped I/O, a ReadFile or WriteFile operation automatically
 advances the file pointer, so the next operation automatically reads the subsequent
 data in the file. When usingoverlapped I/O, you must manage the location
 in the file manually.
• The standard technique of returning a Python string object from
 win32file. ReadFile() doesn't work. Because the I/O operation has not 
completed when the call returns, a Python string can't be used.
As you can imagine, the code for performing overlapped I/O is more complex than 
when performing synchronous I/O. Chapter 18, WindowsNT Services, 
contains some sample code that uses basic 
overlapped I/O on a Windows-named pipe.
```



#Win32file Module解析
`The win32file module contains functions that interface to the File and other I/O-related Win32 API 
functions.`

**CreateFile(FileName,········)**
Opens or creates a file or a number of other objects and returns a handle that can access the object.
`FileName`
The name of the file, pipe, or ***other resource*** to open.
`Result`
A PyHANDLE object to the file.

----
**ReadFile()**
Reads data from an open file.
`errCode, data = ReadFile(FileHandle, Size, Overlapped)`
*FileHandle*
`The file handle identifying the file to read from. This handle typically is obtained from win32file.CreateFile().`
*Size or ReadBuffer*
`If Size is specified as a Python integer, it's the number of bytes to read from the file. When using overlapped I/O, a ReadBuffer shouldbe used, indicating where the ReadFile operation should place the data.`
这句话的意思是：将如你放的是一个整型数据，那么readfile（）啊，你得从这个文件里读这么多的数据回来，但是一旦是异步文件读取的话，你必须指定一个buffer，你想啊：
- 正常的同步文件读取流程是这样的，你给我把数据挖回来，我等着，然后底层I/O操作就吭哧吭哧把数据给挖回来了（同时你一直在矿井口即`FileHandle`这等着），然后把数据放到一个叫做data的“箩筐”内~交接完毕！
- 但是异步读取是这么一个流程：一天一个商人找到一个包工头（应用程序），说：啊，小刘啊（readfile()）啊，你给我去`FileHandle`这个矿井挖一箩筐矿回来！接收到指令后，小刘（readfile()）就去找到挖矿工铁柱（系统底层驱动），说：铁柱啊，你给我去`FileHandle`这个矿井挖一筐矿上来，然后就去干其他事了(走的时候给了铁柱一部手机（overlapped），说“整完了给我电话，我来收”)，但是这其中有个问题啊，铁柱整好的矿往哪放啊？同步模式下，铁柱冒出矿井就可以看到刘工头在井口等着，直接往框内一放，就交接完事了，但是这个不一样啊，铁柱只有在出矿井后才有信号，这个时候打电话给刘工头，然后就是干等着，指不定工头啥时候来呢！说不定等的这段时间都可以再挖一卡车矿了呢！！说不定等着等着铁柱都快饿挂了！而且可以肯定的是这个时间内其他包工头不能给铁柱派活了！！总之就是浪费了大量的资源；因此我们就想啊，要是我们在叫铁柱下去挖矿的时候顺便告诉它你挖完后直接放到我们指定的地点，然后通知我们，我们就会来取的（取到data这个“箩筐”内），放到指定地点后你想干嘛干嘛去，大大提高你的效率！毕竟整个矿场就靠着（**基于**）你们这群底层的干活人员来**运转**呢！ **科科~**
>***感悟：这其实就是一个现代商业社会微剧场的缩影，在实际场景中人们总是很容易的想到优化的方式，并且留心生活的极客们早就把生活写入了代码，优化了代码！因此我们平时留心注意生活，学习生活中的智慧、优化思维然后再拓展发散到各行各业各方个面，余以为这是一种可行的策略。***

*Overlapped*
`An OVERLAPPED object or None if overlapped I/O isn't being performed. This parameter can be omitted, in which case None is used.`
*Result*
`errCode:Either 0, or ERROR_IO_PENDING.`
`Data:The data read from the file.`
这里的data就是上面描述的“箩筐”。异常代码这里是两种要么0要么就是异步I/O挂起提示，0表示数据获取成功，I/O挂起是异步文件读取的时候，提前返回时的返回码。
>这里的***中层哲学***是：不要想管太多（比如，我会很“贪心”地想返回值可以返回异常状态，也许是有的，因为毕竟只描述了0是成功，其他字段并没有描述，这部分得看源码），这只care成功，其他的除了异步态下的 ERROR_IO_PENDING就都是异常了，你开发商只要知道0代表成功了，其他的（除去ERROR_IO_PENDING）都是异常就好了，具体的，你精力旺盛想知道是什么异常也可以去查的，反正我作为一个包工头对上（开发商）是提供**高效率的接口** 和 **高精度的搜查（可选）**，这想必就是严谨的**做事态度**吧！！！对下（铁柱），合理安排策略使得底层利益最大化，实现**共赢**。
----
**WriteFile()**
Writes data to an open file.
`errCode = WriteFile(FileHandle, Data, Overlapped)`
*FileHandle*
`The file handle identifying the file to read from. This handle typically is obtained from win32file.CreateFile().`
*Data*
`The data to write to the file, as a Python string.`

*Overlapped*
`An OVERLAPPED object or None if overlapped I/O isn't being performed. This parameter can be omitted, in which case None is used.`
*Result*
`errCode: Either 0 or ERROR_IO_PENDING.`
----