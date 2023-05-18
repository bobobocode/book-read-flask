# 本地变量


## 为什么会有这么一个问题

如果你理解Java中的ThreadLocal可以跳过以下讲解.

在一个HTTP请求的处理过程中, 一些数据需要经过不同层的多个函数进行处理或者使用.
比如在拦截器获取鉴权后得到的用户信息数据, 在控制器层的函数要使用进行业务处理.

如果把这些数据作为参数, 层层传递, 使用起来非常不便.

我们希望可以用像全局变量那样的方式存取它们, 则需要保障线程安全.
防止多线程情况下, 产生彼此数据的干扰.

所以我们应该让这些全局变量形式的数据以线程本地数据的方式进行存取.

## 使用什么技术来解决

Python内置的threading.local就是来实现这个目的. 
同一个线程中的函数可以像使用全局变量一样存取thread.local管理的数据, 
而无需担心被其它线程中的处理扰乱.

我们看一下Python中的实现.

```
class local:
    __slots__ = '_local__impl', '__dict__'

    def __new__(cls, /, *args, **kw):
        if (args or kw) and (cls.__init__ is object.__init__):
            raise TypeError("Initialization arguments are not supported")
        self = object.__new__(cls)
        impl = _localimpl()
        impl.localargs = (args, kw)
        impl.locallock = RLock()
        object.__setattr__(self, '_local__impl', impl)
        # 我们需要在__init__将要被调用之前创建一个线程字典
        # 以确保我们不会自己再次调用它
        impl.create_dict()
        return self

    def __getattribute__(self, name):
        with _patch(self):
            return object.__getattribute__(self, name)

    def __setattr__(self, name, value):
        if name == '__dict__':
            raise AttributeError(
                "%r object attribute '__dict__' is read-only"
                % self.__class__.__name__)
        with _patch(self):
            return object.__setattr__(self, name, value)

    def __delattr__(self, name):
        if name == '__dict__':
            raise AttributeError(
                "%r object attribute '__dict__' is read-only"
                % self.__class__.__name__)
        with _patch(self):
            return object.__delattr__(self, name)

```
```

我们需要使用 threading 模块中的对象, 但是 threading 模块也可能想要使用我们的 local 类.
如果 thread 模块中没有编译对 locals 的支持。这会产生循环导入的潜在问题。
为此, 我们直到这个文件的底部才导入 threading(这是一种足够巧妙地绕过潜在问题的方法).
请注意, 在 CPython 的所有平台上, thread 模块都支持 locals, 因此没有循环导入问题,
因此在此处调整导入顺序引入的问题不会显现。

class _localimpl:
    """A class managing thread-local dicts"""
    __slots__ = 'key', 'dicts', 'localargs', 'locallock', '__weakref__'

    def __init__(self):
        # The key used in the Thread objects' attribute dicts.
        # We keep it a string for speed but make it unlikely to clash with
        # a "real" attribute.
        self.key = '_threading_local._localimpl.' + str(id(self))
        # { id(Thread) -> (ref(Thread), thread-local dict) }
        self.dicts = {}

    def get_dict(self):
        """Return the dict for the current thread. Raises KeyError if none
        defined."""
        thread = current_thread()
        return self.dicts[id(thread)][1]

    def create_dict(self):
        """Create a new dict for the current thread, and return it."""
        localdict = {}
        key = self.key
        thread = current_thread()
        idt = id(thread)
        def local_deleted(_, key=key):
            # When the localimpl is deleted, remove the thread attribute.
            thread = wrthread()
            if thread is not None:
                del thread.__dict__[key]
        def thread_deleted(_, idt=idt):
            # When the thread is deleted, remove the local dict.
            # Note that this is suboptimal if the thread object gets
            # caught in a reference loop. We would like to be called
            # as soon as the OS-level thread ends instead.
            local = wrlocal()
            if local is not None:
                dct = local.dicts.pop(idt)
        wrlocal = ref(self, local_deleted)
        wrthread = ref(thread, thread_deleted)
        thread.__dict__[key] = wrlocal
        self.dicts[idt] = wrthread, localdict
        return localdict

```



通过这种方式存储的数据只在本线程中有效，而对于其它线程则不可见
使用thread local对象虽然可以基于线程存储全局变量，但是在Web应用中可能会存在如下问题：

有些应用使用的是greenlet协程，这种情况下无法保证协程之间数据的隔离，因为不同的协程可以在同一个线程当中。
即使使用的是线程，WSGI应用也无法保证每个http请求使用的都是不同的线程，因为后一个http请求可能使用的是之前的http请求的线程，这样的话存储于thread local中的数据可能是之前残留的数据。


但是当我们使用异步任务或者协程来执行一个请求的处理过程时, 不能保证每个请求都有自己的线程。可能是一个请求正在重用上一个请求中的线程，因此数据留在线程本地对象中, 不能保证对于请求上下文形成数据隔离.

所以更进一步地, 我们要使用一种可以叫做上下文本地变量的方式来管理这类数
据.
上下文本地变量可以全局性地定义或者导入, 但是它所包含的数据是特定于当前线程、异步任务或者协程的. 这样就防止了不小心获取或者覆盖了其它请求处理者的数据.

在werkzeug包中, 已经包含了上下文本地变量的相关对象.
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

```
>>> loc = Local()
>>> loc.foo = 42
>>> release_local(loc)  # release_local实际调用local对象的__release_local__ method
>>> hasattr(loc, 'foo')
False
```

LocalStack

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

LocalProxy

```
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

```

Werkzeug提供的LocalProxy类允许像一个对象一样直接使用上下文本地变量, 而
不是像Python内置的ContextVar那样需要使用get()方法.
如果一个上下文本地变量被设置了, 它的本地代理会看起来和用起来像一个普通
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
虽然用列表和字典可以实现目的, 但是在上下文本地变量中使用它们需要需要额
外的注意事项.
Werkzeug提供了LocalStack用于封装一个列表, 而Local类用于封装一个字典.
和这些对象关联的了许多性能问题. 因为列表和字典是可变对象.
LocalStack和Local需要做更多工作来确保数据没有被嵌套的上下文中共享.
尽可能的设计你的程序直接使用LocalProxy.
