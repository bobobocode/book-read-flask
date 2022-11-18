# WSGI

Python Web Server Gateway Interface

是一套接口标准

在 Web Server 和 Python Web Application 之间约定的接口通信标准.
符合这套标准的Web应用可以在
以提高web应用在一系列web服务器间的移植性.

这套标准的内容也很简单

web服务器在将请求转交给web应用程序之前，需要先将http报文转换为WSGI规定的格式。
WSGI规定，Web程序必须有一个可调用对象，且该可调用对象接收两个参数，返回一个可迭代对象

environ：字典，包含请求的所有信息
start_response：在可调用对象中调用的函数，用来发起响应，参数包括状态码，headers等


def hello(environ, start_response):
    status = "200 OK"
    response_headers = [('Content-Type', 'text/html')]
    start_response(status, response_headers)
    path = environ['PATH_INFO'][1:] or 'hello'
    return [b'<h1> %s </h1>' % path.encode()]



# coding:utf-8
"""
desc: WSGI服务器实现
"""
from wsgiref.simple_server import make_server
from learn_wsgi.client import hello


def main():
    server = make_server('localhost', 8001, hello)
    print('Serving HTTP on port 8001...')
    server.serve_forever()


if __name__ == '__main__':
    main()
执行python server.py，浏览器打开"http://localhost:8001/a"，即可验证。

通过实现一个简单的WSGI服务，我们可以看到：通过environ可以获取http请求的所有信息，http响应的数据都可以通过start_response加上函数的返回值作为body。

当然，以上只是一个简单的案例，那么在python的Web框架内部是如何遵循WSGI规范的呢？以Flask举例，

Flask与WSGI

Flask中的程序实例app就是一个可调用对象，我们创建app实例时所调用的Flask类实现了__call方法，__call方法调用了wsgi_app()方法，该方法完成了请求和响应的处理，WSGI服务器通过调用该方法传入请求数据，获取返回数据：

def wsgi_app(self, environ, start_response):
    ctx = self.request_context(environ)
    error = None
    try:
        try:
            ctx.push()
            response = self.full_dispatch_request()
        except Exception as e:
            error = e
            response = self.handle_exception(e)
        except:  # noqa: B001
            error = sys.exc_info()[1]
            raise
        return response(environ, start_response)
    finally:
        if self.should_ignore_error(error):
            error = None
        ctx.auto_pop(error)

def __call__(self, environ, start_response):
    return self.wsgi_app(environ, start_response)
Flask的werkzeug库是一个非常优秀的WSGI工具库，具体的实现我们之后再详细学习。
