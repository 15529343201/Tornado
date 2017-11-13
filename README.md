#一,Tornado概述
####**1,Tornado介绍**
&ensp;&ensp;Tornado是使用Python编写的一个强大的可扩展的Web服务器。它在处理高网络流量时表现足够强健,却在创建和编写时有着足够的轻量级,并能够被用在大量的应用和工具中。Tornado作为FriendFeed网站的基础框架,与2009年9月10日发布,目前已经获得了很多社区的支持,并且在一系列不同的场合中得到应用。除FriendFeed和Fackbook外,还有很多公司在生产上转向Tornado,包括Quora,Turntable.fm,Bit.ly,Hipmunk及My Yearbook等。<br>
&ensp;&ensp;相对于其他Python网络框架,Tornado有如下特点。

 - 完备的Web框架:与Django,Flask等一样,Tornado也提供了URL路由映射,Request上下文,基于模板的页面渲染技术等开发Web应用的必备工具。
 - 是一个高效的网络库,性能与Twisted,Gevent等底层Python框架相媲美:提供了异步I/O支持,超时事件处理。这使得Tornado除了可以作为Web应用服务区框架,还可以用来做爬虫应用,物联网关,游戏服务器等后台应用。
 - 提供高效HTTPClient:除了服务器框架,Tornado还提供了基于异步框架的HTTP客户端。
 - 提供高效的内部HTTP服务器:虽然其他Python网络框架(Django,Flask)也提供了内部HTTP服务器,但它们的HTTP服务器由于性能原因只能用于测试环境。而Tornado的HTTP服务器与Tornado异步调用紧密结合,可以直接用于生成环境。
 - 完备的WebSocket支持:WebSocket是HTML5的一种标准,实现了浏览器与服务器之间的双向实时通信。
&ensp;&ensp;因为Tornado的上述特点,Tornado常被用作大型站点的接口服务框架,而不像Django那样着眼于建立完整的大型网站,主要了解Tornado的异步协程编程,身份认证框架,独特的非WSGI部署方式。
####**2,安装Tornado**
&ensp;&ensp;通过命令安装:
``pip install tornado``
#二,异步和协程基础
&ensp;&ensp;协程是Tornado中推荐的编程方式,使用协程可以开发出简捷,高效的异步处理代码。
####**1,同步和异步I/O**
&ensp;&ensp;从计算机硬件发展的角度来看,当今计算机系统的CPU和内存速度日新月异,摩尔定律效果非常明显;同时,硬盘,网络等于I/O相关的速度指标却紧张缓慢。因此,在当今的计算机应用开发中,减少程序在I/O相关操作中的等待是对减少资源消耗,提高并发程度的必要考虑。<br>
&ensp;&ensp;根据Unix Network Programing一书中的定义,同步I/O操作(synchronous I/O operation)导致请求进程阻塞,直到I/O操作完成;异步I/O操作(asynchronous I/O operation)不导致请求进程阻塞。在Python中,同步I/O可以被理解为一个被调用的I/O函数会阻塞调用函数的执行,而异步I/O则不会阻塞调用函数的执行。代码举例如下:
```
from tornado.httpclient import HTTPClient   #Tornado的HTTP客户端类

def synchronous_visit():
    http_client=HTTPClient()
    response=http_client.fetch("www.baidu.com") #阻塞,直到对www.baidu.com访问完成
    print response.bod
```
&ensp;&ensp;HTTPClient是Tornado的同步访问HTTP客户端。上述代码中的synchronous_visit()函数使用了典型的同步I/O操作访问www.baidu.com网站,该函数执行时间取决于网络速度,对方服务器响应速度等。只有当对www.baidu.com的访问完成并获取到结果后,才能完成对synchronous_visit()函数的执行。
&ensp;&ensp;而使用异步I/O访问www.baidu.com网站的函数无需等待访问完成才能返回,比如:
```
from tornado.httpclient import AsyncHTTPClient

def handle_response(response):
    print response.body

def asynchronous_visit():
    http_client=AsyncHTTPClient()
    http_client.fetch("www.baidu.com",callback=handle_response)
```
&ensp;&ensp;AsyncHTTPClient是Tornado的异步访问HTTP客户端。在上述代码的asynchronous_visit()函数中使用AsyncHTTPClient对第三方网站进行异步访问,http_client.fetch()函数会在调用后立刻返回而无需等待实际访问的完成,从而导致asynchronous_visit()也会立刻执行完成。当对www.baidu.com的访问实际完成后,AsyncHTTPClient会调用callback参数指定的函数,开发者可以在其中写入处理访问结果的逻辑代码。
####**2.Python关键字yield**

 1. 迭代器
 &ensp;&ensp;迭代器(Iterator)是访问集合的一种方式。迭代器对象从集合的第1个元素开始访问,直到所有元素都被访问一遍后结束。迭代器不能回退,只能往前进行迭代。
&ensp;&ensp;Python中最常使用迭代器的场景是循环语句for,它用于迭代器封装集合,并且逐个访问集合元素以执行循环体。比如:
```
for number in range(5):       #range返回一个列表
    print number
```
&ensp;&ensp;其中的range()返回一个包含指定元素的集合,而for语句将其封装成一个迭代器后访问。使用iter()调用可以将列表,集合转换为迭代器,比如:
![image](https://github.com/15529343201/Tornado/blob/master/image/iter1.PNG)
&ensp;&ensp;代码中的t即为迭代器。
&ensp;&ensp;迭代器与普通Python对象的区别是迭代器有一个netx()方法,每次调用该方法可以返回一个元素。调用者(比如for语句)可以通过不断调用next()方法来逐个访问集合元素。比如:
![image](https://github.com/15529343201/Tornado/blob/master/image/iter2.PNG)
 2. 使用yield
&ensp;&ensp;迭代器在Python编程中的使用范围很广,那么开发者如何定制自己的迭代器呢?答案是使用yield关键字。调用任何定义中包含yield关键字的函数都不会执行该函数,而会获得一个对应该函数的迭代器。
![image](https://github.com/15529343201/Tornado/blob/master/image/yield1.PNG)
&ensp;&ensp;执行该部分代码的结果如下:
![image](https://github.com/15529343201/Tornado/blob/master/image/yield2.PNG)
&ensp;&ensp;每次调用迭代器的next()函数,将执行迭代器函数,并返回yield的结果作为迭代返回元素。当迭代器函数return时,迭代器会抛出StopIteration异常使迭代终止。
&ensp;&ensp;在Python中,使用yield关键字定义的迭代器也被称为"生成器"

