# 设计Flask

## route

URL rule

endpoint
端点就是程序中一组逻辑处理单元的ID，该ID对应的代码决定了对此ID请求应该作出何种响应。通常，端点与视图函数同名，但是你也可以修改它，例如：

@app.route('/greeting/<name>', endpoint='say_hello')

Endpoint有什么作用

端点通常用作反向查询URL地址（viewfunction-->endpoint-->URL）。例如，在flask中有个视图，你想把它关联到另一个视图上（或从站点的一处连接到另一处）。不用去千辛万苦的写它对应的URL地址，直接使用URL_for()就可以啦：

@app.route('/')
def index():
    print url_for('give_greeting', name='Mark') # 打印出 '/greeting/Mark'

@app.route('/greeting/<name>')
def give_greeting(name):
    return 'Hello, {0}!'.format(name)
备注：url_for()中give_greeting是端点名.
这样做是大有裨益的：我们可以随意改变应用中的URL地址，却不用修改与之关联的资源的代码。

为何要多此一举

那么问题来了：为何要多此一举，为何要先把URL映射到端点上，再通过端点映射到视图函数上，为何不跳过中间的这个步骤？
原因就是采用这种方法能够使程序更高、更快、更强。例如蓝本。蓝本允许我们把应用分割为一个个小的部分，现在admin蓝本中含有超级管理员级的资源，user蓝本中则含有用户一级的资源。
蓝本允许咱们把应用分割为一个个以命名空间区分的小部分：
main.py:

from flask import Flask, Blueprint
from admin import admin
from user import user

app = Flask(__name__)
app.register_blueprint(admin, url_prefix='admin')
app.register_blueprint(user, url_prefix='user')
admin.py:

admin = Blueprint('admin', __name__)

@admin.route('/greeting')
def greeting():
    return 'Hello, administrative user!'
user.py:

user = Blueprint('user', __name__)
@user.route('/greeting')
def greeting():
    return 'Hello, lowly normal user!'
注意，在两个蓝本中路由地址'/greeting'的函数都叫"greeting"。如果我想调用admin对应的greeting函数，我不能说“我想要greeting”，因为这里还有一个user对应的greeting函数。端点这时就发挥作用了：指定一个蓝本名称作为端点的一部分--通过这种方式端点实现了对命名空间的支持。所以，我们可以这样写：

print url_for('admin.greeting') # Prints '/admin/greeting'
print url_for('user.greeting') # Prints '/user/greeting'



.. code-block:: python

            @app.route("/")
            def index():
                ...

        .. code-block:: python

            def index():
                ...

            app.add_url_rule("/", view_func=index)


URL Route Registrations
-----------------------

Generally there are three ways to define rules for the routing system:

1.  You can use the :meth:`flask.Flask.route` decorator.
2.  You can use the :meth:`flask.Flask.add_url_rule` function.
3.  You can directly access the underlying Werkzeug routing system
    which is exposed as :attr:`flask.Flask.url_map`.

Variable parts in the route can be specified with angular brackets
(``/user/<username>``). By default a variable part in the URL accepts any
string without a slash however a different converter can be specified as
well by using ``<converter:name>``.

Variable parts are passed to the view function as keyword arguments.

The following converters are available:

=========== ===============================================
`string`    accepts any text without a slash (the default)
`int`       accepts integers
`float`     like `int` but for floating point values
`path`      like the default but also accepts slashes
`any`       matches one of the items provided
`uuid`      accepts UUID strings
=========== ===============================================

Custom converters can be defined using :attr:`flask.Flask.url_map`.

Here are some examples::

    @app.route('/')
    def index():
        pass

    @app.route('/<username>')
    def show_user(username):
        pass

    @app.route('/post/<int:post_id>')
    def show_post(post_id):
        pass

An important detail to keep in mind is how Flask deals with trailing
slashes. The idea is to keep each URL unique so the following rules
apply:

1. If a rule ends with a slash and is requested without a slash by the
   user, the user is automatically redirected to the same page with a
   trailing slash attached.
2. If a rule does not end with a trailing slash and the user requests the
   page with a trailing slash, a 404 not found is raised.

This is consistent with how web servers deal with static files. This
also makes it possible to use relative link targets safely.

You can also define multiple rules for the same function. They have to be
unique however. Defaults can also be specified. Here for example is a
definition for a URL that accepts an optional page::

    @app.route('/users/', defaults={'page': 1})
    @app.route('/users/page/<int:page>')
    def show_users(page):
        pass

This specifies that ``/users/`` will be the URL for page one and
``/users/page/N`` will be the URL for page ``N``.

If a URL contains a default value, it will be redirected to its simpler
form with a 301 redirect. In the above example, ``/users/page/1`` will
be redirected to ``/users/``. If your route handles ``GET`` and ``POST``
requests, make sure the default route only handles ``GET``, as redirects
can't preserve form data. ::

   @app.route('/region/', defaults={'id': 1})
   @app.route('/region/<int:id>', methods=['GET', 'POST'])
   def region(id):
      pass

Here are the parameters that :meth:`~flask.Flask.route` and
:meth:`~flask.Flask.add_url_rule` accept. The only difference is that
with the route parameter the view function is defined with the decorator
instead of the `view_func` parameter.

=============== ==========================================================
`rule`          the URL rule as string
`endpoint`      the endpoint for the registered URL rule. Flask itself
                assumes that the name of the view function is the name
                of the endpoint if not explicitly stated.
`view_func`     the function to call when serving a request to the
                provided endpoint. If this is not provided one can
                specify the function later by storing it in the
                :attr:`~flask.Flask.view_functions` dictionary with the
                endpoint as key.
`defaults`      A dictionary with defaults for this rule. See the
                example above for how defaults work.
`subdomain`     specifies the rule for the subdomain in case subdomain
                matching is in use. If not specified the default
                subdomain is assumed.
`**options`     the options to be forwarded to the underlying
                :class:`~werkzeug.routing.Rule` object. A change to
                Werkzeug is handling of method options. methods is a list
                of methods this rule should be limited to (``GET``, ``POST``
                etc.). By default a rule just listens for ``GET`` (and
                implicitly ``HEAD``). Starting with Flask 0.6, ``OPTIONS`` is
                implicitly added and handled by the standard request
                handling. They have to be specified as keyword arguments.
=============== ==========================================================


## 拦截器
## debug模式

## 内嵌开发server
