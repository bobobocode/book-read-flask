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
