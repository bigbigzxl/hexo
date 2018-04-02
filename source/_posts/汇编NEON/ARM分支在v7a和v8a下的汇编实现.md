
---
title: ARM分支在v7a和v8a下的汇编实现
categories: 汇编NEON # 配置
---


[**链接**](https://yq.aliyun.com/articles/57989?spm=a2c4e.11153940.blogcont57985.12.2ed113a7u6wyfW)

### Java分支结构

栗子：
```java
    public static long bigger(long a, long b){
        if(a>=b){
            return a;
        }else{
            return b;
        }
    }

    public static int less(int a,int b){
        if(a<=b){
            return a;
        }else{
            return b;
        }
    }
```

### ARM的分支结构

对于这种根据不同情况选择不同值的语句，ARM在设计时专门有一条强大的csel指令，专门做这个事情。

先用cmp指令比较两个值，然后设置条件:
+ ge是大于或等于;
+ le是小于或等于;

**如果条件满足，则x0就是x0，否则x0取x1的值。**

```cpp
000000000000079c <_Z6biggerll>:
 79c:    eb01001f     cmp    x0, x1
 7a0:    9a81a000     csel    x0, x0, x1, ge
 7a4:    d65f03c0     ret

00000000000007a8 <_Z4lessii>:
 7a8:    6b01001f     cmp    w0, w1
 7ac:    1a81d000     csel    w0, w0, w1, le
 7b0:    d65f03c0     ret
```

### Thumb2指令的分支结构

**csel这样的指令需要32位的长度才放得下.**

**Thumb2本着能省则省的原则，改用16位的条件mov指令来实现，可以节省2个字节指令空间。**

```cpp
0000101c <_Z6biggerll>:
    101c:    4288          cmp    r0, r1
    101e:    bfb8          it    lt
    1020:    4608          movlt    r0, r1
    1022:    4770          bx    lr

00001024 <_Z4lessii>:
    1024:    4288          cmp    r0, r1
    1026:    bfa8          it    ge
    1028:    4608          movge    r0, r1
    102a:    4770          bx    lr
```

### Thumb指令的分支结构

Thumb指令没有带条件的mov操作，更不可能有csel这样复杂的指令了。

那也没问题，返璞归真，我们直接跳转就是了呗〜


bge.n，是说大于或等于，也就是不小于的时候直接跳到12aa，就是bx lr返回这条指令上去。

adds r0, r1, #0其实也可以翻译成movs r0,r1。前面我们讲过，movs r0,r1其实是adds r0, r1, #0的别名。本质上是一回事。

```cpp
000012a4 <_Z6biggerll>:
    12a4:    4288          cmp    r0, r1
    12a6:    da00          bge.n    12aa <_Z6biggerll+0x6>
    12a8:    1c08          adds    r0, r1, #0
    12aa:    4770          bx    lr

000012ac <_Z4lessii>:
    12ac:    4288          cmp    r0, r1
    12ae:    dd00          ble.n    12b2 <_Z4lessii+0x6>
    12b0:    1c08          adds    r0, r1, #0
    12b2:    4770          bx    lr
```