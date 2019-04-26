---
title: python爬虫_part5_scrapy框架学习之处理不同格式文件
date: 2018-02-22 22:00
categories: Python
tags:
- 爬虫
- scrapy 
---

使用 XMLFeedSpider 处理 xml，使用 CSVFeedSpider 处理 csv。

<!--more-->

**{% post_link python爬虫-part6-scrapy架构学习 python爬虫-part6-scrapy架构学习 %}**

###  用 XMLFeedSpider 分析 XML 源

如果想用 Scrapy 来处理 XML 文件，可以使用 XMLFeedSpider 去实现。

首先运行 `scrapy startproject myxml` 创建一个分析 XML 文件的项目

然后运行 `scrapy genspider -t xmlfeed myxmlspider baidu.com` 生成分析 XML 源的 spider 文件。


myxmlspider.xml：

```python
# -*- coding: utf-8 -*-
from scrapy.spiders import XMLFeedSpider


class MyxmlspiderSpider(XMLFeedSpider):
    name = 'myxmlspider'
    allowed_domains = ['sina.com.cn']
    start_urls = ['http://sina.com.cn/feed.xml']
    iterator = 'iternodes' # you can change this; see the docs
    itertag = 'item' # change it accordingly

    def parse_node(self, response, selector):
        i = {}
        #i['url'] = selector.select('url').extract()
        #i['name'] = selector.select('name').extract()
        #i['description'] = selector.select('description').extract()
        return i
```

items.py

```python
# -*- coding: utf-8 -*-

# Define here the models for your scraped items
#
# See documentation in:
# https://doc.scrapy.org/en/latest/topics/items.html

import scrapy


class MyxmlItem(scrapy.Item):
    # define the fields for your item here like:
    name = scrapy.Field()
    title = scrapy.Field()
    link = scrapy.Field()
    author = scrapy.Field()

```

代码分析：

* iterator: 使用哪个迭代器，默认为 `iternodes`, 还有其它诸如 `html`, `xml` 的迭代器。
* itertag: 主要用来设置开始迭代的节点
* parse_node： 该方法在节点与所提供的标签名相符合的时候会被调用。在方法中，可以进行一些信息的提取和处理操作。

除此之外， XMLFeedSpider 还有一些常见的属性和方法


名称 | 属性 or 方法 | 含义
---|---|---|
namespaces | 属性 | 以列表的形式存在，主要定义在文档中会被 spider 处理的可用命名空间
adapt_response | 方法 | 该方法主要在 spider 分析响应 （Response）前被调用
process_results(respon, results) | 方法 | 该方法主要在 spider 返回结果时被调用，主要对结果在返回前进行最后处理

接下来提取想要的信息：

```python
# -*- coding: utf-8 -*-
from scrapy.spiders import XMLFeedSpider
from myxml.items import MyxmlItem

class MyxmlspiderSpider(XMLFeedSpider):
    name = 'myxmlspider'
    allowed_domains = ['sina.com.cn']
    # xml 源地址
    start_urls = ['http://blog.sina.com.cn/rss/1615888477.xml']
    
    iterator = 'iternodes' # you can change this; see the docs
    
    # 开始迭代的节点
    itertag = 'rss' # change it accordingly

    def parse_node(self, response, selector):
        i = MyxmlItem()

        # 使用 Xpath 表达式将对应信息提取处理，并存储到对应的 item 中
        i['title'] = selector.select('/rss/channel/item/title/text()').extract()
        i['link'] = selector.select('/rss/channel/item/link/text()').extract()
        i['author'] = selector.select('/rss/channel/item/author/text()').extract()

        # 通过 for 循环以此遍历提取出来存在 item 中的信息并输出
        for j in range(len(i['title'])):
            print('第' + str(j+1) + '篇文章')
            print('标题:', i['title'][j])
            print('对应的链接: ', i['link'][j])
            print('对应的作者: ', i['author'][j])

            print('---------------------------------------')
        return i

```

### 学会使用 CSVFeedSpider

* scrapy startproject mycsv
生成 mycsv 的处理 csv 文件的项目

* scrapy genspider -t csvfeed mycsvspider iqianyue.com
以 csvfedd 为模板，生成 mycsvfeedspider.py, 指定 allowed_domains 为 iqianyue.com

* items.py

```python
# -*- coding: utf-8 -*-

# Define here the models for your scraped items
#
# See documentation in:
# https://doc.scrapy.org/en/latest/topics/items.html

import scrapy


class MycsvItem(scrapy.Item):
    # define the fields for your item here like:
    # name = scrapy.Field()
    name = scrapy.Field()
    sex = scrapy.Field()

```

* mycsvspider.py

```python

# -*- coding: utf-8 -*-
from scrapy.spiders import CSVFeedSpider
from mycsv.items import MycsvItem


class MycsvspiderSpider(CSVFeedSpider):
    name = 'mycsvspider'
    allowed_domains = ['iqianyue.com']

    # 定义要处理的 csv 文件所在地址
    start_urls = ['http://yum.iqianyue.com/weisuenbook/pyspd/part12/mydata.csv']
    # 定义 header
    headers = ['name', 'sex', 'addr', 'email']
    # 定义 分隔符
    delimiter = ','

    # Do any adaptations you need here
    #def adapt_response(self, response):
    #    return response

    def parse_row(self, response, row):
        i = MycsvItem()
        
        i['name'] = row['name'].encode()
        i['sex'] = row['sex'].encode()

        print('name: ', i['name'], ' sex: ', i['sex'])
        print('---------------------------------------')
        return i
```
