---
title: python 爬虫小demo
date: 2021-04-20 22:51:20
tags:
	[爬虫]
categories: Python

---

# Python 爬虫入门篇

​	看过《图解 http 》就应该会知道客户端在发起网络请求，服务器在收到请求以后，就会发送我们请求的一些数据，客户端接受以后，经过浏览器进行解析，最后可视化形成我们所浏览的网页的模样。其实所发送的所有数据都在我们的浏览器里，当我们不想机械地通过浏览器按照前端程序员所希望的模式去接收我们请求的数据时，我们就可以通过爬虫去定向的爬取我们所希望得到的数据。在互联网上的许许多多的服务器 24 小时运行着，时时刻刻等待着别人的请求，爬虫就是通过模拟请求，就好像在浏览器输入网址，通过一些 http 库向指定的服务器发起请求，这个时候爬虫可以假装自己是浏览器（通过添加一些 header ），大多是服务器就会直接返回数据给爬虫，当然一些大型的、涉及安全性的网站就会有很多反爬虫的机制，这个时候要爬数据就没那么简单了。

<!--more-->

## 一、Request 模块

### 1. 请求方法

针对 http 不同类型的请求， requests 库封装了相应的同名方法，比如 get、put、post、delete 等，对应于请求的各种参数，诸如 get、post 等方法里内嵌各种参数，比如 params，header，body 等（在 python 方法里对应于一个json格式的字典对象），而这些方法的返回值就是我们网络请求得到的返回值

### 2. 返回对象的处理

针对请求方法里返回的 response，requests 提供以下几种方式方便我们处理爬取下来的数据。

- response.text 会利用 response 的 header 中的编码方式将返回的数据流解码成一个纯文本模式
- response.content 二进制模式
- response.raw 返回成一个 urllib3 的对象
- response.json() 将返回的对象转化为json格式，在 python 中构成字典或者字典列表

### 3. 参数传递

无论是 header 还是 params 或者是 body，只需要将其写成字典对象，最后传入到请求方法里就可以了

## 二、demo （一）

我们可以从 [GitHub 的 API 文档](https://docs.github.com/en/rest/reference/actions#list-artifacts-for-a-repository)里找到一些简单的 API ，进行网络爬虫的测试，示例：

```python
import requests


def request(url, param):
    header = {'Accept': 'application/vnd.github.v3+json'}
    try:
        response = requests.get(url, params=param, headers=header)
        if response.status_code == 200:
            return response
        else:
            return None
    except requests.RequestException:
        return None


if __name__ == '__main__':
    params = {
        'q': 'python',
        'sort': 'stars',
        'order': 'desc'
    }
    response = request(
        'https://api.github.com/search/repositories', params).json()
    print("一共有" + str(response["total_count"]) + "条结果")
    list = response["items"]
    for item in list:
        print("仓库名：%-30s\t" % str(item['name']), end=' ')
        print("Stars数：%10s" % str(item['forks_count']))

```

![image-20210425132141599](https://tva1.sinaimg.cn/large/008i3skNgy1gpvx6tdilnj30xm0qcn3m.jpg)

但是在大多数情况下，我们对于一个API 的了解情况是非常少的，也就没有办法这么直接的去解析数据，而是通过分析和处理网页的内容去摸索爬虫。

比如，我们打开豆瓣 Top 250 的网页，我们要如何一次性拿到这 250 部电影的信息呢？这个时候就可以用 Chrome 去监控我们打开这个网页的方式了，看下图：

![image-20210425134533636](https://tva1.sinaimg.cn/large/008i3skNgy1gpvxvlyu5ej31t00u07wh.jpg)

这里就可以看到打开这个页面时。浏览器所发起的网络请求，有 url，状态码，方法名，后面还有传递的参数，我们打开 postman 去测试这样的一个接口

![image-20210425135119669](https://tva1.sinaimg.cn/large/008i3skNgy1gpvy1lq0u0j310u0u0wlt.jpg)

意外的发现这个借口返回的时一个 html 格式的内容，这时候我们就需要从这个 html 格式的文件中去查找我们所需要的内容，而用到的方法是正则表达式的匹配,可以用 bs4 这个模块

## 三、demo（二）

```python
import requests
import re
from bs4 import BeautifulSoup
import json


def getFilms(url, header, page):
    try:
        pos = page*25
        payload = {
            'start': str(pos),
            'filter': ''
        }
        response = requests.get(
            url, params=payload, headers=header)
        if response.status_code == 200:
            return response.text
    except requests.RequestException:
        return None


'''有的api为了反爬虫。就会给api加一个header去确认浏览器访问，这里就拿到浏览器信息就好'''
header = {
    'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/89.0.4389.82 Safari/537.36'
}
base_url = 'https://movie.douban.com/top250'
for i in range(0, 11):
    html = getFilms(base_url, header, i)
    soup = BeautifulSoup(html, 'lxml')
    if soup.find(class_='grid_view'):
        list = soup.find(class_='grid_view').find_all('li')

        fileName = 'films.txt'
        with open(fileName, 'a+') as f:

            for item in list:
                item_name = item.find(class_='title').string
                item_img = item.find('a').find('img').get('src')
                item_index = item.find(class_='').string
                item_score = item.find(class_='rating_num').string
                item_author = item.find('p').text
                if item.find(class_='inq'):

                    item_intr = item.find(class_='inq').string
                    print('爬取电影：' + item_index + ' | ' + item_name +
                          ' | ' + item_score + ' | ' + item_intr)
                    f.write(str('爬取电影：' + item_index + ' | ' + item_name +
                                ' | ' + item_score + ' | ' + item_intr+'\n'))

                else:
                    f.writelines('爬取电影：' + item_index + ' | ' + item_name +
                                 ' | ' + item_score + ' | '+'None'+'\n')
                    print('爬取电影：' + item_index + ' | ' + item_name +
                          ' | ' + item_score + ' | '+'None')
                    # print('爬取电影：' + item_index + ' | ' + item_name + ' | ' + item_img + ' | '
                    #       + item_score + ' | ' + item_author + ' | ''' + item_intr)

```

