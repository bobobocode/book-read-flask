# 主要路线（流程）

使用wsgi
werkzeug
app.run -> werkzeug.run_simple 需要一个wsgi函数
Flask对象__call__是一个wsgi函数

wsgi_app -> full_dispatch_request

self.ensure_sync(self.view_functions[rule.endpoint])(**view_args)
