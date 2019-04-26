---
title: python 爬虫实战之-google related search
date: 2018-01-15 20:30:00
categories: Python
tags:
- 爬虫
---

当你使用 google 搜索某个关键词是，在页面底部会有 `search related to xxx` 模块，这里就实现爬取这个模块给出的关键字。

<!--more-->

**{% post_link python爬虫实战-百度搜索联想词抓取 python爬虫实战-百度搜索联想词抓取 %}**

```python
import random
import re
import time
import urllib

import requests

brs_col_pattern = '<div class="brs_col">(.*?)</div>'
_e4b_pattern = '<p class="_e4b">(.*?)</p>'
keyword_pattern = '<a .*?>(.*?)</a>'
base_url = 'https://www.google.com/search?hl=en&q='

# 网上找的免费代理
PROXIES = [
    "https://0.0.0.0:0",
    "https://0.0.0.0:0",
    "https://0.0.0.0:0",
    "https://0.0.0.0:0",
    "https://0.0.0.0:0"
]

# 根据正则表达式取出关键字
def parse_html(html_content):
    card_section = re.compile(brs_col_pattern).findall(html_content)
    target_keyword_list = []
    for section in card_section:
        print('section: ', section)
        p_content = re.compile(_e4b_pattern).findall(section)
        for p in p_content:
            print('p: ', p)
            target_keyword = re.compile(keyword_pattern).findall(p)[0]
            target_keyword_list.append(target_keyword.replace('<b>', '').replace('</b>', ''))

    print(target_keyword_list)
    return target_keyword_list

# 使用代理
def use_proxy(proxy_addr, url, header):
    s = requests.session()
    html = s.get(url=url, proxies={"http": proxy_addr}, headers=header)
    return html

# 构建请求的 header 信息
def get_header():
    return {
        'User-Agent': 'Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:25.0) Gecko/20100101 Firefox/25.0',
        'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8',
        'Accept-Encoding': 'gzip, deflate, br',
        'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8',
        'referer': 'https://www.google.com/'
    }

# 随机获取一个代理服务器
def get_random(array):
    return array[random.randrange(0, len(array))]


def request_google_search(source_keyword_list):
    total_keyword = []
    for keyword in source_keyword_list:
        for i in range(1, 4):
            keyword_code = urllib.request.quote(keyword)
            search_url = base_url + keyword_code + '&start=' + str(i * 10)
            print(search_url)
            flag = True
            total_time = 1
            while flag:
                proxy_ = get_random(PROXIES)
                print('本次代理IP: ', proxy_)
                header = get_header()
                print(header)

                try:
                    html = use_proxy(proxy_, search_url, header)  # 请求回来的 html 页面代码
                    if html.status_code == 200:
                        flag = False
                except urllib.request.URLError as e:
                    if hasattr(e, 'code'):
                        print(e.code)
                    if hasattr(e, 'reason'):
                        print(e.reason)
                    print('代理 IP 无法访问，先休眠5s, 再换个 IP 尝试')
                    time.sleep(5)
                    total_time += 1
                except Exception as e:
                    print('程序出现异常: ', str(e))
                    exit(-1)

                if total_time > 10:
                    print('多次尝试使用代理失败，程序退出...')
                    exit(-2)

            keyword_list = parse_html(html.text)
            total_keyword.extend(keyword_list)

    target_keyword_set = set(total_keyword)
    total_keyword = list(target_keyword_set)
    return total_keyword


if __name__ == '__main__':
    source_keyword_list = ['java', 'python', 'scala']
    result = request_google_search(source_keyword_list)
    print(result)

```