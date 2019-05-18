---
title: python爬虫-手机号归属地查询
date: 2019-05-18 17:53:10
categories: Python
tags:
- 爬虫
---

好久没有写python的记录，前两天工作原因，需要根据手机号前7位爬取手机号归属地以及运营商的信息，于是写了个爬虫脚本。原先只是使用requests库，进行了单线程的爬取，发现太慢了，而且会出现停顿获取不到结果的问题，后面果断使用了scrapy框架，那是相当的快，👍。

<!--more-->

## 基于requests
创建了一个python文件夹，然后创建了两个python文件，run.py, spider.py。前者是程序运行的入口，后者进行数据爬取。

- run.py
![run.py](runpy.png)
- spider.py
![spider.py](spiderpy.png)

`requests.get(url)`就是请求url，然后返回结果。需要观察response的内容，然后对内容解析，想这个响应内容就是一个json格式数据，那么就用json库序列化，就可以轻松地取出想要的数据。for 循坏是为了如果响应码不为200，进行的重试策略。timeout设置最大等待时间。

建议: 字符串的拼接不要用`+`，python中字符串和数字是不能用`+`拼接的，除非将数字转换成字符串。作为代替，可以使用format()方法，例如:
```python
'ljming{}'.format(666);
```
如果要拼接多个字符串，可以在大括号里设置一个标识，在format函数里为这个标识赋值。
```python
'ljming{age},{address}'.format(age=18, address=beijing)
```

## 使用scrapy

首先是有个scrapy命令新建一个scrapy项目`scrapy startproject [projectName]`，然后在你的运行命令的那个目录就会生成一个projectName的文件夹。

![scrapy-start-project](startproject.png)

文件夹里的每个文件是干嘛的，可以自行百度。

然后通过命令 `scrapy genspider -t basic [spiderName]`生成一个spider文件。scrapy命令可以看 **{% post_link python爬虫_part3_scrapy学习之入门 python爬虫_part3_scrapy学习之入门 %}**

- python_spider.py

```python
class PhoneSpiderSpider(scrapy.Spider):
    name = 'phone_spider'
    allowed_domains = ['mobsec-dianhua.baidu.com']
    base_url = "http://mobsec-dianhua.baidu.com/dianhua_api/open/location?tel={}"
    start_urls = []

    def __init__(self, *args, **kwargs):
        super(PhoneSpiderSpider, self).__init__(*args, **kwargs)
        phone_head = [1480000, 1650000, 1670000, 1910000, 1620000, 1700000, 1710000]

        for head in phone_head:
            stop_flag = head + 9999
            while head <= stop_flag:
                self.start_urls.append(self.base_url.format(head))
                head += 1

        print("start_urls size={}".format(len(self.start_urls)))

    def parse(self, response):
        rsp_content = response.text
        json_obj = json.loads(rsp_content)

        data_json = json_obj['response']
        rsp_header = json_obj['responseHeader']
        if rsp_header['status'] != 200:
            print("response not success")

        request_phone = response.url.split('=')[1]
        print('request_phone={}'.format(request_phone))

        item = PhoneInfoItem()
        item['phone'] = request_phone
        item['head'] = request_phone[:3]
        if data_json[str(request_phone)] is not None and data_json[request_phone]['detail'] is not None:
            item['operator'] = data_json[request_phone]['detail']['operator']
            item['province'] = data_json[request_phone]['detail']['province']
            item['city'] = data_json[request_phone]['detail']['area'][0]['city']

            print("item={}".format(item))
        else:
            item['operator'] = ''
            item['province'] = ''
            item['city'] = ''

        yield item
```

`__init__` 函数对PhoneSpiderSpider进行初始化，主要是创建了需要爬去的号码，然后将号码放进start_uls集合中。

`parse` 函数是就响应结果进行处理，主要是观察reponse的内容格式，然后进行处理。`PhoneInfoItem`这个类是`item.py`的一个类，是strat project命令自动生成的。可以根据具体情况，设置属性。

- item.py

```python
class PhoneInfoItem(scrapy.Item):
    # define the fields for your item here like:
    # name = scrapy.Field()
    phone = scrapy.Field()
    head = scrapy.Field()
    province = scrapy.Field()
    city = scrapy.Field()
    operator = scrapy.Field()
```

在parse函数中，取出要的值，然后实例化PhoneInfoItem类，最后 yield item。yield item之后pipelines.py就会对item进行处理，这里一般是将保存数据的操作（写文件or数据库）。

这里需要打开`settings.py`文件的一个注释
![item_pipelines](itempipelines.png)

- settings.py
该文件内容主要是对你的爬虫进行一些设置，默认所有的配置都是被注释掉的，如果要用打开就行。

- pipelines.py

```python
MYSQL_HOST = '127.0.0.1'
MYSQL_PORT = 3306
MYSQL_USER = 'ljm'
MYSQL_PASSWD = ''
MYSQL_DB = 'spider'

db = MySQLDatabase(MYSQL_DB, host=MYSQL_HOST, port=MYSQL_PORT, user=MYSQL_USER, passwd=MYSQL_PASSWD)


class BaseModel(Model):
    class Meta:
        database = db


class Phone_Info(BaseModel):
    id = PrimaryKeyField()
    number = CharField()
    head = CharField()
    province = CharField()
    city = CharField()
    operator = CharField()


class PhoneInfoPipeline(object):

    def process_item(self, item, spider):
        phone_info = Phone_Info(number=item['phone'], operator=item['operator'],
                                head=item['head'], province=item['province'], city=item['city'])
        phone_info.save()
        return item

```
pipelines.py 生成时只有 PhoneInfoPipeline这个类，上面的是为了数据入库用的，这里用到了python的一个orm框架-peewee，怎么用？看**{% post_link peewee入门 peewee入门 %}**。

`process_item`函数就是具体处理数据的函数，最终通过peewee保存在数据库。