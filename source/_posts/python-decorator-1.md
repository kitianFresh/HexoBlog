---
title:
  Python Decorator(1)
date:
  2017-04-24
categories:
- Python
tags:
- python
- functional programming
- decorator
---


# 函数式

装饰器的实现,得益于 Python 支持函数式编程! 因此, 记住, **装饰器的本质是函数式!** 函数式编程,可以查看我的一篇基础教程! 从另一个角度来讲, 更多的是动态的修改代码! 需要注意的是, Python Decorator 并不是装饰器模式, 虽然他可以很方便的实现装饰器模式. 我更多的理解是, **Python Decorator 是 code decorator, 就是动态修改代码的特性!**

## 递归


```python
def fib(n):
    if n is 0 or n is 1:
        return 1
    else:
        return fib(n-1) + fib(n-2)
old = fib # 后面的 fib 可能会被修改, 先保存起来!
```

首先实现一个递归函数, 然后写出递归函数的递归深度跟踪函数!对于深度的跟踪, C style code 当然是需要全局变量做深度记录和跟踪, 最后写出来的代码是这样的!


```python
indent = 0
def fib_verbose_tree(n):
    global indent 
    print '| ' * indent + '|--', 'fib', n
    indent += 1
    if n is 0 or n is 1:
        print '| ' * indent + '|--', 'return', repr(1)
        indent -= 1
        return 1
    else:
        value = fib_verbose_tree(n-1) + fib_verbose_tree(n-2)
        print '| ' * indent + '|--', 'return', repr(value)
        indent -= 1
        return value
```


```python
print '-------------fib_verbose_tree-----------'
print fib_verbose_tree(4)
```

    -------------fib_verbose_tree-----------
    |-- fib 4
    | |-- fib 3
    | | |-- fib 2
    | | | |-- fib 1
    | | | | |-- return 1
    | | | |-- fib 0
    | | | | |-- return 1
    | | | |-- return 2
    | | |-- fib 1
    | | | |-- return 1
    | | |-- return 3
    | |-- fib 2
    | | |-- fib 1
    | | | |-- return 1
    | | |-- fib 0
    | | | |-- return 1
    | | |-- return 2
    | |-- return 5
    5


如果使用函数式编程, 我们可以得到一个递归跟踪的外壳函数,负责深度的加减处理!其实就是函数调用之前,深度加1,出栈后,深度减1.


```python
def trace_tree(f):
    f.indent = 0
    def wrapped(*args):
        print '| ' * f.indent + '|--', f.__name__, str(args)
        f.indent += 1
        value = f(*args)
        print '| ' * f.indent + '|--', 'return', repr(value)
        f.indent -= 1
        return value
    return wrapped
```


```python
fib = old
print '-------------trace_tree-----------'
fib = trace_tree(fib)
print fib(4)
```

    -------------trace_tree-----------
    |-- fib (4,)
    | |-- fib (3,)
    | | |-- fib (2,)
    | | | |-- fib (1,)
    | | | | |-- return 1
    | | | |-- fib (0,)
    | | | | |-- return 1
    | | | |-- return 2
    | | |-- fib (1,)
    | | | |-- return 1
    | | |-- return 3
    | |-- fib (2,)
    | | |-- fib (1,)
    | | | |-- return 1
    | | |-- fib (0,)
    | | | |-- return 1
    | | |-- return 2
    | |-- return 5
    5


最后,**我们调用 fib , 其实就是在调用 wrapped() 函数!因为此时的 fib 已经等于另外一个函数了! fib 的代码已经成为 wrapped 了, 即 fib 的代码现在已经是 wrapped 函数的代码了.因为 fib 指针对象现在指向的是 wrapped 函数.**

下面我们来做 dp 编程.


```python
def memoize(f):
    cache = {}
    def wrapped(n):
        if n not in cache:
            cache[n] = f(n)
        return cache[n]
    return wrapped

print '-------------memoize_trace_tree-----------'
fib = old
fib = trace_tree(fib)
fib = memoize(fib)
print fib(4)
```

    -------------memoize_trace_tree-----------
    |-- fib (4,)
    | |-- fib (3,)
    | | |-- fib (2,)
    | | | |-- fib (1,)
    | | | | |-- return 1
    | | | |-- fib (0,)
    | | | | |-- return 1
    | | | |-- return 2
    | | |-- return 3
    | |-- return 5
    5


以上是 DP 打表过程, 也可以当作装饰器来使用!


```python
import time
def profile(f):
    def wrapped(n):
        start = time.time()
        value = f(n)
        end = time.time()
        print "time taken: %lf sec" % (end - start)
        return value
    return wrapped

fib = old
print '-------------timing_memoized-----------'
fib = memoize(fib)
f = profile(fib)
print f(20)

fib = old
print '-------------timing_unmemoized-----------'
f = profile(fib)
print f(20)
```

    -------------timing_memoized-----------
    time taken: 0.000021 sec
    10946
    -------------timing_unmemoized-----------
    time taken: 0.003210 sec
    10946


# Syntax Sugar
Python 其实允许另一种形式的装饰器, 直接使用 @decorator 加在需要装饰的函数定义上面, 即可完成装饰! Magic!


```python
@trace_tree
def my_pow(x, n):
    if n == 0:
        return 1
    if n == 1:
        return x
    return my_pow(x, n/2) * my_pow(x, n/2) if n % 2 == 0 else my_pow(x, n/2) * my_pow(x, n/2) * x

my_pow(2, 5)
```

    |-- my_pow (2, 5)
    | |-- my_pow (2, 2)
    | | |-- my_pow (2, 1)
    | | | |-- return 2
    | | |-- my_pow (2, 1)
    | | | |-- return 2
    | | |-- return 4
    | |-- my_pow (2, 2)
    | | |-- my_pow (2, 1)
    | | | |-- return 2
    | | |-- my_pow (2, 1)
    | | | |-- return 2
    | | |-- return 4
    | |-- return 32

    32



这就是 Python 中的语法糖, `@trace_tree` 之后, Python 自动执行了 `my_pow = trace_tree(my_pow)`.

# Decorator
Python 中有很多不同的 decorator, 根据装饰器本身的定义分两大类:
 - 函数定义的decorator, **decorator 本身是使用 函数定义的.**
   * 不带参数, 即装饰器本身不带参数的.
   * 带参数, 即装饰器本身带参数的.
 - 类定义的decorator, **decorator 本身是使用 类定义的.**
   * 不带参数, 即装饰器本身不带参数的.
   * 带参数, 即装饰器本身带参数的. 

按照 装饰器的用途, 即装饰的是 普通函数, 还是类方法又分两大类.

Python 对 decorator 的定义:
>A decorator is a function which accepts a function and returns a new function. Since it’s a function, we must provide three pieces of information: the name of the decorator, a parameter, and a suite of statements that creates and returns the resulting function.

>The suite of statements in a decorator will generally include a function def statement to create the new function and a return statement.

>A common alternative is to include a class definition statement . If a class definition is used, that class must define a callable object by including a definition for the \_\_call\_\_() method and (usually) being a subclass of collections.Callable.

>There are two kinds of decorators, decorators without arguments and decorators with arguments. In the first case, the operation of the decorator is very simple. In the case where the decorator accepts areguments the definition of the decorator is rather obscure, we’ll return to this in Defining Complex Decorators.



## Function Decorator


```python
def entry_exit(f):
    def new_f():
        print("Entring", f.__name__)
        f()
        print("Exited", f.__name__)
        #new_f.__name__ = f.__name__
    return new_f
    #return new_f()

@entry_exit
def func1():
    print("call func1()")

@entry_exit
def func2():
    print("call func2()")

func1()
func2()
print(func1.__name__)
```

    ('Entring', 'func1')
    call func1()
    ('Exited', 'func1')
    ('Entring', 'func2')
    call func2()
    ('Exited', 'func2')
    new_f


这个简单的例子, 可以看出, 经过装饰器 `@entry_exit` 装饰的函数, 他的相关信息发生改变了! 函数名字变成了 new\_f. **因此, 我们一般会在返回 new\_f 之前修改 new\_f.\_\_name\_\_ 等函数本身自带的属性信息为 原来的函数!** 稍后,我们还会看到, 可以实现一个装饰器实现这个功能, 而且 Python 的一些包已经实现了!


```python
def p_decorator(func):
   def wrapper(*args, **kwargs):
       return "<p>{0}</p>".format(func(*args, **kwargs))
   return wrapper

class Person(object):
    def __init__(self):
        self.name = "kinny"
        self.family = "Tian"

    @p_decorator
    def get_fullname(self):
        return self.name+" "+self.family

my_person = Person()

print my_person.get_fullname()
```

    <p>kinny Tian</p>


### 无参数的 function decorator

无参数的 function decorator 模式比较固定, 由于 decorator 本身没有额外参数, 写法也很简单, **遵循传入函数, 返回一个经过包装的函数的原则即可, 只有两个层次**. 这个内部返回函数的参数一般通用的写法是 `resultFunction(*args, **kwargs)`. 这个参数同时也可以接受被包装函数的参数, 因此写法更加通用! 当然如果你的装饰器只给某些特定的函数用, 那么参数也可以是这些特定的被包装函数参数保持一致即可!
```python
def myDecorator( argumentFunction ):
    def resultFunction(*args, **kwargs):
        dosomething()
        argumentFunction(*args, **kwargs)
        dosomething()
    resultFunction.__doc__= argumentFunction.__doc__
    return resultFunction
```

### 有参数的 function decorator
**带参数的 function decorator 写法稍微复杂, 因为他自己也有参数. 所以第一层次函数, 就不能把 被包装函数当参数了. 那么只有内部多加一层次, 用来接受被包装函数参数了! 因此他的经典写法是 三层次!**


```python
def decorator_function_with_arguments(arg1, arg2, arg3):
    def wrap(f):
        print("Enter wrap()")
        
        def wrapped_f(*args):
            print("Inside wrapped_f()")
            print("Decorator arguments: ", arg1, arg2, arg3)
            f(*args)
            print("After f(*args)")
        
        print("Exit wrap()")
        return wrapped_f

    return wrap

@decorator_function_with_arguments("hello", "world", 22)
def sayHello(a1, a2, a3, a4):
    print('sayHello arguments:', a1, a2, a3, a4)

print("After decoration")

print("----------------Preparing to call sayHello()----------------")
sayHello("say", "hello", "argument", "list")
print("----------------After first sayHello() call----------------")
sayHello("a", "different", "set of", "arguments")
print("----------------After second sayHello() call----------------")
```

    Enter wrap()
    Exit wrap()
    After decoration
    ----------------Preparing to call sayHello()----------------
    Inside wrapped_f()
    ('Decorator arguments: ', 'hello', 'world', 22)
    ('sayHello arguments:', 'say', 'hello', 'argument', 'list')
    After f(*args)
    ----------------After first sayHello() call----------------
    Inside wrapped_f()
    ('Decorator arguments: ', 'hello', 'world', 22)
    ('sayHello arguments:', 'a', 'different', 'set of', 'arguments')
    After f(*args)
    ----------------After second sayHello() call----------------


>来看看常见的函数decorator为啥都要写几层包裹函数，这是很多例子都有的，但是没告诉你为啥要这样写。

Python 解释器读取到 `@decorator_function_with_arguments("hello", "world", 22)` 时，首先查找 `decorator_function_with_arguments` 并调用，但是 Python 知道这是一个装饰器啊，他还需要一个 被装饰的函数 `f` 做参数，和 class decorator 一样， 在完成 `dfwa = decorator_function_with_arguments(arg1, arg2, arg3)` 之后，继续 寻找 `f`， 然后 `f = dfwa(f)`, 此时还是装饰过程， 因此 返回的还必须是 callable 对象，即函数。 也就是说， **Python 的 带参数函数装饰器 解释过程 就是 `f = dfwa(arg)(f) => f()` 模型，三个调用才发生真正的调用，因此 function decorator 里面 必须是 嵌套 2 层 wrap func.**

## Class Decorator
**因为 Python 要求装饰器必须返回一个 callable 对象, 因此, class decorator 必须要实现 \_\_call\_\_ 函数, 这样, 返回的对象才能被调用!**

### 无参数的 class decorator
**如果无参数, 那么解释器碰到 `@decorator_without_arguments` 就会直接寻找 被装饰函数 `f`, 然后构造装饰后的对象! 因此 `decorator_without_arguments` 的 初始化方法 \_\_init\_\_() 的参数就是 f 了. 然后新构造的对象, 每次被调用的时候, 就是 `decorated.__call__(*args)`.**


```python
class decorator_without_arguments(object):

    def __init__(self, f):
        """
        If there are no decorator arguments, the function
        to be decorated is passed to the constructor.
        """
        print("Inside __init__()")
        self.f = f

    def __call__(self, *args):
        """
        The __call__ method is not called until the
        decorated function is called.
        """
        print("Inside __call__()")
        self.f(*args)
        print("After self.f(*args)")

@decorator_without_arguments
def sayHello(a1, a2, a3, a4):
    print('sayHello arguments:', a1, a2, a3, a4)

print("After decoration")

print("----------------Preparing to call sayHello()----------------")
sayHello("say", "hello", "argument", "list")
print("----------------After first sayHello() call----------------")
sayHello("a", "different", "set of", "arguments")
print("----------------After second sayHello() call----------------")
```

    Inside __init__()
    After decoration
    ----------------Preparing to call sayHello()----------------
    Inside __call__()
    ('sayHello arguments:', 'say', 'hello', 'argument', 'list')
    After self.f(*args)
    ----------------After first sayHello() call----------------
    Inside __call__()
    ('sayHello arguments:', 'a', 'different', 'set of', 'arguments')
    After self.f(*args)
    ----------------After second sayHello() call----------------


当 Python 解释器读取到 `@decorator_without_arguments` 时，由于 `decorator_without_arguments` 是装饰器，初始化需要被装饰的 `f` 做参数，因此继续读取 `def sayHello`， 拿到参数后， 就执行 `sayHello = decorator_without_arguments(sayHello)`, 是个 `callable` 对象。以后 `sayHello()` 的调用就是 `sayHello.__call__()`.

### 带参数的 class decorator
**如果 class decorator 本身还需要 参数 做初始化, 那么就和 带参数的function decorator 一样, 初始化的时候, 不能加 `f` 做参数了, 然后碰到 `f` 时候会再次被调用一次, 返回一个 callable 对象或者说一个函数. 此时, \_\_call\_\_ 就必须加 `f` 作为参数, 导致 \_\_call\_\_ 内部必须再写一个函数, 返回.**


```python
class decorator_with_arguments(object):

    def __init__(self, arg1, arg2, arg3):
        """
        If there are decorator arguments, the function
        to be decorated is not passed to the constructor!
        """
        print("Inside __init__()")
        self.arg1 = arg1
        self.arg2 = arg2
        self.arg3 = arg3

    def __call__(self, f):
        """
        If there are decorator arguments, __call__() is only called
        once, as part of the decoration process! You can only give
        it a single argument, which is the function object.
        """
        print("Inside __call__()")
        def wrapped_f(*args):
            print("Inside wrapped_f()")
            print("Decorator arguments:", self.arg1, self.arg2, self.arg3)
            f(*args)
            print("After f(*args)")
        return wrapped_f

@decorator_with_arguments("hello", "world", 22)
def sayHello(a1, a2, a3, a4):
    print('sayHello arguments:', a1, a2, a3, a4)

print("After decoration")

print("----------------Preparing to call sayHello()----------------")
sayHello("say", "hello", "argument", "list")
print("----------------After first sayHello() call----------------")
sayHello("a", "different", "set of", "arguments")
print("----------------After second sayHello() call----------------")
```

    Inside __init__()
    Inside __call__()
    After decoration
    ----------------Preparing to call sayHello()----------------
    Inside wrapped_f()
    ('Decorator arguments:', 'hello', 'world', 22)
    ('sayHello arguments:', 'say', 'hello', 'argument', 'list')
    After f(*args)
    ----------------After first sayHello() call----------------
    Inside wrapped_f()
    ('Decorator arguments:', 'hello', 'world', 22)
    ('sayHello arguments:', 'a', 'different', 'set of', 'arguments')
    After f(*args)
    ----------------After second sayHello() call----------------


当 Python 解释器读取到 `@decorator_with_arguments("hello", "world", 22)` 时完成初始化，带参数的 decorator 首先使用其参数初始化，`dwa = decorator_with_arguments("hello", "world", 22)`。由于 decorator_with_arguments 是装饰器，需要被装饰的 f 做参数，因此继续读取 `def sayHello`， 拿到参数后， 就执行 `sayHello = dwa(sayHello)`, 返回的是 `wrapped_f`。**class decorator的 解释模型 就是 `f = dcwa(args)(f) => f()`.**

# 深入理解decorator
**decorator 本身很容易修改原来函数的名字等属性, 因此我们需要保持原来的属性! 将我们返回的 callable 对象的 \_\_name\_\_ \_\_doc\_\_ 和 \_\_module\_\_ 修改为被装饰的对象的即可!**



```python
class entry_exit(object):

    def __init__(self, f):
        self.f = f
        self.__name__ = f.__name__
        self.__doc__ = f.__doc__
        self.__module__ = f.__module__

    def __call__(self):
        print("Entering", self.f.__name__)
        self.f()
        print("Exited", self.f.__name__)

@entry_exit
def func1():
    """函数func1"""
    print("call func1()")

@entry_exit
def func2():
    """函数func2"""
    print("call func2()")

func1()
func2()
print func1.__name__
print func1.__doc__
print func1.__module__
print func1.f.__name__
print func1.f.__doc__
print func1.f.__module__
```

    ('Entering', 'func1')
    call func1()
    ('Exited', 'func1')
    ('Entering', 'func2')
    call func2()
    ('Exited', 'func2')
    func1
    函数func1
    __main__
    func1
    函数func1
    __main__


其实 functools 包里面提供来了一个 wraps 装饰器, 可以完成函数属性不变的功能!


```python
from functools import wraps
def entry_exit(f):
    @wraps(f)
    def new_f():
        print("Entring", f.__name__)
        f()
        print("Exited", f.__name__)
    return new_f

@entry_exit
def func1():
    """函数func1"""
    print("call func1()")

func1()
print(func1.__name__)
print(func1.__doc__)
print(func1.__module__)
```

    ('Entring', 'func1')
    call func1()
    ('Exited', 'func1')
    func1
    函数func1
    __main__


以上装饰器的实现其实也很简单, 给出一个自己实现的简单版本!
```python
def wraps(decoratedFunc):
    def wrapper(decorator):
        decorator.__name__ = decoratedFunc.__name__
        decorator.__module__ = decoratedFunc.__module__
        decorator.__doc__ = decoratedFunc.__doc__
        return decorator
    
    return wrapper
```

# 复杂decorator
Python 内置了三个 基本的 decorator, `@staticmethod`, `@classmethod`, `@property`, 还有许多更加复杂的 decorator, 可以装饰类等, 还涉及到 metaprogramming, 下一次准备深入一下现实开源代码中的decorator, 先总结到这里!
```python
def bar():
    pass
bar = staticmethod(bar)

等价于
@staticmethod
def bar():
    pass
```

# 总结

什么是装饰器，就是装饰函数或者类的函数或类。那么他就必须接受 一个函数或者类当参数， 然后装饰它， 最后返回 装饰好的 函数或类。类比大象放冰箱分三步，decorator三步走：

 1. 把 被装饰者 传递给 装饰者;
 2. 在 装饰者 内部装饰 这个 被装饰者。由于 需要返回一个 被装饰的 对象， 因此 在装饰者 内部 一定至少会 定义一个新的 函数;
 3. 返回 装饰者 新定义的 callable 对象 即经过装饰后的对象;



装饰器并不仅仅体现了函数式编程， 它体现的更加本质的东西是**一处代码被另一处代码动态修改或者说替换**，这也正是编译型语言做不到的事情，例如C/C++就做不到。一个**语言要想动态的修改自己，绝对需要靠虚拟机或者解释器在运行的时候来帮忙**， 编译器只能在还没有运行的时候修改代码, 动态运行时候是没有办法做的；

Java具有反射机制，动态产生和构造一个类，实际上就是因为它有一个虚拟机，虚拟机提供了这种接口。所以，**解释型语言都具备很大的灵活性和可操作性。**

当然你也可以从另一个层面来理解装饰器， 就是以 decorator 为代码主体，即把里面的 decorator 看成是 C 语言里的 micro 宏定义， 把 decorated 看成实际的宏替换实例。例如 `define micro 777` 就是这里的 `@decorator decorated`, 只不过这里的 **宏 替换换过程是 静态的，装饰器替换是 动态的, 因为 C 语言是 静态 编译类型，Python 是 动态 解释类型语言嘛^\_^**;因此本质上就是**少写代码，动态修改** ^\_^

再来看看 decorator 的 @ 写法， 实际上是带领我们进入另一种 程序设计思维， 虽然本质上是 函数式 或者 宏替换。即 把 代码 运用到 代码 中。记住一个观点，软件工程的所有设计方法，有99%其实是因为懒。即为了 少 写 代码。代码写的越少，越易读，易维护，易扩展，易重构。但是少写代码并不意味着你真的少写了代码，实际上你看看开源社区，我们的代码 肯定是 越来越多的。但是 从 历史的 角度来看， 代码是越写越少。

# 参考
 - [Decorators](http://python-3-patterns-idioms-test.readthedocs.io/en/latest/PythonDecorators.html)
 - [Understanding Python Decorators in 12 Easy Steps!](http://simeonfranklin.com/blog/2012/jul/1/python-decorators-in-12-steps/#footnote_2)
 - [Python Decorators](https://pythonconquerstheuniverse.wordpress.com/2012/04/29/python-decorators/)
 - [Primer on Python Decorators](https://realpython.com/blog/python/primer-on-python-decorators/)web flask为例子讲解decorator使用场景
 - [guide-to-python-function-decorators](http://thecodeship.com/patterns/guide-to-python-function-decorators/)以html渲染为例子，还讲了functools.wraps的由来
 - [pep-0318](https://www.python.org/dev/peps/pep-0318/#current-syntax)
