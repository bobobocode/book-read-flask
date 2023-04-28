Flask的命令行功能使用了第三方模块Click来实现.
使用Click的注解可以快速将一个普通函数变成命令行接口.

看一下Click官方文档中的例子, 非常简单便捷
```
import click

@click.command()
@click.option("--count", default=1, help="Number of greetings.")
@click.option("--name", prompt="Your name",
              help="The person to greet.")
def hello(count, name):
    """Simple program that greets NAME for a total of COUNT times."""
    for _ in range(count):
        click.echo("Hello, %s!" % name)

if __name__ == '__main__':
    hello()
```
其中在hello函数上, 使用 @click.command() 装饰一个函数, 使之成为命令行接口;
使用 @click.option() 装饰函数, 为其添加命令行选项.
该脚本执行起来如下:
```
$ python hello.py --count=3
Your name: Click
Hello, Click!
Hello, Click!
Hello, Click!
```

Flask用到的Click中一个特别重要的功能, 就是可任意嵌套命令行工具的概念.
也就是实现我们在命令行中经常用到的子命令.这个能力是通过Command类和Group类设计实现的.
还有MultiCommand

Flask的命令行接口
=================

安装Flask时，会在你的虚拟环境中安装一个***flask***脚本，该脚本是一个
***Click***命令行接口。
从终端执行该脚本，可以使用内置的可扩展的面向应用程序的命令。
使用***--help***选项能给出关于所有命令和选项的更多信息


应用程序发现
------------

***flask***命令是由Flask安装的，而不是你的应用程序。
它为了使用你的应用程序而必须被告知如何找到它。
***--app***选项用于指定如何去加载应用程序。
具体来说就是去哪里找到名为***app***或者***application***应用程序实例,
或者名为***create_app***或***make_app***的能够返回应用程序实例的工厂函数。

该选项可以使用多种取值方式，它的取值包含三个部分:
1. 一个可选的设置当前工作目录的路径；
2. 一个Python文件或者导入模块路径；
3. 一个可选的实例对象或者工厂函数(可以带参数,将按字面意义解释)的名字;

如果没有设置改选项的值, 命令会尝试导入***app***或者***wsgi***, 并尝试
去检测一个应用程序实例或者工厂函数.


运行开发服务器
--------------

***run***命令将启动一个开发服务器.
如下所示:

```sh
$ flask --app hello run
 * Serving Flask app "hello"
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```

需要注意的是:
    不要在生产环境使用该server, 因为它只提供开发便捷性, 从设计上不支持
    用于生产环境所需要的特定的安全、稳定性和性能保障.


调试模式
--------


在调试模式, ***flask run***命令将默认启用交互式调试器和热部署器.
查看和调试错误都变得容易
使用***--debug***的选项来启用调试模式


```sh
$ flask --app hello --debug run
* Serving Flask app "hello"
* Debug mode: on
* Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
* Restarting with inotify reloader
* Debugger is active!
* Debugger PIN: 223-456-919
```


让热部署器监视和忽略文件
-----------------------

当使用调试模式时, 当你的Python代码或者导入模块变化时将会触发热部署器.
通过***--extra-files***选项, 热部署器可以附加监视文件.
而通过***--exclude-patterns***选项可忽略监控一些文件.

```sh
$ flask run --extra-files file1:dirA/file2:dirB/
 * Running on http://127.0.0.1:8000/
 * Detected change in '/path/to/file1', reloading
```


打开Shell
---------

为探查你应用程序的数据, 你可以使用***shell***命令打开一个交互的Python shell.
一个应用程序的上下文将会被激活, 并且应用程序的实例将会在这个shell环境
中导入.

```sh
$ flask shell
Python 3.10.0 (default, Oct 27 2021, 06:59:51) [GCC 11.1.0] on linux
App: example [production]
Instance: /home/david/Projects/pallets/flask/instance
>>>
```

使用方法***Flask.shell_context_processor***可以添加其它动态的导入.


使用dotenv获取的环境变量
------------------------

***flask***命令支持使用环境变量设置任意命令的任意选项.
这些环境变量命名像这样:

***FLASK_OPTION***或者***FLASK_COMMAND_OPTION***, 
例如***FLASK_APP***或者***FLASK_RUN_PORT***.

你可以使用Flask的dotenv支持来自动化设置环境变量, 而不是每次运行命令时
传递选项值或者每次打开一个新的终端重新设置环境变量.

如果安装了***python-dotenv***, 运行***flask***命令将会按文件***.env***
和***.flaskenv***中的定义设置环境变量.

你也可以使用选项***--env-file***来指定一个额外的文件加载环境变量.

Dotenv files can be used to avoid having to set ``--app`` or
``FLASK_APP`` manually, and to set configuration using environment
variables similar to how some deployment services work.

Variables set on the command line are used over those set in :file:`.env`,
which are used over those set in :file:`.flaskenv`. :file:`.flaskenv` should be
used for public variables, such as ``FLASK_APP``, while :file:`.env` should not
be committed to your repository so that it can set private variables.

Directories are scanned upwards from the directory you call ``flask``
from to locate the files.

The files are only loaded by the ``flask`` command or calling
:meth:`~Flask.run`. If you would like to load these files when running in
production, you should call :func:`~cli.load_dotenv` manually.


设置命令选项
-----------

Click is configured to load default values for command options from
environment variables. The variables use the pattern
``FLASK_COMMAND_OPTION``. For example, to set the port for the run
command, instead of ``flask run --port 8000``:

.. tabs::

   .. group-tab:: Bash

      .. code-block:: text

         $ export FLASK_RUN_PORT=8000
         $ flask run
          * Running on http://127.0.0.1:8000/

   .. group-tab:: Fish

      .. code-block:: text

         $ set -x FLASK_RUN_PORT 8000
         $ flask run
          * Running on http://127.0.0.1:8000/

   .. group-tab:: CMD

      .. code-block:: text

         > set FLASK_RUN_PORT=8000
         > flask run
          * Running on http://127.0.0.1:8000/

   .. group-tab:: Powershell

      .. code-block:: text

         > $env:FLASK_RUN_PORT = 8000
         > flask run
          * Running on http://127.0.0.1:8000/

These can be added to the ``.flaskenv`` file just like ``FLASK_APP`` to
control default command options.


Disable dotenv
~~~~~~~~~~~~~~

The ``flask`` command will show a message if it detects dotenv files but
python-dotenv is not installed.

.. code-block:: bash

    $ flask run
     * Tip: There are .env files present. Do "pip install python-dotenv" to use them.

You can tell Flask not to load dotenv files even when python-dotenv is
installed by setting the ``FLASK_SKIP_DOTENV`` environment variable.
This can be useful if you want to load them manually, or if you're using
a project runner that loads them already. Keep in mind that the
environment variables must be set before the app loads or it won't
configure as expected.

.. tabs::

   .. group-tab:: Bash

      .. code-block:: text

         $ export FLASK_SKIP_DOTENV=1
         $ flask run

   .. group-tab:: Fish

      .. code-block:: text

         $ set -x FLASK_SKIP_DOTENV 1
         $ flask run

   .. group-tab:: CMD

      .. code-block:: text

         > set FLASK_SKIP_DOTENV=1
         > flask run

   .. group-tab:: Powershell

      .. code-block:: text

         > $env:FLASK_SKIP_DOTENV = 1
         > flask run


Environment Variables From virtualenv
-------------------------------------

If you do not want to install dotenv support, you can still set environment
variables by adding them to the end of the virtualenv's :file:`activate`
script. Activating the virtualenv will set the variables.

.. tabs::

   .. group-tab:: Bash

      Unix Bash, :file:`venv/bin/activate`::

          $ export FLASK_APP=hello

   .. group-tab:: Fish

      Fish, :file:`venv/bin/activate.fish`::

          $ set -x FLASK_APP hello

   .. group-tab:: CMD

      Windows CMD, :file:`venv\\Scripts\\activate.bat`::

          > set FLASK_APP=hello

   .. group-tab:: Powershell

      Windows Powershell, :file:`venv\\Scripts\\activate.ps1`::

          > $env:FLASK_APP = "hello"

It is preferred to use dotenv support over this, since :file:`.flaskenv` can be
committed to the repository so that it works automatically wherever the project
is checked out.


Custom Commands
---------------

The ``flask`` command is implemented using `Click`_. See that project's
documentation for full information about writing commands.

This example adds the command ``create-user`` that takes the argument
``name``. ::

    import click
    from flask import Flask

    app = Flask(__name__)

    @app.cli.command("create-user")
    @click.argument("name")
    def create_user(name):
        ...

::

    $ flask create-user admin

This example adds the same command, but as ``user create``, a command in a
group. This is useful if you want to organize multiple related commands. ::

    import click
    from flask import Flask
    from flask.cli import AppGroup

    app = Flask(__name__)
    user_cli = AppGroup('user')

    @user_cli.command('create')
    @click.argument('name')
    def create_user(name):
        ...

    app.cli.add_command(user_cli)

::

    $ flask user create demo

See :ref:`testing-cli` for an overview of how to test your custom
commands.


Registering Commands with Blueprints
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If your application uses blueprints, you can optionally register CLI
commands directly onto them. When your blueprint is registered onto your
application, the associated commands will be available to the ``flask``
command. By default, those commands will be nested in a group matching
the name of the blueprint.

.. code-block:: python

    from flask import Blueprint

    bp = Blueprint('students', __name__)

    @bp.cli.command('create')
    @click.argument('name')
    def create(name):
        ...

    app.register_blueprint(bp)

.. code-block:: text

    $ flask students create alice

You can alter the group name by specifying the ``cli_group`` parameter
when creating the :class:`Blueprint` object, or later with
:meth:`app.register_blueprint(bp, cli_group='...') <Flask.register_blueprint>`.
The following are equivalent:

.. code-block:: python

    bp = Blueprint('students', __name__, cli_group='other')
    # or
    app.register_blueprint(bp, cli_group='other')

.. code-block:: text

    $ flask other create alice

Specifying ``cli_group=None`` will remove the nesting and merge the
commands directly to the application's level:

.. code-block:: python

    bp = Blueprint('students', __name__, cli_group=None)
    # or
    app.register_blueprint(bp, cli_group=None)

.. code-block:: text

    $ flask create alice


Application Context
~~~~~~~~~~~~~~~~~~~

Commands added using the Flask app's :attr:`~Flask.cli` or
:class:`~flask.cli.FlaskGroup` :meth:`~cli.AppGroup.command` decorator
will be executed with an application context pushed, so your custom
commands and parameters have access to the app and its configuration. The
:func:`~cli.with_appcontext` decorator can be used to get the same
behavior, but is not needed in most cases.

.. code-block:: python

    import click
    from flask.cli import with_appcontext

    @click.command()
    @with_appcontext
    def do_work():
        ...

    app.cli.add_command(do_work)


Plugins
-------

Flask will automatically load commands specified in the ``flask.commands``
`entry point`_. This is useful for extensions that want to add commands when
they are installed. Entry points are specified in :file:`setup.py` ::

    from setuptools import setup

    setup(
        name='flask-my-extension',
        ...,
        entry_points={
            'flask.commands': [
                'my-command=flask_my_extension.commands:cli'
            ],
        },
    )


.. _entry point: https://packaging.python.org/tutorials/packaging-projects/#entry-points

Inside :file:`flask_my_extension/commands.py` you can then export a Click
object::

    import click

    @click.command()
    def cli():
        ...

Once that package is installed in the same virtualenv as your Flask project,
you can run ``flask my-command`` to invoke the command.


.. _custom-scripts:

Custom Scripts
--------------

When you are using the app factory pattern, it may be more convenient to define
your own Click script. Instead of using ``--app`` and letting Flask load
your application, you can create your own Click object and export it as a
`console script`_ entry point.

Create an instance of :class:`~cli.FlaskGroup` and pass it the factory::

    import click
    from flask import Flask
    from flask.cli import FlaskGroup

    def create_app():
        app = Flask('wiki')
        # other setup
        return app

    @click.group(cls=FlaskGroup, create_app=create_app)
    def cli():
        """Management script for the Wiki application."""

Define the entry point in :file:`setup.py`::

    from setuptools import setup

    setup(
        name='flask-my-extension',
        ...,
        entry_points={
            'console_scripts': [
                'wiki=wiki:cli'
            ],
        },
    )

Install the application in the virtualenv in editable mode and the custom
script is available. Note that you don't need to set ``--app``. ::

    $ pip install -e .
    $ wiki run

.. admonition:: Errors in Custom Scripts

    When using a custom script, if you introduce an error in your
    module-level code, the reloader will fail because it can no longer
    load the entry point.

    The ``flask`` command, being separate from your code, does not have
    this issue and is recommended in most cases.

.. _console script: https://packaging.python.org/tutorials/packaging-projects/#console-scripts


PyCharm Integration
-------------------

PyCharm Professional provides a special Flask run configuration to run the development
server. For the Community Edition, and for other commands besides ``run``, you need to
create a custom run configuration. These instructions should be similar for any other
IDE you use.

In PyCharm, with your project open, click on *Run* from the menu bar and go to *Edit
Configurations*. You'll see a screen similar to this:

.. image:: _static/pycharm-run-config.png
    :align: center
    :class: screenshot
    :alt: Screenshot of PyCharm run configuration.

Once you create a configuration for the ``flask run``, you can copy and change it to
call any other command.

Click the *+ (Add New Configuration)* button and select *Python*. Give the configuration
a name such as "flask run".

Click the *Script path* dropdown and change it to *Module name*, then input ``flask``.

The *Parameters* field is set to the CLI command to execute along with any arguments.
This example uses ``--app hello --debug run``, which will run the development server in
debug mode. ``--app hello`` should be the import or file with your Flask app.

If you installed your project as a package in your virtualenv, you may uncheck the
*PYTHONPATH* options. This will more accurately match how you deploy later.

Click *OK* to save and close the configuration. Select the configuration in the main
PyCharm window and click the play button next to it to run the server.

Now that you have a configuration for ``flask run``, you can copy that configuration and
change the *Parameters* argument to run a different CLI command.
