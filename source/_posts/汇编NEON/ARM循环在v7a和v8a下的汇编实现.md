
---
title: ARM循环在v7a和v8a下的汇编实现
categories: 汇编NEON # 配置
---


[**链接**](https://yq.aliyun.com/articles/57989?spm=a2c4e.11153940.blogcont57985.12.2ed113a7u6wyfW)

### Java循环结构

```java
 public static long testWhile(int value){
        int i=0;
        long sum = 0;
        while(i<=value){
            sum += i;
            i++;
        }
        return sum;
    }

    public static long testDoWhile(int value){
        long sum = 0;
        int i=0;
        do{
            sum += i;
            i++;
        }while(i<value);
        return sum;
    }

    public static long testFor(int value){
        long sum = 0;
        for(int i=0;i<=value;i++){
            sum += i;
        }
        return sum;
    }
```

### Thumb的循环结构

C++代码与Java代码几乎是一字未改：
```cpp
long testWhile(int value){
    int i=0;
    long sum = 0;
    while(i<=value){
        sum += i;
        i++;
    }
    return sum;
}

long testDoWhile(int value){
    long sum = 0;
    int i=0;
    do{
        sum += i;
        i++;
    }while(i<value);
    return sum;
}

long testFor(int value){
    long sum = 0;
    for(int i=0;i<=value;i++){
        sum += i;
    }
    return sum;
}
```

我们下面来看看16位的Thumb指令是如何实现循环的。

1. **value参数传进来在r0;**

2. **将r2,r3清0;**

3. **r2与参数传进来的r0比较;**

4. **如果0已经比r0大了，循环结束，将结果r3赋给r0，返回。**

5. **如果0比r0小，则继续循环，r3结果等于r3值加上r2值，r2值随后加1。**

6. **然后无条件回12b8那条比较语句，再判断r2是不是大于r0，如此循环。**

```cpp
000012b4 <_Z9testWhilei>:
    12b4:    2300          movs    r3, #0
    12b6:    1c1a          adds    r2, r3, #0
    12b8:    4282          cmp    r2, r0
    12ba:    dc02          bgt.n    12c2 <_Z9testWhilei+0xe>
    12bc:    189b          adds    r3, r3, r2
    12be:    3201          adds    r2, #1
    12c0:    e7fa          b.n    12b8 <_Z9testWhilei+0x4>
    12c2:    1c18          adds    r0, r3, #0
    12c4:    4770          bx    lr
```
后面两个也是语句位置变了变，本质是一样的。

```cpp
000012c6 <_Z11testDoWhilei>:
    12c6:    2300          movs    r3, #0
    12c8:    1c1a          adds    r2, r3, #0
    12ca:    18d2          adds    r2, r2, r3
    12cc:    3301          adds    r3, #1
    12ce:    4283          cmp    r3, r0
    12d0:    dbfb          blt.n    12ca <_Z11testDoWhilei+0x4>
    12d2:    1c10          adds    r0, r2, #0
    12d4:    4770          bx    lr

000012d6 <_Z7testFori>:
    12d6:    2300          movs    r3, #0
    12d8:    1c1a          adds    r2, r3, #0
    12da:    4283          cmp    r3, r0
    12dc:    dc02          bgt.n    12e4 <_Z7testFori+0xe>
    12de:    18d2          adds    r2, r2, r3
    12e0:    3301          adds    r3, #1
    12e2:    e7fa          b.n    12da <_Z7testFori+0x4>
    12e4:    1c10          adds    r0, r2, #0
    12e6:    4770          bx    lr
```

### Thumb2的循环结构
#### arm v7a的循环结构
除了地址不同，与前面的Thumb指令完全相同.
```cpp
0000102c <_Z9testWhilei>:
    102c:    2300          movs    r3, #0
    102e:    461a          mov    r2, r3
    1030:    4282          cmp    r2, r0
    1032:    dc02          bgt.n    103a <_Z9testWhilei+0xe>
    1034:    4413          add    r3, r2
    1036:    3201          adds    r2, #1
    1038:    e7fa          b.n    1030 <_Z9testWhilei+0x4>
    103a:    4618          mov    r0, r3
    103c:    4770          bx    lr
```

#### arm v8a的循环结构

下面我们看第一个while循环在v8a上的实现：

竟然动用了强大的**NEON指令**！

```cpp
00000000000007c0 <_Z9testWhilei>:
 7c0:    2a0003e3     mov    w3, w0
 7c4:    37f80643     tbnz    w3, #31, 88c <_Z9testWhilei+0xcc>
 7c8:    51000c60     sub    w0, w3, #0x3
 7cc:    7100147f     cmp    w3, #0x5
 7d0:    53027c00     lsr    w0, w0, #2
 7d4:    11000464     add    w4, w3, #0x1
 7d8:    11000400     add    w0, w0, #0x1
 7dc:    531e7402     lsl    w2, w0, #2
 7e0:    5400050d     b.le    880 <_Z9testWhilei+0xc0>
 7e4:    100005e5     adr    x5, 8a0 <_Z9testWhilei+0xe0>
 7e8:    4f000484     movi    v4.4s, #0x4
 7ec:    4f000400     movi    v0.4s, #0x0
 7f0:    52800001     mov    w1, #0x0                       // #0
 7f4:    3dc000a1     ldr    q1, [x5]
 7f8:    0f20a422     sxtl    v2.2d, v1.2s
 7fc:    11000421     add    w1, w1, #0x1
 800:    4f20a423     sxtl2    v3.2d, v1.4s
 804:    6b01001f     cmp    w0, w1
 808:    4ea48421     add    v1.4s, v1.4s, v4.4s
 80c:    4ee08440     add    v0.2d, v2.2d, v0.2d
 810:    4ee08460     add    v0.2d, v3.2d, v0.2d
 814:    54ffff28     b.hi    7f8 <_Z9testWhilei+0x38>
 818:    5ef1b800     addp    d0, v0.2d
 81c:    6b02009f     cmp    w4, w2
 820:    4e083c00     mov    x0, v0.d[0]
 824:    540002c0     b.eq    87c <_Z9testWhilei+0xbc>
 828:    11000444     add    w4, w2, #0x1
 82c:    8b22c000     add    x0, x0, w2, sxtw
 830:    6b04007f     cmp    w3, w4
 834:    5400024b     b.lt    87c <_Z9testWhilei+0xbc>
 838:    11000841     add    w1, w2, #0x2
 83c:    8b24c000     add    x0, x0, w4, sxtw
 840:    6b01007f     cmp    w3, w1
 844:    540001cb     b.lt    87c <_Z9testWhilei+0xbc>
 848:    11000c44     add    w4, w2, #0x3
 84c:    8b21c000     add    x0, x0, w1, sxtw
 850:    6b04007f     cmp    w3, w4
 854:    5400014b     b.lt    87c <_Z9testWhilei+0xbc>
 858:    11001041     add    w1, w2, #0x4
 85c:    8b24c000     add    x0, x0, w4, sxtw
 860:    6b01007f     cmp    w3, w1
 864:    540000cb     b.lt    87c <_Z9testWhilei+0xbc>
 868:    11001442     add    w2, w2, #0x5
 86c:    8b21c000     add    x0, x0, w1, sxtw
 870:    6b02007f     cmp    w3, w2
 874:    8b22c001     add    x1, x0, w2, sxtw
 878:    9a80a020     csel    x0, x1, x0, ge
 87c:    d65f03c0     ret
 880:    d2800000     mov    x0, #0x0                       // #0
 884:    52800002     mov    w2, #0x0                       // #0
 888:    17ffffe8     b    828 <_Z9testWhilei+0x68>
 88c:    d2800000     mov    x0, #0x0                       // #0
 890:    d65f03c0     ret
 894:    d503201f     nop
 898:    d503201f     nop
 89c:    d503201f     nop
 8a0:    00000000     .inst    0x00000000 ; undefined
 8a4:    00000001     .inst    0x00000001 ; undefined
 8a8:    00000002     .inst    0x00000002 ; undefined
 8ac:    00000003     .inst    0x00000003 ; undefined
```