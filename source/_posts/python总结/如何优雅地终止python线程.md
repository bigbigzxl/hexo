---
title: 如何优雅地终止python线程
categories: python总结
---
# 前言 · 零
我们知道，在python里面要终止一个线程，常规的做法就是**设置/检查 --->标志或者锁**方式来实现的。

这种方式好不好呢？

应该是不大好的！因为在所有的程序语言里面，**突然地终止一个线程，这无论如何都不是一个好的设计模式**。

**同时**

有些情况下更甚，比如：
+ 线程打开一个必须合理关闭的临界资源时，比如打开一个可读可写的文件；
+ 线程已经创建了好几个其他的线程，这些线程也是需要被关闭的(这可存在子孙线程游离的风险啊！)。

简单来说，就是我们一大群的线程共线了公共资源，你要其中一个线程“离场”，假如这个线程刚好占用着资源，那么强制让其离开的结局就是资源被锁死了，大家都拿不到了！怎么样是不是有点类似修仙类小说的情节！
>**知道为啥threading仅有start而没有end不？**
>
>你看，线程一般用在网络连接、释放系统资源、dump流文件，这些都跟IO相关了，你突然关闭线程那这些
>没有合理地关闭怎么办？是不是就是给自己造bug呢？啊？！

>因此这种事情中最重要的不是终止线程而是**线程的清理**啊。

# 解决方案 · 壹

一个比较nice的方式就是每个线程都带一个退出请求标志，在线程里面间隔一定的时间来检查一次，看是不是该自己离开了！

```python
import threading

class StoppableThread(threading.Thread):
    """Thread class with a stop() method. The thread itself has to check
    regularly for the stopped() condition."""

    def __init__(self):
        super(StoppableThread, self).__init__()
        self._stop_event = threading.Event()

    def stop(self):
        self._stop_event.set()

    def stopped(self):
        return self._stop_event.is_set()
```

在这部分代码所示，当你想要退出线程的时候你应当显示调用stop()函数，并且使用join()函数来等待线程**合适地退出**。线程应当周期性地检测停止标志。

然而，还有一些使用场景中你真的需要kill掉一个线程：比如，当你封装了一个外部库，但是这个外部库**在长时间调用**，因此你想中断这个过程。


# 解决方案 · 貳

接下来的方案是允许在python线程里面raise一个Exception（当然是有一些限制的）。

```python
def _async_raise(tid, exctype):
    '''Raises an exception in the threads with id tid'''
    if not inspect.isclass(exctype):
        raise TypeError("Only types can be raised (not instances)")
    res = ctypes.pythonapi.PyThreadState_SetAsyncExc(tid,
                                                  ctypes.py_object(exctype))
    if res == 0:
        raise ValueError("invalid thread id")
    elif res != 1:
        # "if it returns a number greater than one, you're in trouble,
        # and you should call it again with exc=NULL to revert the effect"
        ctypes.pythonapi.PyThreadState_SetAsyncExc(tid, 0)
        raise SystemError("PyThreadState_SetAsyncExc failed")

class ThreadWithExc(threading.Thread):
    '''A thread class that supports raising exception in the thread from
       another thread.
    '''
    def _get_my_tid(self):
        """determines this (self's) thread id

        CAREFUL : this function is executed in the context of the caller
        thread, to get the identity of the thread represented by this
        instance.
        """
        if not self.isAlive():
            raise threading.ThreadError("the thread is not active")

        # do we have it cached?
        if hasattr(self, "_thread_id"):
            return self._thread_id

        # no, look for it in the _active dict
        for tid, tobj in threading._active.items():
            if tobj is self:
                self._thread_id = tid
                return tid

        # TODO: in python 2.6, there's a simpler way to do : self.ident

        raise AssertionError("could not determine the thread's id")

    def raiseExc(self, exctype):
        """Raises the given exception type in the context of this thread.

        If the thread is busy in a system call (time.sleep(),
        socket.accept(), ...), the exception is simply ignored.

        If you are sure that your exception should terminate the thread,
        one way to ensure that it works is:

            t = ThreadWithExc( ... )
            ...
            t.raiseExc( SomeException )
            while t.isAlive():
                time.sleep( 0.1 )
                t.raiseExc( SomeException )

        If the exception is to be caught by the thread, you need a way to
        check that your thread has caught it.

        CAREFUL : this function is executed in the context of the
        caller thread, to raise an excpetion in the context of the
        thread represented by this instance.
        """
        _async_raise( self._get_my_tid(), exctype )
```

正如注释里面描述，这不是啥“灵丹妙药”，因为，假如线程在python解释器之外busy，这样子的话终端异常就抓不到啦~

这个代码的合理使用方式是：让线程抓住一个**特定的异常**然后执行清理操作。这样的话你就能终端一个任务并能合适地进行清除。


# 解决方案 · 叁

假如我们要做个啥事情，类似于中断的方式，那么我们就可以用thread.join方式。
```
join的原理就是依次检验线程池中的线程是否结束，没有结束就阻塞直到线程结束，如果结束则跳转执行下一个线程的join函数。

先看看这个：

1. 阻塞主进程，专注于执行多线程中的程序。

2. 多线程多join的情况下，依次执行各线程的join方法，前头一个结束了才能执行后面一个。

3. 无参数，则等待到该线程结束，才开始执行下一个线程的join。

4. 参数timeout为线程的阻塞时间，如 timeout=2 就是罩着这个线程2s 以后，就不管他了，继续执行下面的代码。
```
```python
# coding: utf-8
# 多线程join
import threading, time  
def doWaiting1():  
    print 'start waiting1: ' + time.strftime('%H:%M:%S') + "\n"  
    time.sleep(3)  
    print 'stop waiting1: ' + time.strftime('%H:%M:%S') + "\n" 

def doWaiting2():  
    print 'start waiting2: ' + time.strftime('%H:%M:%S') + "\n"   
    time.sleep(8)  
    print 'stop waiting2: ', time.strftime('%H:%M:%S') + "\n"  

tsk = []    
thread1 = threading.Thread(target = doWaiting1)  
thread1.start()  
tsk.append(thread1)

thread2 = threading.Thread(target = doWaiting2)  
thread2.start()  
tsk.append(thread2)

print 'start join: ' + time.strftime('%H:%M:%S') + "\n"   
for tt in tsk:
    tt.join()
print 'end join: ' + time.strftime('%H:%M:%S') + "\n"
```
**默认join方式，也就是不带参，阻塞模式，只有子线程运行完才运行其他的。**

1、 两个线程在同一时间开启，join 函数执行。

2、waiting1 线程执行（等待）了3s 以后，结束。

3、waiting2 线程执行（等待）了8s 以后，运行结束。

4、join 函数（返回到了主进程）执行结束。
>**这里是默认的join方式，是在线程已经开始跑了之后，然后再join的，注意这点，join之后主线程就必须等子线程结束才会返回主线。**



join的参数，也就是timeout参数，改为2，即join（2），那么结果就是如下了：

1. 两个线程在同一时间开启，join 函数执行。

2. wating1 线程在执行（等待）了三秒以后，完成。

3. join 退出（两个2s，一共4s，36-32=4，无误）。

4. waiting2 线程由于没有在 join 规定的等待时间内（4s）完成，所以自己在后面执行完成。

>**join(2)就是:我给你子线程两秒钟，每个的2s钟结束之后我就走，我不会有丝毫的顾虑！**