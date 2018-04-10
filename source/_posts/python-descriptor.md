---
title: python描述符
date: 2018-04-07 16:30:29
tags:
	- python
---

## 概述
在上一次的`session`源码分析中，我提到了一个比较有趣的东西，就是`cached_property`这个类装饰器，一开始我认为它是类装饰器，但是，实际上它是属于描述符的一种。在了解了描述符之后，发现它的功能非常的强大，而且用处非常的多，所以特地来总结一下。

<!--more-->

### 描述符
#### 定义
官方给出的定义是：描述符是具有`绑定行为`的对象的属性，其属性的访问被描述符协议方法覆写。这些方法包括`__set__()`, `__get__()`, `__delete__()`, 一个对象只要包含了这三个方法中的任意一种，我们就称它为描述符。

#### 属性访问的默认行为
属性访问的默认行为是从一个对象的字典中获取(get), 设置(set), 删除(delete)属性。例如：`a.x`的查找连始于`a.__dict__['x']`，接着查找`type(a).__dict__['x']`, 然后是`type(a)`除了元类之外的基类。如果查找到的值是一个包含描述符方法的对象，那么python可能会重写该对象的默认行为并调用那个描述符方法。

####协议

```
descr.__get__(self, obj, objtype=None) ---> value
descr.__set__(self, obj, value) ---> None
descr.__delete__(self, obj) --->None
```

这些就是描述符协议，定义这些方法中的任意一个，对象就被认为是描述符并且在其被当做属性访问时调用相应的描述符方法。

如果一个对象同时定义了`__get__()`，和`__set__()`方法，它就成为了一个资料描述符(data descriptor), 只定义了`__get__()`方法的描述符被称为是非资料描述符(他们通常用于方法，但是也有其他用处)

资料描述符和非资料描述符的区别在于访问实例时获取结果的顺序。如果一个实例字典中有一个和资料描述符同名的实例变量， 那么访问时优先访问到资料描述符，反之，如果实例字典里面有一个和非资料描述符同名的项，那么优先访问到到实例字典里面的项。

为了构造一个只读的资料描述符，需同时定义`__get__()`和`__set__()`方法并且在`__set__()`方法中抛出`AttributeError`异常。在`__set()__`方法中加上抛出异常的逻辑就足够让一个对象成为资料描述符。

#### 调用描述符的优先级规则
描述符最常见的调用场景是属性访问时描述符被自动调用。比如obj.d在obj的字典中查找obj.d.如果d定义了`__get__()`方法，那么`d.__get__(obj)`会依据如下的优先规则被调用。调用的细节取决于`obj`是一个对象还是类。

对于对象来说，关键在于`object.__getattribute__()`，它将`b.x`转换为`type(b).__dict__['x'].__get__(b, type(b))`。这种实现依据这样的一个优先链：资料描述符优先于实例变量，实例变量优先于非资料描述符，如果对象包含`__getattr__()`方法，那么这个方法 (`__getattr__()`)的访问优先级最低。完整的C实现在`Objects/object.c`中`PyObject_GenericGetAttr()`。
而对于类来讲，关键之处在于`type.__getattribute__()`方法，它将`B.x`转换为`B.__dict__['x'].__get__(None, B)`。


这里需要记住的要点是:

* 描述符被`__getattribute__()`方法调用
* 重写`__getattribute__()`方法会阻止描述符的自动调用
* `object.__getattribute__()`和`type.__getattribute__()`对`__get__()`方法的调用不一样
* 资料描述符总是覆写实例字典
* 非资料描述符可能被实例字典覆写


#### 描述符实例

##### 属性
这里展示两个资料描述符的实例。第一个是定义一个资料描述符来监听对象属性的变动情况，第二个是用原生的python来定义`property()`。

```python
class RevalAccess:
	def __init__(self, initval=None, name='var'):
	    self.val = initval
	    self.name = name
	
	def __get__(self, obj, objtype):
	    print('Retrieving ', self.name)
	    return self.val
	    
	def __set__(self, obj, value):
		print('Updating ', self.name)
		self.val = val
		
		
>>> class MyClass(object):
    x = RevealAccess(10, 'var "x"')
    y = 5

>>> m = MyClass()
>>> m.x
Retrieving var "x"
10
>>> m.x = 20
Updating var "x"
>>> m.x
Retrieving var "x"
20
>>> m.y
5


class Property:
    def __init__(self, fget=None, fset=None, fdel=None, doc):
    	self.fget = fget
    	self.fset = fset
    	self.fdel  =fdel
    	if doc is None and fget is not None:
    		doc = fget.__doc__
    	self.doc = doc
    	
    def __get__(self, obj, objtype=None):
    	if obj is None:
    		return self
    	if self.fget is None:
            raise AttributeError("unreadable attribute")
        return self.fget(obj)

    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(obj, value)

    def __delete__(self, obj):
        if self.fdel is None:
            raise AttributeError("can't delete attribute")
        self.fdel(obj)

    def getter(self, fget):
        return type(self)(fget, self.fset, self.fdel, self.__doc__)

    def setter(self, fset):
        return type(self)(self.fget, fset, self.fdel, self.__doc__)

    def deleter(self, fdel):
        return type(self)(self.fget, self.fset, fdel, self.__doc__)
```

#### 函数和方法
Python面向对象特性的建立是基于一个函数环境的。利用非资料描述符，这两者无缝结合在一起了。

类字典将方法存作函数。在类定义中，方法是用def和lambda来定义的，这和定义函数是一样的。唯一不同的是，在类的方法定义中，第一个参数被保留用来指代对象实例。依据Python的惯例，实例引用被称为self，但是它可以是诸如this等的任意其它变量名。

为了支持方法调用，函数包括了__get__()方法来在属性访问时绑定方法。这意味着所有的函数都是非资料描述符，它们依据其是从对象还是类中被调用来返回绑定(bound)或非绑定(unbound)方法。

函数有一个`__get__()`方法，当函数被当做属性访问时它将函数转成了方法。非资料描述符将`obj.f(*args)`调用转换成`f(obj, *args)`；而调用`klass.f(*args)`将变成`f(*args)`。

下面的表格总结了绑定和它最有用的两种变种：

|  转换     |  从对象中调用  |  从类中调用  |
|--------  |--------------|-----------  |
|  函数     |	`f(obj, *args)`|	`f(*args)`|
|静态方法	 |`f(*args)`	|`f(*args)`|
|类方法	    |`f(type(obj), *args)`|  `f(klass, *args)`|


静态方法会毫无更改地返回底层函数。调用c.f和C.f分别等价于直接调用`object.__getattribute__(c, "f")`和`object.__getattribute__(C, "f")`。这样，无论是从一个对象还是一个类中，这个函数都可以正确地访问到。
静态方法(static methods)最好的实现是利用不引用self变量的方法。

```python

>>> class E(object):
        def f(x):
            print(x)
        f = staticmethod(f)

>>> print(E.f(3))
3
>>> print(E().f(3))
3
```
利用非资料描述符协议，staticmethod()的纯Python实现看起来像这样：

```python
class StaticMethod:
    def __init__(self, func):
    	self.func = func
    
    def __get__(self, obj, objtype=None):
    	return self.func
```

与静态方法不一样的是，类方法在调用函数之前将到该类的引用至于参数列表之前（即将该类的引用作为函数的第一个参数）。无论调用者是对象还是类，其格式是一样的：

```python
>>> class E(object):
        def f(klass, x):
            return klass.__name__, x
        f = classmethod(f)

>>> print(E.f(3))
('E', 3)
>>> print(E().f(3))
('E', 3)
```

利用非资料描述符协议，classmethod的纯Python实现看起来像这样：

```python
class ClassMethod(object):
    "Emulate PyClassMethod_Type() in Objects/funcobject.c"

    def __init__(self, f):
        self.f = f

    def __get__(self, obj, klass=None):
        if klass is None:
            klass = type(obj)
        def newfunc(*args):
            return self.f(klass, *args)
        return newfunc
```

### 来说说@
以前我一直认为python中的`@`符和`装饰器`是等价的，好像只要出现了`@`符号，那么它后面跟这的就是一个装饰器。但是实际上来看不是这样的。比如python中的类装饰器，只有定义了`__call__()`的魔法方法，一个类才能是`装饰器`。 但是我们从上文可以发现，其实用描述符来修饰一个函数的时候，也是可以使用`@`符号的，但是描述符并不是一个装饰器。

所以，我觉得可以这样认为：`@`是一个修饰符，修饰符后面的对象会把被修饰的函数作为参数，并返回一个可调用的对象。

### 参考资料

[https://harveyqing.gitbooks.io/python-read-and-write/content/python_advance/python_descriptor.html](https://harveyqing.gitbooks.io/python-read-and-write/content/python_advance/python_descriptor.html)