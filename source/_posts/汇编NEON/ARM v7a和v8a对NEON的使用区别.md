
---
title: ARM v7a和v8a对NEON的使用区别
categories: 汇编NEON # 配置
---


[**原作大神的链接在这！！！**](https://yq.aliyun.com/articles/57985?spm=a2c4e.11153940.blogcont57934.9.4a251ce1oxxKxE)

 ### ARM浮点运算

强大的ARM v8A芯片，已经不输于JVM的设计了，也是很简单。
源代码：

```cpp
double dadd(double a,double b){
    return a+b;
}

double dsub(double a,double b){
    return a-b;
}

double dmul(double a,double b){
    return a*b;
}

double ddiv(double a,double b){
    return a/b;
}
```
 
 ### ARM v8a的浮点运算

汇编代码：
```cpp
0000000000000760 <_Z4dadddd>:
 760:    1e612800     fadd    d0, d0, d1
 764:    d65f03c0     ret

0000000000000768 <_Z4dsubdd>:
 768:    1e613800     fsub    d0, d0, d1
 76c:    d65f03c0     ret

0000000000000770 <_Z4dmuldd>:
 770:    1e610800     fmul    d0, d0, d1
 774:    d65f03c0     ret

0000000000000778 <_Z4ddivdd>:
 778:    1e611800     fdiv    d0, d0, d1
 77c:    d65f03c0     ret
 ```
 我们可以看到，**寄存器已经不是x开头的通用寄存器了，而变成了d开头的NEON寄存器**。我们实际上是借用了ARM v7a才出现的NEON指令才使得指令变得这么简单。
 
 > 也就是说，在ARM v8a架构下使用NEON就不要像ARM v7a一样的显示调用了，直接指明我要用的是D寄存器，具体怎么调用你们自己看着办吧！
 
 ### ARM v7a的浮点运算：

同样是NEON指令，但是v7a的就比v8a的看起来要复杂一点。不过倒更清晰地反映了逻辑事实。

v7a的NEON指令需要用vmov将通用寄存器中的数传送到NEON寄存器中，然后再进行计算。结果再通过vmov送回到通用寄存器中。

```cpp
00000fde <_Z4dadddd>:
     fde:    ec41 0b17     vmov    d7, r0, r1
     fe2:    ec43 2b16     vmov    d6, r2, r3
     fe6:    ee37 7b06     vadd.f64    d7, d7, d6
     fea:    ec51 0b17     vmov    r0, r1, d7
     fee:    4770          bx    lr

00000ff0 <_Z4dsubdd>:
     ff0:    ec41 0b17     vmov    d7, r0, r1
     ff4:    ec43 2b16     vmov    d6, r2, r3
     ff8:    ee37 7b46     vsub.f64    d7, d7, d6
     ffc:    ec51 0b17     vmov    r0, r1, d7
    1000:    4770          bx    lr

00001002 <_Z4dmuldd>:
    1002:    ec41 0b17     vmov    d7, r0, r1
    1006:    ec43 2b16     vmov    d6, r2, r3
    100a:    ee27 7b06     vmul.f64    d7, d7, d6
    100e:    ec51 0b17     vmov    r0, r1, d7
    1012:    4770          bx    lr

00001014 <_Z4ddivdd>:
    1014:    ec41 0b17     vmov    d7, r0, r1
    1018:    ec43 2b16     vmov    d6, r2, r3
    101c:    ee87 7b06     vdiv.f64    d7, d7, d6
    1020:    ec51 0b17     vmov    r0, r1, d7
    1024:    4770          bx    lr
```

### 传统ARM的浮点运算

没啥说的，都得函数实现了：

```cpp
00001248 <_Z3addll>:
    1248:    1840          adds    r0, r0, r1
    124a:    4770          bx    lr

0000124c <_Z3subll>:
    124c:    1a40          subs    r0, r0, r1
    124e:    4770          bx    lr

00001250 <_Z3mulll>:
    1250:    4348          muls    r0, r1
    1252:    4770          bx    lr

00001254 <_Z3divll>:
    1254:    b508          push    {r3, lr}
    1256:    f001 ff47     bl    30e8 <_Unwind_GetTextRelBase+0x8>
    125a:    bd08          pop    {r3, pc}

0000125c <_Z3modll>:
    125c:    b508          push    {r3, lr}
    125e:    f001 ff4b     bl    30f8 <_Unwind_GetTextRelBase+0x18>
    1262:    1c08          adds    r0, r1, #0
    1264:    bd08          pop    {r3, pc}
```