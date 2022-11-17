# 一个"请求"的路程

打开地图, 我们一定一样, 都是迫不及待地想了解首要一个问题:
一个HTTP请求是如何通过Flask到达我们编写的视图函数的?

这个过程分成两个部分:

一个部分是请求到达HTTP服务器, 被服务器解析包装成Request对象的过程.
第二个部分是该Request对象被路由到我们写的视图函数的过程.

## 获得Request对象

我们从使用Flask时常写的一行启动语句开始

```
app.run() # app为Flask类的对象
```

该方法内调用werkzeug的run_simple函数

Flask类中实现了特殊方法__call__, 也就是Flask对象是一个可调用的对象,
它被调用时执行wsgi_app方法. 所以Flask对象本身是一个wsgi web应用.
所以werkzeug.run_simple方法被调用时传递的参数是Flask对象本身(self).

flask.request_class(environ)

## 路由Request对象到视图函数

路由即是依据HTTP请求的URL, 查找到对应的视图函数.
Flask把我们编写的处理某种url路径请求的函数, 称作视图函数

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
