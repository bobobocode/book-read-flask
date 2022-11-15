# 主要路线（流程）

使用wsgi
werkzeug
app.run -> werkzeug.run_simple 需要一个wsgi函数
Flask对象__call__是一个wsgi函数

wsgi_app -> full_dispatch_request

```
self.ensure_sync(self.view_functions[rule.endpoint])(**view_args)
```

Flask 的路由主要是根据前端请求 (request) 中 url path 信息，找到对应的 endpoint，再根据 endpoint 找到对应的 view function 进行处理。在这个关系中，url 与 endpoint 的映射，由 werkzeug 的 Rule 和 Map 来实现，endpoint 与 view function 的映射由 Flask 用 Python 标准数据结构 dict 实现。 在 werkzeug 中，Rule 表示 url rule 和 endpoint 的映射，Map 实现与多个 Rule 的绑定
