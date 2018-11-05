---
title: python爬虫scrapy框架
date: 2017-09-04 20:17:00
categories: Python
tags:
- 爬虫
- scrapy
---

## 安装

```
pip install Scrapy
```

<!--more-->

## 使用教程

- 创建一个新的Scrapy项目
- 编写一个spider，爬取网站数据
- 使用命令行导出爬取的数据
- 修改spider，递归访问链接
- 使用spider参数

## 创建一个项目

```
scrapy startproject tutorial
```

文件夹目录应该是这样的

```
tutorial/
    scrapy.cfg            # deploy configuration file

    tutorial/             # project's Python module, you'll import your code from here
        __init__.py

        items.py          # project items definition file

        pipelines.py      # project pipelines file

        settings.py       # project settings file

        spiders/          # a directory where you'll later put your spiders
            __init__.py
```

下面就是创建的第一个spider

```python
import scrapy

class QuotesSpider(scrapy.Spider):
    name = 'quotes'

    def start_requests(self):
        urls = [
            'http://quotes.toscrape.com/page/1/',
            'http://quotes.toscrape.com/page/2/',
        ]
        for url in urls:
           yield scrapy.Request(url=url, callback=self.parse())

    def parse(self, response):
        page = response.url.split('/')[-2]
        filename = 'quotes-%s.html' % page
        with open(filename, 'wb') as f:
            f.write(response.body)

        self.log('Saved file %s' % filename)
```

参数和方法

- name: 定义spider，在项目中唯一
- start_requests(): 必须返回一个spider爬取可迭代的Requests，后续的请求将从这些初始化的请求中自动生成。
- parse(): 处理每个请求产生的响应的方法，参数是一个`TextResponse`对象，它保存页面内容，以及有一些其他的方法处理它。

parse()方法，通常用来解析response，提取爬出的数据，作为dicts；找到新的urls，创建新的requests

## 运行spider

在`tutorial/`目录下

```
scrapy crawl quotes # spider name
```

## 相对于`start_requests`方法的快捷方式

你可以定义一个`start_urls`的类属性，是个url的集合，去代替实现从URLs自动生成`scrapy.Request`对象的`start_requests`方法

```python
import scrapy


class QuotesSpider(scrapy.Spider):
    name = "quotes"
    start_urls = [
        'http://quotes.toscrape.com/page/1/',
        'http://quotes.toscrape.com/page/2/',
    ]

    def parse(self, response):
        page = response.url.split("/")[-2]
        filename = 'quotes-%s.html' % page
        with open(filename, 'wb') as f:
            f.write(response.body)
```

## 抓取数据

最好学习怎样抓取数据方法是：在`scrapy shell`交互下，使用选择器。

- response.css('title') 
list-like object -> SelectorList

```
[<Selector xpath=u'descendant-or-self::title' data=u'<title>Quotes to Scrape</title>'>]
```

- response.css('title::text').extract() 
获取`<title>`标签里面的内容，return a list

```
[u'Quotes to Scrape']
```

- response.css('title::text').extract_first()
获取第一个结果

```
'Quotes to Scrape'
```

- response.css('title:text').re(r'Quotes.*')
使用正则，获取匹配正则表达的内容

```
['Quotes to Scrape']
```

## Xpath简介

除了css，scrapy选择器也支持Xpath表达式

```
response.xpath('//title')
[<Selector xpath='//title' data='<title>Quotes to Scrape</title>'>]

response.xpath('//title/text()').extract_first()
'Quotes to Scrape'
```

## 抓取quotes和authors

```
scrapy shell 'http://quotes.toscrape.com'
response.css('div.quote')

quote = response.css('div.quote')[0]
title = quote.css('span.text::text').extract_first()
author = quote.css('small.author::text').extract_first()
tags = quote.css('div.tags a.tag::text').extract()

```

## 使用spider抓取数据

```python
import scrapy

class QuotesSpider(scrapy.Spider):
    name = 'quotes'

    start_urls = [
        'http://quotes.toscrape.com/page/1/',
        'http://quotes.toscrape.com/page/2/',
    ]

    def parse(self, response):
        for quote in response.css('div.quote'):
            yield {
                'text': quote.css('span.text::text').extract_first(),
                'author': quote.css('small.author::text').extract_first(),
                'tags': quote.css('div.tags a.tag::text').extract()
            }

```

## 存储已经爬取的数据

```
scrapy crawl quotes -o quotes.json
```

这条命令将自动生成quotes.json文件，但是运行两次，它会在原有的基础上直接添加，这将会产生一个格式损坏的json文件

```
scrapy crawl qutos -o quotes.jl
```
这不会产生和上面一样的问题，

## following link

```python
import scrapy

class QuotesSpider(scrapy.Spider):
    name = 'quotes'

    start_urls = [
        'http://quotes.toscrape.com/page/1/'
    ]

    def parse(self, response):
        for quote in response.css('div.quote'):
            yield {
                'text': quote.css('span.text::text').extract_first(),
                'author': quote.css('small.author::text').extract_first(),
                'tags': quote.css('div.tags a.tag::text').extract()
            }
        next_page = response.css('li.next a::attr(href)').extract_first()
        if next_page is not None:
            ## shortcut
            ## yield response.follow(next_page, callback=self.parse)
            next_page = response.urljoin(next_page)
            yield scrapy.Request(next_page, callback=self.parse())

```

## 使用spider参数

当运行下面的命令时，你可以通过`-a`可选项为spiders传一个参数：

```
scrapy crawl quotes -o quotes-humor.json -a tag=humor
```

参数被传给spider的`__init__`方法，默认成为spider的属性


在这个例子中，`tag`的值将会在`self.tag`被使用，您可以使用此方法使您的蜘蛛只获取带有特定标记的`quotes`，然后根据参数构建URL：

```python
import scrapy

class ArgumentSpider(scrapy.Spider):
    name = 'argument'

    def start_requests(self):
        url = 'http://quotes.toscrape.com/'
        tag = getattr(self, 'tag', None)
        print tag
        if tag is not None:
            url = url + 'tag/' + tag
        yield scrapy.Request(url, self.parse)

    def parse(self, response):
        for quote in response.css('div.quote'):
            yield {
                'text': quote.css('span.text::text').extract_first(),
                'author': quote.css('small.author::text').extract_first()
            }
        next_page = response.css('li.next a::attr(href)').extract_first()
        if next_page is not None:
            yield response.follow(next_page, self.parse)

```


