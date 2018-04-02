---
title: Python增强建议书(PEP8Python Enhancement Proposals)
categories: python总结
---
# PEP8 python官方出品
面对编码风格选择，比较常见的包括 [PEP-8](https://www.python.org/dev/peps/pep-0008/) 和 [Google Python Style Guide](http://zh-google-styleguide.readthedocs.org/en/latest/google-python-styleguide/)。

每个PEP文件可能是描述某种功能、信息、或进程等。大部分情况下我们可以把他当成设计文档，里面包含了**技术规范**和**功能的基本原理说明**等。

PEP8是Python官方提出的：Style Guide for Python Code，算是社区规范，如果你是自己写给自己看的，那爱怎么来就怎么来，如果你是在一个公司，一个团队，那良好的代码习惯，才是缩减别人读懂你代码的时间成本有效方法，当然对于自己，良好的编写规范，也会让你的代码显得更为美观，紧急情况下，可以快速查找。
```python
#单行过长
with open('/path/to/some/file/you/want/to/read') as file_1, \
     open('/path/to/some/file/being/written', 'w') as file_2:
    file_2.write(file_1.read())
```

```python
#类实现demo
class Rectangle(Blob):

    def __init__(self, width, height,
                 color='black', emphasis=None, highlight=0):
        if (width == 0 and height == 0 and
                color == 'red' and emphasis == 'strong' or
                highlight > 100):
            raise ValueError("sorry, you lose")
        if width == 0 and height == 0 and (color == 'red' or
                                           emphasis is None):
            raise ValueError("I don't think so -- values are %s, %s" %
                             (width, height))
        Blob.__init__(self, width, height,
                      color, emphasis, highlight)
```

```python
# 括号里边避免空格
# Yes
spam(ham[1], {eggs: 2})
# No
spam( ham[ 1 ], { eggs: 2 } )
```

```python
# 逗号，冒号，分号之前避免空格
# Yes
if x == 4: print x, y; x, y = y, x
# No
if x == 4 : print x , y ; x , y = y , x
```

```python
# Yes函数调用的左括号之前不能有空格
spam(1)
dct['key'] = lst[index]

# No
spam (1)
dct ['key'] = lst [index]
```

```python
# Yes赋值等操作符前后不能因为对齐而添加多个空格
x = 1
y = 2
long_variable = 3

# No
x             = 1
y             = 2
long_variable = 3
```

```python
# Yes优先级高的运算符或操作符的前后不建议有空格。
i = i + 1
submitted += 1
x = x*2 - 1
hypot2 = x*x + y*y
c = (a+b) * (a-b)

# No
i=i+1
submitted +=1
x = x * 2 - 1
hypot2 = x * x + y * y
c = (a + b) * (a - b)
```

```python
# Yes关键字参数和默认值参数的前后不要加空格
def complex(real, imag=0.0):
    return magic(r=real, i=imag)

# No
def complex(real, imag = 0.0):
    return magic(r = real, i = imag)
```
下面讲述首尾有下划线的情况:
+ ` _single_leading_underscore`:(单前置下划线): 弱内部使用标志。 例如"from M import " 不会导入以下划线开头的对象。

+ `single_trailing_underscore_`(单后置下划线): 用于避免与 Python关键词的冲突。
eq. `Tkinter.Toplevel(master, class_='ClassName')`

+ ` __double_leading_underscore`(双前置下划线): 当用于命名类属性，会触发名字重整。 (在类FooBar中，__boo变成 _FooBar__boo)。

+ `__double_leading_and_trailing_underscore__`(双前后下划线)：用户名字空间的魔法对象或属性。例如:__init__ , __import__ or __file__，不要自己发明这样的名字。


**同函数命名规则**:
非公开方法和实例变量增加一个前置下划线。
为避免与子类命名冲突，采用两个前置下划线来触发重整。类Foo属性名为__a， 不能以 Foo.__a访问。(执著的用户还是可以通过Foo._Foo__a。) 通常双前置下划线仅被用来避免与基类的属性发生命名冲突。

**谨记这些Python指南：**

+ 公开属性应该没有前导下划线。

+ 如果公开属性名和保留关键字冲突，可以添加后置下划线

+ 简单的公开数据属性，最好只公开属性名，没有复杂的访问/修改方法，python的Property提供了很好的封装方法。 d.如果不希望子类使用的属性，考虑用两个前置下划线(没有后置下划线)命名。

**公共和内部接口:**

+ 任何向后兼容的保证只适用于公共接口。

+ 文档化的接口通常是公共的，除非明说明是临时的或为内部接口、其他所有接口默认是内部的。

+ 为了更好地支持内省，模块要在__all__属性列出公共API。

+ 内部接口要有前置下划线。

+ 如果命名空间(包、模块或类)是内部的，里面的接口也是内部的。

+ 导入名称应视为实现细节。其他模块不能间接访名字，除非在模块的API文档中明确记载，如os.path中或包的__init__暴露了子模块。

**此外所有try/except子句的代码要尽可的少，以免屏蔽其他的错误。**
```python
# Yes
try:
    value = collection[key]
except KeyError:
    return key_not_found(key)
else:
    return handle_value(value)

# No
try:
    # 太泛了!
    return handle_value(collection[key])
except KeyError:
    # 会捕捉到handle_value()中的KeyError
    return key_not_found(key)
```


---

# python风格指南(Google出品)
[python风格指南(Google出品)](http://zh-google-styleguide.readthedocs.io/en/latest/google-python-styleguide/#)

### 函数注释部分
**背景**：Python 是 Google主要的脚本语言。这本风格指南主要包含的是针对python的编程准则。
```python
def fetch_bigtable_rows(big_table, keys, other_silly_variable=None):
    """Fetches rows from a Bigtable.

    Retrieves rows pertaining to the given keys from the Table instance
    represented by big_table.  Silly things may happen if
    other_silly_variable is not None.

    Args:
        big_table: An open Bigtable Table instance.
        keys: A sequence of strings representing the key of each table row
            to fetch.
        other_silly_variable: Another optional variable, that has a much
            longer name than the other args, and which does nothing.

    Returns:
        A dict mapping keys to the corresponding table row data
        fetched. Each row is represented as a tuple of strings. For
        example:

        {'Serak': ('Rigel VII', 'Preparer'),
         'Zim': ('Irk', 'Invader'),
         'Lrrr': ('Omicron Persei 8', 'Emperor')}

        If a key from the keys argument is missing from the dictionary,
        then that row was not found in the table.

    Raises:
        IOError: An error occurred accessing the bigtable.Table object.
    """
    pass
```

### 类
类应该在其定义下有一个用于描述该类的文档字符串. 如果你的类有公共属性(Attributes), 那么文档中应该有一个属性(Attributes)段. 并且应该遵守和函数参数相同的格式.
```python
class SampleClass(object):
    """Summary of class here.

    Longer class information....
    Longer class information....

    Attributes:
        likes_spam: A boolean indicating if we like SPAM or not.
        eggs: An integer count of the eggs we have laid.
    """

    def __init__(self, likes_spam=False):
        """Inits SampleClass with blah."""
        self.likes_spam = likes_spam
        self.eggs = 0

    def public_method(self):
        """Performs operation blah."""
```
在逗号后面空一格，前面不空，赋值符号两边都空。
```python
Yes: x = a + b
     x = '%s, %s!' % (imperative, expletive)
     x = '{}, {}!'.format(imperative, expletive)
     x = 'name: %s; score: %d' % (name, n)
     x = 'name: {}; score: {}'.format(name, n)
```