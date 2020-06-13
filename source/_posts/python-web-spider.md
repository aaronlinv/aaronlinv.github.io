---
title: 爬虫是咋回事
date: 2019-10-31 16:46:19
categories: Python笔记
---
## 背景
之前零零碎碎接触过几次爬虫，有的是公开课，有的是实体课。老师都是偏实践，对于理论的部分都是一笔带过，所以对于那些库都是一知半解。自己也想写一些东西，尽可能在吸收那些大佬的总结的同时，加入一些自己的理解。把大佬一越而过，而我们要挣扎很久的坑铺平。


这样学的知识更像是一盘散沙，我会简单地使用这些函数去实现一些功能，但却不清楚它们之间的关系，遇到一些复杂的问题就很难去处理。系统地学习虽然比较耗时，但是会对整个体系有个了解，而这在你遇到一些问题的时候会发挥重要作用。

网上教程的质量参差不齐，而且代码案例爬取的网站的结构也可能随着时间的推移而改变，对于初学者来说是个不小的挑战。

我看的是嵩天老师的[Python网络爬虫与信息提取](https://www.icourse163.org/learn/BIT-1001870001)，嵩天老师讲的比较通俗易懂，不需要有特别扎实的Python基础也能听懂。
## 遇到的问题
- 爬取网页，程序没有反应：可能是网络连接问题，可以用通用代码架构来提高爬虫的可靠性
- F12开发者工具-Network选项卡里可以看的文本但是爬取的文本却没有：网站防爬机制，可以定义Headers,Cookie等解决


##  爬虫原理

对于非异步加载的网站，一般都可以通过requests库和BeautifulSoup库爬取信息
1. 获取网页的源代码：通过Requests库，来获取网页的HTML代码
2. 解析网页代码获得期望的数据：通过BeautifulSoup库解析网页源代码，把我们需要的信息按照一定规则提取出来
## requests库
requests库有7个主要方法，后6个方法都是对request方法的封装（使用更方便）

![嵩天](pythonWebSpider/1.png)

6个方法对应的就是HTTP协议对资源的操作

![嵩天](pythonWebSpider/2.png)

HTTP请求状态：status_code 200表示成功，返回其他值说明连接出现问题
``` python
r = requests.get(url) #url传入爬取的网址
print(r.status_code)  #200表示连接成功
```
爬虫通用框架：网络连接可能出现各种异常，所以我们可以用try-except处理异常，增强代码可靠性
``` python
import requests  # 导入requests库，用于获得网页源代码

def getHTMLText(url):
    try:
        r = requests.get(url, timeout=30) # 设置超时时间为30毫秒
        r.raise_for_status() # 这个方法就是判断status_code是否为200，否的话引发HTTPError异常
        r.encoding = r.apparent_encoding
        return r.text # 返回爬取到的文本
    except:
        return "产生异常"
```

编码：如果header没有chartset指定编码，默认就是ISO-8859-1，这个编码不能解析中文，可能会出现乱码，而apparent_encoding会根据内容猜测编码，所以我们一般把编码设置为apparent_encoding
``` python
r.encoding = r.apparent_encoding
```
## BeautifulSoup库
BeautifulSoup使用pip安装要注意，如果没有加末尾的4，实际安装的是BeautifulSoup3 
``` bash
pip insatll beautifulsoup4
```
在import的时候也要注意，从bs4库(beautifulsoup4)中引入BeautifulSoup这个类
``` python
from bs4 import BeautifulSoup
#注意B和S是大写
```
创建一个BeautifulSoup对象，传入的一个参数是需要解析的HTML或者XML文档，第二个参数是使用的解析器，这里需要lxml，需要安装一下
``` bash
pip install lxml
```

``` python
soup = BeautifulSoup(html, 'lxml')
```

BeautifulSoup类的基本元素：
* Tag：标签，HTML中由<>和</>构成的那些标签
* Name：就是标签的名字，a标签、p标签
* Attributes：标签的属性
* NavigableString：标签的<>和</>之间的文字
* Comment：HTML中的注释 

爬取数据需要先找到数据对应Tag标签，而Name,Attributes,NavigableString,Comment就是围绕着Tag标签，Comment相对来说用的少

HTML标签树遍历：
* 下行遍历：.contents  .children  .descendants
* 上行遍历：.parent  .parents
* 平行遍历：.next_sibling  .next_siblings  .previous_sibling  .previous_sibling

有复数含义的属性返回的都是列表，而像.parent .next_sibling  .previous_sibling 这些都是返回一个Tag标签节点 <class 'bs4.element.Tag'> 或者是一个字符串节点（有可能是\n）,在遍历这些列表的时候如果没有判断类型，使用的时候可能会报错

>TypeError: 'NavigableString' object is not callable

find_all( )方法：
* 可检索标签名、标签属性、标签内的字符串，返回一个列表
* 还有find( ),find_parent( ),find_parents( )等方法
* 因为dind_all比较常用，所以对象是Tag或者是soup都可以省略find_all,<tag>find_all(...)等价于<tag>(...),soup.find_all(..)等价于soup(...)
## 爬取大学排名实例
爬取最好大学网的大学排名数据，将排名、学校名称、地区输出到屏幕

![爬取大学排名效果](pythonWebSpider/3.png)

### 遇到的问题
* 遍历每个tr获取每个大学信息，出现object is not callable
* 输出到屏幕排版问题

### 分析网页结构
Chrome浏览器打开[最好大学网的网页](http://zuihaodaxue.com/zuihaodaxuepaiming2019.html)，按F12打开开发者工具

![Chrome开发者工具](pythonWebSpider/4.png)

需要刷新一下网页才会显示，双击这个document文件

![Document文件](pythonWebSpider/5.png)

选择Response选项卡就可以看到HTML代码，全选复制到编辑器里

![VScode](pythonWebSpider/6.png)

排名的信息在tbody标签里，每一个tr标签里面都是一所大学的信息，每一个td标签就是每所大学具体的信息。我们可以遍历tbody里面的每个tr标签，然后把每所大学的信息都保存在list里。

``` python
soup = BeautifulSoup(html, 'html.parser')
    for tr in soup.find('tbody').children:
        if isinstance(tr, bs4.element.Tag):
            # import bs4 才能使用对应的标签定义
            tds = tr('td')  # 查询tr 中的td标签
            ulist.append([tds[0].string, tds[1].string, tds[2].string])
```

在遍历的时候报错,会遇到一个问题：

>发生异常: TypeError'NavigableString' object is not callable

![comment](pythonWebSpider/7.png)

原因是：HTML代码tbody标签中最后有一段Comment,而Comment对象和NavigableString对象不能像Tag对象一样被调用，所以我们在遍历时需要对类型进行判断，判断是否为：bs4.element.Tag
### 排版问题
在输出时候中文排版会出现一些小问题
![排版错误](pythonWebSpider/8.png)

``` python
def printUnivList(ulist, num):
    print("{:^10}\t{:^6}\t{:^10}".format("排名", "学校名称", "总分"))

    for i in range(num):
        u = ulist[i]
        print("{:^10}\t{:^6}\t{:^10}".format(u[0], u[1], u[2]))
```

原因就是我们在使用format方法在我们定义好的布局里填充大学信息的时候，排版默认使用西文的空格，这样会导致排版错误。

``` python
def printUnivList(ulist, num):
    tplt = "{0:^10}\t{1:{3}^10}\t{2:^10}"
    print(tplt.format("排名", "学校名称", "总分", chr(12288)))
    # 使用中文空格填充

    for i in range(num):
        u = ulist[i]
        # print(u)
        print(tplt.format(u[0], u[1], u[2], chr(12288)))
```

我们可以使用中文空格来进行填充，这样可以解决排版问题，chr(12288)就是中文空格

``` python
tplt = "{0:^10}\t{1:{3}^10}\t{2:^10}"
print(tplt.format(u[0], u[1], u[2], chr(12288)))

# 这个tplt就是我们定义每行的模板，{}里面就是要填充的内容
# 冒号前面是填充内容的索引号，^10 ^的表居中对齐10代表宽度为10
# \t 表示一个制表符，在冒号后面是填充字符，不指定默认使用西文空格
# 这里我们指定{3},就是使用索引为3的 chr(12288) 来填充


```


### 具体代码

先把大体的框架写完
``` python
import requests
from bs4 import BeautifulSoup


def getHTMLText(url):
    pass


def fillUnivList(ulist, html):
    pass


def printUnivList(ulist, num):
    pass


def main():
    uinfo = []
    url = 'http://zuihaodaxue.com/zuihaodaxuepaiming2019.html'
    html = getHTMLText(url)
    fillUnivList(uinfo, html)
    printUnivList(uinfo, 20)  # 爬取前20名


main()
```

再填充每个具体的方法

``` python
import requests
from bs4 import BeautifulSoup
import bs4


def getHTMLText(url):
    try:
        r = requests.get(url, timeout=30)
        r.raise_for_status()
        r.encoding = r.apparent_encoding
        return r.text
    except:
        return ""


def fillUnivList(ulist, html):
    soup = BeautifulSoup(html, 'html.parser')
    for tr in soup.find('tbody').children:
        if isinstance(tr, bs4.element.Tag):
            # import bs4 才能使用对应的标签定义
            tds = tr('td')  # 查询tr 中的td标签
            ulist.append([tds[0].string, tds[1].string, tds[2].string])


# def printUnivList(ulist, num):
#     print("{:^10}\t{:^6}\t{:^10}".format("排名", "学校名称", "总分"))

#     for i in range(num):
#         u = ulist[i]
#         print("{:^10}\t{:^6}\t{:^10}".format(u[0], u[1], u[2]))


def printUnivList(ulist, num):
    tplt = "{0:^10}\t{1:{3}^10}\t{2:^10}"
    print(tplt.format("排名", "学校名称", "总分", chr(12288)))
    # 使用中文空格填充

    for i in range(num):
        u = ulist[i]
        # print(u)
        print(tplt.format(u[0], u[1], u[2], chr(12288)))


def main():
    uinfo = []
    url = 'http://zuihaodaxue.com/zuihaodaxuepaiming2019.html'
    html = getHTMLText(url)
    fillUnivList(uinfo, html)
    printUnivList(uinfo, 20) 


main()
```