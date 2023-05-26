应用程序上下文
==============

在一个请求或者CLI命令等活动的过程中,应用程序上下文保持跟踪应用程序层面的数据.
应用程序上下文使得这些数据可以通过访问:data:`current_app` 和:data:`g` 的代理来使用, 
而无需向每个函数传递应用程序对象.

和:doc:`/reqcontext` 类似, 能够在一个请求过程中保持跟踪请求层面的数据.
当一个请求上下文被推出时, 一个相应的应用程序上下文也被推出.

上下文的目标
------------

类:class:`Flask` 应用程序对象具有很多属性,  像属性:attr:`~Flask.config`, 
能够用于接入视图和:doc:`CLI commands </cli>`. 

However, importing the ``app`` instance
within the modules in your project is prone to circular import issues.
When using the :doc:`app factory pattern </patterns/appfactories>` or
writing reusable :doc:`blueprints </blueprints>` or
:doc:`extensions </extensions>` there won't be an ``app`` instance to
import at all.

Flask solves this issue with the *application context*. Rather than
referring to an ``app`` directly, you use the :data:`current_app`
proxy, which points to the application handling the current activity.

Flask automatically *pushes* an application context when handling a
request. View functions, error handlers, and other functions that run
during a request will have access to :data:`current_app`.

Flask will also automatically push an app context when running CLI
commands registered with :attr:`Flask.cli` using ``@app.cli.command()``.


Lifetime of the Context
-----------------------

The application context is created and destroyed as necessary. When a
Flask application begins handling a request, it pushes an application
context and a :doc:`request context </reqcontext>`. When the request
ends it pops the request context then the application context.
Typically, an application context will have the same lifetime as a
request.

See :doc:`/reqcontext` for more information about how the contexts work
and the full life cycle of a request.


Manually Push a Context
-----------------------

If you try to access :data:`current_app`, or anything that uses it,
outside an application context, you'll get this error message:

.. code-block:: pytb

    RuntimeError: Working outside of application context.

    This typically means that you attempted to use functionality that
    needed to interface with the current application object in some way.
    To solve this, set up an application context with app.app_context().

If you see that error while configuring your application, such as when
initializing an extension, you can push a context manually since you
have direct access to the ``app``. Use :meth:`~Flask.app_context` in a
``with`` block, and everything that runs in the block will have access
to :data:`current_app`. ::

    def create_app():
        app = Flask(__name__)

        with app.app_context():
            init_db()

        return app

If you see that error somewhere else in your code not related to
configuring the application, it most likely indicates that you should
move that code into a view function or CLI command.


Storing Data
------------

The application context is a good place to store common data during a
request or CLI command. Flask provides the :data:`g object <g>` for this
purpose. It is a simple namespace object that has the same lifetime as
an application context.

.. note::
    The ``g`` name stands for "global", but that is referring to the
    data being global *within a context*. The data on ``g`` is lost
    after the context ends, and it is not an appropriate place to store
    data between requests. Use the :data:`session` or a database to
    store data across requests.

A common use for :data:`g` is to manage resources during a request.

1.  ``get_X()`` creates resource ``X`` if it does not exist, caching it
    as ``g.X``.
2.  ``teardown_X()`` closes or otherwise deallocates the resource if it
    exists. It is registered as a :meth:`~Flask.teardown_appcontext`
    handler.

For example, you can manage a database connection using this pattern::

    from flask import g

    def get_db():
        if 'db' not in g:
            g.db = connect_to_database()

        return g.db

    @app.teardown_appcontext
    def teardown_db(exception):
        db = g.pop('db', None)

        if db is not None:
            db.close()

During a request, every call to ``get_db()`` will return the same
connection, and it will be closed automatically at the end of the
request.

You can use :class:`~werkzeug.local.LocalProxy` to make a new context
local from ``get_db()``::

    from werkzeug.local import LocalProxy
    db = LocalProxy(get_db)

Accessing ``db`` will call ``get_db`` internally, in the same way that
:data:`current_app` works.


Events and Signals
------------------

The application will call functions registered with
:meth:`~Flask.teardown_appcontext` when the application context is
popped.

If :data:`~signals.signals_available` is true, the following signals are
sent: :data:`appcontext_pushed`, :data:`appcontext_tearing_down`, and
:data:`appcontext_popped`.
