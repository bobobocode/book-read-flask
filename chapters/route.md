# 一个"请求"的路程

.. flow::  
   :width: 500  
   :height: 400  
   
   st=>start: Start  
   e=>end: End  
   op1=>operation: Operation 1  
   op2=>operation: Operation 2  
   
   st->op1->op2->e  



Flask应用的HTTP请求处理过程可以分为以下几个步骤：

1. 接收请求

Flask应用首先需要接收到HTTP请求。这个过程是由Python的内置包socket和werkzeug库中的BaseWSGIServer和WSGIRequestHandler实现的。其中，socket库负责创建服务器套接字，BaseWSGIServer负责监听套接字并获取客户端连接，WSGIRequestHandler负责解析请求头和请求体，并构造environ字典作为参数传递给Flask应用处理。

2. 路由匹配

当Flask应用接收到请求后，会根据路由规则匹配对应的视图函数。这个过程是由Flask中的路由系统实现的。路由系统会根据URL路径、HTTP方法和其他条件来匹配对应的视图函数。

3. 视图函数处理

一旦路由匹配成功，Flask就会调用对应的视图函数来处理请求。视图函数可以访问environ字典中包含的所有信息，包括URL参数、查询参数、请求头和请求体等。

4. 响应生成

在视图函数中处理完业务逻辑后，需要把结果返回给客户端。这个过程是通过构造响应对象实现的。响应对象可以设置状态码、响应头和响应体等信息，并最终将整个响应发送回客户端。

5. 响应发送

最后，Flask应用会调用BaseWSGIServer和WSGIRequestHandler中的方法将响应发送回客户端。这个过程是通过socket套接字实现的，其中包括响应头和响应体的发送。

总的来说，Flask应用的HTTP请求处理过程是一个由多个组件协同工作完成的复杂过程。理解这个过程对于深入了解Flask框架和Web开发非常重要。

当一个HTTP请求到达Flask应用时，它会先被Flask内置的Werkzeug Server接收。接着，Werkzeug会将请求交给Flask应用来处理。

Flask应用首先会从请求中解析出HTTP方法（GET、POST等）、URL、HTTP头和请求体等信息。接着，它会查找一个与URL匹配的路由，找到对应的视图函数进行处理。

视图函数会处理请求并返回一个HTTP响应，响应中包含了HTTP状态码、HTTP头和响应体等信息。Flask应用将响应发送回Werkzeug Server，最终由Server将响应发送回客户端。

以下是HTTP请求的处理过程示意图：

HTTP Request Handling in Flask

其中，客户端发送的HTTP请求首先被Werkzeug Server接收。接着，Werkzeug将请求交给Flask应用处理。Flask应用会查找对应的路由，并将请求交给路由对应的视图函数处理。视图函数返回HTTP响应，Flask应用将其发送回Werkzeug Server，最终由Server将响应发送回客户端。

我们先看 Flask 官方提供的最基本最微型的使用示例：

```
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello_world():
    return "<p>Hello, World!</p>"

if __name__ == "__main__":
    app.run()
```

这段代码展示了 Flask 最高层次的设计抽象，即使用 Flask 构建 Web 应用的最基本形式。在这个例子中，我们定义了一个路由为“/”的视图函数 hello_world()，当用户访问该路由时，该函数将被执行，将其返回值作为 HTTP 响应返回。

接下来，我们将从 HTTP 请求的处理过程的角度，解析 Flask 的主高速公路。整个处理过程分为两个部分：

第一个部分是请求到达 HTTP 服务器，被服务器解析包装成 Request 对象的过程；
第二个部分是该 Request 对象被路由到我们编写的视图函数的过程。
接下来，我们分别看一下这两个过程。

解析 HTTP 请求构造 Request 对象
首先，我们来看启动 Flask 应用时经常用到的一行代码：

```
if __name__ == "__main__":
    app.run()
```

该语句会调用 Werkzeug 库的 run_simple 函数，该函数会构建一个 WSGI 服务器并启动。这个 WSGI 服务器是 Werkzeug 自己定义的 BaseWSGIServer，它基于 Python 内置的 HTTPServer 模板，通过自定义 handler 将请求和响应处理转换成 WSGI 标准。最后，它调用 Flask 实例的 wsgi_app 方法，将其作为 WSGI 应用传递给 BaseWSGIServer。因此，Flask 实例本身就是一个 WSGI 应用。

在 wsgi_app 方法内部，Flask 使用 request_class(environ) 构造一个 werkzeug.Request 对象，并将其返回。这个 request_class 方法会将 HTTP 请求中的环境变量 environ 转化为一个 wrapper.Request 对象，再使用 werkzeug.Request 对象对其进行封装。

此外，为了方便用户在一个请求上下文中访问相关信息，Flask 还使用 ctx.RequestContext 构建了一个请求上下文，并为每个请求绑定了一个 url_adapter 对象和一个 match_request 函数。

综上所述，这个过程分为三个层次：

Python 内置的 HTTPServer 基于 BaseServer 形成了运行 Web 服务器的调用框架；
Werkzeug 继承下来这个调用关系，更具体地定义了请求解析转换和调用 Web 应用的流程；
Flask 定义了 Web 应用的基本形式，并启动 Werkzeug 的服务器运行机制，将自身对象传递给 run_simple
接下来, 我们分别看一下这两个过程.

## 解析HTTP请求构造Request对象

我们从使用Flask时常写的一行启动语句开始

该方法内调用werkzeug的run_simple函数

run_simple构建wsgi server, 并且启动server的serve_forever函数

这个wsgi server是werkzeug自己定义的BaseWSGIServer, 它基于Python内置的
HTTPServer模板, 通过自定义handler将请求和响应处理转换成wsig标准. 并调
用参数app, 该参数只需是一个wsgi函数.

Flask类中实现了特殊方法__call__, 也就是Flask对象是一个可调用的对象,
它被调用时执行wsgi_app方法. 所以Flask对象本身是一个wsgi web应用.
所以werkzeug.run_simple方法被调用时传递的参数是Flask对象本身(self).

flask.request_class(environ) -> wrapper.Request -> werkzeug.Request

ctx.RequestContext
    url_adapter = Flask create_url_adapter
    match_request

所以这里分为三个层次
Python内置的HttpServer基于BaseServer形成了运行web服务器的调用框架
werkzeug继承下来这个调用关系, 更具体定义了请求解析转换和调用Web App
Flask定义了Web App的基本形式, 并启动werkzeug的server运行机制, 将自身对
象self传递进去.

## 路由Request对象到视图函数

路由即是依据HTTP请求的URL, 查找到对应的视图函数.

1. 构建请求路径与视图函数的对应关系

要告诉Flask URL与视图函数的对应规则, Flask提供的便捷的方法是在视图函数
上加route装饰器, 装饰器的参数即为URL规则.

装饰器在脚本被加载时执行, 则这种对应信息被保存在Flask对象中的两个属性
中.
这两个属性首先一个保存URL规则到endpoint的对应关系, 另一个保存endpoint
到视图函数的对应关系

其中endpoint如果没有指定的话, 默认为函数名称

之所以用endpoint做一次中介传递对应关系, 是因为Blueprint的设计概念.

Flask使用Blueprint蓝图概念, 用于模块化组织代码.
同一个工程中, 不同的Blueprint下的视图函数, 在url规则对应上
形成独立的区域, 像命名空间一样隔离.
这样就可以对这批视图函数设置独立的权限控制等, 实则在一个工程下管理多个
App.
常见的用法, 可以在一个互联网服务App中包含其对应数据的后台管理App

当定义了Blueprint蓝图之后, 不同的蓝图下可以独立定义各种的视图函数,
对于同名函数, 它们的endpoint是以蓝图名称和函数名称共同定义的, 所以
endpoint名称不同.


