# 一、介绍WSGI
## 1.1 WSGI边界
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2579230/1655179183262-99bb2573-0bbd-4a22-9a2c-20375e9d3e72.png)
WSGI（Web Server Gateway Interface)主要规定了服务器端和应用程序间的接口。 
WEB Server主要负责HTTP协议请求和响应，但`不一定支持WSGI接口`访问。

## 1.2 客户请求流程
![](https://cdn.nlark.com/yuque/0/2022/jpeg/2579230/1655280755609-0ad40895-ec0f-4cde-8007-b81ffc63abe5.jpeg)
关键三处：

- `environ`是简单封装的请求报文的字典
- `start_response`解决响应报文头的函数
- `app函数`返回响应报文正文，简单理解就是HTML
# 二、WSGI服务器实现-wsgiref
## 2.1 实现简单WSGI HTTP服务端
wsgiref是Python提供的一个WSGI参考实现库，`不适合生产环境`使用。
`wsgiref.simple_server模块`实现一个简单的WSGI HTTP服务器。
```python
from wsgiref.simple_server import make_server, demo_app

ip = '127.0.0.1'
port = 8080
server = make_server(ip, port, demo_app) # demo_app应用程序，可调用
server.serve_forever() # server.handle_request() 执行一次
# 浏览器请求访问 http://127.0.0.1:8080/
# 或者 curl http://127.0.0.1:8080/
```
## 2.2 make_server
启动一个WSGI服务器
```python
def make_server(
    host, port, app, server_class=WSGIServer, handler_class=WSGIRequestHandler
):
    """Create a new WSGI server listening on `host` and `port` for `app`"""
    server = server_class((host, port), handler_class)
    server.set_app(app)
    return server
```
## 2.2 demo_app
实现了(environ,start_response)两参数函数，小巧完整的WSGI的应用程序app的实现
```python
def demo_app(environ,start_response):
    from io import StringIO
    stdout = StringIO()
    print("Hello world!", file=stdout)
    print(file=stdout)
    h = sorted(environ.items())
    for k,v in h:
        print(k,'=',repr(v), file=stdout)
    start_response("200 OK", [('Content-Type','text/plain; charset=utf-8')])
    return [stdout.getvalue().encode("utf-8")]
```
 
# 三、WSGI APP应用程序
## 3.1 WSGI APP必须实现的三点

1. APP应用程序应该是一个`可调用对象`
   - `函数`
   - `类`
   - 实现了`__call__方法`的`类的实例`
2. APP可调用对象应该接收`两个参数`
2. 可调用对象都必须`返回一个可迭代对象`
```python
res_str = b'www.baidu.com\n'

# 1、函数实现
def application(environ, start_response):
    start_response("200 OK", [('Content-Type','text/plain; charset=utf-8')])
    return  [res_str]

# 2 类实现
class Application:
    def __init__(self, environ, start_response):
        self.env = environ
        self.start_response = start_response
    # 可迭代对象
    def __iter__(self):
        self.start_response("200 OK", [('Content-Type','text/plain; charset=utf-8')])
        yield res_str
        # yield from [res_str]
        # return iter([res_str])

# 3 类的实例实现
class Application:
    def __call__(self, environ, start_response):
        start_response("200 OK", [('Content-Type','text/plain; charset=utf-8')])
        return [res_str]
```
environ和start_response这两个参数名可以是任何合法名，但是一般默认都是这2个名字。 
应用程序端还有些其他的规定，暂不用关心 
`注意`：第2、第3种实现调用时的不同

## 3.2 environ
environ是包含http request请求信息的dict字典对象

| **名称 ** | **含义** |
| --- | --- |
| REQUEST_METHOD  | 请求方法，GET、POST等 |
| PATH_INFO** ** | URL中的路径部分  |
| QUERY_STRING  | 查询字符串 |
| SERVER_NAME, SERVER_PORT | 服务器名、端口 |
| HTTP_HOST  | 地址和端口  |
| SERVER_PROTOCOL | 协议  |
| HTTP_USER_AGENT  | UserAgent信息 |

```python
CONTENT_TYPE = 'text/plain'
HTTP_HOST = '127.0.0.1:9999'
HTTP_USER_AGENT = 'Mozilla/5.0 (Windows; U; Windows NT 6.1; zh-CN) 
AppleWebKit/537.36 (KHTML, like Gecko) Version/5.0.1 Safari/537.36'
PATH_INFO = '/'
QUERY_STRING = ''
REMOTE_ADDR = '127.0.0.1'
REMOTE_HOST = ''
REQUEST_METHOD = 'GET'
SERVER_NAME = 'DESKTOP-D34H5HF'
SERVER_PORT = '9999'
SERVER_PROTOCOL = 'HTTP/1.1'
SERVER_SOFTWARE = 'WSGIServer/0.2'
```
##  3.3 start_response

- start_response应该在`返回可迭代对象之前调用`，因为它是Response Header。返回的可迭代对象是ResponseBody。
- 它是一个`可调用对象`。有3个参数，如下：
   - `start_response(status, response_headers, exc_info=None)`
| 参数名称 | 说明 |
| --- | --- |
| status | 状态码和状态描述，例如 200OK  |
| response_headers  | 一个元素为二元组的列表，例如[('Content-Type','text/plain;charset=utf-8')]  |
| exc_info  | 在错误处理的时候使用 |

# 四、自定义APP的服务端
服务器程序需要调用符合上述定义的可调用对象APP，传入environ、start_response，APP处理后，返回响应头和可迭代对象的正文，由服务器封装返回浏览器端。
```python
from wsgiref.simple_server import make_server

def application(environ, start_response):
    status = '200 OK'
    headers = [('Content-Type', 'text/html;charset=utf-8')]
    start_response(status, headers)
    html = '<h1>xxx</h1>'.encode("utf-8")
    return [html]

ip = '127.0.0.1'
port = 8080
server = make_server(ip, port, application)
server.serve_forever()
```
`simple_server`只是参考用，不能用于生产环境
测试访问：

- -I 使用HEAD方法
- -X 指定方法POST
- -d 传输数据
```python
curl -I http://127.0.0.1:8080/xxx?id=5
curl -X POST http://127.0.0.1:8080/yyy -d '{"x":2}'
```
# 五、总结
## 5.1 WSGI 服务器作用 

1. 监听HTTP服务端口（TCPServer，默认端口80）接收浏览器端的HTTP请求，这是WEB Server的 

作用 

2. 解析请求报文封装成`environ`环境数据 
2. 负责调用应用程序app，将`environ`数据和`start_response`方法两个实参传入给Application 
2. 利用app的返回值和`start_response`返回的值，`构造HTTP响应报文 `
2. 将响应报文返回浏览器端 


2、3、4要实现WSGI协议，该协议约定了和应用程序之间接口（参看PEP333，https://www.python.or 
g/dev/peps/pep-0333/） 
## 5.2 WSGI APP应用程序 

- 遵从WSGI协议 
- 本身是一个可调用对象 
- 调用start_response，返回响应头部 
- 返回包含正文的可迭代对象 
## 5.3 WEB框架
Django、Flask都是符合WSGI协议且可以快速开发的框架，但本质上是编写Application，说白了，就是 
编写一个函数，这个函数签名为`app(environ, start_response) `，这不过在app函数内部调用非常复杂而 
已。比如要解决访问数据库、静态HTML页面读取、动态网页生成等。

 

 



 



