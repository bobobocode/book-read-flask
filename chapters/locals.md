# 上下文局部变量

在处理一个web请求的过程中, 一些数据需要跨越多个函数进行处理或者使用,
比如鉴权通过后得到的用户信息数据.

这些数据作为参数层层传递将会造成增加程序的复杂度, 所以我们此时需要以全
局数据的方式使用它们. 但这样无法保障线程安全, 不同的处理者可能会彼此干扰对方的数据.

我们可以让这些全局变量中的数据以线程本地数据的方式进行存储和获取. Python内置的threading.local可以很方便地管理线程本地数据. 类似的有Java中的ThreadLocal.

在Python的标准库中提供了thread local对象用于存储thread-safe和thread-specific的数据，通过这种方式存储的数据只在本线程中有效，而对于其它线程则不可见。正是基于这样的特性，我们可以把针对线程全局的数据存储进thread local对象，举个简单的例子

```
from threading import local
thread_local_data = local()
thread_local_data.user_name="Jim"
thread_local_data.user_name
'Jim'
```
使用thread local对象虽然可以基于线程存储全局变量，但是在Web应用中可能会存在如下问题：

有些应用使用的是greenlet协程，这种情况下无法保证协程之间数据的隔离，因为不同的协程可以在同一个线程当中。
即使使用的是线程，WSGI应用也无法保证每个http请求使用的都是不同的线程，因为后一个http请求可能使用的是之前的http请求的线程，这样的话存储于thread local中的数据可能是之前残留的数据。


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

Local


```
# 在有greenlet的情况下，get_indent实际获取的是greenlet的id，而没有greenlet的情况下获取的是thread id
try:
    from greenlet import getcurrent as get_ident
except ImportError:
    try:
        from thread import get_ident
    except ImportError:
        from _thread import get_ident

class Local(object):
    __slots__ = ('__storage__', '__ident_func__')

    def __init__(self):
        object.__setattr__(self, '__storage__', {})
        object.__setattr__(self, '__ident_func__', get_ident)

    def __iter__(self):
        return iter(self.__storage__.items())

    # 当调用Local对象时，返回对应的LocalProxy
    def __call__(self, proxy):
        """Create a proxy for a name."""
        return LocalProxy(self, proxy)

    # Local类中特有的method，用于清空greenlet id或线程id对应的dict数据
    def __release_local__(self):
        self.__storage__.pop(self.__ident_func__(), None)

    def __getattr__(self, name):
        try:
            return self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)

    def __setattr__(self, name, value):
        ident = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            storage[ident] = {name: value}

    def __delattr__(self, name):
        try:
            del self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)

```
这段代码实际是对__storage__ dict的封装，而这个dict中的key使用的就是get_indent函数获取的id（当有greenlet时使用greenlet id，没有则使用thread id）
__storage__ dict中的value也是一个dict，这个dict就是该greenlet(或者线程)对应的local存储空间
通过重新实现__getattr__, __setattr__等魔术方法，我们在greenlet或者线程中使用local对象时，实际会自动获取greenlet id(或者线程id)，从而获取到对应的dict存储空间，再通过name key就可以获取到真正的存储的对象。这个技巧实际上在编写线程安全或协程安全的代码时是非常有用的，即通过线程id(或协程id)来分别存储数据。
当我们需要释放local数据的内存时，可以通过调用release_local()函数来释放当前context的local数据，如下

```
>>> loc = Local()
>>> loc.foo = 42
>>> release_local(loc)  # release_local实际调用local对象的__release_local__ method
>>> hasattr(loc, 'foo')
False
```

LocalStack

LocalStack与Local对象类似，都是可以基于Greenlet协程或者线程进行全局存储的存储空间(实际LocalStack是对Local进行了二次封装），区别在于其数据结构是栈的形式。示例如下：

>>> ls = LocalStack()
>>> ls.push(42)
>>> ls.top
42
>>> ls.push(23)
>>> ls.top
23
>>> ls.pop()
23
>>> ls.top
42
从示例看出Local对象存储的时候是类似字典的方式，需要有key和value，而LocalStack是基于栈的，通过push和pop来存储和弹出数据
另外，当我们想释放存储空间的时候，也可以调用release_local()
LocalStack在Flask框架中会频繁的出现，其Request Context和App Context的实现都是基于LocalStack，具体可以参考Github上的Flask源码

LocalProxy

LocalProxy用于代理Local对象和LocalStack对象，而所谓代理就是作为中间的代理人来处理所有针对被代理对象的操作，如下图所示：


proxy.jpg
接下来我们将重点讲下如下内容：

LocalProxy的使用
LocalProxy代码解析
为什么要使用LocalProxy
LocalProxy的使用

初始化LocalProxy有三种方式：

通过Local或者LocalStack对象的__call__ method
from werkzeug.local import Local
l = Local()

# these are proxies
request = l('request')
user = l('user')


from werkzeug.local import LocalStack
_response_local = LocalStack()

# this is a proxy
response = _response_local()
上述代码直接将对象像函数一样调用，这是因为Local和LocalStack都实现了__call__ method，这样其对象就是callable的，因此当我们将对象作为函数调用时，实际调用的是__call__ method，可以看下本文开头部分的Local的源代码，会发现__call__ method会返回一个LocalProxy对象

通过LocalProxy类进行初始化
l = Local()
request = LocalProxy(l, 'request')
实际上这段代码跟第一种方式是等价的，但这种方式是最'原始'的方式，我们在Local的源代码实现中看到其__call__ method就是通过这种方式生成LocalProxy的

使用callable对象作为参数
request = LocalProxy(get_current_request())
通过传递一个函数，我们可以自定义如何返回Local或LocalStack对象

那么LocalProxy是如何实现这种代理的呢？接下来看下源码解析

LocalProxy代码解析

下面截取LocalProxy的部分代码，我们来进行解析

# LocalProxy部分代码

@implements_bool
class LocalProxy(object):
    __slots__ = ('__local', '__dict__', '__name__', '__wrapped__')

    def __init__(self, local, name=None):
        object.__setattr__(self, '_LocalProxy__local', local)
        object.__setattr__(self, '__name__', name)
        if callable(local) and not hasattr(local, '__release_local__'):
            # "local" is a callable that is not an instance of Local or
            # LocalManager: mark it as a wrapped function.
            object.__setattr__(self, '__wrapped__', local)

    def _get_current_object(self):
        """Return the current object.  This is useful if you want the real
        object behind the proxy at a time for performance reasons or because
        you want to pass the object into a different context.
        """
        # 由于所有Local或LocalStack对象都有__release_local__ method, 所以如果没有该属性就表明self.__local为callable对象
        if not hasattr(self.__local, '__release_local__'):
            return self.__local()
        try:
            # 此处self.__local为Local或LocalStack对象
            return getattr(self.__local, self.__name__)
        except AttributeError:
            raise RuntimeError('no object bound to %s' % self.__name__)

    @property
    def __dict__(self):
        try:
            return self._get_current_object().__dict__
        except RuntimeError:
            raise AttributeError('__dict__')

    def __getattr__(self, name):
        if name == '__members__':
            return dir(self._get_current_object())
        return getattr(self._get_current_object(), name)

    def __setitem__(self, key, value):
        self._get_current_object()[key] = value

    def __delitem__(self, key):
        del self._get_current_object()[key]

    if PY2:
        __getslice__ = lambda x, i, j: x._get_current_object()[i:j]

        def __setslice__(self, i, j, seq):
            self._get_current_object()[i:j] = seq

        def __delslice__(self, i, j):
            del self._get_current_object()[i:j]

    # 截取部分操作符代码
    __setattr__ = lambda x, n, v: setattr(x._get_current_object(), n, v)
    __delattr__ = lambda x, n: delattr(x._get_current_object(), n)
    __str__ = lambda x: str(x._get_current_object())
    __lt__ = lambda x, o: x._get_current_object() < o
    __le__ = lambda x, o: x._get_current_object() <= o
    __eq__ = lambda x, o: x._get_current_object() == o
首先在__init__ method中传递的local参数会被赋予属性_LocalProxy__local,该属性可以通过self.__local进行访问，关于这一点可以看StackOverflow的问题回答
LocalProxy通过_get_current_object来获取代理的对象。需要注意的是当初始化参数为callable对象时，则直接调用以返回Local或LocalStack对象，具体看源代码的注释。
重载了绝大多数操作符，以便在调用LocalProxy的相应操作时，通过_get_current_object method来获取真正代理的对象，然后再进行相应操作
为什么要使用LocalProxy

可是说了这么多，为什么一定要用proxy，而不能直接调用Local或LocalStack对象呢？这主要是在有多个可供调用的对象的时候会出现问题，如下图：


multiple objects
我们再通过下面的代码也许可以看出一二：

# use Local object directly
from werkzeug.local import LocalStack
user_stack = LocalStack()
user_stack.push({'name': 'Bob'})
user_stack.push({'name': 'John'})

def get_user():
    # do something to get User object and return it
    return user_stack.pop()


# 直接调用函数获取user对象
user = get_user()
print user['name']
print user['name']
打印结果是：

John
John
再看下使用LocalProxy

# use LocalProxy
from werkzeug.local import LocalStack, LocalProxy
user_stack = LocalStack()
user_stack.push({'name': 'Bob'})
user_stack.push({'name': 'John'})

def get_user():
    # do something to get User object and return it
    return user_stack.pop()

# 通过LocalProxy使用user对象
user = LocalProxy(get_user)
print user['name']
print user['name']
打印结果是：

John
Bob
怎么样，看出区别了吧，直接使用LocalStack对象，user一旦赋值就无法再动态更新了，而使用Proxy，每次调用操作符(这里[]操作符用于获取属性)，都会重新获取user，从而实现了动态更新user的效果。见下图：


proxy auto select object
Flask以及Flask的插件很多时候都需要这种动态更新的效果，因此LocalProxy就会非常有用了。



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
