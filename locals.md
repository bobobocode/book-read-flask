# 上下文局部变量

在处理一个web请求的过程中, 一些数据需要跨越多个函数进行处理或者使用,
比如鉴权通过后得到的用户信息数据.

这些数据作为参数层层传递将会造成增加程序的复杂度, 所以我们此时需要以全
局数据的方式使用它们. 但这样无法保障线程安全, 不同的处理者可能会彼此干扰对方的数据.

我们可以让这些全局变量中的数据以线程本地数据的方式进行存储和获取. Python内置的threading.local可以很方便地管理线程本地数据. 类似的有Java中的ThreadLocal.

但是当我们使用异步任务或者协程来执行一个请求的处理过程时, 不能保证每个请求都有自己的线程。可能是一个请求正在重用上一个请求中的线程，因此数据留在线程本地对象中, 不能保证对于请求上下文形成数据隔离.

所以更进一步地, 我们要使用一种可以叫做上下文局部变量的方式来管理这类数
据.
上下文局部变量可以全局性地定义或者导入, 但是它所包含的数据是特定于当前线程、异步任务或者协程的. 这样就防止了不小心获取或者覆盖了其它请求处理者的数据.

在werkzeug包中, 已经包含了上下文局部变量的相关对象.
Flask充分使用了它们.

所以我们要先介绍一下werkzeug的local模块.

当前Python内置的contextvars模块提供了保存每个上下文数据的方法.
该模块相对于早前的threading.local模块, 可以存储每个线程、异步任务和协
程的的数据.

Werkzeug中的local模块提供了对ContextVar的封装, 使之更容易使用.

## Werkzeug LocalProxy

* ContextVar

* LocalProxy

Werkzeug提供的LocalProxy类允许像一个对象一样直接使用上下文局部变量, 而
不是像Python内置的ContextVar那样需要使用get()方法.
如果一个上下文局部变量被设置了, 它的本地代理会看起来和用起来像一个普通
的对象变量被设置了一样.
如果没有被设置, 则对于大多数操作会抛出RuntimeError异常.

```
from contextvars import ContextVar
from werkzeug.local import LocalProxy

_request_var = ContextVar("request")
request = LocalProxy(_request_var)

from werkzeug.wrappers import Request

@Request.application
def app(r):
    _request_var.set(r)
    check_auth()
    ...

from werkzeug.exceptions import Unauthorized

def check_auth():
    if request.form["username"] != "admin":
        raise Unauthorized()
```


ContextVar一次存储一个变量. 你会发现你需要存储一堆变量, 或者在一个命名
空间下的
多个属性.
虽然用列表和字典可以实现目的, 但是在上下文局部变量中使用它们需要需要额
外的注意事项.
Werkzeug提供了LocalStack用于封装一个列表, 而Local类用于封装一个字典.
和这些对象关联的了许多性能问题. 因为列表和字典是可变对象.
LocalStack和Local需要做更多工作来确保数据没有被嵌套的上下文中共享.
尽可能的设计你的程序直接使用LocalProxy.


释放数据
