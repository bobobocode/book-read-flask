# Flask中的Python

Flask源代码中用到的Python一些特别的地方

* functool包中的update_wrapper方法

更新一个 wrapper 函数以使其类似于 wrapped 函数.

可选参数为指明原函数的哪些属性要直接被赋值给 wrapper 函数的匹配属性的元组，并且这些 wrapper 函数的属性将使用原函数的对应属性来更新.
这些参数的默认值是模块级常量 WRAPPER_ASSIGNMENTS 
(它将被赋值给 wrapper 函数的 __module__, __name__, __qualname__, __annotations__ 和 __doc__ 即文档字符串) 以及 WRAPPER_UPDATES (它将更新 wrapper 函数的 __dict__ 即实例字典)

为了允许出于内省和其他目的访问原始函数（例如绕过 lru_cache() 之类的缓存装饰器），此函数会自动为 wrapper 添加一个指向被包装函数的 __wrapped__ 属性。

此函数的主要目的是在 decorator 函数中用来包装被装饰的函数并返回包装器。 如果包装器函数未被更新，则被返回函数的元数据将反映包装器定义而不是原始函数定义，这通常没有什么用处。

update_wrapper() 可以与函数之外的可调用对象一同使用。 在 assigned 或 updated 中命名的任何属性如果不存在于被包装对象则会被忽略（即该函数将不会尝试在包装器函数上设置它们）
如果包装器函数自身缺少在 updated 中命名的任何属性则仍将引发 AttributeError。

* typing

cast方法其实什么都没做. 它的源代码是这样:

```
def cast(typ, val):
    return val
```

它只是示意val是指定的类型, 不会做任何检查和转换.
Python是动态类型语言, 所以这里也不需求做任何转换.


TypeVar

    """Type variable.

    Usage::

      T = TypeVar('T')  # Can be anything
      A = TypeVar('A', str, bytes)  # Must be str or bytes

    Type variables exist primarily for the benefit of static type
    checkers.  They serve as the parameters for generic types as well
    as for generic function definitions.  See class Generic for more
    information on generic types.  Generic functions work as follows:

      def repeat(x: T, n: int) -> List[T]:
          '''Return a list containing n references to x.'''
          return [x]*n

      def longest(x: A, y: A) -> A:
          '''Return the longest of two strings.'''
          return x if len(x) >= len(y) else y

    The latter example's signature is essentially the overloading
    of (str, str) -> str and (bytes, bytes) -> bytes.  Also note
    that if the arguments are instances of some subclass of str,
    the return type is still plain str.

    At runtime, isinstance(x, T) and issubclass(C, T) will raise TypeError.

    Type variables defined with covariant=True or contravariant=True
    can be used to declare covariant or contravariant generic types.
    See PEP 484 for more details. By default generic types are invariant
    in all type variables.

    Type variables can be introspected. e.g.:

      T.__name__ == 'T'
      T.__constraints__ == ()
      T.__covariant__ == False
      T.__contravariant__ = False
      A.__constraints__ == (str, bytes)

    Note that only type variables defined in global scope can be pickled.
    """

