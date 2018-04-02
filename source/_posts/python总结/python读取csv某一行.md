---
title: python读取csv某一行
categories: python总结
---
## 源起 · 壹
我要读取csv文件中的某一行，写到这里的时候就不知道咋整了：
```python
csv_reader = csv.reader(open('20180105.csv', "rb"))
for row in csv_reader:
#哄逗泥！！！这输出的是第一列哟！
    print(row[0])
```

至此，我想啊，怎么输出一行来呢？明明很简单的问题，看来是python基础不过关呐！ 不过没关系，所有的牛逼不都是这么一点一滴积累起来的么？！

**保持好奇心和求知欲就好啦！！！**

## 思考 · 貳
我把文件全部输出(`print row`)是这样子的：
```
['TestResult', 'InitTime', 'MainsceenCurrVDD12', 'MainsceenCurrVDD18', 'SleepCurrVDD12', 'SleepCurrVDD18', 'RunappTime']
['1', '23.8', '148.00', '18.70', '2.66', '0.97', '205.3']
['0', '0.5', '', '', '', '', '']
```
假设我完全不知道上面的`csv_reader`就是一个迭代器，那么我可能会认为这就是一个二维数组，那么我直接`csv_reader[0]`输出一行试试嘛！反正又不要钱！
```
[code]:
csv_reader = csv.reader(open('20180105.csv', "rb"))
    print csv_reader[0]

[outcome]:
TypeError: '_csv.reader' object has no attribute '__getitem__'
```
失败！*1

既然这样子不行，那么我把它整成二维数组不就好啦！
```python
for row in csv_reader:
    rows.append(row)

print rows[0]
```
噢啦~打完收工！

但是这种写法......**一点都不优雅啊！你直接写个0是什么意思啊？裸奔吗？你append是啥意思啊？不嫌累的慌吗？这么写跟C有什么区别啊！作为贵族你的尊严呢？！**

---
### 为尊严而战 · 及格
```python
  for i,row in enumerate(csv_reader):
        if i == 0:
            print row
```
>**点评：**嗯，不错，用枚举的方式给迭代器里面的每一行都**自动编了一个顺序号**，那么我查找起来就方便多拉！，其次由于我们要的是第一行，因此很快就可以找出来啦！比起上面的方式不知快了多少倍！

---
### 为尊严而战 · 进化

上面是我们根据行号来查找数据，但是假如我们要**根据行内数据特征来查找**呢？

duang~介是嘛？！ **DictReader !!!**

```python
with open('20180105.csv','rb') as csvfile:
    reader = csv.DictReader(csvfile)
    rows = [row for row in reader]
print rows
```

输出的是字典了哟~**第一行就是字典的key**，下面对应的就是value：
```
[{'SleepCurrVDD12': '2.66', 'TestResult': '1', 'SleepCurrVDD18': '0.97', 'InitTime': '23.8', 'RunappTime': '205.3', 'MainsceenCurrVDD18': '18.70', 'MainsceenCurrVDD12': '148.00'},
 {'SleepCurrVDD12': '', 'TestResult': '0', 'SleepCurrVDD18': '', 'InitTime': '0.5', 'RunappTime': '', 'MainsceenCurrVDD18': '', 'MainsceenCurrVDD12': ''}]
```

假如我要找TestResult=“1”的行，咋整？

```python
with open('20180105.csv','rb') as csvfile:
    reader = csv.DictReader(csvfile)
    # rows = [row for row in reader]
    for row in reader:
        if row["TestResult"] == "1":
            print row
```
输出：
```
{'SleepCurrVDD12': '2.66', 'TestResult': '1', 'SleepCurrVDD18': '0.97', 'InitTime': '23.8', 'RunappTime': '205.3', 'MainsceenCurrVDD18': '18.70', 'MainsceenCurrVDD12': '148.00'}
```

