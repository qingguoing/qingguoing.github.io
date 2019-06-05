---
title: Python 3 学习笔记（1）
date:  2018-01-21 15:29:45
tags: ['python', 'learning']
category: python
---

python3 学习笔记系列之基础部分，学习网址：Python教程 - 廖雪峰的官方网站。本文主要学习基础知识：安装、变量声明、循环和条件判断、list 和 tuple、dict 和 set 以及函数声明和调用。

随着人工智能和机器学习的发展，python 作为最火的传播媒介之一，是需要掌握其简单的语法，保证能看懂平常的代码和运行方式。本文学习的 python3。

<!--more-->

### 安装

mac 上默认内置的 python 版本是 2.7，通过 brew install python3 来安装 python3。终端输入 python3 进入终端交互界面表示安装成功。

### 基础

python 作为一种弱类型语言，同 javascript 一样，对于变量的声明无需具体的类型指定，而且不需要关键字指定，而且变量指向的类型可以随时更改。通常采用全部大写字母声明一个常量。

```python
a = 123; // 声明一个变量
a = true;
python 中的布尔值是 True 和 False，布尔值运算符分别是 and， or 和 not。
```

### 字符串

对于单个字符的编码，Python 提供了 ord 函数获取字符的整数表示，chr 函数把编码转换为对应的字符，用带 b 前缀的单引号或双引号表示以字节为单位的字符串。要注意区分 'ABC' 和 b'ABC'，前者是 str，后者虽然内容显示得和前者一样，但bytes的每个字符都只占用一个字节。len() 函数计算的是 str 的字符数，如果换成 bytes，len() 函数就计算字节数。

```python
ord('A') # 65
chr('66') # 66
x = b'ABC'
'ABC'.encode('ascii') # b'ABC'
'中文'.encode('utf-8') # b'\xe4\xb8\xad\xe6\x96\x87'
b'\xe4\xb8\xad\xe6\x96\x87'.decode('utf-8') # '中文'
# 1个中文字符经过 UTF-8 编码后通常会占用3个字节，而1个英文字符只占用1个字节
len(b'\xe4\xb8\xad\xe6\x96\x87') # 6
len(b'ABC') # 3
```

### 格式化

在 Python 中，采用的格式化方式和C语言是一致的，用 % 实现。

```python
'Hello, %s' % 'world' # hello world
```

### 条件判断和循环

python 中的判断和循环中的代码块通过缩进来表示，有别于其它语言的 {} 包裹形式。

```python
if <条件判断1>:
    <执行1>
elif <条件判断2>:
    <执行2>
elif <条件判断3>:
    <执行3>
else:
    <执行4>
```

range() 函数生成一个整数序列，下面中的 range(101) 就是生成 0 到 100 中的所有整数。 for x in ... 循环就是把每个元素代入变量x，然后执行缩进块的语句。

```python
sum = 0
for x in range(101):
    sum = sum + x
print(sum)
```

### list 和 tuple

Python 内置的一种数据类型是列表 list。list 是一种有序的集合，可以随时添加和删除其中的元素。(等同于其它语言中的数组)。操作 list 的方法有 append、insert、pop。

```python
classmates = ['Michael', 'Bob', 'Tracy']
len(classmates) # 3
classmates.append('Adam') # ['Michael', 'Bob', 'Tracy', 'Adam']
classmates.insert(1, 'Jack') # ['Michael', 'Jack', 'Bob', 'Tracy', 'Adam']
classmates.pop() # ['Michael', 'Jack', 'Bob', 'Tracy']
classmates.pop(1) # ['Michael', 'Bob', 'Tracy']
```

另一种有序列表叫元组：tuple。tuple 和 list 非常类似，但是tuple一旦初始化就不能修改，即 tuple 的每个元素指向永不变。

```python
classmates = ('Michael', 'Bob', 'Tracy')
# 只有1个元素的tuple定义时必须加一个逗号 ,
t = (1,)
```

### dict 和 set

Python 内置了字典：dict 的支持，dict 全称 dictionary，在其他语言中也称为 map，使用键-值（key-value）存储，具有极快的查找速度。

```python
d = {'Michael': 95, 'Bob': 75, 'Tracy': 85}
d['Michael'] # 95
```

还通过 dict 提供的 get() 方法，如果 key 不存在，可以返回 None，或者自己指定的 value。

```python
'Thomas' in d # False
d.get('Thomas') # None
d.get('Thomas', -1) # -1
```

要删除一个 key，用 pop(key) 方法，对应的 value 也会从 dict 中删除：

```python
d.pop('Bob') # 75
```

set 和 dict 类似，也是一组 key 的集合，但不存储 value。由于 key 不能重复，所以，在 set 中，没有重复的 key。要创建一个 set，需要提供一个 list 作为输入集合，重复的元素会被自动过滤。

```python
s = set([1, 1, 2, 2, 3, 3]) # {1, 2, 3}
```

通过 add(key) 方法可以添加元素到set中，可以重复添加，但不会有效果。通过 remove(key) 方法可以删除元素。

```python
s.add(4) # {1, 2, 3, 4}
s.remove(2) # {1, 3, 4}
```

### 函数

```python
def functionName(param):
  #body
```

上面是 python 中函数定义的格式。如果没有 return 语句，函数执行完毕后也会返回结果，只是结果为 None。`return None` 可以简写为return。同时，函数体内可以存在多个 return 语句，随时返回函数的结果。

一个函数可以同时返回多个值，此时的返回值是一个 tuple。在语法上，返回一个tuple可以省略括号，而多个变量可以同时接收一个tuple，按位置赋给对应的值。

```python
def move(x, y):
    nx = x + 3
    ny = y - 5
    return nx, ny
x, y = move(1, 5)
```

### 空函数

```python
def noop1():
  return
def noop2():
  pass
```

第二个空函数的定义使用了一个 pass 占位符，实际上什么都没做。缺少了pass，代码运行就会有语法错误。两个空函数默认返回都是 None。

### 函数参数

```python
# 位置参数
def power(x):
    return x * x
power(2);
```

### 默认参数

```python
# 默认参数，必选参数在前
def power(x, n=2, base=0):
    s = 1
    while n > 0:
        n = n - 1
        s = s * x
    return s + base
power(2)
```

有多个默认参数时，调用的时候，既可以按顺序提供默认参数，未传入的参数采用默认值。也可以不按顺序提供部分默认参数，此时需要把参数名写上。例如 power(2, base=3) 时，参数 n 采用默认值 2，base 采用传入的值 3。

### 可变参数

函数的参数前面加个 *，标识函数接受到的参数是一个 tuple。调用时，可以传入任意个参数，包括 0 个。类似 JavaScript 中 ES6 版本新增的 ... 运算符。

```python
def calc(*numbers):
    sum = 0
    for n in numbers:
        sum = sum + n * n
    return sum
calc() # 0
nums = [0, 1, 2]
# *nums 表示把 nums 这个 list 的所有元素作为可变参数传进去
calc(*nums)
```

### 关键字参数

可变参数允许你传入0个或任意个参数，这些可变参数在函数调用时自动组装为一个tuple。而关键字参数允许你传入0个或任意个含参数名的参数，这些关键字参数在函数内部自动组装为一个dict。

```python
def person(name, age, **kw):
    print('name:', name, 'age:', age, 'other:', kw)
# 只传入必选参数
person('test', 18) # name: test age: 18 other: {}
# 任意数量的关键字参数
person('test', 18, gender='F', job='FE') # name: test age: 18 other: {'gender': 'F', 'job': 'FE'}
extra = {'city': 'Beijing', 'job': 'FE'}
# **extra表示把 extra 这个 dict 的所有 key-value 用关键字参数传入到函数的 **kw 参数，kw 将获得一个 dict，注意 kw 获得的 dict 是 extra 的一份拷贝，对 kw 的改动不会影响到函数外的 extra
person('test', 18, **extra) # name: test age: 18 other: {'city': 'Beijing', 'job': 'FE'}
```

### 命名关键字参数

用来限制关键字参数的名字。命名关键字参数需要一个特殊分隔符 *，* 后面的参数被视为命名关键字参数。如果函数定义中已经有了一个可变参数，后面跟着的命名关键字参数就不再需要一个特殊分隔符 * 了。函数调用时，必须传入参数名，否则将被视为位置参数而报错（实参大于形参个数）。

```python
# 只接收city和job作为关键字参数
def person(name, age, *, city, job):
    print(name, age, city, job)
def person(name, age, *args, city, job):
    print(name, age, args, city, job)
# 命名关键字参数的默认值
def person(name, age, *, city='Beijing', job):
    print(name, age, city, job)
```

### 总结

在Python中定义函数，可以用必选参数、默认参数、可变参数、关键字参数和命名关键字参数，这5种参数都可以组合使用。但是请注意，参数定义的顺序必须是：必选参数、默认参数、可变参数、命名关键字参数和关键字参数。

```python
def f1(a, b, c=0, *args, **kw):
    print('a =', a, 'b =', b, 'c =', c, 'args =', args, 'kw =', kw)
def f2(a, b, c=0, *, d, **kw):
    print('a =', a, 'b =', b, 'c =', c, 'd =', d, 'kw =', kw)
args = (1, 2, 3, 4)
kw = {'d': 99, 'x': '#'}
f1(*args, **kw) # a = 1 b = 2 c = 3 args = (4,) kw = {'d': 99, 'x': '#'}
args = (1, 2, 3)
f2(*args, **kw) # a = 1 b = 2 c = 3 d = 99 kw = {'x': '#'}
```
