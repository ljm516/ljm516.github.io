---
title: python 爬虫学习 part1-urllib 库的使用
date: 2018-01-03 22:00:00
categories: Python
tags:
- 爬虫
---

学习 python 爬虫，我是认真的。跟着书本走吧《精通 Python 网络爬虫》

<!--more--> 

**{% post_link python爬虫_part2_正则 python爬虫_part2学习正则 %}**

### 入门

- 一个简单的例子
```python
import urllib.request

file = urllib.request.urlopen(url)
data = file.read() # 读取全部数据
dataline = file.readline() # 读取一行数据
print(data)

with open(filepath, 'wb') as fhandle:
    fhandle.write(data)
    
```


- 直接将对应的网页信息写入文件

`urllib.request.urlretrieve('url', 'filepath')`

- urlretrieve 执行过程中会产生一些缓存，可以使用 urlcleanup() 进行清除。

`urllib.request.urlcleanup()`

- 获取爬取网页信息

```python
file.info()

out: <http.client.HTTPMessage object at 0x00000208350B9278>
```
- 获取爬取网页状态码

```python
file.getcode()

out: 200
```

- 获取爬取网页的url

```python
file.geturl()

out：http://www.baidu.com
```

由于 url 标准中只会允许一部分 ASCII 字符，但是比如汉字，或者 `:`, `&` 等不符合标准的字符时，需要对 url 进行编码。可以使用 urllib.request.quote() 进行编码。

```python
urllib.request.quote('http://www.sina.com.cn')

out: http%3A//www.sina.com.cn
```

相应的，如果要对 url 进行解码，应该怎么办？ 可使用 urllib.request.unquote()

```python
urllib.request.unquote('http%3A//www.sina.com.cn')

out: 'http://www.sina.com.cn'
```

### 模拟浏览器

- 方法一： 使用 build_opener() 修改报头
 
```python
import urllib.request

url = 'www.baidu.com'
headers = ('Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.84 Safari/537.36')
opener = urllib.request.build_opener()  # 创建自定义的 opener 对象
opener.addheaders = [headers]
data = opener.open(url).read

```

- 方法二: 使用 add_header() 添加报头

```python
import urllib.request

url = 'http://www.baidu.com'
req = urllib.request.Request(url)  # 创建一个 Request 对象
req.add_header('Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.84 Safari/537.36')
data = urllib.request.urlopen(req).read()
    
```

### 请求超时设置

```python
import urllib.request

for i in range(100):
    
    try:
        file = urllib.request.urlopen('www.baidu.com', timeout=1)
        data = file.read()
        print(len(data))
    except Exception as e:
        print('exception: ' + str(e))

```

### url 包含中文问题

```python
import urllib.request

keyword = '李江明'
key_code = urllib.reqeust.quote(keyword)
url = 'http://www.baidu.com/s?wd=' + key_code
req = urllib.request.Request(url)
data = urllib.request.urlopen(req).read()

```

==以上内容 2018-01-03 更新==

### 提交 POST 请求

```python
>>> import urllib.request
>>> import urllib.parse
>>> url = 'http://www.iqianyue.com/mypost'
>>> postdata = {'name': 'test', 'pass': 'test'}
>>> data = urllib.parse.urlencode(postdata).encode('utf-8')
>>> req = urllib.request.Request(url, data)
>>> req.add_header('User-Agent', 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.84 Safari/537.36')
>>> html = urllib.request.urlopen(req).read()
>>> print(html)
b'<html>\r\n<head>\r\n<title>Post Test Page</title>\r\n</head>\r\n\r\n<body>\r\n<form action="" method="post">\r\nname:<input name="name" type="text" /><br>\r\npasswd:<input name="pass" type="text" /><br>\r\n<input name="" type="submit" value="submit" />\r\n<br />\r\nyou input name is:test<br>you input passwd is:test</body>\r\n</html>'

```

### 代理服务器的设置

[选择代理服务器](http://www.xicidaili.com/)

```python
>>> def use_proxy(proxy_addr, url):
...     proxy = urllib.request.ProxyHandler({'http': proxy_addr})
...     opener = urllib.request.build_opener(proxy, urllib.request.HTTPHandler)
...     urllib.request.install_opener(opener)
...     data = urllib.request.urlopen(url).read().decode('utf-8')
...     return data
...
>>> proxy_add = '119.134.115.55:3128'
>>> data = use_proxy(proxy_addr, 'http://www.baidu.com')
>>> print(len(data))
112103

```

### 开启 debug 模式

```python
>>> httphd = urllib.request.HTTPHandler(debuglevel=1)
>>> httpshd = urllib.request.HTTPSHandler(debuglevel=1)
>>> opener = urllib.request.build_opener(httphd, httpshd)
>>> urllib.request.install_opener(opener)
>>> data = urllib.request.urlopen('http://edu.51cto.com')

send: b'GET / HTTP/1.1\r\nAccept-Encoding: identity\r\nHost: edu.51cto.com\r\nUser-Agent: Python-urllib/3.6\r\nConnection: close\r\n\r\n'
reply: 'HTTP/1.1 200 OK\r\n'
header: Date header: Content-Type header: Transfer-Encoding header: Connection header: Set-Cookie header: Server header: Vary header: Vary header: X-Powered-By header: Set-Cookie header: Set-Cookie header: Set-Cookie header: Load-Balancing header: Load-Balancing 
>>>

```

### 异常处理 URLError

```python
>>> import urllib.error
>>> try:
...     urllib.request.urlopen('http://blog.csdn.net.ccc')
... except urllib.error.HTTPError as e:
...     print(e.code)
...     print(e.reason)
... except urllib.error.URLError as e:
...     print(e.reason)
...

```



==以上内容 2018-01-04 更新==






