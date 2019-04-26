---
title: python 爬虫学习 part2-正则表达式
date: 2018-01-08 22:20:00
categories: Python
tags:
- 爬虫
---

在爬虫网页信息的时候，正则表达式可以匹配你想要的信息。正则在爬虫学习中是很重要的。

<!--more-->

**{% post_link python爬虫_part3_scrapy学习之入门 python爬虫_part3_scrapy学习之入门 %}**

### 正则表达式

- 通用字符


符号 | 含义
---|---
\w | 匹配任意一个字母、数字或者下划线
\W | 匹配除字母、数字和下划线以外的任意一个字符
\d | 匹配任意一个十进制数
\D | 匹配除十进制以外的任意一个其他字符
\s | 匹配任意一个空白字符 
\S | 匹配除空白字符以外的任意一个其他字符

- 元字符

> 正则表达式中具有一些特殊含义的字符


符号 | 含义
---|---
. | 匹配除换行符以外的任意字符
^ | 匹配字符串的开始位置
$ | 匹配字符串的结束位置
* | 匹配 0 次，1 次或多次前面的原子
? | 匹配 0 次或 1 次前面的原子
+ | 匹配 1 次或多次前面的原子
{n} | 前面的原子恰好出现 n 次
{n,}| 前面的原子至少出现 n 次
{n, m}| 前面的原子至少出现 n 次，之多出现 m 次
竖杠 | 模式选择符
() | 模式单元符


- 模式修正


符号 | 含义
---|---
I | 匹配时忽略大小写
M | 多行匹配
L | 做本地化识别匹配
U | 根据 Unicode 字符及解析字符
S | 让 `.` 匹配包含换行符，即用了该模式修正后，`.`匹配就可以匹配任意字符了


- 贪婪模式和懒惰模式

贪婪模式的核心点就是尽可能多的匹配，懒惰模式就是尽可能少的匹配。

example: 

```python
>>> p1= 'p.*y'  # 贪婪匹配
>>> p2 = 'p.*?y'  # 懒惰匹配
>>> s = 'adcdfphp345pythony_py'
>>> r1 = re.search(p1, s)
>>> r2 = re.search(p2, s)
>>> print(r1, r2)

<_sre.SRE_Match object; span=(5, 21), match='php345pythony_py'> <_sre.SRE_Match object; span=(5, 13), match='php345py'>
```

### 正则表达式常用的函数

#### re.match()
从字符串起始位置匹配

#### re.search()
从整个字符串进行扫描匹配

#### 全局匹配函数
将符合模式的内容全部匹配出来

思路：
* 使用 re.compile() 对正则表达式进行预编译
* 使用 findall() 根据正则表达式从源字符串中将匹配的结果全部找出

```python
>>> import re
>>> string = 'hellomypythonhispythonourpythonend'
>>> pattern = re.compile('.python.')
>>> result = pattern.findall(string)
>>> print(result)

['ypythonh', 'spythono', 'rpythone']
```
#### re.sub()

实现替换某些字符串的功能。re.sub(pattern. rep, string, max)

* pattern: 正则表达式
* rep: 目标字符串
* string: 源字符串
* max: 最多替换次数(默认全部)

```python
>>> import re
>>> pattern = re.compile('python.')
>>> result1 = re.sub(pattern, 'php', string)
>>> result2 = re.sub(pattern, 'php', string, 2)
>>> print(result1, result2)

hellomyphpisphpurphpnd hellomyphpisphpurpythonend
```

#### some protices

```python

>>> import re
>>> url_pattern = '[a-zA-Z]+://[^\s]*[.com|.cn]'  # .com 或者 .cn 网址
>>> url_string = '<a href="http://www.baidu.com">百度首页</a>'
>>> r1 = re.search(url_pattern, url_string)
>>> print(r1)
<_sre.SRE_Match object; span=(9, 29), match='http://www.baidu.com'>

>>> phone_pattern = '\d{4}-\d{7}|\d{3}-\d{8}'  # 电话号码
>>> phone_string = '021-111111111111111'
>>> r2 = re.search(phone_pattern, phone_string)
>>> print(r2)
<_sre.SRE_Match object; span=(0, 12), match='021-11111111'>

>>> email_pattern = '\w+([.+-]\w+)*@\w+([.-]\w+)*\.\w+([.-]\w+)*' # 电子邮箱
>>> email_string = '<a href="http://www.baidu.com">百度首页></a><br><a href="18370600881@163.com">电子邮箱</a>'
>>> r3 = re.search(email_pattern, email_string)
>>> print(r3)
<_sre.SRE_Match object; span=(53, 72), match='18370600881@163.com'>

```