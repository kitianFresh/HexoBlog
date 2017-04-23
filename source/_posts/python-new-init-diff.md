---
title:
  Python __new__ 和 __init__
date:
  2017-04-011
categories:
- python
tags:
- python
- object
- class
---


# Python \_\_new\_\_() & \_\_init\_\_()

Python 对象的创建涉及到两个函数, \_\_new\_\_() 和 \_\_init\_\_(). 这两个函数的区别是什么呢?

本文解决两个问题:
 1. 什么是静态方法
 2. 如何使用 \_\_new\_\_()
 
## static methods
和Java 一样, **Python 也拥有静态方法, 静态方法属于类,而不是类实例. Python 中申明一个静态方法通过装饰器 @staticmethod 完成!**


```python
class Alpha():
    def hack(self, a, b):
        self.a = a
        self.b = b
        print a, b
    def unhack(self):
        print self.a, self.b
```


```python
obj = Alpha()
obj.hack(1,2)
```

    1 2



```python
obj.unhack()
```

    1 2



```python
Alpha.hack(1,2)
```


    ---------------------------------------------------------------------------

    TypeError                                 Traceback (most recent call last)

    <ipython-input-4-36142b2a0e7c> in <module>()
    ----> 1 Alpha.hack(1,2)
    

    TypeError: unbound method hack() must be called with Alpha instance as first argument (got int instance instead)


>我们看到非静态方法属于类实例, 必须要一个实例对象做参数才能调用!那么是否可以这样呢?


```python
Alpha.hack(obj, 3, 4)
```

    3 4



```python
Alpha.unhack(obj)
```

    3 4


正常运行! 你会发现,其实 Python 使用 obj.hack() 只是像C++/Java一样的语法糖而已, 本质上是在调用一个方法(代码), 真实情况是 Alpha.hack(obj, 3, 4). 这就是为什么在 Python 中写 一个类, 实例方法必须要加上 self 的原因了! 

`unbound method hack() must be called with Alpha instance as first argument (got int instance instead)`.

**未绑定的方法, 被调用的时候 必须使用 实例变量做第一个参数!** 

>未绑定是什么意思? 

**正常情况下, obj.hack(\*args) 其实会被 Python 解释器 翻译成 Alpha.hack(obj, \*args), 和我们的猜测是一致的! 如果类的内部定义的方法, 没有 self 参数或者 @staticmethod, 那么方法其实就是一个孤立的方法, 命名空间在 Alpha 之下的一个普通方法而已, 并没有和某个实例或者类绑定起来!** 因此抛出 `unbound method` 异常!

这也是 Python 著名的动态绑定特性! 你写的一个 类的代码, 不一定就是这个类或者的实例最终的形态! 方法和属性都是可以动态的绑定到类实例的!


```python
class Beta():
    @staticmethod
    def hack(a, b):
#         self.a = a # 显然不能在静态方法中绑定 成员属性吧
#         self.b = b
        print a, b
    def unhack(self):
        pass
```


```python
Beta.hack(1,2)
```

    1 2


## \_\_new\_\_() & \_\_init\_\_()


```python
class A(object):
    def __init__(self, a, b):
        print "init called"
        print "self is", self
        self.a, self.b = a, b
```


```python
a = A(1,2)
```

    init called
    self is <__main__.A object at 0x7f64c004a210>


我们看到, **\_\_init\_\_() 并不创建对象, 而是直接使用对象了! 说明在 \_\_init\_\_() 调用之前, 对象已经被创建了!**

> 谁创建的对象?

其实是 **\_\_new\_\_() 方法创建的对象!** 以下是关于 \_\_new\_\_() 的几点规则:

1. **\_\_new\_\_() 肯定不是实例方法, 因为这个时候还没的实例!**
2. **\_\_new\_\_() 被调用当看到 `a=A()` 这种语句的时候.**
3. **\_\_new\_\_() 必须返回一个对象.**
4. **只有当 \_\_new\_\_() 返回一个对象的时候, \_\_init\_\_() 才会被调用.**
5. **\_\_new\_\_() 获取所有调用 class 时候传递过来的参数, 另外, 还有一个额外的 cls 参数!**
6. **\_\_new\_\_() 是从 object 类继承来的, 因为 Python 中所有类的基类都是 object.**

### 覆盖 \_\_new\_\_() 方法


```python
class A(object):
    def __new__(cls, *args, **kwargs):
        print cls
        print "args is", args
        print "kwargs is", kwargs
```


```python
a = A()
```

    <class '__main__.A'>
    args is ()
    kwargs is {}



```python
a = A(1,2, named="sad")
```

    <class '__main__.A'>
    args is (1, 2)
    kwargs is {'named': 'sad'}



```python
print a
```

    None


由于\_\_new\_\_() 没有返回对象, 因此 a 是 None.


```python
import datetime

class B(object):
    def __new__(cls, *args, **kwargs):
        instance = super(B, cls).__new__(cls, *args, **kwargs)
        setattr(instance, 'created_at', datetime.datetime.now())
        return instance
    def __init__(self, a, b):
        print "inside init"
        print self.created_at
        self.a, self.b = a, b
```


```python
obj = B(1,2)
```

    inside init
    2017-04-23 20:47:57.021831


    /usr/local/lib/python2.7/dist-packages/ipykernel/__main__.py:5: DeprecationWarning: object() takes no parameters



```python
class B(object):
    pass

class A(object):
    def __new__(cls, *args, **kwargs):
        new_instance = object.__new__(B, *args, **kwargs)
        return new_instance
a = A()
print a
```

    <__main__.B object at 0x7f64c005d590>


### Python2 & Python3

```python
#python2 写法, 兼容 Python3
return super(A, cls).__new__(cls, *args, **kwargs)

#python3 写法
return super().__new__(cls, *args, **kwargs)
```
