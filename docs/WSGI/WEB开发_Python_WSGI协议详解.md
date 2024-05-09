# WEB开发——Python WSGI协议详解

## Web应用程序开发

### Web应用程序的本质是什么

简单描述Web应用程序的本质，就是我们通过浏览器访问互联网上指定的网页文件展示到浏览器上。

流程如下图:
![图片描述](https://cdn.jsdelivr.net/gh/Killer-89757/PicBed/images/2024%2F05%2F5cdbc88e000193a009630234-40a6b9.jpeg)

从更深层次一点的技术角度来看，由以下几个步骤：

- 浏览器，将要请求的内容按照HTTP协议发送服务端
- 服务端，根据请求内容找到指定的HTML页面
- 浏览器，解析请求到的HTML内容展示出来

#### HTTP协议的全称是HyperText Transfer Protocol（超文本传输协议）

HTTP协议是我们常用的五层协议中的应用层(5层从上到下是应用层，传输层，网络层，数据链路层，物理层)，HTTP协议中协定的内容称之为消息，消息主要包括消息头——Header和消息体——Body。
客户端请求时的消息称为Request，服务端响应时的消息称为Response.

Header：包括请求方法，HTTP版本，URI，状态码，COOKIE等
Body：是响应或者请求时的内容，包含HTML,CSS,JS等

**HTTP协议这里就不做过多的描述，可以到[点击这里](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Messages)深入了解HTTP协议**

#### HTML的全称是Hyper Text Markup Language（超文本标记语言）

简单点说，HTML 是一种由不同元素组成的标记语言，它定义了网页内容的含义和结构，所有我们在浏览器中看到的内容都是由一个一个的元素组成。除 HTML 以外的其它技术则通常用来描述一个网页的表现与展示效果（如 CSS），或功能与行为（如 JavaScript）。

**HTML就不再往深处描述，可以到[点击这里](https://developer.mozilla.org/zh-CN/docs/Web/HTML)深入了解HTML**

### WEB开发的历程

#### 静态开发

```
直接将写好的HTML页面放在服务器上，然后直接通过浏览器访问指定服务器的文件。
```

#### 动态开发

```python
随着我们的需求变化单独使用静态开发已经不能完全满足我们。

例如我们查看的页面只有部分内容会变化，那我们再去开发相同的页面。

一是开发上是一种重复工作，完全是一种浪费。
二是数据量变化巨大时，完全是跟不上速度，并且数据变化也不是定时更新。

为了应对这种问题，动态网页技术也就诞生了。早期的动态网页开发技术是CGI

CGI全称：Common Gateway Interface，通用网关接口，它是一段程序，运行在服务器上如：HTTP 服务器，
提供同客户端 HTML 页面的接口。
CGI 程序可以是 Python 脚本，PERL 脚本，SHELL 脚本，C 或者 C++ 程序等。

各种编程语言也针对动态网页开发给出不同的解决方案，JAVA的servlet，Python的WSGI协议等。
```

Python的WSGI协议也是我们本章要讲的内容

**CGI流程**

![图片描述](https://cdn.jsdelivr.net/gh/Killer-89757/PicBed/images/2024%2F05%2F5cdbcb5d000190a812930422-f0ce79.jpeg)

**WSGI的流程**
![图片描述](https://cdn.jsdelivr.net/gh/Killer-89757/PicBed/images/2024%2F05%2F5cdbcb710001a54808421154-b178b2.jpeg)

## 什么是WSGI

WSGI全称是Web Server Gateway Interface，其主要作用是Web服务器与Python Web应用程序或框架之间的建议标准接口，以促进跨各种Web服务器的Web应用程序可移植性。

**WSGI并不是框架**而只是一种协议，我们可以将WSGI协议分成三个组件Application，Server，Middleware和协议中传输的内容。

将这三个组件对映射到我们具体使用的组件是：

- Server：常用的有uWSGI，gunicorn等
- Application：Django，Flask等
- Middleware： Flask等框架中的装饰器

[点击这里查看官方关于WSGI协议的定义](https://www.python.org/dev/peps/pep-3333)

### 组件Application

应用程序，是一个可重复调用的可调用对象，在Python中可以是一个函数，也可以是一个类，如果是类的话要实现__call__方法，要求这个可调用对象接收2个参数，返回一个内容结果

接收的2个参数分别是environ和start_response。

- environ是web服务器解析HTTP协议的一些信息，例如请求方法，请求URI等信息构成的一个Dict对象。
- start_response是一个函数，接收2个参数，一个是HTTP状态码，一个HTTP消息中的响应头。

依照官方提供的示例用函数实现应用程序

```python
def simple_app(environ, start_response):
    """Simplest possible application object"""
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain; charset=utf-8')]
    start_response(status, response_headers)
    
    return_body = []
    
    for key, value in environ.items():
        return_body.append("{} : {}".format(key, value))
    
    return_body.append("\nHello WSGI!")
    # 返回结果必须是bytes
    return ["\n".join(return_body).encode("utf-8")]
```

### 组件Server

Web服务器，主要是实现相应的信息转换，将网络请求中的信息，按照HTTP协议将内容拿出，同时按照WSGI协议组装成新的数据，同时将提供的start_response传递给Application。最后接收Application返回的内容，按照WSGI协议解析出。最终按照HTTP协议组织好内容返回就完成了一次请求。

Server操作的步骤如下：

1. 根据HTTP协议内容构建envrion
2. 提供一个start_response函数，接收HTTP STATU 和 HTTP HEADER
3. 将envrion和start_response作为参数调用Application
4. 接收Application返回的结果
5. 按照HTTP协议，顺序写入HTTP响应头（start_response接收），HTTP响应体（Application返回结果）

下面这个是pep3333协议中的一个server例子，按照CGI请求的方式来实现。

```python
import os, sys

enc, esc = sys.getfilesystemencoding(), 'surrogateescape'

def unicode_to_wsgi(u):
    # Convert an environment variable to a WSGI "bytes-as-unicode" string
    return u.encode(enc, esc).decode('iso-8859-1')

def wsgi_to_bytes(s):
    return s.encode('iso-8859-1')

def run_with_cgi(application):
	# 按照WSGI协议，构建environ内容
	# 1类 CGI相关的变量，此脚本就是用于cgi执行，所以前面的web服务器已经将CGI变量封装好，这里直接使用
    environ = {k: unicode_to_wsgi(v) for k,v in os.environ.items()}
    # 2类 wsgi定义的变量
    environ['wsgi.input']        = sys.stdin.buffer
    environ['wsgi.errors']       = sys.stderr
    environ['wsgi.version']      = (1, 0)
    environ['wsgi.multithread']  = False
    environ['wsgi.multiprocess'] = True
    environ['wsgi.run_once']     = True

    if environ.get('HTTPS', 'off') in ('on', '1'):
        environ['wsgi.url_scheme'] = 'https'
    else:
        environ['wsgi.url_scheme'] = 'http'

    headers_set = []
    headers_sent = []

    def write(data):
	    # 将内容返回
        out = sys.stdout.buffer

        if not headers_set:
             raise AssertionError("write() before start_response()")

        elif not headers_sent:
             # Before the first output, send the stored headers
             status, response_headers = headers_sent[:] = headers_set
             out.write(wsgi_to_bytes('Status: %s\r\n' % status))
             for header in response_headers:
                 out.write(wsgi_to_bytes('%s: %s\r\n' % header))
             out.write(wsgi_to_bytes('\r\n'))

        out.write(data)
        out.flush()
	
	
    def start_response(status, response_headers, exc_info=None):
        if exc_info:
            try:
                if headers_sent:
                    # Re-raise original exception if headers sent
                    raise exc_info[1].with_traceback(exc_info[2])
            finally:
                exc_info = None     # avoid dangling circular ref
        elif headers_set:
            raise AssertionError("Headers already set!")

        headers_set[:] = [status, response_headers]

        # Note: error checking on the headers should happen here,
        # *after* the headers are set.  That way, if an error
        # occurs, start_response can only be re-called with
        # exc_info set.

        return write
	
	# 将上面处理的参数交给应用程序
    result = application(environ, start_response)
    try:
	    # 将请求到的结果写回。
        for data in result:
            if data:    # don't send headers until body appears
                write(data)
        if not headers_sent:
            write('')   # send headers now if body was empty
    finally:
        if hasattr(result, 'close'):
            result.close()
```

### 组件Middleware

中间件，可以理解为对应用程序的一组装饰器。
在应用程序端看来，它可以提供一个类start_response函数，可以想start_response函数一样接收HTTP STATU和Headers；和environ。
在服务端看来，他可以接收2个参数，并且可以返回一个类Application对象。
下面看一个例子，记录每次请求的消耗时间：

```python
import time
class ResponseTimingMiddleware(object):
    """记录请求耗时"""
    def __init__(self, app):
        self.app = app

    def __call__(self, environ, start_response):
        start_time = time.time()
        response = self.app(environ, start_response)
        response_time = (time.time() - start_time) * 1000
        timing_text = "记录请求耗时中间件输出\n\n本次请求耗时: {:.10f}ms \n\n\n".format(response_time)
        response.append(timing_text.encode('utf-8'))
        return response
```

### 协议内容

重点看environ有哪些内容，这里面才是浏览器每次请求时的信息。再深入一点探索，就是HTTP请求消息中的请求头和请求体都是怎么定义及怎么回去的。
environ是一个字典，environ中要包含CGI定义的变量，主要是将HTTP协议中的内容，比如请求方法，POST/GET，请求URI等，另外是WSGI协议自己定义的变量，比如请求body中要读取的信息等。列一下主要变量项如下：

**CGI相关变量**

| 变量            | 说明                                          |
| :-------------- | :-------------------------------------------- |
| REQUEST_METHOD  | POST,GET等,HTTP请求的动词标识                 |
| SERVER_PROTOCOL | 服务器运行的HTTP协议. 这里当是HTTP/1.0.       |
| PATH_INFO       | 附加的路径信息, 由浏览器发出.                 |
| QUERY_STRING    | 请求URL的“？”后面的部分                       |
| CONTENT_TYPE    | HTTP请求中任何Content-Type字段的内容          |
| CONTENT_LENGTH  | 标准输入口的字节数.                           |
| HTTP_[变量]     | 其他一些变量，例如HTTP_ACCEPT，HTTP_REFERER等 |

**上述内容是动态开发的根基，只有根据上述内容才可以标准化的动态处理请求。**

**WSGI定义变量**

| 变量              | 说明                                      |
| :---------------- | :---------------------------------------- |
| wsgi.version      | WSGI版本,要求是元组(1,0),标识WSGI 1.0协议 |
| wsgi.url_scheme   | 表示调用应用程序的URL的协议，http或https  |
| wsgi.input        | 类文件对象，读取HTTP请求体字节的输入流    |
| wsgi.errors       | 类文件对象，写入错误输出的输出流          |
| wsgi.multithread  | 如果是多线程，则设置为True，否则为False。 |
| wsgi.multiprocess | 如果是多进程，则设置为True，否则为False。 |
| wsgi.run_once     | 如果只需要运行一次，设置为True            |

WSGI协议对于两个输入输出流有一些方法必须要实现

| 流          | 方法            |
| :---------- | :-------------- |
| wsgi.input  | read(size)      |
| wsgi.input  | readline()      |
| wsgi.input  | readlines(hint) |
| wsgi.input  | **iter**()      |
| wsgi.errors | flush()         |
| wsgi.errors | write(str)      |
| wsgi.errors | writelines(seq) |

这些基本上就是WSGI协议中定义的主要变量，也基本上涵盖了我们开发时所需要的变量。

Server端按照协议的内容生成这些environ字典，然后将请求信息交给Application，Application根据这些信息确认请求要处理的内容，然后返回响应消息。从头顺下来就是这个流程。

### 示例展示

Server端涉及到实现http相关内容，我们直接使用python内置wsgiref来实现，具体代码如下：

```python
import time
from wsgiref.simple_server import make_server

class ResponseTimingMiddleware(object):
    """记录请求耗时"""
    def __init__(self, app):
        self.app = app

    def __call__(self, environ, start_response):
        start_time = time.time()
        response = self.app(environ, start_response)
        response_time = (time.time() - start_time) * 1000
        timing_text = "记录请求耗时中间件输出\n\n本次请求耗时: {:.10f}ms \n\n\n".format(response_time)
        response.append(timing_text.encode('utf-8'))
        return response

def simple_app(environ, start_response):
    """Simplest possible application object"""
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain; charset=utf-8')]
    start_response(status, response_headers)
    
    return_body = []
    
    for key, value in environ.items():
        return_body.append("{} : {}".format(key, value))
    
    return_body.append("\nHello WSGI!")
    # 返回结果必须是bytes
    return ["\n".join(return_body).encode("utf-8")]

# 创建应用程序
app = ResponseTimingMiddleware(simple_app)
# 启动服务，监听8080
httpd = make_server('localhost', 8080,  app)  
httpd.serve_forever()
```

启动服务后，我们打开浏览器访问http://localhost:8080，执行结果如下。

![图片描述](https://img1.sycdn.imooc.com//5cdd3be70001329c19201011.png)

上图可以看到我们前面提到的中间件以及Application中执行返回的结果全都实现。

WSGI协议内容就到这，下次我们阅读python wsgiref库的源码，看其如何实现wsgi协议。