# Flask对象

Flask类定义在源代码中的app.py里.

url_map: 储存 url 和 endpoint 的映射（url_map 的数据类型是 werkzeug.routing.Map）
其中url由Rule对象封装. 因为用户在route装饰器中定义并且Flask要保存的不
只是一个url字符串, 而是url path的某种规则.
view_functions: 储存 endpoint 和 view function 的映射 (dict 类型)
