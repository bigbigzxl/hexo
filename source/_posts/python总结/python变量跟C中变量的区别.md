---
title: python变量跟C中变量的区别
categories: python总结
---
[**英文元档**](http://python.net/~goodger/projects/pycon/2007/idiomatic/handout.html#other-languages-have-variables)

# 其他语言的变量有"variables"
比如c语言中，定义一个变量，就是把值放到变量盒子里面去：

`int a = 1` | `int a = 2` | `int b = a`
---|---|---
![](http://upload-images.jianshu.io/upload_images/4749583-05c76f863f713203..png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) | ![](http://upload-images.jianshu.io/upload_images/4749583-44ec650d8ced4929..png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) |![](http://upload-images.jianshu.io/upload_images/4749583-c487e0c573c5afd0..png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)![](http://upload-images.jianshu.io/upload_images/4749583-ea79b4c17af2f23f..png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


如上图所示，赋值b=a，就是新建一个盒子，然后把值赋值一份放过去，特点是：**两个值之间完全独立。**

# python的则是 "names"

python的变量就是贴标签！`（python里面不是什么都是对象么，我们的变量名就是一个标签而已，类属性啥的都在变量自己内部保存着呢！）`

`a = 1` | `a = 2` | `b = a`
---|---|---
![](http://upload-images.jianshu.io/upload_images/4749583-35d3f68b63f7806c..png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) | ![](http://upload-images.jianshu.io/upload_images/4749583-fbbc79b5fed9d6eb..png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)         ![](http://upload-images.jianshu.io/upload_images/4749583-6b118a1d05fc63c3..png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) |![](http://upload-images.jianshu.io/upload_images/4749583-794a8953eddd863f..png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面的a=2之后，1就没有归属了，就无法调用了，python的基于引用的内存管理器很快就会把这个对象的内存给清理掉的。

# 理解举例：默认的参数值

这是一个很容易出错的情景：

```python
def bad_append(new_item, a_list=[]):
    a_list.append(new_item)
    return a_list
```

**上面的a_list在函数定义的时候就赋值为空数组了，因此每次你调用函数的时候，你得到同样的默认值**
```python
>>> print bad_append('one')
['one']

>>> print bad_append('two')
['one', 'two']
```
第一次进去的时候就是一个空的数组，然后你添加一个`“one”`，第二次进去的时候，a_list已经有一个元素了（因为这个是在函数定义的时候就初始化的哟，不是每次调用函数的时候才临时初始化的哈！）

由于list是**可变对象**,因此你是可以随时变量内容的。

正确的操作方法是（得到一个默认的list or dictionary or set）：在每次运行的时候才创建这个可变对象，也就是说在函数里面创建对象。

```python
def good_append(new_item, a_list=None):
    if a_list is None:
        a_list = []
    a_list.append(new_item)
    return a_list
```

每次均初始化一个空的list，然后a_list指向它，这样的话就不会每次都往a_list累计填值了。