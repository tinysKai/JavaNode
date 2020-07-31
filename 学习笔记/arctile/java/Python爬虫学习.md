## Python爬虫学习

### urllib

```python
#  学习URL相关的库 urlLib爬虫库

from urllib import request  # 必须得这么引入,因为urllib属于模块request中的一个模块
from urllib import parse    # 引入解析库

print("=======get请求========")

url = "https://www.baidu.com"
response = request.urlopen(url, timeout=3)
# 获取网页返回内容,以utf-8编码解码
print(response.read().decode("utf-8"))


print("========post请求===============")
data = bytes(parse.urlencode({"world": "hello"}), encoding="utf-8")
response2 = request.urlopen("http://httpbin.org/post", data=data,timeout=3)
print(response2.read().decode("utf-8"))


```

### 携带header信息

```python
# 主要学习携带header来发起请求

from urllib import request, parse

url = 'http://httpbin.org/post'

headers = {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",
    "Accept-Encoding": "gzip, deflate, sdch",
    "Accept-Language": "zh-CN,zh;q=0.8",
    "Connection": "close",
    "Cookie": "_gauges_unique_hour=1; _gauges_unique_day=1; _gauges_unique_month=1; _gauges_unique_year=1; _gauges_unique=1",
    "Referer": "http://httpbin.org/",
    "Upgrade-Insecure-Requests": "1",
    "User-Agent": "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/57.0.2987.98 Safari/537.36 LBBROWSER"
}

dictHeader = {
    'name': 'value'
}

data = bytes(parse.urlencode(dictHeader), encoding='utf8')
req = request.Request(url=url, data=data, headers=headers, method='POST')
response = request.urlopen(req)
print(response.read().decode('utf-8'))

```

### request库

```python
# 学习request库,引入request库使http请求容易操作一些

# get请求
import requests
url = 'http://httpbin.org/get'
data = {'key': 'value', 'abc': 'xyz'}
# .get是使用get方式请求url，字典类型的data不用进行额外处理
response = requests.get(url, data)
print(response.text)


print("========开始post请求=======")

# post请求
url = 'http://httpbin.org/post'
data = {'key': 'value', 'abc': 'xyz'}
# .post表示为post方法
response = requests.post(url, data)
# 返回类型为json格式
print(response.json())

```

```python
# http请求配合正则来抓取内容

import requests
import re

content = requests.get('http://www.cnu.cc/discoveryPage/hot-人像').text
# print(content)

# 抓取的内容如下
# <a href = "http://www.cnu.cc/works/387303" class ="thumbnail" target="_blank" >
# < div class ="title" >
#     "ANN"
# </ div>


# 简化后的正则表达式为 : < a href="(.*?)".*?title">(.*?)</div>
# "re.S"叫做单行模式，简单来说，就是你用正则要匹配的内容在多行里，会增加要匹配的难度，这时候使用"re.S"把每行最后的换行符\n当做正常的一个字符串来进行匹配的一种小技巧
pattern = re.compile(r'<a href="(.*?)".*?title">(.*?)</div>', re.S)
# 使用findAll来匹配查找,会以元组方式来得到结果如:  ('http://www.cnu.cc/selectedPage', '\n                            果子\n                            ')
results = re.findall(pattern, content)
# print(results)


for result in results:
    # 将数组中的每个元祖拆分为两个值分别赋予给url以及name
    url, name = result
    # 处理name的空格以及换行问题,"\s"表示空格和换行,这里是将其全部替换为空串
    print(url, re.sub('\s', '', name))

```

### soup

```python
# 学习爬虫工具 BeautifulSoup
# 安装工具库命令 : pip3 install bs4
# 安装第二个工具库 : pip3 install lxml

html_doc = """
<html><head><title>The Dormouse's story</title></head>
<body>
<p class="title"><b>The Dormouse's story</b></p>
<p class="story">Once upon a time there were three little sisters; and their names were
<a href="http://example.com/elsie" class="sister" id="link1">Elsie</a>,
<a href="http://example.com/lacie" class="sister" id="link2">Lacie</a> and
<a href="http://example.com/tillie" class="sister" id="link3">Tillie</a>;
and they lived at the bottom of a well.</p>
<p class="story">...</p>
"""

from bs4 import BeautifulSoup

soup = BeautifulSoup(html_doc, 'lxml')

#print(soup.prettify())

# # 找到title标签
print("============找到title标签============")
print(soup.title)

# # title 标签里的内容
print("==============title 标签里的内容==========")
print(soup.title.string)


# # 找到p标签
#print(soup.p)
#
# # 找到p标签class的名字
print("============找到p标签class的名字===========")
print(soup.p['class'])
#
# # 找到第一个a标签
print("===============找到第一个a标签==========")
print(soup.a)
#
# # 找到所有的a标签
print("================找到所有的a标签=============")
print(soup.find_all('a'))


# # 找到id为link3的的标签
print("================找到id为link3的的标签=============")
print(soup.find(id="link3"))

# # 找到所有<a>标签的链接
print("=================找到所有<a>标签的链接===========")
for link in soup.find_all('a'):
    print(link.get('href'))

# # 找到文档中所有的文本内容
# print(soup.get_text())

```

```python
# 学习使用soup来爬取infoQ的信息

from bs4 import BeautifulSoup
import requests

headers = {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",
    "Accept-Language": "zh-CN,zh;q=0.8",
    "Connection": "close",
    "Cookie": "_gauges_unique_hour=1; _gauges_unique_day=1; _gauges_unique_month=1; _gauges_unique_year=1; _gauges_unique=1",
    "Referer": "http://www.infoq.com",
    "Upgrade-Insecure-Requests": "1",
    "User-Agent": "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/57.0.2987.98 Safari/537.36 LBBROWSER"
}

url = 'https://www.infoq.com/news/'


# 取得网页完整内容
def craw(url):
    response = requests.get(url, headers=headers)
    print(response.text)

# craw(url)


# 取得新闻标题
def craw2(url):
    # 获取返回体
    response = requests.get(url, headers=headers)
    # 使用soup来解析返回报文
    soup = BeautifulSoup(response.text, 'lxml')

    # 抓取"<div class="items__content"></div>"中的带title标签的链接
    for title_href in soup.find_all('div', class_='items__content'):
        # 如果链接包含title(默认可省略,同文本内容),则获取其链接
        hrefLinks = ([title.get('href') for title in title_href.find_all('a') if title.get('title')])
        for href in hrefLinks:
            print(href)
#craw2(url)


# 翻页,大于15小于46,以15为步长来循环
for i in range(15, 46, 15):
    url = 'http://www.infoq.com/news/' + str(i)
    craw2(url)

```

### 图片爬取

```python
from bs4 import BeautifulSoup
import requests
import os
import shutil

headers = {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",
    "Accept-Language": "zh-CN,zh;q=0.8",
    "Connection": "close",
    "Cookie": "_gauges_unique_hour=1; _gauges_unique_day=1; _gauges_unique_month=1; _gauges_unique_year=1; _gauges_unique=1",
    "Referer": "http://www.infoq.com",
    "Upgrade-Insecure-Requests": "1",
    "User-Agent": "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/57.0.2987.98 Safari/537.36 LBBROWSER"
}

url = 'http://www.infoq.com/presentations'


# 下载图片
# Requests 库封装复杂的接口，提供更人性化的 HTTP 客户端，但不直接提供下载文件的函数。
# 需要通过为请求设置特殊参数 stream 来实现。当 stream 设为 True 时，
# 上述请求只下载HTTP响应头，并保持连接处于打开状态，
# 直到访问 Response.content 属性时才开始下载响应主体内容


def download_jpg(image_url, image_localpath):
    # 下载文件所以使用流模式,这里stream为true
    response = requests.get(image_url, stream=True)
    # 判断http响应状态
    if response.status_code == 200:
        # 准备写文件
        with open(image_localpath, 'wb') as f:
            response.raw.deconde_content = True
            # 使用shutil来复制文件
            shutil.copyfileobj(response.raw, f)


# 取得演讲图片
def craw3(url):
    response = requests.get(url, headers=headers)
    soup = BeautifulSoup(response.text, 'lxml')
    for pic_href in soup.find_all('div', class_='items__content'):
        # 抓取所有的img标签
        for pic in pic_href.find_all('img'):
            # 获取其图片路径
            imgurl = pic.get('src')
            dir = os.path.abspath('../pic')
            # 获取文件名(basename方法能帮我们提取到链接中的文件名)
            filename = os.path.basename(imgurl)
            # 拼装本地路径
            imgpath = os.path.join(dir, filename)
            print('开始下载 %s' % imgurl)
            download_jpg(imgurl, imgpath)


craw3(url)
```

