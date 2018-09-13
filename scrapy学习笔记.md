[TOC]

### 一、Scrapy基础知识

#### 1、Scrapy的基本架构和主要组件

**主要组件**

* ENGINE：引擎，框架核心，控制其他组件协调工作。内部组件
* SCHEDULER：调度器，负责协调请求。内部组件
* DOWNLOADER：下载器，负责发送请求和接收响应。内部组件
* SPIDER：爬虫文件，负责生成请求，提取数据，再次生成请求。用户实现
* MIDDLEWARE：中间件，负责处理请求和响应。可选组件
* ITEM PIPELINE：数据管道，负责对提取的数据进行处理。可选组件

主要流程：

1. SPIDER生成请求，提交给ENGINE，ENGINE将请求添加到SCHEDULER的请求队列
2. SCHEDULER协调调度请求，按照算法发送给DOWNLOADER
3. DOWNLOADER发送请求，并接收响应，响应再由ENGINE发送给SPIDER
4. SPIDER的页面解析函数，由响应提取数据，并封装到ITEM，如果有需要，再次生成请求，发送给ENGINE
5. ITEM可以由ITEM PIPELINE进一步处理，并导出为csv或者json格式。

#### 2、重要的两个对象：请求Request和响应Response

* **Request**

Scrapy中Request的构造器方法：

> `Request(url,[callback, method='GET', headers, body, cookies, meta, encoding='utf-8', priority = 0, dont_filter = False， errback])`
>
> 主要参数：
>
> > 1. **url** ： 请求页面的url，bytes或者str类型
> > 2. **callback**：Callback类型，请求收到响应后的回调函数
> > 3. **method**：请求方法，默认为GET
> > 4. **headers**：请求头文件，dict类型
> > 5. **cookies**： Cookies信息，dict类型
> > 6. **body**：请求的正文，bytes或者str类型
> > 7. **meta**：元数据字典，dict类型，用于给框架的其他组件传递信息。
> > 8. encoding：编码，默认为utf-8
> > 9. priority：请求优先级，默认为0
> > 10. dont_filter：默认为False，过滤对同一个url地址多次提交下载请求，如果为True，强制下载
> > 11. errback：请求出现错误时的回调函数，如404

* **Response**

Scrapy中响应有三个分类

>TextResponse和子类HtmlResqonse、XmlResponse

下载器获得的响应具有多个属性

> 1. url ：Response的url地址，str类型
> 2. status ： 状态码，int类型
> 3. headers ： 响应头，dict类型，可以调用get或者getlist方法访问
> 4. body：响应体，bytes类型
> 5. text：文本形式的响应正文，str类型，由body解码得到
> 6. encoding：响应编码
> 7. request：该响应的请求
> 8. meta：即request对象中的meta，可以用来传递信息
> 9. selector：Selector对象，用于从Response中提取数据
> 10. **xpath(query)**：response.selector.xpath的快捷方式
> 11. **css(query)**：response.selector.css的快捷方式
> 12. **urljoin(url)**：构造绝对url，传入的url为相对地址，与response.url拼接生成绝对地址

#### 3、Spider子类的基本组成

一个简单的Spider示例：

```python
import scrapy
class DemoSpider(scrapy.spiders.Spider):
    # 唯一标识
    name = 'Demo'
    # 爬虫起始网址
    start_urls = ['http://books.toscrape.com/']
    # 页面解析函数，继承Spider必须实现的方法，类似接口函数
    def parse(self, response):
        pass
```

Spider子类编写的基本流程：

> 1. 继承scrapy.Spider类
> 2. 取别名name
> 3. 设定爬虫起始网址start_urls
> 4. 页面解析函数parse，从response中提取数据

从**Spider源代码**看爬虫具体过程

> Spider基类部分源代码
>
> ```python
> class Spider(object_ref):
>     ...
>     def start_request(self):
>         for url in start_urls:
>             yield self.make_requests_from_url(url)
>     ...
>     def make_requests_from_url(self, url):
>         return Request(url,dont_filter = True)
>     ...
>     def parse(self,response):
>         raise NotImplementedError
> ```
>
> * 由框架主动调用start_request函数，从用户设定的start_urls中逐条获得url，传递给make_requests_from_url函数
> * make_requests_from_url函数实际上返回构造好的Request对象
> * start_request函数返回Request对象的迭代器yield
> * parse作为默认的页面解析函数，需要用户具体实现，类似借口。
> * 如果需要传递请求头，或者POST方式请求，就需要自定义start_request函数

**页面解析函数**

>页面解析的作用
>
>* 使用选择器从Response中提取出需要的数据，封装到Item
>* 使用选择器或者LinkExtractor提出Response中的链接，构造新的请求

***

### 二、Selector数据提取

页面数据解析一般有BeautifulSoup和lxml

* BeautifulSoup：使用简单，API简介，但是速度慢
* lxml：速度快，但是API复杂，较难使用

Selector是基于lxml库构建，并优化的API。

#### 1. 构造Selector对象

>* 从text创建
>* 从response创建
>* Scrapy的Response对象内置了Selector对象，并预先设定了xpath和css方法

##### 2.数据提取XPath

`XPath: 即 XML Path Language，解析xml文档的语言`

xml文档结构：

* 根节点：整个文档的起点
* 元素节点：一般使用<>包裹
* 文本节点：处于元素节点<>和</>之间
* 属性节点：元素节点的属性部分

**XPath** 的基本语法：

| 表达式    | 描述                                           |
| --------- | ---------------------------------------------- |
| /         | 选中根节点（root）                             |
| .         | 选中当前节点                                   |
| ..        | 选中当前节点的父节点                           |
| ELEMENT   | 选中所有ELEMENT元素节点                        |
| //ELEMENT | 选中后代节点中所有ELEMENT元素节点              |
| *         | 选中所有元素子节点                             |
| text()    | 选中所有文本节点                               |
| @ATTR     | 选中名为ATTR的属性节点                         |
| @*        | 选中所有属性节点                               |
| [谓语]    | 用来查找某个特定节点，或者包含某个特定值得节点 |

