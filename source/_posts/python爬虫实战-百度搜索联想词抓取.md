---
title: python 爬虫实战之-百度联想词抓取
date: 2018-04-19 14:35:00
categories: Python
tags:
- 爬虫
---

我们都有这样的经历，在百度搜索输入框输入想要搜索的词时，输入框下面会出现百度联想推荐搜索词。这里就是想抓取这部分的词。

<!--more-->

在浏览器输入 `www.baidu.com`。 打开 chrome 浏览的检查界面，选择 Network，当在百度搜索输入框中输入词时，可以看到 network 中出现了一个网络请求，这个请求会返回百度联想推荐搜索词。我们可以发现，这个请求的 url 是这样的: `https://sp0.baidu.com/5a1Fazu8AA54nxGko9WTAnF6hhy/su?wd=&json=1&p=3&sid=1428_21113_18559_20927&req=2&sc=eb&csor=0&pwd=aa&cb=jQuery110207135102943980893_1524119759664&_=1524119759667`。然后我去除不需要的部分，提取出基础的请求 url `https://sp0.baidu.com/5a1Fazu8AA54nxGko9WTAnF6hhy/su?wd=`。 这个 url 后面加上要搜索的词，就可以获取推荐词。

现在我们就编码实现：

```python
BASE_REQUEST_URL = 'https://sp0.baidu.com/5a1Fazu8AA54nxGko9WTAnF6hhy/su?wd='

# 构建请求头的部分，可以网上找些内容，这里随机获取构建请求头，避免请求次数过多被屏蔽
ACCEPT = [
    'text/html',
    'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8'
]

ACCEPT_ENCODING = [
    'gzip, deflate',
    'gzip, deflate, compress',
    'gzip,deflate,sdch'
]

ACCEPT_LANGUAGE = [
    "en-us",
    "en-uk",
    "en,de;q=0.8,de-de;q=0.5,en-us;q=0.3",
    "en,de;q=0.8,de-at;q=0.5,en-uk;q=0.3",
    "en-US,en;q=0.8"
]

USER_AGENTS = [

    "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:25.0) Gecko/20100101 Firefox/25.0",

    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/30.0.1599.101 Safari/537.36",

    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9) AppleWebKit/537.71 (KHTML, like Gecko) Version/7.0 Safari/537.71",

    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:24.0) Gecko/20100101 Firefox/24.0",

    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/30.0.1599.101 Safari/537.36",

    "Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 6.1; WOW64; Trident/6.0)",

    "Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 6.1; Trident/6.0)",

    "Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 6.1; Trident/5.0)",

    "Mozilla/4.0 (compatible; MSIE 10.0; Windows NT 6.1; Trident/5.0)",

    "Mozilla/1.22 (compatible; MSIE 10.0; Windows 3.1)"

    ]


def get_random(array):
    return array[random.randrange(0, len(array))]


def get_header():
    return {

        'User-Agent': get_random(USER_AGENTS),

        'Connection': 'Keep-Alive',

        'Accept-Language': get_random(ACCEPT_LANGUAGE),

        'Accept-Encoding': get_random(ACCEPT_ENCODING),

        'Accept': get_random(ACCEPT)

    }


def spider():
    request_header = None

    for keyword in source_keyword_list:
        request_url = BASE_REQUEST_URL + keyword

        if not request_header:
            request_header = get_header()

        rsp = requests.get(request_url, headers=request_header)
        print('request status_code: {code}'.format(code=rsp.status_code))

        text = rsp.text
        get_imagination_keyword(text=text, source_keyword=keyword)
```

`spider()` 方法请求了那个 url，例如我们搜索 NBA，那么请求的 url 将会是这样 `https://sp0.baidu.com/5a1Fazu8AA54nxGko9WTAnF6hhy/su?wd=NBA`，响应返回一个字符串，字符串内容是: `window.baidu.sug({q:"nba",p:false,s:["nba直播","nba直播吧","nba录像","nba2k online","nba2k17","nba98","nba数据","nba排名","nba历史得分榜","nba回放"]});`。 那么我们就来解析这个字符串，获取里面的词。

```python
def get_imagination_keyword(text, source_keyword):
    print('return text: {text}'.format(text=text))

    result = re.compile('(\[.*?\])').findall(text)[0]
    json_result = json.loads(result)
    print('{source keyword}, result: {r}'.format(r=json_result))
    
```

`(\[.*?\])` 这个正则就可以匹配除字符串中 `[]` 的内容，即我们想要的词。这就大功告成，相当简单。 
