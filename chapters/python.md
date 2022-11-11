# Flask中的Python

Flask源代码中用到的Python一些特别的地方

* functool包中的update_wrapper方法

更新一个 wrapper 函数以使其类似于 wrapped 函数。 可选参数为指明原函数的哪些属性要直接被赋值给 wrapper 函数的匹配属性的元组，并且这些 wrapper 函数的属性将使用原函数的对应属性来更新。 这些参数的默认值是模块级常量 WRAPPER_ASSIGNMENTS (它将被赋值给 wrapper 函数的 __module__, __name__, __qualname__, __annotations__ 和 __doc__ 即文档字符串) 以及 WRAPPER_UPDATES (它将更新 wrapper 函数的 __dict__ 即实例字典)。

为了允许出于内省和其他目的访问原始函数（例如绕过 lru_cache() 之类的缓存装饰器），此函数会自动为 wrapper 添加一个指向被包装函数的 __wrapped__ 属性。

此函数的主要目的是在 decorator 函数中用来包装被装饰的函数并返回包装器。 如果包装器函数未被更新，则被返回函数的元数据将反映包装器定义而不是原始函数定义，这通常没有什么用处。

update_wrapper() 可以与函数之外的可调用对象一同使用。 在 assigned 或 updated 中命名的任何属性如果不存在于被包装对象则会被忽略（即该函数将不会尝试在包装器函数上设置它们）。 如果包装器函数自身缺少在 updated 中命名的任何属性则仍将引发 AttributeError。
