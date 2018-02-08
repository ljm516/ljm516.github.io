---
title: python爬虫_part4_scrapy框架学习之进阶
date: 2018-01-21 22:00
categories: python
tags:
- 爬虫
- scrapy 
---

学习 scrapy 基本命令之后，开始学习 scrapy 的高级用法。包括如何机构化你爬取的数据, 怎样编写你的 spider 文件, spider 的一些属性和方法, xpath 的基础以及怎样进行传递参数运行 spider 文件。 

<!--more-->

### scrapy 进阶

* items.py: 存储爬取的数据，将非结构化数据转成结构化数据

```python
>>> import scrapy
>>> class person(scrapy.Item):
...     name = scrapy.Field()
...     job = scrapy.Field()
...     email = scrapy.Field()
...
>>> ljm = person(name='ljming', job='coder', email='xxxxxxx@qq.com')
>>> print(ljm)
{'email': 'xxxxxxx@qq.com', 'job': 'coder', 'name': 'ljming'}
>>> print(ljm.keys())
dict_keys(['name', 'job', 'email'])
>>> print(ljm.items())
ItemsView({'email': 'xxxxxxx@qq.com', 'job': 'coder', 'name': 'ljming'})

```

* Spider 的编写

在项目中可以使用 genspider 命令创建一个爬虫文件，然后我们对它来进行修改。

```python

# -*- coding: utf-8 -*-
import scrapy


class LjmSpider(scrapy.Spider):  # 继承 scrapy.Spider
    name = 'ljm'  # 代表爬虫名称
    allowed_domains = ['baidu.com']  # 表示允许爬取的域名
    start_urls = ['http://baidu.com/']  # 表示开始爬取的网址


    '''
    parse 方法，如果没有特别指定回调函数，该方法是处理 Scrapy 
    爬虫爬行到的网页响应的默认方法。通过该方法可以e对响应进行
    处理并返回处理后的数据，同时该方法也负责链接的跟进。
    '''
    def parse(self, response):
        pass

```

* spider 的其它一些属性和方法


名称 | 属性 or 方法 | 含义
---|--- |---
start_requests() | 方法 | 该方法会默认读取 start_urls 属性（当然可以是自定义）中定义的网址，为每一个网址生成一个 Request 请求对象，并返回可迭代对象。
make_requests_from_url(url) | 方法 | 该方法会被 start_requests() 调用，该方法负责实现生成 Request 请求对象
closed(reason) | 方法 | 关闭 Spider时， 该方法会被调用
log(message[, level, component]) | 方法 | 使用该方法可以实现在 Spider 中添加 log
__init__() | 方法 | 该方法主要负责爬虫的初始化，为构造函数


##### items.py

```python

# -*- coding: utf-8 -*-

# Define here the models for your scraped items
#
# See documentation in:
# https://doc.scrapy.org/en/latest/topics/items.html

import scrapy


class ScrapylearnItem(scrapy.Item):
    # define the fields for your item here like:
    # name = scrapy.Field()
    pass

class MySpiderItem(scrapy.Item):
	url_name = scrapy.Field()
	url_key = scrapy.Field()
	url_cr = scrapy.Field()
	url_addr = scrapy.Field()

```

##### ljm.py
```python

# -*- coding: utf-8 -*-

import scrapy

from scrapyLearn.items import MySpiderItem

class LjmSpider(scrapy.Spider):
    name = 'ljm'
    allowed_domains = ['baidu.com']
    start_urls = [
        'https://mbd.baidu.com/newspage/data/landingsuper?context=%7B%22nid%22%3A%22news_15501735110847759526%22%7D&n_type=0&p_from=1',
        'https://mbd.baidu.com/newspage/data/landingsuper?context=%7B%22nid%22%3A%22news_17413885825184600007%22%7D&n_type=0&p_from=1'
    ]

    def parse(self, response):
        item = MySpiderItem()
        # print(response.text)
        item['url_name'] = response.xpath("/html/head/title/text()")
        print('url_name:', item['url_name'])

```

执行 `scrapy crawl [spider_name]` 运行此程序


### XPath 基础

XPath 是一种 XML 路径语言， 通过改语言可以再 XML 文档中迅速地查找到相应的信息。 XPath 表达式通常叫做 XPath selector。

想获取标签中的文本信息，可以通过 text() 实现。例如：`/html/body/h2/text()`

可以使用 `//` 提取某个标签的所有信息。例如：`//p`

如果想获取所有属性 `X` 的值为 `Y` 的 `<Z>` 标签的内容，可以通过 `//Z[@X='Y']` 的方式获取。 例如： `//img[@class='f1']`

### Spider 类参数传递

在 Spider 类中可以通过 `-a` 选项实现参数的传递。

首先，在 Spider 文件中重写 `__init__` 方法, 在构造方法中设置一个变量用于接受用户在执行爬虫文件时传递过来的参数。

```python
# -*- coding: utf-8 -*-

import scrapy

from scrapyLearn.items import MySpiderItem


class LjmSpider(scrapy.Spider):
    name = 'ljm'
    allowed_domains = ['baidu.com']
    start_urls = [
        'https://mbd.baidu.com/newspage/data/landingsuper?context=%7B%22nid%22%3A%22news_15501735110847759526%22%7D&n_type=0&p_from=1',
        'https://mbd.baidu.com/newspage/data/landingsuper?context=%7B%22nid%22%3A%22news_17413885825184600007%22%7D&n_type=0&p_from=1'
    ]

    # 实现参数传递
    def __init__(self, myurl=None, *args, **kwargs):
        super(LjmSpider, self).__init__(*args, **kwargs)
        print('要爬取的网页地址: ', myurl)

        # 重新定义 start_urls 属性，属性值为传进来的参数值
        self.start_urls = [myurl]

    def parse(self, response):
        item = MySpiderItem()
        item['url_name'] = response.xpath("/html/head/title/text()")
        print('url_name:', item['url_name'])

```

运行： `scrapy crawl ljm -a myurl=http://www.baidu.com`

**如要传递多个url， 可以使用某个符号将每个url分隔，然后再后台接收后，使用 split 方法进行分隔**
