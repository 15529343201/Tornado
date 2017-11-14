# 一,Tornado概述<br>
### **1,Tornado介绍**<br>
&ensp;&ensp;Tornado是使用Python编写的一个强大的可扩展的Web服务器。它在处理高网络流量时表现足够强健,却在创建和编写时有着足够的轻量级,并能够被用在大量的应用和工具中。Tornado作为FriendFeed网站的基础框架,与2009年9月10日发布,目前已经获得了很多社区的支持,并且在一系列不同的场合中得到应用。除FriendFeed和Fackbook外,还有很多公司在生产上转向Tornado,包括Quora,Turntable.fm,Bit.ly,Hipmunk及My Yearbook等。<br>
&ensp;&ensp;相对于其他Python网络框架,Tornado有如下特点。

 - 完备的Web框架:与Django,Flask等一样,Tornado也提供了URL路由映射,Request上下文,基于模板的页面渲染技术等开发Web应用的必备工具。
 - 是一个高效的网络库,性能与Twisted,Gevent等底层Python框架相媲美:提供了异步I/O支持,超时事件处理。这使得Tornado除了可以作为Web应用服务区框架,还可以用来做爬虫应用,物联网关,游戏服务器等后台应用。
 - 提供高效HTTPClient:除了服务器框架,Tornado还提供了基于异步框架的HTTP客户端。
 - 提供高效的内部HTTP服务器:虽然其他Python网络框架(Django,Flask)也提供了内部HTTP服务器,但它们的HTTP服务器由于性能原因只能用于测试环境。而Tornado的HTTP服务器与Tornado异步调用紧密结合,可以直接用于生成环境。
 - 完备的WebSocket支持:WebSocket是HTML5的一种标准,实现了浏览器与服务器之间的双向实时通信。
&ensp;&ensp;因为Tornado的上述特点,Tornado常被用作大型站点的接口服务框架,而不像Django那样着眼于建立完整的大型网站,主要了解Tornado的异步协程编程,身份认证框架,独特的非WSGI部署方式。
### **2,安装Tornado**
&ensp;&ensp;通过命令安装:
``pip install tornado``
# 二,异步和协程基础
&ensp;&ensp;协程是Tornado中推荐的编程方式,使用协程可以开发出简捷,高效的异步处理代码。
### **1,同步和异步I/O**
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
&ensp;&ensp;AsyncHTTPClient是Tornado的异步访问HTTP客户端。在上述代码的asynchronous_visit()函数中使用AsyncHTTPClient对第三方网站进行异步访问,http_client.fetch()函数会在调用后立刻返回而无需等待实际访问的完成,从而导致asynchronous_visit()也会立刻执行完成。当对www.baidu.com的访问实际完成后,AsyncHTTPClient会调用callback参数指定的函数,开发者可以在其中写入处理访问结果的逻辑代码。<br>
### **2.Python关键字yield**

 1. 迭代器
 &ensp;&ensp;迭代器(Iterator)是访问集合的一种方式。迭代器对象从集合的第1个元素开始访问,直到所有元素都被访问一遍后结束。迭代器不能回退,只能往前进行迭代。
&ensp;&ensp;Python中最常使用迭代器的场景是循环语句for,它用于迭代器封装集合,并且逐个访问集合元素以执行循环体。比如:
```
for number in range(5):       #range返回一个列表
    print number
```
&ensp;&ensp;其中的range()返回一个包含指定元素的集合,而for语句将其封装成一个迭代器后访问。使用iter()调用可以将列表,集合转换为迭代器,比如:<br>
![image](https://github.com/15529343201/Tornado/blob/master/image/iter1.PNG)<br>
&ensp;&ensp;代码中的t即为迭代器。
&ensp;&ensp;迭代器与普通Python对象的区别是迭代器有一个netx()方法,每次调用该方法可以返回一个元素。调用者(比如for语句)可以通过不断调用next()方法来逐个访问集合元素。比如:<br>
![image](https://github.com/15529343201/Tornado/blob/master/image/iter2.PNG)<br>
 2. 使用yield
&ensp;&ensp;迭代器在Python编程中的使用范围很广,那么开发者如何定制自己的迭代器呢?答案是使用yield关键字。调用任何定义中包含yield关键字的函数都不会执行该函数,而会获得一个对应该函数的迭代器。<br>
![image](https://github.com/15529343201/Tornado/blob/master/image/yield1.PNG)<br>
&ensp;&ensp;执行该部分代码的结果如下:<br>
![image](https://github.com/15529343201/Tornado/blob/master/image/yield2.PNG)<br>
&ensp;&ensp;每次调用迭代器的next()函数,将执行迭代器函数,并返回yield的结果作为迭代返回元素。当迭代器函数return时,迭代器会抛出StopIteration异常使迭代终止。
&ensp;&ensp;在Python中,使用yield关键字定义的迭代器也被称为"生成器"<br>
### **3.协程**<br>
&emsp;&emsp;使用Tornado协程可以开发出类似同步代码的异步行为。同时,因为协程本身不使用线程,所以减少了线程上下文切换的开销,是一种更高效的开发模式。<br>
1.编写协程函数<br>
```
from tornado import gen           #引入协程库gen
from tornado.httpclient import AsyncHTTPClient

@gen.coroutine                    #使用gen.coroutine装饰器
def coroutine_visit():
    http_client=AsyncHTTPClient()
    response=yield http_client.fetch("www.baidu.com")
    print response.body
```
&emsp;&emsp;本例中仍然使用了异步客户端AsyncHTTPClient进行页面访问,装饰器@gen.coroutine声明这是一个协程函数。由于yield关键字的使用,使得代码中不用在编写回调函数用于处理访问结果,而可以直接在yield语句的后面编写结果处理语句。<br>
2.调用协程函数<br>
&emsp;&emsp;由于Tornado协程基于Python的yield关键字实现,所以不能像调用普通函数一样调用协程函数,比如用下面的代码不能调用之前编写的coroutine_visit()协程函数:<br>
```
def bad_call():
    coroutine_visit()
```
&emsp;&emsp;协程函数可以通过以下三种方式进行调用。

 - 在本身是协程的函数内通过yield关键字调用
 - 在IOLoop尚未启动时,通过IOLoop的run_sync()函数调用
 - 在IOLoop已经启动时,通过IOLoop的spawn_callback()函数调用。
&emsp;&emsp;下面是一个"通过协程函数调用协程函数"的例子:
```
from tornado import gen

@gen.coroutine
def outer_coroutine():
    print "start call another coroutine"
    yeild cortoutine_visit()
    print "end of outer_cortoutine"
```
&emsp;&emsp;本例中outer_coroutine和coroutine_visit都是协程函数,所以它们之间可以通过yield关键字进行调用。<br>
&emsp;&emsp;IOLoop是Tornado的主事件循环对象,Tornado程序通过它监听外部客户端的访问请求,并执行相应的操作。当程序尚未进入IOLoop的running状态时,可以通过run_sync()函数调用协程函数,比如:<br>
```
from tornado.ioloop import IOLoop

def func_normal():
    print "start to call a coroutine"
    IOLoop.current().run_sync(lambda:coroutine_visit())
    print "end of calling a coroutine"
```
&emsp;&emsp;本例中spawn_callback()函数将不会等待被调用协程执行完成,所以spawn_callback()之前和之后的print语句将会被连续执行,而coroutine_visit本身将会由IOLoop在合适的时机进行调用。
&emsp;&emsp;IOLoop的spawn_callback()函数没有为开发者提供获取协程函数调用返回值的方法,所以只能用spawn_callback()调用没有返回值的协程函数。<br>
3.在协程中调用阻塞函数<br>
&emsp;&emsp;在协程中直接调用阻塞函数会影响协程本身的性能,所以Tornado提供了在协程中利用线程池调度阻塞函数,从而不影响协程本身继续执行的方法。示例代码如下:<br>
```
from concurrent.futures import ThreadPoolExecutor

thread_pool=ThreadPoolExecutor(2)

def mySleep(count):
    import time
    for I in range(count):
        time.sleep(1)
        
@gen.coroutine
def call_blocking():
    print "start of call_blocking"
    yield thread_pool.submit(mySleep,10)
    print "end of call_blocking"
```
&emsp;&emsp;代码中首先引用了concurrent.futures中的ThreadPoolExecutor类,并实例化了一个有两个线程的线程池thread_pool。在需要调用阻塞函数的协程call_blocking中,使用thread_pool.submit调用阻塞函数,并通过yield返回。这样便不会阻塞协程所在线程的继续执行,也保证了阻塞函数前后代码的执行顺序。<br>
4.在协程中等待多个异步调用<br>
&emsp;&emsp;其实,Tornado允许在协程中用一个yield关键字等待多个异步调用,只需把这些调用用列表(list)或字典(dictionary)的方式传递给yield关键字即可。
&emsp;&emsp;使用列表方式传递多个异步调用的示例代码如下:
```
from tornado import gen
from tornado.httpclient import AsyncHTTPClient

@gen.coroutine
def coroutine_visit():
    http_client=AsyncHTTPClient()
    list_response=yield [http_client.fetch("www.baidu.com"),
        http_client.fetch("www.163.com"),
        http_client.fetch("www.google.com")
    ]
    
for response in list_response:
    print response.body
```
&emsp;&emsp;在代码中仍然用@gen.coroutine装饰器定义协程,在需要yield的地方用列表传递若干个异步调用,只有在列表中的所有调用都执行完成后,yield才会返回并继续执行。yield以列表方式返回N个调用的输出结果,可以通过for语句逐个访问。<br>
&emsp;&emsp;用字典方式传递多个异步调用的示例代码如下:
```
from tornado import gen
from tornado.httpclient import AsyncHTTPClient

@gen.coroutine
def coroutine_visit():
    http_client=AsyncHTTPClient()
    dict_response=yield {"baidu":http_client.fetch("www.baidu.com"),
        "163":http_client.fetch("www.163.com"),
        "google":http_client.fetch("www.google.com")
    }

print dict_respone["sina"].body
```
&emsp;&emsp;本例中以字典形式给yield关键字传递异步调用要求,并且Tornado以字典形式返回异步调用结果。
# **三,实战演练:开发Tornado网站**
### **1.网站结构**
示例:通过如下HelloWorld程序学习Tornado网站的基本结构:
```
import tornado.ioloop
import tornado.web

class MainHandler(tornado.web.RequestHandler):
    def get(self):
	self.write("Hello world");


def make_app():
    return tornado.web.Application([
	(r"/",MainHandler),
    ])


def main():
    app=make_app()
    app.listen(8888)
    tornado.ioloop.IOLoop.current().start()


if __name__=="__main__":
    main()
```
![image](https://github.com/15529343201/Tornado/blob/master/image/HelloWorld1.PNG)<br>
![image](https://github.com/15529343201/Tornado/blob/master/image/HelloWorld2.PNG)<br>
&emsp;&emsp;下面逐行解析上面的代码做了些什么。<br>
&emsp;&emsp;(1)首先通过import语句引入tornado包中的ioloop和web类。引入这两个类是Tornado程序的基础。<br>
&emsp;&emsp;(2)实现一个web.RequestHandler子类,重载其中的get()函数,该函数负责相应定位到该RequestHandler的HTTP GET请求的处理。本例中简单的通过self.write()函数输出"Hello World"。<br>
&emsp;&emsp;(3)定义了make_app()函数,该函数返回一个web.Application对象。该对象的第1个参数用于定义Tornado程序的路由映射。本例将对根URL的访问映射到了RequestHandler子类MainHandler中。<br>
&emsp;&emsp;(4)用web.Applicaiton.listen()函数指定服务器监听的端口。<br>
&emsp;&emsp;(5)用tornado.ioloop.IOLoop.current().start()启动IOLoop,该函数将一直运行且不退出,用于处理完所有客户端的访问请求。
### **2.路由解析**
&emsp;&emsp;向web.Application对象传递的第1个参数URL路由映射列表的配置方式与Django类似,用正则字符串进行路由匹配。Tornado的路由字符串有两种:固定字串路径和参数字串路径。
#### **1.固定字串路径**
&emsp;&emsp;固定字串即是普通的字符串固定匹配,比如:<br>
```
Handlers=[("/",MainHandler),               #只匹配根路径
        ("/entry",EntryHandler),           #只匹配/entry
        ("/entry/2015",Entry2015Handler),  #只匹配/entry/2015
]
```
#### **2.参数字串路径**
&emsp;&emsp;参数字串可以将具备一定模式的路径映射到同一个RequesetHandler中处理,其中路径中的参数部分用小括号"()"标识,下面是一个参数路径的例子。<br>
```
# url handler
handlers = [(r"/entry/([^/]+)",EntryHandler),]

class EntryHandler(tornado.web.RequestHandler):
    def get(self,slug):
        entry=self.db.get("SELECT * FROM entries WHERE slug=%s",slug)
        if not entry:
            raise tornado.web.HTTPError(404)
        self.render("entry.html",entry=entry)
```
&emsp;&emsp;本例中用"/entry([^/]+)"定义"以/entry/开头的URL"模式,小括号中的内容是正则表达式。URL尾部的变量部分以参数形式传递给RequestHandler的get()函数,本例中将该参数命名为slug。<br>
#### **3.带默认值的参数路径**
&emsp;&emsp;之前例子中的handlers=[(r"/entry/([^/]+)",EntryHandler),]模式定义了客户端必须输入路径参数。比如,其能够匹配如下路径:
```
http://xx.xx.xx.xx/entry/abc
http://xx.xx.xx.xx/entry/2015-09-10
```
&emsp;&emsp;但是其无法匹配:
```
http://xx.xx.xx.xx/entry
```
&emsp;&emsp;对于需要匹配客户端未传入时的路径,则需要用如下方法改变URL路径和对get()函数的定义:<br>
```
# url handler
handlers=[(r"/entry/([^/]*)",EntryHandler),]

class EntryHandler(tornado.web.RequestHandler):
    def get(self,slug='default'):
        entry=self.db.get("SELECT * FROM entries WHERE slug=%s",slug)
        if not entry:
            raise tornado.web.HTTPError(404)
        self.render("entry.html",entry=entry)
```
&emsp;&emsp;本例中首先用星号"*"取代加号"+"定义了URL模式"/entry/([^/]*)",然后为RequestHandler子类的get()函数的slug参数配置了默认值default。
#### **4.多参数路径**
&emsp;&emsp;参数路径还允许在一个URL模式中定义多个可变参数,比如:<br>
```
handlers  = [
    (r'/(\d{4})/(\d{2})/(\d{2})/([a-zA-Z\-0-9\.:,_]+)/?',DetailHandler)
    
class DetailHandler(tornado.web.RequestHandler):
    def get(self,year,month,day,slug):
        self.write("%d-%d-%d %s"%(year,month,day,slug))
```
