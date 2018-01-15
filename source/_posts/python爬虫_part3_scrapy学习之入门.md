---
title: python爬虫_part3_scrapy框架学习之入门
date: 2018-01-11 23:40
categories: python
tags:
- 爬虫
- scrapy 
---

开启爬虫框架 scrapy 学习之旅

<!--more-->

* 安装

`pip install scrapy`

#### 全局命令

`scrapy -h` 查看有哪些全局命令

```
scrapy -h

Scrapy 1.5.0 - no active project

Usage:
  scrapy <command> [options] [args]

Available commands:
  bench         Run quick benchmark test
  fetch         Fetch a URL using the Scrapy downloader
  genspider     Generate new spider using pre-defined templates
  runspider     Run a self-contained spider (without creating a project)
  settings      Get settings values
  shell         Interactive scraping console
  startproject  Create new project
  version       Print Scrapy version
  view          Open URL in browser, as seen by Scrapy

  [ more ]      More commands available when run from project directory

Use "scrapy <command> -h" to see more info about a command

```

* fetch 命令

主要用来显示爬虫爬取的过程

一些参数
```
scrapy fetch --headers: 控制显示对应的爬虫爬取网页时候的头信息
scrapy fetch --nolog: 控制不显示日志信息
scrapy fetch --spider=[spiderName]: 控制使用哪个爬虫
scrapy fetch --logfile=[filePath]: 指定存储日志信息的文件
scrapy fetch --loglevel=[level]： 控制日志等级

```

* runspider 命令

通过该命令可以实现不依托 scrapy 的爬虫项目，直接运行一个爬虫文件（首先要写好一个 scrapy 爬虫文件）

```python

from scrapy.spiders import Spider

class FirstSpider(Spider):
	name = 'first'
	allowed_domains = ['baidu.com']
	start_urls = [
		'http://www.baidu.com'
	]

	def parse(self, response):
		pass

```

执行 `scrapy runspider [.py]`

* settings 命令

该命令用来查看 scrapy 的配置信息

`scrapy settings --get [conf_name]`

* shell 命令

启动 scrapy 的交互终端

```python
scrapy shell http://www.baidu.com

2018-01-11 22:54:53 [scrapy.utils.log] INFO: Scrapy 1.5.0 started (bot: scrapybot)
2018-01-11 22:54:53 [scrapy.utils.log] INFO: Versions: lxml 4.1.1.0, libxml2 2.9.5, cssselect 1.0.3, parsel 1.3.1, w3lib 1.18.0, Twisted 17.9.0, Python 3.6.3 (v3.6.3:2c5fed8, Oct  3 2017, 18:11:49) [MSC v.1900 64 bit (AMD64)], pyOpenSSL 17.5.0 (OpenSSL 1.1.0g  2 Nov 2017), cryptography 2.1.4, Platform Windows-10-10.0.16299-SP0
2018-01-11 22:54:53 [scrapy.crawler] INFO: Overridden settings: {'DUPEFILTER_CLASS': 'scrapy.dupefilters.BaseDupeFilter', 'LOGSTATS_INTERVAL': 0}
2018-01-11 22:54:53 [scrapy.middleware] INFO: Enabled extensions:
['scrapy.extensions.corestats.CoreStats',
 'scrapy.extensions.telnet.TelnetConsole']
2018-01-11 22:54:53 [scrapy.middleware] INFO: Enabled downloader middlewares:
['scrapy.downloadermiddlewares.httpauth.HttpAuthMiddleware',
 'scrapy.downloadermiddlewares.downloadtimeout.DownloadTimeoutMiddleware',
 'scrapy.downloadermiddlewares.defaultheaders.DefaultHeadersMiddleware',
 'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware',
 'scrapy.downloadermiddlewares.retry.RetryMiddleware',
 'scrapy.downloadermiddlewares.redirect.MetaRefreshMiddleware',
 'scrapy.downloadermiddlewares.httpcompression.HttpCompressionMiddleware',
 'scrapy.downloadermiddlewares.redirect.RedirectMiddleware',
 'scrapy.downloadermiddlewares.cookies.CookiesMiddleware',
 'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware',
 'scrapy.downloadermiddlewares.stats.DownloaderStats']
2018-01-11 22:54:53 [scrapy.middleware] INFO: Enabled spider middlewares:
['scrapy.spidermiddlewares.httperror.HttpErrorMiddleware',
 'scrapy.spidermiddlewares.offsite.OffsiteMiddleware',
 'scrapy.spidermiddlewares.referer.RefererMiddleware',
 'scrapy.spidermiddlewares.urllength.UrlLengthMiddleware',
 'scrapy.spidermiddlewares.depth.DepthMiddleware']
2018-01-11 22:54:53 [scrapy.middleware] INFO: Enabled item pipelines:
[]
2018-01-11 22:54:53 [scrapy.extensions.telnet] DEBUG: Telnet console listening on 127.0.0.1:6023
2018-01-11 22:54:53 [scrapy.core.engine] INFO: Spider opened
2018-01-11 22:54:54 [scrapy.core.engine] DEBUG: Crawled (200) <GET http://www.baidu.com> (referer: None)
2018-01-11 22:54:55 [traitlets] DEBUG: Using default logger
2018-01-11 22:54:55 [traitlets] DEBUG: Using default logger
[s] Available Scrapy objects:
[s]   scrapy     scrapy module (contains scrapy.Request, scrapy.Selector, etc)
[s]   crawler    <scrapy.crawler.Crawler object at 0x0000018E991AAC88>
[s]   item       {}
[s]   request    <GET http://www.baidu.com>
[s]   response   <200 http://www.baidu.com>
[s]   settings   <scrapy.settings.Settings object at 0x0000018E9B881CC0>
[s]   spider     <DefaultSpider 'default' at 0x18e9bb08550>
[s] Useful shortcuts:
[s]   fetch(url[, redirect=True]) Fetch URL and update local objects (by default, redirects are followed)
[s]   fetch(req)                  Fetch a scrapy.Request and update local objects
[s]   shelp()           Shell help (print this help)
[s]   view(response)    View response in a browser
In [1]: title = response.xpath('/html/head/title')
In [2]: print(title)
[<Selector xpath='/html/head/title' data='<title>百度一下，你就知道</title>'>]
In [3]: exit()  # 退出
```

* startproject 命令

用于创建项目 `scrapy startprject [project_name]`

* version 命令

用于显示 scrapy 版本信息 `scrapy verion` or `scrapy -v`

* view 命令

实现下载某个页面并用浏览器查看的功能。

```python
scrapy view http://news.sina.com.cn

2018-01-11 23:03:15 [scrapy.utils.log] INFO: Scrapy 1.5.0 started (bot: scrapybot)
2018-01-11 23:03:15 [scrapy.utils.log] INFO: Versions: lxml 4.1.1.0, libxml2 2.9.5, cssselect 1.0.3, parsel 1.3.1, w3lib 1.18.0, Twisted 17.9.0, Python 3.6.3 (v3.6.3:2c5fed8, Oct  3 2017, 18:11:49) [MSC v.1900 64 bit (AMD64)], pyOpenSSL 17.5.0 (OpenSSL 1.1.0g  2 Nov 2017), cryptography 2.1.4, Platform Windows-10-10.0.16299-SP0
2018-01-11 23:03:15 [scrapy.crawler] INFO: Overridden settings: {}
2018-01-11 23:03:15 [scrapy.middleware] INFO: Enabled extensions:
['scrapy.extensions.corestats.CoreStats',
 'scrapy.extensions.telnet.TelnetConsole',
 'scrapy.extensions.logstats.LogStats']
2018-01-11 23:03:16 [scrapy.middleware] INFO: Enabled downloader middlewares:
['scrapy.downloadermiddlewares.httpauth.HttpAuthMiddleware',
 'scrapy.downloadermiddlewares.downloadtimeout.DownloadTimeoutMiddleware',
 'scrapy.downloadermiddlewares.defaultheaders.DefaultHeadersMiddleware',
 'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware',
 'scrapy.downloadermiddlewares.retry.RetryMiddleware',
 'scrapy.downloadermiddlewares.redirect.MetaRefreshMiddleware',
 'scrapy.downloadermiddlewares.httpcompression.HttpCompressionMiddleware',
 'scrapy.downloadermiddlewares.redirect.RedirectMiddleware',
 'scrapy.downloadermiddlewares.cookies.CookiesMiddleware',
 'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware',
 'scrapy.downloadermiddlewares.stats.DownloaderStats']
2018-01-11 23:03:16 [scrapy.middleware] INFO: Enabled spider middlewares:
['scrapy.spidermiddlewares.httperror.HttpErrorMiddleware',
 'scrapy.spidermiddlewares.offsite.OffsiteMiddleware',
 'scrapy.spidermiddlewares.referer.RefererMiddleware',
 'scrapy.spidermiddlewares.urllength.UrlLengthMiddleware',
 'scrapy.spidermiddlewares.depth.DepthMiddleware']
2018-01-11 23:03:16 [scrapy.middleware] INFO: Enabled item pipelines:
[]
2018-01-11 23:03:16 [scrapy.core.engine] INFO: Spider opened
2018-01-11 23:03:16 [scrapy.extensions.logstats] INFO: Crawled 0 pages (at 0 pages/min), scraped 0 items (at 0 items/min)
2018-01-11 23:03:16 [scrapy.extensions.telnet] DEBUG: Telnet console listening on 127.0.0.1:6023
2018-01-11 23:03:25 [scrapy.core.engine] DEBUG: Crawled (200) <GET http://news.sina.com.cn> (referer: None)
2018-01-11 23:03:26 [scrapy.core.engine] INFO: Closing spider (finished)
2018-01-11 23:03:26 [scrapy.statscollectors] INFO: Dumping Scrapy stats:
{'downloader/request_bytes': 215,
 'downloader/request_count': 1,
 'downloader/request_method_count/GET': 1,
 'downloader/response_bytes': 132044,
 'downloader/response_count': 1,
 'downloader/response_status_count/200': 1,
 'finish_reason': 'finished',
 'finish_time': datetime.datetime(2018, 1, 11, 15, 3, 26, 66996),
 'log_count/DEBUG': 2,
 'log_count/INFO': 7,
 'response_received_count': 1,
 'scheduler/dequeued': 1,
 'scheduler/dequeued/memory': 1,
 'scheduler/enqueued': 1,
 'scheduler/enqueued/memory': 1,
 'start_time': datetime.datetime(2018, 1, 11, 15, 3, 16, 51050)}
2018-01-11 23:03:26 [scrapy.core.engine] INFO: Spider closed (finished)

```

#### 项目命令

bench, check, crawl, edit, genspider, list, parse 等

* bench 命令

测试本地硬件性能

* genspider 命令

创建 scrapy 文件，快速创建爬虫文件的方式。 需要在 scrapy 爬虫项目目录中，才能使用该命令。

可以使用 `scrapy genspider -l` 查看模板

```python
scrapy genspider -l

Available templates:
  basic
  crawl
  csvfeed
  xmlfeed

```
可以选取一个模板生成一个爬虫文件: `scrapy genspider basic [file_name] [url]`

查看某个模板的内容 `scrapy genspider -d [template_name]`

```python

from scrapy.spiders import CSVFeedSpider

class $classname(CSVFeedSpider):
    name = '$name'
    allowed_domains = ['$domain']
    start_urls = ['http://$domain/feed.csv']
    # headers = ['id', 'name', 'description', 'image_link']
    # delimiter = '\t'

    # Do any adaptations you need here
    #def adapt_response(self, response):
    #    return response

    def parse_row(self, response, row):
        i = {}
        #i['url'] = row['url']
        #i['name'] = row['name']
        #i['description'] = row['description']
        return i
```

* check 命令

在 scrapy 中使用合同（contract）[一种交互式的检查方式] 的方式对爬虫进行测试

格式： `scrapy check [spider_name]`  # 不是文件名，即不带 `.py` 后缀

比如：

```python
# -*- coding: utf-8 -*-
import scrapy


class LjmSpider(scrapy.Spider):
    name = 'ljm'  # 是这个 name
    allowed_domains = ['baidu.com']
    start_urls = ['http://baidu.com/']

    def parse(self, response):
        pass

>>> scrapy check ljm

----------------------------------------------------------------------
Ran 0 contracts in 0.000s

OK

```

* crawl 命令

启动某个爬虫: `scrapy crawl [spider_name]`

* list 命令

列出当前可使用的爬虫文件：`scrapy list`

* edit 命令

直接打开对应编辑器对爬虫文件进行编辑（windows）环境会出现问题，比较实用 linux 环境。

在 windows 环境打开:

```
scrapy edit ljm
'%s' 不是内部或外部命令，也不是可运行的程序
或批处理文件。

```

* parse 命令

可以实现获取指定的 url 网址，并使用对用的爬虫文件进行处理和分析。没有指定爬虫文件或处理函数，会使用默认的。

scrapy parse -h 查看参数

```
Usage
=====
  scrapy parse [options] <url>

Parse URL (using its spider) and print the results

Options
=======
--help, -h              show this help message and exit
--spider=SPIDER         use this spider without looking for one
-a NAME=VALUE           set spider argument (may be repeated)
--pipelines             process items through pipelines
--nolinks               don't show links to follow (extracted requests)
--noitems               don't show scraped items
--nocolour              avoid using pygments to colorize the output
--rules, -r             use CrawlSpider rules to discover the callback
--callback=CALLBACK, -c CALLBACK
                        use this callback for parsing, instead looking for a
                        callback
--meta=META, -m META    inject extra meta into the Request, it must be a valid
                        raw json string
--depth=DEPTH, -d DEPTH
                        maximum depth for parsing requests [default: 1]
--verbose, -v           print each depth level one by one

Global Options
--------------
--logfile=FILE          log file. if omitted stderr will be used
--loglevel=LEVEL, -L LEVEL
                        log level (default: DEBUG)
--nolog                 disable logging completely
--profile=FILE          write python cProfile stats to FILE
--pidfile=FILE          write process ID to FILE
--set=NAME=VALUE, -s NAME=VALUE
                        set/override setting (may be repeated)
--pdb                   enable pdb on failure
```

参数说明


参数 | 含义
---|---
--spider=SPIDER | 强行指定某个爬虫文件 spider 进行处理
-a Name=VALUE | 设置 spider 的参数，可能会重复
--pipelines | 通过 pipelines 来处理 items
--nolinks | 不展示提取到的链接信息
--noitems | 不展示得到的 items
--nocolour | 输出的结果颜色不高亮
--rules, -r| 使用 CrawlSpider 规则去处理回调函数
--callback=CALLBACk, -c CALLBACK | 使用 spider 中用于处理返回的响应的回调函数
--depth=DEPTH, -d DEPTH | 设置爬行深度， 默认为1
--verbose, -v | 显示每层的详细信息

如： `scrapy parse http://www.baidu.com --spider=[spider_name]`

