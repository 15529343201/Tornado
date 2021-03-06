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
![image](https://github.com/15529343201/Tornado/blob/master/image/basic1.PNG)<br>
![image](https://github.com/15529343201/Tornado/blob/master/image/basic2.PNG)<br>
![image](https://github.com/15529343201/Tornado/blob/master/image/basic3.PNG)<br>
![image](https://github.com/15529343201/Tornado/blob/master/image/basic4.PNG)<br>
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
### **3.RequestHandler**
#### **1.接入点函数**
**(1)RequestHandler.initialize()**<br>
&emsp;&emsp;该方法被子类重写,实现了RequestHandler子类实例的初识化过程。可以为该函数传递参数,参数来源于配置URL映射时的定义。比如:
```
from tornado.web import RequestHandler
from tornado.web import Application

class ProfileHandler(RequestHandler):
    def initialize(self,database):
        self.database=database
        
    def get(self)
        pass
    
    def post(self)
        pass
        
app=Application([
    (r'/account,ProfileHandler,dict(database="c:\\example.db")),
])
```
&emsp;&emsp;本例中的initialize有参数database,该参数有Application定义URL映射时以dict方式给出。<br>
**(2)RequestHandler.prepare(),RequestHandler.on_finish()**<br>
&emsp;&emsp;prepare()方法用于调用请求处理(get,post等)方法之前的初始化处理。而on_finish()用于请求处理结束后的一些清理工作。这两种方法一个在处理前,一个在处理后,可以根据实际需要进行重写。通常用prepare()方法做资源初始化操作,而用on_finish()方法可做清理对象占用的内存或者关闭数据库连接等工作。<br>
**(3)HTTP Action处理函数**<br>
&emsp;&emsp;每个HTTP Action在RequestHandler中都以单独的函数进行处理。

 - RequestHandler.get(*args,**kwargs)
 - RequestHandler.head(*args,**kwargs)
 - RequestHandler.post(*args,**kwargs)
 - RequestHandler.delete(*args,**kwargs)
 - RequestHandler.patch(*args,**kwargs)
 - RequestHandler.put(*args,**kwargs)
 - RequestHandler.options(*args,**kwargs)

&emsp;&emsp;每个处理函数都以它们对应的HTTP Action小写的方式命名,此处不再赘述器应用方法。
#### **2.输入捕获**
&emsp;&emsp;输入捕获是指在RequestHandler中用于获取客户端输入的工具函数和属性,比如获取URL查询字符串,Post提交参数。
**(1)RequestHandler.get_argument(name),RequestHandler.get_arguments(name)**<br>
&emsp;&emsp;都是返回给定参数的值。get_argument获取单个值;而get_arguments是针对参数存在多个值的情况下使用的,返回多个值的列表。<br>
&emsp;&emsp;用get_argument/get_arguments()方法获取的是URL查询字符串参数与Post提交参数的参数合集。
**(2)RequestHandler.get_query_argument(name),RequestHandler.get_query_arguments(name)**<br>
&emsp;&emsp;它们与get_argument,get_arguments的功能类似,但是仅从URL查询参数中获取参数值。
**(3)RequestHandler.get_body_argument(name),RequestHandler.get_body_arguments(name)**<br>
&emsp;&emsp;与get_argument,get_arguments功能类似,但是仅从Post提交参数中获取参数值。
**(4)RequestHandler.get_cookie(name,default=None)**<br>
&emsp;&emsp;根据Cookie名称获取Cookie值。<br>
**(5)RequestHandler.request**<br>
&emsp;&emsp;返回tornado.httputil.HTTPServerRequest对象实例的属性,通过该对象可以获取关于HTTP请求的一切信息,比如:
```
import tornado.web
class DetailHandler(tornado.web.RequestHandler):
    def get():
        remote_ip=self.request.remote_ip      #获取客户端ip地址
        host=self.request.host                #获取请求的主机地址
```
&emsp;&emsp;常用的httputil.HTTPServerRequest对象属性
| 属性名 | 说明 |
| - | - |
| method | HTTP请求方法,比如GET,POST等 |
| uri | 客户端请求的uri的完整内容 |
| path | uri路径名,即不包括查询字符串 |
| query | uri中的查询字符串 |
| version | 客户端发送请求时使用的HTTP版本,比如HTTP/1.1 |
| headers | 以字典方式表达的HTTP Headers |
| body | 以字符串方式表达的HTTP消息体 |
| remote_ip | 客户端的IP地址 |
| protocol | 请求协议,比如HTTP,HTTPS |
| host | 请求消息中的主机名 |
| arguments | 客户端提交的所有参数 |
| files | 以字典方式表达的客户端上传的文件,每个文件名对应一个HTTPFile |
| cookies | 客户端提交的Cookie字典 |

#### **3.输出响应函数**
&emsp;&emsp;输出响应函数是指一组为客户端生成处理结果的工具函数,开发者调用它们以控制URL的处理结果。常用的输出响应函数如下。<br>
**(1)RequestHandler.set_status(status_code,reason=None)**<br>
&emsp;&emsp;设置HTTP Response中的返回码。如果有描述性的语句,则可以赋值给reason参数。
**(2)RequestHandler.set_header(name,value)**<br>
&emsp;&emsp;以键值对的方式设置HTTP Response中的HTTP头参数。使用set_header配置的Header值将覆盖之前配置的Header,比如:
```
import tornado.web
class DetailHandler(tornado.web.RequestHandler):
    def get():
        self.set_header("NUMBER",9)
        self.set_header("LANGUAGE","France")
        self.set_header("LANGUAGE","Chinese")
```
&emsp;&emsp;本例中的get()函数调用了3次set_header,但是只配置了两个Header参数,最后的HTTP Header中的参数将会是:
```
NUMBER:9
LANGUAGE:Chinese
```
**(3)RequestHandler.add_header(name,value)**<br>
&emsp;&emsp;以键值对的方式设置HTTP Response中的HTTP头参数。与set_header不同的是add_header配置的Header值将不会覆盖之前配置的Header,比如:
```
import tornado.web
class DetailHandler(tornado.web.RequestHandler):
    def get():
        self.set_header("NUMBER",8)
        self.set_header("LANGUAGE","France")
        self.add_header("LANGUAGE","Chinese")
```
&emsp;&emsp;最后HTTP Header中的参数将会是:
```
NUMBER:8
LANGUAGE:France
LANGUAGE:Chinese
```
**(4)RequestHandler.write(chunk)**<br>
&emsp;&emsp;将给定的块作为HTTP Body发送给客户端。在一般情况下,用本函数输出字符串给客户端。如果给定的块是一个字典,则会将这个块以JSON格式发送给客户端,同时将HTTP Header中的Content_Type设置成application/json。
**(5)RequestHandler.finish(chunk=None)**<br>
&emsp;&emsp;本方法通知Tornado:Response的生成工作完成,chunk参数是需要传递给客户端的HTTP body。调用finish()后,Tornado将向客户端发送HTTP Response。本方法适用于对RequestHandler的异步请求处理。<br>
&emsp;&emsp;**注意**:在同步或协程访问处理的函数中,无需调用finish()函数。<br>
**(6)ReqeustHandler.render(template_name,\**kwargs)**<br>
&emsp;&emsp;用给定的参数渲染模板,可以在本函数中传入模板文件名称和模板参数,比如:
```
import tornado.web
class MainHandler(tornado.web.RequestHandler):
    def get(self):
        items=["Python","C++","Java"]
        self.render["template.html",title="Tornado Templates",items=items)
```
&emsp;&emsp;render()的第一个参数是对模板文件的命名,之后以命名参数的形式传入多个模板参数。<br>
&emsp;&emsp;Tornado的基本模板语法与Django相同,但是功能弱化,高级过滤器不可用。<br>
**(7)RequestHandler.redirect(url,permanent=False,status=None)**<br>
&emsp;&emsp;进行页面重定向。在RequestHandler处理过程中,可以随时调用redirect()函数进行页面重定向,比如:
```
import tornado.web
class LoginHandler(tornado.web.RequestHandler):
    def get(self):
        self.render("login.html",next=self.get_argument("next","/"))
    
    def post(self):
        username=self.get_argument("username","")
        password=self.get_argument("password","")
        auth=self.db.authenticate(username,password)
        if auth:
            self.set_current_user(username)
            self.redirect(self.get_argument("next",u"/"))
        else:
            error_msg=u"?error="+tornado.escape.url_escape("Login incorrect.")
            self.redirect(u"/login"+error_msg)
```
&emsp;&emsp;在本例LoginHandler的post处理函数中,根据验证是否成功将客户端重定向到不同的页面;如果成功则重定向到next参数所指向的URL;如果不成功,则重定向到"/login"页面。<br>
**(8)RequestHandler.clear()**<br>
&emsp;&emsp;清空所有在本次请求中之前写入的Header和Body内容。比如:
```
import tornado.web
class DetailHandler(tornado.web.RequestHandler):
    def get():
        self.set_header("NUMBER",8)
        self.clear()
        self.set_header("LANGUAGE","France")
```
&emsp;&emsp;最后的Header中将不包括参数NUMBER。<br>
**(9)RequestHandler.set_cookie(name,value)**<br>
&emsp;&emsp;按键/值对设置Response中的Cookie值。<br>
**(10)RequestHandler.clear_all_cookies(path='/',domain=None)**<br>
&emsp;&emsp;清空本次请求中的所有Cookie。<br>
### **4.异步化及协程化**
&emsp;&emsp;在本节之前学习的例子都是用同步的方法处理用户的请求,即在RequestHandler的get()或post()函数中完成所有处理,当退出get(),post()等函数后马上向客户端返回Response。但是当处理逻辑比较复杂或需要等待外部I/O时,这样的处理机制会阻塞服务器线程,所以并不适合大量客户端的高并发请求场景。<br>
&emsp;&emsp;Tornado有两种方式可改变同步的处理流程。

 - 异步化:针对RequestHandler的处理函数使用@tornado.web.asynchronous修饰器,将默认的同步机制改为异步机制。
 - 协程化:针对RequestHandler的处理函数使用@tornado.gen.coroutine修饰器,将默认的同步机制改为协程机制

**1.异步化**<br>
&emsp;&emsp;异步化的RequestHandler处理如下:
```
#!/usr/bin/env python
# -*- coding: utf-8 -*-



import tornado.ioloop
import tornado.web
import tornado.httpclient

class MainHandler(tornado.web.RequestHandler):
    @tornado.web.asynchronous
    def get(self):
        http = tornado.httpclient.AsyncHTTPClient()
        http.fetch("http://www.baidu.com",
                   callback=self.on_response)

    def on_response(self, response):
        if response.error: raise tornado.web.HTTPError(500)
        self.write(response.body)
        self.finish()


def make_app():
    return tornado.web.Application([
        (r"/", MainHandler),
    ])

def main():
    app = make_app()
    app.listen(8888)
    tornado.ioloop.IOLoop.current().start()

if __name__ == "__main__":
    main()
```
![image](https://github.com/15529343201/Tornado/blob/master/image/7-3.PNG)<br>
&emsp;&emsp;本例中用装饰器tornado.web.asynchronous定义了HTTP访问处理函数get()。这样,当get()函数返回时,对该HTTP访问的请求尚未完成,所以Tornado无法发送HTTP Response给客户端。只有当在随后的on_response()中的finish()函数被调用时,Tornado才知道本次处理已完成,可以发送Response给客户端。<br>
&emsp;&emsp;异步编程虽然提高了服务器的并发能力,但如本例所示,在编程方法上较同步方法显得繁琐。<br>
**2.协程化**<br>
&emsp;&emsp;Tornado协程结合了同步处理和异步处理的优点,使得代码既清晰易懂,又能够适应海量客户端的高并发请求。
&emsp;&emsp;协程的编程方法举例如下:
```
#!/usr/bin/env python
# -*- coding: utf-8 -*-



import tornado.ioloop
import tornado.web
import tornado.httpclient


class MainHandler(tornado.web.RequestHandler):
    @tornado.gen.coroutine
    def get(self):
        http = tornado.httpclient.AsyncHTTPClient()
        response = yield http.fetch("http://www.baidu.com")
        self.write(response.body)


def make_app():
    return tornado.web.Application([
        (r"/", MainHandler),
    ])

def main():
    app = make_app()
    app.listen(8888)
    tornado.ioloop.IOLoop.current().start()

if __name__ == "__main__":
    main()
```
![image](https://github.com/15529343201/Tornado/blob/master/image/7-3.PNG)<br>
&emsp;&emsp;本例所示仍然是一个转发网站内容的处理器,代码量与相应的同步版差不多。协程化的关键技术点如下。

 - 用tornado.gen.coroutine装饰MainHandler的get(),post()等处理函数。
 - 使用异步对象处理耗时操作,比如本例的AsyncHTTPClient。
 - 调用yield关键字获取异步对象的处理结果。
# **用户身份认证框架**
&emsp;&emsp;用户身份认证是几乎所有网站的必要功能,对于Tornado的开发源头FriendFeed和Facebook这样的社交网站尤其如此,所以Tornado框架本身较其他Python框架集成了最为丰富的用户身份验证功能。使用该框架,开发者能够快速开发出既安全又强大的用户身份认证机制。
### **1.安全Cookie机制**
&emsp;&emsp;Cookie是很多网站为了辨别用户的身份而储存在用户本地终端(Client Side)的数据,定义与RFC2109。在Tornado中使用RequestHandler.get_cookie(),RequestHandler.get_cookie()可以方便的对Cookie进行读写,比如:
```
import tornado.web
session_id=1
class MainHandler(tornado.web.RequestHandler):
    def get(self):
        if not self.get_cookie("session"):
            self.set_cookie("session",str(session_id))
                session_id=session_id+1
            self.write("Your session got a new session!")
        else:
            self.write("Your session was set!")
```
&emsp;&emsp;本例中用get_cookie()函数判断Cookie名"session"是否存在,如果不存在则为其赋予新的session_id。<br>
&emsp;&emsp;因为Cookie总是被保存在客户端,所以如何保证其不被篡改是服务端必须解决的问题。Tornado提供了为Cookie信息加密的机制,使得客户端无法随意解析和修改Cookie的键值。<br>
&emsp;&emsp;一个使用安全Cookie的网站例子如下所示:
```
#!/usr/bin/env python
# -*- coding: utf-8 -*-


import tornado.web
import tornado.ioloop

session_id = 1



class MainHandler(tornado.web.RequestHandler):
    def get(self):
        global session_id
        if not self.get_secure_cookie("session"):
            self.set_secure_cookie("session",str(session_id))
            session_id = session_id + 1
            self.write("Your session got a new session!")
        else:
            self.write("Your session was set!")

application = tornado.web.Application([
    (r"/", MainHandler),
], cookie_secret="SECRET_DONT_LEAK")

def main():
    application.listen(8888)
    tornado.ioloop.IOLoop.current().start()

if __name__ == "__main__":
    main()
```
![image](https://github.com/15529343201/Tornado/blob/master/image/7-5-1.PNG)<br>
![image](https://github.com/15529343201/Tornado/blob/master/image/7-5-2.PNG)<br>
&emsp;&emsp;本例网站只提供了一个根目录页面,解析其关键点如下。

 - 在tornado.web.Application对象初始化时赋予cookie_secret参数,该参数值是一个字符串,用于保存本网站Cookie加密时的密钥。
 - 在需要读取Cookie的地方用RequestHandler.get_secure_cookie替换原来的RequestHandler.get_cookie调用。
 - 在需要写入Cookie的地方用RequestHandler.get_secure_cookie替换原来的RequestHandler.set_cookie调用。<br>
&emsp;&emsp;**注意:cookie_secret参数值是Cookie的加密密钥,需要做好保护工作,不能泄露给外部人员。**
### **2.用户身份认证**
&emsp;&emsp;在Tornado的RequestHandler类中有一个current_user属性用于保存当前请求的用户名。RequestHandler.current_user的默认值是None,在get(),post()等处理函数中可以随时读取该属性以获得当前的用户名。RequestHandler.current_user是一个只读属性,所以开发者需要重载RequestHandler.get_current_user()函数以设置该属性值。<br>
&emsp;&emsp;下面是使用RequestHandler.current_user属性及RequestHandler.get_current_user()方法来实现的用户身份控制的例子:
```python
import tornado.web
import tornado.ioloop
import uuid

dict_sessions = {}
class BaseHandler(tornado.web.RequestHandler):
    def get_current_user(self):
        session_id = self.get_secure_cookie("session_id")
        return dict_sessions.get(session_id)


class MainHandler(BaseHandler):
    def get(self):
        if not self.current_user:
            self.redirect("/login")
            return
        name = tornado.escape.xhtml_escape(self.current_user)
        self.write("Hello, " + name)

class LoginHandler(BaseHandler):
    def get(self):
        self.write('<html><body><form action="/login" method="post">'
                   'Name: <input type="text" name="name">'
                   '<input type="submit" value="Sign in">'
                   '</form></body></html>')

    def post(self):
        if len(self.get_argument("name"))<3:
            self.redirect("/login")
        session_id = str(uuid.uuid1())
        dict_sessions[session_id] = self.get_argument("name")
        self.set_secure_cookie("session_id", session_id)
        self.redirect("/")

application = tornado.web.Application([
    (r"/", MainHandler),
    (r"/login", LoginHandler),
], cookie_secret="SECRET_DONT_LEAK")


def main():
    application.listen(8888)
    tornado.ioloop.IOLoop.current().start()
    

if __name__ == "__main__":
    main()
```
&emsp;&emsp;本例演示了一个完整的身份认证编程框架,对其代码解析如下。

 - 用全局字典dict_sessions保存已经登录的用户信息,为简单起见,本例只用其保存"会话ID:用户名"的键/值。
 - 定义公共基类BaseHandler,该类继承自tornado.web.RequestHandler,用于定义本网站所有处理器的公共属性和行为。重载它的get_current_user()函数,其在开发者访问RequestHandler.current_user属性时自动被Tornado调用。该函数首先用get_secure_cookie()获得本次访问的会话ID,然后用会话ID从dict_sessions中获得用户名并返回。
 - MainHandler类是一个要求用户经过身份认证才能访问的处理器实例。该处理器中的处理函数get()使用了装饰器tornado.web.authenticated,具有该装饰器的处理函数在执行之前根据current_user是否已经被赋值来判断用户的身份认证情况。如果已经被赋值则可以进行正常逻辑,否则自动重定向到网站的登录页面。
 - LoginHandler类是登录页面处理器,其get()函数用于渲染登录页面,post()函数用于验证是否允许用户登录。本例中只要用户输入的用户名大于等于3个字节即允许用户登录。
 - 在tornado.web.Application的初始化函数中通过login_url参数给出网站的登录页面地址。改地址被用于tornado.web.authenticated装饰器在发现用户尚未验证时重定向到一个URL。
### **3.防止跨站攻击**
&emsp;&emsp;跨站请求伪造(Cross-site request forgery,CSRF或XSRF)是一种对网站的恶意利用。通过CSRF,攻击者可以冒用用户的身份,在用户不知情的情况下执行恶意操作。<br>
&emsp;&emsp;**1.CSRF攻击原理**
![image](https://github.com/15529343201/Tornado/blob/master/image/csrf.PNG)<br>

 - 用户首先访问了存在CSRF漏洞的网站Web(A),成功登陆并获取到了Cookie。此后,所有该用户对Web(A)的访问均会携带Web(A)的Cookie,因此被Web(A)认为是有效操作。
 - 此时用户又访问了带有攻击行为的站点Web(B),而Web(B)的返回页面中带有一个访问Web(A)进行恶意操作的链接,但被伪装成了合法内容。比如如下超链接看上去是一个抽奖信息,实际上却是向Web(A)站点提交提款请求:<br>
```
<a href="http://www.site1.com/get_money?amount=500&dest_card=340734569234">
三百万元大抽奖
</a>
```
 - 用户一旦点击恶意链接,就在不知情的情况下向Web(A)站点发送了请求,因为之前用户在Web(A)进行过登录且尚未退出,所以Web(A)在收到用户的请求和附带的Cookie时将认为该请求是用户发出的正常请求。此时,恶意站点的目的已经达到。
&emsp;&emsp;**2.用Tornado防范CSRF攻击**<br>
&emsp;&emsp;为了防范CSRF攻击,要求每个请求包括一个参数值作为令牌来匹配存储在Cookie中的对应值。<br>
&emsp;&emsp;Tornado应用可以通过一个Cookie头和一个隐藏的HTML表单元素向页面提供令牌。这样,当一个合法页面的表单被提交时,他将包括表单值和已存储的Cookie。如果两者匹配,则Tornado应用认定请求有效。<br>
&emsp;&emsp;开启Tornado的CSRF防范功能需要两个步骤。<br>
(1)在实例化tornado.web.Application时传入xsrf_cookies=True参数,即
```
application=tornado.web.Application([
    (r'/',MainHandler),
    (r'/purchase',PurchaseHandler),
],
cookie_secret="DONT_LEAK_SECRET",
xsrf_cookies=True,
)
```
或者
```
settings = [
    "cookie_secret":"DONT_LEAK_SECRET",
    "xsrf_cookies":True
]
application=tornado.web.Application([
    (r'/',MainHandler),
    (r'/login',LoginHandler),
],**settings)
```
(2)在每个具有HTML表单的模板文件中,为所有表单添加xsrf_form_html()函数标签,比如:<br>
```
<form action="/login" method="post">
{% module xsrf_form_html() %}
<input type="text" name="message"/>
<input type="submit" value="Post"/>
</form>
```
&emsp;&emsp;这里的{%module xsrf_form_html()%}起到了为表单添加隐藏元素防止跨站请求的作用。<br>
&emsp;&emsp;Tornado的安全Cookies支持和XSRF防范框架减轻了应用开发者的很多负担。没有它们,开发者需要思考很过防范的细节措施,因此Tornado內建的安全功能也非常有用。<br>
# HTML5 WebSocke概念及应用
&emsp;&emsp;Tornado的异步特性使得其非常适合服务器的高并发处理,客户端与服务器的持久连接应用架构就是高并发的典型应用。而WebSocket正是在HTTP客户端与服务器之间建立持久连接的HTML5标准技术。<br>
### **1.WebSocket概念**
&emsp;&emsp;WebSocket protocol是HTML5定义的一种新的标准协议(RFC6455),它实现了浏览器与服务器的全双工通信(full-duplex)。<br>
 **1. WebSocket的应用场景**
&emsp;&emsp;传统的HTTP和HTML技术使用客户端主动向服务器发送请求并获得回复的应用场景。但是随着即时通信需求的增多,这样的通信模型有时并不能满足应用的需求。<br>
&emsp;&emsp;WebSocket与普通Socket通信类似,它打破了原来HTTP的Request和Response一对一的通信模型,同时打破了服务器只能被动地接受客户端请求的应用场景。下图解释了HTTP+HTML的应用局限性。<br>
&emsp;&emsp;传统HTTP+HTML方案只适用于客户端主动发出请求的场景,而无法满足服务器端发起的通信要求。也许读者听说过Ajax,Long poll等基于传统HTTP的动态客户端技术,但这些技术无不采用轮询技术,耗费了大量的网络带宽和计算资源。<br>
&emsp;&emsp;而WebSocket正是为了应对这样的场景而制定的HTML5标准,相对于普通的Socket通信,WebSocket又在应用层定义了基本的交互流程,使得Tornado这样的服务器器框架和JavaScript客户端可以构建标准的Websocket模块。<br>
&emsp;&emsp;总结WebSocket的特点如下:<br>
 
 
 - WebSocket适合服务器端主动推送的场景。
 - 相对于Ajax和Long poll等技术,WebSocket通信模型更高效
 - WebSocket仍然与HTTP完成Internet通信。
 - 因为是HTML5的标准协议,所以不受企业防火墙的拦截。

**2. WebSocket的通信原理**<br>
&emsp;&emsp;WebSocket的通信原理是在客户端与服务器之间建立TCP持久连接,从而使得当服务器有消息需要推送给客户端时能够进行即时通信。<br>
&emsp;&emsp;虽然WebSocket不是HTTP,但由于Internet上HTML本身是由HTTP封装并进行传输的,所以WebSocket仍然需要与HTTP进行协作。IETF在RFC6455中定义了基于HTTP链路建立WebSocket信道的标准流程。<br>
&emsp;&emsp;客户端通过发送如下HTTP Request告诉服务器需要建立一个WebSocket长连接信道:<br>
```python
GET /stock_info/?encodin=text HTTP/1.1
Host:echo.websocket.org
Origin:http://websocket.org
Cookie:__token=ubcxx13
Connection:Upgrade
Sec-WebSocket-Key:uRovscZjNo1/umbTt5uKmw==
Upgrade:websocket
Sec-WebSocket-Version:13
```

 - HTTP请求谓词:GET
 - 请求地址:/stock_info
 - HTTP版本号:1.1
 - 服务器主机域名:echo.websocket.org
 - Cookie信息:__token=ubcxx13
&emsp;&emsp;但是在HTTP Header中出现了4个特殊的字段,它们是:<br>
```python
Connection:Upgrade
Sec-WebSocket-Key:uRovscZjNo1/umbTt5uKmw==
Upgrade:websocket
Sec-WebSocket-Version:13
```
&emsp;&emsp;这就是WebSocket建立链路的核心,它告诉Web服务器:客户端希望建立一个WebSocket链接,客户端使用的WebSocket版本是13,密钥是``uRovscZjNo1/umbTt5uKmw==``。<br>
&emsp;&emsp;服务器收到该Request后,如果同意建立WebSocket链接则返回类似如下的Response:<br>
```python
HTTP/1.1 101 WebSocket Protocol Handshake
Date:Fri,10 Feb 2012 17:38:18 GMT
Connection:Upgrade
Server:Kaazing Gateway
Upgrade:WebSocket
Access-Control-Allow-Origin:http://websocket.org
Access-Control-Allow-Credentials:true
Sec-WebSocket-Accept:rLHCkw/SKsO9GAH/ZSFhBATDKrU=
Access-Control-Allow-Headers:content-type
```
&emsp;&emsp;这仍旧是一个标准的HTTP Response,其中与WebSocket相关的Header信息是:<br>
```python
Connection:Upgrade
Upgrade:WebSocket
Sec-WebSocket-Accept:rLHCkw/SKsO9GAH/ZSFhBATDKrU=
```
&emsp;&emsp;前面的两条数据告诉客户端:服务器已经将本地链接转换为WebSocket链接,而Sec-WebSocket-Accept是将客户端发送的Sec-WebSocket-Key加密后产生的数据,已让客户端确认服务器能够正常工作。<br>
&emsp;&emsp;至此,在客户端与服务器之间已经建立了一个TCP持久长连接,双发已经可以随时向对方发送消息。<br>
### **2.服务端编程**
&emsp;&emsp;Tornado定义了tornado.websocket.WebSocketHandler类用于处理WebSocket连接诶的请求,应用开发者应该继承该类并实现其中的``open(),on_message(),on_close()``函数。<br>

 - WebSocketHandler.open()函数:在一个新的WebSocket链接建立时,Tornado框架会调用此函数。在本函数中,开发者可以和在get(),post()等函数中一样用get_argument()函数获取客户端提交的参数,以及用get_secure_cookie/set_secure_cookie操作Cookie等。
 - WebSocketHandler.on_message(message)函数:建立WebSocket链接后,当收到来自客户端的消息时,Tornado框架会调用本函数。通常,这是服务器端WebSocket编程的核心函数,通过解析收到的消息作出相应的处理。
 - WebSocketHandler.on_close()函数:当WebSocket链接关闭时,Tornado框架会调用本函数。在本函数中,可以通过访问self.close_code和self.close_reason查询关闭的原因。
&emsp;&emsp;除了这3个tornado框架自动调用的入口函数,WebSocketHandler还提供了两个开发者主动操作WebSocket的函数。<br>

 - WebSocketHandler.write_message(message,binary=False)函数,用于向与本链接相对应的客户端信息。
 - WebSocketHandler.close(code=None,reason=None)函数:主动关闭WebSocket链接。其中的code和reason用于告诉客户端链接被关闭的原因。参数code必须是一个数值,而reason是一个字符串。
&emsp;&emsp;下面是持续为客户端推送时间的Tornado WebSocket程序:<br>
```python
import tornado.ioloop
import tornado.web
import tornado.websocket

from tornado.options import define, options, parse_command_line

define("port", default=8888, help="run on the given port", type=int)

# we gonna store clients in dictionary..
clients = dict()

class IndexHandler(tornado.web.RequestHandler):
    @tornado.web.asynchronous
    def get(self):
        #self.write("This is your response")
        #self.finish()
        self.render("index.html")

class MyWebSocketHandler(tornado.websocket.WebSocketHandler):
    def open(self, *args):
        self.id = self.get_argument("Id")
        self.stream.set_nodelay(True)
        clients[self.id] = {"id": self.id, "object": self}

    def on_message(self, message):        
        """
        when we receive some message we want some message handler..
        for this example i will just print message to console
        """
        print "Client %s received a message : %s" % (self.id, message)
        
    def on_close(self):
        if self.id in clients:
            del clients[self.id]
            print "Client %s is closed" % (self.id)

app = tornado.web.Application([
    (r'/', IndexHandler),
    (r'/websocket', MyWebSocketHandler),
])


import threading
import time
def sendTime():
    import datetime
    while True:
        for key in clients.keys():
            msg = str(datetime.datetime.now())
            clients[key]["object"].write_message(msg)
            print "write to client %s: %s" % (key,msg)
        time.sleep(1)

  
if __name__ == '__main__':
    threading.Thread(target=sendTime).start()
    parse_command_line()
    app.listen(options.port)
    tornado.ioloop.IOLoop.instance().start()

```
&emsp;&emsp;解析上述代码如下。<br>

 - 定义了全局变量字典clients,用于保存所有与服务器建立WebSocket链接的客户端信息。字典的键是客户端id,值是由一个id与相应的WebSocketHandler实例构成的元组(Tuple)。
 - IndexHandler是一个普通的页面处理器,用于向客户端渲染主页index.html。本页面中包含了WebSocket的客户端程序。
 - MyWebSocketHandler是本例的核心处理器,继承自tornado.web.WebSocketHandler。其中的open()函数将所有客户端链接保存到clients字典中;on_message()用于显示客户端发来的消息;on_close()用于将已经关闭的WebSocket链接从clients字典中移除。
 - 函数sendTime()运行在单独的线程中,每隔1秒轮询clients中的所有客户端并通过MyWebSocketHandler.write_message()函数向客户端推送时间消息。
 - 本例的tornado.web.Application实例中只配置了两个路由,分别指向IndexHandler和MyWebSocketHandler,仍然由Tornado IOLoop启动并运行。
### **3.客户端编程**
&emsp;&emsp;由于WebSocket是HTML5的标准之一,所以主流浏览器的Web
客户端编程语言JavaScript已经支持WebSocket的客户端编程。<br>
&emsp;&emsp;客户端编程围绕着WebSocket对象展开,在JavaScript中可以通过如下代码初始化WebSocket对象:<br>
```javascript
var Socket=new WebSocket(url);
```
&emsp;&emsp;在代码中只需给WebSocket构造函数传入服务器的url地址,比如http://mysite.com/point。可以为该对象的如下事件指定处理函数以响应它们。<br>

 - WebSocket.onopen:此事件发生在WebSocket链接建立时。
 - WebSocket.onmessage:此事件发生在收到了来自服务器的消息时。
 - WebSocket.onerror:此事件发生在通信过程中有任何差错时。
 - WebSocket.onclose:此事件发生在与服务器的链接关闭时。
&emsp;&emsp;除了这些事件处理函数,还可以通过WebSocket对象的两个方法进行主动操作。<br>
 - WebSocket.send(data):向服务器发送消息。
 - WebSocket.close():主动关闭现有链接
&emsp;&emsp;客户端的WebSocket编程示例程序如下:<br>
```html
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
	
    </head>
    <body>
        <a href="javascript:WebSocketTest()">Run WebSocket</a>
        <div id="messages" style="height:200px;background:black;color:white;"></div>
    </body>
	
    <script type="text/javascript">
        var messageContainer = document.getElementById("messages");
        function WebSocketTest() {
            if ("WebSocket" in window) {
                messageContainer.innerHTML = "WebSocket is supported by your Browser!";
                var ws = new WebSocket("ws://localhost:8888/websocket?Id=12345");
                ws.onopen = function() {
                    ws.send("Message to send");
                };
                ws.onmessage = function (evt) { 
                    var received_msg = evt.data;
                    messageContainer.innerHTML =
     			  messageContainer.innerHTML+
  				  "<br/>Message is received:"+received_msg;
                };
                ws.onclose = function() { 
                    messageContainer.innerHTML =
  				  messageContainer.innerHTML+"<br/>Connection is closed...";
                };
            } else {
                messageContainer.innerHTML = "WebSocket NOT supported by your Browser!";
            }
        }
    </script>
</html>

```
&emsp;&emsp;对上述代码解析如下。<br>

 - 客户端页面主体由两部分构成:一个Run WebSocket链接用于让用户启动WebSocket;另一个id=message的<div>标签用于显示服务器端的消息。
 - 使用JavaScript语句if("WebSocket" in window)可以判断当前浏览器是否支持WebSocket对象。
 - 如果浏览器支持WebSocket对象,则定义实例ws链接到服务器的WebSocket地址ws://localhost:8888/websocket,并传入表示自己的参数id=12345。然后通过JavaScript语法定义事件onopen,onmessage,onclose的处理函数。除了在onopen事件中客户端向服务器用WebSocket.send()函数发送了消息,其余事件均只将事件结果显示在页面<div>标签中。

 
