importlib
* spec

一个命名空间，其中包含用于加载模块的相关导入信息。是 importlib.machinery.ModuleSpec 的实例.
包括模块全限定名, 模块的加载器, 模块的原始地址, 子模块地址列表, 模块所在的包的全限名等


* from asgiref.sync import async_to_sync

Flask 的路由机制是在 werkzeug 中实现的， Flask 只是调用而已。Flask 的路由包括三个主要过程：

路由的构建 （werkzeug 实现相关的数据结构，Rule, Map 等）
路由的匹配 （werkzueg 实现）
请求的分派（dispatch， Flask 实现）


Rule('/', endpoint='index', methods=['GET']),

Map(url_rules)

Map([<Rule '/' (HEAD, GET) -> index>])

Rule 定义 url rule 和 endpoint 之间的映射，url rule 与 endpoint 是多对一关系

rule 的标准格式是 <converter(arguments):name>，由转换器 (converter）、转换器参数(argument) 和名称（name）构成。converter 在路由匹配的时候会用到
Url rule 必须从 / 开始。/ 的英文为 slash，了解其英文有助于看懂代码。如果不以 slash 开始，抛出如下错误：

# FILE: werkzeug/routing.py
if not string.startswith("/"):
    raise ValueError("urls must start with a leading slash")

url path 的结尾，有以 / 结尾和不以 / 结尾两种可能 。比如有些人用 /about 表示，有些用 /about/ 表示（多了一个斜杠）。其实这两个应该是同一个 url。为了保证 url 的唯一性，避免被搜索引擎索引两次，Rule 和 Map 提供了相关属性对此进行区分：


Flask 路由信息包括 url_map 和 view_functions

from flask import Flask
from werkzeug.routing import Map, Rule

app = Flask(__name__)

def index():
    return 'Index Page'

def about():
    return 'About Page'

# create url rules and map instances
url_rules = [
    Rule('/', endpoint='index', methods=['GET']),
    Rule('/about', endpoint='about', methods=['GET'])
]
map = Map(url_rules)

# set app.url_map and view_functions
app.url_map = map
app.view_functions = {
    'index': index,
    'about': about
}

if __name__ == '__main__':
    app.run()

但这段代码有一个小问题。我们知道 Flask 为了能处理静态文件，在.__init__() 方法中构造了一个 static rule，上面的代码直接对 url_map 赋值，初始化方法中创建的 static rule 就被丢掉了。为了保留 static rule，可以模拟 Map 添加 Rule 的方式，稍作变更，以下是代码，我略去了重复的部分。

url_rules = [
    Rule('/', endpoint='index', methods=['GET']),
    Rule('/about', endpoint='about', methods=['GET'])
]

for rule in url_rules:
    app.url_map.add(rule)

app.view_functions['index'] = index
app.view_functions['about'] = about

--------Flask.add_url_rule()


动态 url 由 werkzeug 通过转换器 (converter) 来实现

绑定到环境bind_to_environ() 方法在内部调用了 Map.bind() 方法， Map.bind() 方法创建 MapAdapter 的实例。MapAdapter 类负责 URL 匹配的工作。
MapAdapter.match() 方法
该方法的作用是，传入 path_info 和 method，返回 tuple 类型包括 endpoint + arguments 的信息或者 rule + arguments 信息 (You get a tuple in the form (endpoint, arguments) if there is a match (unless return_rule is True, in which case you get a tuple in the form (rule, arguments)))。该方法内部调用 Rule.match() 方法：

werkzueg 实现了 7 中预定义的 converter，能满足绝大部分需求。在 routing.py 中，我们可以看到如下的代码：

DEFAULT_CONVERTERS = {
    "default": UnicodeConverter,
    "string": UnicodeConverter,
    "any": AnyConverter,
    "path": PathConverter,
    "int": IntegerConverter,
    "float": FloatConverter,
    "uuid": UUIDConverter,
}
7 种 converter，都直接或者间接继承自 BaseConverter。BaseConverter 类的代码如下：

class BaseConverter(object):
    """Base class for all converters."""

    regex = "[^/]+"
    weight = 100

    def __init__(self, map):
        self.map = map

    def to_python(self, value):
        return value

    def to_url(self, value):
        if isinstance(value, (bytes, bytearray)):
            return _fast_url_quote(value)
        return _fast_url_quote(text_type(value).encode(self.map.charset))

每一种 converter 的 __init__() 方法确定了 converter 可以使用哪些 arguments。比如 UnicodeConverter 的 __init__() 方法是这样的：

def __init__(self, map, minlength=1, maxlength=None, length=None):
	# 代码略
所以我们在使用 UnicodeConverter 的时候能够使用 length 、minlenght 和 maxlenght 这些参数。比如下面的示例：

rules = [
    Rule('/message/<string(length=11):mobile_number>', endpoint='mobile')
]
BaseConveter 类的 to_python() 方法在 MapAdapter.match() 方法中匹配成功后被调用，将请求的 path 中动态 url 部分解析出 argument value， 传递给该方法的value 参数。在上面的示例中，13811112222 手机号码被解析出来，传递给 to_python() 方法。

to_url() 方法在 url_for() 函数反向构建url 的时候，将 arguments 参数传递给该方法。后面结合具体的示例来帮助大家理解。

UnicodeConverter 是默认的转换器，用于实现 string 类型的动态 url。

自定义转换器

刚刚给出的例子没有针对手机号码的校验规则，假设我们要对请求中传递的手机号码进行校验，可以用自定义转换器来实现。如果只是想增加校验规则，在转换器的 __init__() 方法中改写 BaseConverter 的 regex 属性。

自定义转换器代码如下：

# CustomConverter.py

from werkzeug.routing import BaseConverter

class MobileConverter(BaseConverter):
    def __init__(self, map):
        BaseConverter.__init__(self, map)
        self.regex = r'1[35678]\d{9}'
然后在创建 Map 的时候 converters 参数从自定义转换器赋值：

from CustomConverter import MobileConverter

mobile_converter = [{'mobile', MobileConverter}]
rules = [
    Rule('/', endpoint='index'),
    Rule('/message/<mobile:mobile_number>', endpoint='mobile')
]

url_map = Map(rules, converters=mobile_converter)
to_python() 方法如何使用呢？假设为了增加友好性，将返回到前端的手机号码用 - 分割，比如 13811112222 显示为 138-1111-2222。根据刚才的说明, to_python() 方法在匹配成功后将被调用，所以可以改写 BaseConverter 的 to_python() 方法，编写如下代码：

from werkzeug.routing import BaseConverter

class MobileConverter(BaseConverter):
    def __init__(self, map):
        BaseConverter.__init__(self, map)
        self.regex = r'1[35678]\d{9}'

    def to_python(self, value):
        return '{}-{}-{}'.format(value[:3], value[3:7], value[7:12])
to_url() 方法在 url_for() 函数中被调用，url_for() 函数的 argument 参数被传递给该方法的 value 参数。下面的示例演示了 to_url() 的用法。

from werkzeug.routing import BaseConverter

class MobileConverter(BaseConverter):
    def __init__(self, map):
        BaseConverter.__init__(self, map)
        self.regex = r'1[35678]\d{9}'

    def to_python(self, value):
        return '{}-{}-{}'.format(value[:3], value[3:7], value[7:12])

    def to_url(self, value):
        print(value)
        return value

from flask import Flask, url_for
from CustomConverter import MobileConverter

app = Flask(__name__)
app.url_map.converters['mobile'] = MobileConverter

@app.route('/')
def index():
    return 'Index page'

@app.route('/message/<mobile:mobile_number>')
def send_message(mobile_number):
    print(url_for('send_message', mobile_number='13833334444'))
    return 'Message was sent to {}'.format(mobile_number)

if __name__ == '__main__':
    app.run()
