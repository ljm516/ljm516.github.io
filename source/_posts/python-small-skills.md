---
title: python 中的一些小技巧合集
date: 2018-01-31 22:45:00
categories: Python
tags: 
- notes 
---

这里汇总一些不容易记住语法小技巧。

<!--more-->

## 删除空白

* str.rstrip(): 去除右空白
* str.lstrip(): 去除左空白
* str.strip(): 去除两端空白

```python
>>> s = ' l j m '
>>> print(s.rstrip())
 l j m
>>> print(s.lstrip())
l j m
>>> print(s.rstrip(), len(s.rstrip()))
 l j m 6
>>> print(s.lstrip(), len(s.lstrip()))
l j m  6
>>> print(s.strip(), len(s.strip()))
l j m 5
>>>

```

** str.strip() 并不能将字符之间的空格去掉哦 **

## 判断一个元素是否任意匹配集合中的元素

```python
>>> item_list = ['a', 'ab', 'abc', 'abcd']
>>> item = 'abc'
>>> result = [x for i, x in enumerate(item_list) if x in item]
>>> for i in result:
...     print(i)
...
a
ab
abc
```

## 循环集合，并删除符合条件的元素


* this way will be throw index out of range error
```python
>>> item_list = ['a', 'ab', 'abc', 'abcd']
>>> for index in range(len(item_list)):
...     if 'a' in item_list[index]:
...             item_list.remove(item_list[index])
...
Traceback (most recent call last):
  File "<stdin>", line 2, in <module>
IndexError: list index out of range
```

* copy itself each loop
```python
>>> item_list = ['a', 'ab', 'abc', 'abcd']
>>> for item in item_list[:]:
...     if 'a' in item:
...             item_list.remove(item)
...
>>> print(item_list)
[]
```

* Reverse order traversal
```python
>>> item_list = ['a', 'ab', 'abc', 'abcd']
>>> for index in range(len(item_list) - 1, -1, -1):
...     if 'a' in item_list[index]:
...             item_list.remove(item_list[index])
...
>>> print(item_list)
[]
```

## 时间格式化

```python
>>> import time
>>> print(time.strftime('%Y-%m-%d %H:%m:%S', time.localtime(time.time())))
2018-01-25 22:01:10
>>> format_time = time.strftime('%Y-%m-%d %H:%m:%S', time.localtime(time.time()))
>>> target = format_time.replace('-','').replace(':','').replace(' ','')
>>> print(target)
20180125220134
```