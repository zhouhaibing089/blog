### 对象(objects), 值(values)和类型(types)

*对象(objects)*是关于数据(data)的抽象, 在python中, 所有的数据都以对象或者对象间的关系来表示.(代码(code)我们也用对象来表示, 在某种意义上, 这正符合了冯.诺依曼的程序存储模型).

每个对象都有id, 类型和值三个属性. 对象的id从创建之后就不再改变, 比如, 你可以用内存地址来表示对象的id. `is`操作符用来检查两个对象的id是否相同, `id()`函数返回一个整数来表示对象的id.

> **CPython的实现细节**: `id(x)`返回x的内存地址.

对象的类型决定了该对象支持的操作集(operations)(例: 该对象是否有长度)和该对象可能的值(values). `type()`函数返回对象的类型(该类型本身也是一个对象), 和id一样, 对象的类型也是不可改变的.

有些对象的值是可变的, 我们称其为*mutable*, 对于不可变的对象则称之为*immutable*. 当一个immutable的容器(container)中包含的引用是指向mutable的对象时, 那么我们说该容器的值也是可变的, 但该容器对象仍然被认作是不可变的, 所以对象的不可变并不完全等同于该对象值的不可变. 对象的可变性取决于对象的类型, 例如: 数字(numbers), 字符串(strings)和元组(tuples)是不可变的, 列表(lists)和字典(dictionaries)是可变的.

对象从来不会被显示地清除, 但是, 当该对象不再被其他对象使用时, 它可能会被当做垃圾回收.

> **CPython的实现细节**: 回收策略目前使用的是引用计数和延迟循环引用检测(可选), 这种策略能够尽可能快地回收那些不可用的对象, 但是不保证能够立马回收那些具有循环引用的对象. 用户可以查看`gc`模块来获取关于如何控制循环引用下垃圾回收的信息. 其他的实现方式可能不同, CPython也可能改变其实现, 因此, 不要依赖于对象不可用时的终止化(finalization)过程(所以你总是应该显式地关闭文件).

注意, 当你使用一些跟踪(tracing)或调试(debugging)工具时, 可能会导致一些正常情况下被回收的对象仍然存活(alive), 另外当你使用`try...except`时也会有一些对象不被立马回收.

有一些对象会包含对外部资源(如文件和窗口)的引用, 因此我们期望当该对象被回收后, 它所指向的那些资源也能够被释放(freed), 但是由于回收过程不保证能立马执行, 所以那些对象通常都需要提供显式的方式来释放那些资源, 比如常见的`close()`方法.我们建议在程序中始终显式地执行那些方法. `with`和`try...finally`为你执行那些方法提供了便利.

有一些对象包含有对其他对象的引用, 我们称之为容器(container). 元组, 列表, 字典都是容器. 容器包含的引用是该容器值的一部分, 大部分情况下, 当我们说容器的值时, 我们说的不是容器包含的那些引用, 而是容器本身. 但是当我们说容器的可变性时, 我们说的是容器包含的对象的可变性, 所以一个不可变的容器(比如元组)包含可变对象的引用时, 该容器的值则随着所包含对象的变化而变化.

类型几乎影响了对象的方方面面, 甚至包括对象的id: 对于不可变的类型, 那些计算出新值的操作可能返回一个指向已存对象的引用, 而这在可变对象的情况下是不允许的. 比如, 在执行`a = 1; b = 1`后, a和b可能指向相同的对象, 但是执行`c = []; d = []`后, c和d一定会指向不同的对象.(注意如果`c = d = []`则是将相同的对象赋给c和d).

### 标准类型结构(The standard type hierarchy)

下面是一些内建于python中的类型. 扩展模块(extension modules)可以定义额外的类型, 未来的版本也可能添加新的类型(例如实数, 高效存储整数数组等), 尽管这些类型经常会通过标准库的形式来提供.

下面有些类型的描述中会包含关于特殊属性(special attributes)的段落, 这些属性通常用来访问实现中的相关细节, 并不适合于通常情况下的使用.(那些属性的定义可能会在将来进行调整).

**`None`**: 此类型只包含一个值且只有一个对象代表此值, 该对象可以通过`None`名称来访问. 在很多情况下, 我们会用该对象来表示某个值的缺失(absence), 比如我们当一个函数不返回任何值时, 我们用`None`来代替. 它的真值测试(truth value)为`False`.

**`NotImplemented`**: 此类型只包含一个值且只有一个对象代表此值, 该对象可以通过`NotImplemented`名称来访问. 数值方法(numeric methods)以及富比较(rich comparison methods)方法在未实现时应该返回此值(取决于具体的方法, 解释器会接着尝试该方法的镜像操作(reflected operation)或其他后备操作). 它的真值测试为`True`.[注: 镜像操作举例, `==`的镜像操作为`!=`]

**`Ellipsis`**: 此类型只包含一个值且只有一个对象代表该值, 该对象可以通过`...`字面量或者`Ellipsis`名称来访问. 它的真值测试为`True`.

**`numbers.Number`**: 数值类型的对象通常由字面量创建或由相关算术操作函数返回. 数值类型是不可变的, 一旦被创建, 他们的值就不能被改变. python中的数值和数学中的数值有着很强的相关性, 除了多了些关于计算机方面的限制.

Python将数值区分为三种子类型: 整数(integers), 浮点数(floating point numbers)和复数(complex numbers).

*   **`numbers.Integral`**: 表示了数学中的整数(正整数与负整数), 具体又分为两类.
    *    Integers(`int`): 这个表示了所有的整数, 范围取决于可用的(虚拟)内存. 位移(shift)和掩码(mask)操作都基于整数的二进制表示, 负整数会以补码的形式表示, 同时左边补上符号位.
    *    Booleans(`bool`): 表示了真值测试中的两个值`True`和`False`, 分别用对象`True`和`False`表示. Boolean是integer的子类, 大部分情况下, 它的两个值在整数中的行为分别等同于`1`和`0`, 除了在转换成字符串时, 他们会分别转成`"True"`和`"False"`.

*   **`numbers.Real(float)`**: 表示了机器层面的双精度浮点数. 关于浮点数的范围以及如何处理溢出都依赖于底层系统的架构. Python不支持单精度浮点数, 因为使用单精度浮点数在处理器以及内存方面的节省(savings)通常都不及使用objects带来的开销, 所以我们没有必要使用两种精度的浮点数来使语言变得更复杂.

*   **`numbers.Complex(complex)`**: 表示了一对(pair)机器层面的双精度浮点数. 因此约束与限制都可以参考上面的浮点数. 复数`z`的实部和虚部可以通过它的两个只读属性(`z.real`和`z.imag`)来获得.

**`Sequences`**: 表示有穷(finite)有序(ordered)通过非负整数来索引的序列. `len()`函数返回序列中元素的个数. 索引值从0开始, 因此当序列中元素个数为`n`时, 索引(index)的最大值为`n-1`, `a[i]`表示第i个元素.

序列支持切片(slicing)操作, `a[i:j]`选择了序列a中索引k大于等于i小于j(i <= k < j)的所有元素. 切片操作作为表达式时, 它表示的是一个新的同类型的序列(也就是说返回的序列的索引重新调整为从0开始).

有些序列还支持扩展的切片操作(extended slicing), 它允许提供第三个参数指定步长(step). `a[i:j:k]`选择了所有索引x的值, 且x满足`x = i + n * k`和`i <= x < j`.

序列根据它的可变性分为两种:

*    **不可变序列**: 不可变序列对象一旦创建就不可再改变.(它包含的其他对象可能是可变的, 因而它们的值也是可以变化的, 然而由不可变的引用直接指向的对象是不能改变的). 以下类型是不可变序列:

    *   **`Strings`**: string是unicode码的序列. 它的每个unicode码值处在`U+0000-U+10FFFF`范围中. Python没有`char`这种类型, 取而代之的是一个长度为1的string. `ord()`函数将一个unicode码的string形式转成一个`0 - 10FFFF`范围内的整数, `chr()`函数则将一个处在`0 - 10FFFF`范围内的整数转成对应的长度为1的string. `str.encode()`方法可以将string以特定编码转成bytes对象. `bytes.decode()`方法则相反, 它将bytes对象转成特定编码的string.

    *   **`Tuples`**: tuple中的项可以是任意的Python对象. 含两项及两项以上的tuple由逗号分隔的表达式组成. 只含有一项的tuple需要在该表达式后附加一个逗号(因为括号必须用来将表达式分组(grouping), 同时单个表达式本身也无法组成tuple). 空的元tuple由一对空括号表示.

    *   **`Bytes`**: bytes是不可变的数组, 它的每一项都是由8个bit组成的byte(用整数来表示就处在`0-256`之间). bytes字面量(`b'abc'`)和`bytes()`函数可以用来创建bytes对象. 同样, bytes对象可以通过`decode()`方法可以转成字符串.

*   **可变序列**: 可变序列在创建之后仍然可以改变. 下标(subscription)和切片标记(slicing)都可以用来作为赋值的对象和`del`语句的参数. 目前有两种内建可变的序列类型(扩展模块`array`和`collections`提供了一些其他的可变序列类型):

    *   **`Lists`**: list中的每一项可以是任意的Python对象. 它由用逗号分隔的表达式加上方括号组成. (针对长度为0或1的list, 我们并没有特殊的标记).

    *   **`Byte Arrays`**: bytearray对象是一个可变的数组. 该对象由`bytearray()`构造方法创建. 它和bytes对象的区别只在于它是不可变的(因而也是无法进行hash的), 其他的接口及功能都是一样的.

**`Set`**: 集合类型表示无序(unordered)有穷(finite)唯一(unique)不可变(immutable)的对象集. 因此, 集合中的对象无法用数值下标(subscription)进行索引. 但是, 它可以被遍历(iterated over), `len()`函数返回集合中的元素数量. 集合类型的常见使用场景: 快速成员检查(membership testing), 序列去重(remove duplicates)和一些常见的数学计算(交集, 并集, 差集, 对称差集等).

集合中的元素, 和字典中的键值一样, 需要是不可变类型. 注意数值类型遵循数值上的比较规则: 如果两个数值比较起来是相等的(如`1`和`1.0`), 那么只有其中一个能出现在集合中.

目前有两种内建的集合类型:

*   **`Sets`**: 表示可变的集合, 通过内建构造函数`set()`创建. 创建之后可以通过若干方法来改变它(如`add()`方法).

*   **`Frozen sets`**: 表示不可变的集合, 通过内建构造函数`frozenset()`创建. 因为frozenset是不可变的, 也是可hash的, 因此它本身也可以作为集合中的元素或作为字典的键值.

**`Mappings`**: 表示有穷(finite)的通过任意集合索引的对象集. `a[k]`表示从映射`a`中取出由`k`索引的对象值, 它可以作为赋值的对象也可以作为`del`语句的参数. `len()`函数返回映射中的所有元素个数.

目前只有一种内建的映射类型:

*   **`Dictionaries`**: 表示有穷(finite)的通过几乎任意对象索引的对象集. 不能用来作为索引的类型包括列表, 字典以及其他任何通过值比较而不是通过对象id来比较的可变类型, 之所以有这个限制是因为字典的实现要求索引的hash值是一个固定的常数. 数值类型作为索引遵循数值比较的规则: 如果两个数值比较起来相等的话, 那么不管用哪个做索引, 访问的值都是一样的. 字典类型是可变的, 它可以通过`{...}`标记来创建. 扩展模块`dbm.ndbm`, `dbm.gnu`和`collections`中提供了其他的映射类型.

**`Callable types`**: 有两种类型可以执行函数操作.

*   **`User-defined functions`**: 用户自定义的函数通过函数定义来创建, 调用该函数时需要提供和形参数量相同的参数列表. 它拥有以下特殊属性:

| 属性               | 含义                                                              |       |
|-------------------|------------------------------------------------------------------|-------|
| `__doc__`         | 函数的文档字符串, 如果没有, 则为`None`, 不会被子类继承                    | 可写   |
| `__name__`        | 函数的名字                                                         | 可写   |
| `__qualname__`    | 函数的全限定名称, Python3.3中新加的                                   | 可写   |
| `__module__`      | 函数所定义在的模块, 如果没有, 则为`None`                                | 可写   |
| `__defaults__`    | 函数中所有拥有默认值的参数值组成的元组, 如果没有, 则为`None`                | 可写   |
| `__code__`        | 编译后函数体的代码对象                                                | 可写   |
| `__globals__`     | 指向全局变量字典的引用, 即所定义在的模块的全局命名空间                      | 只读   |
| `__dict__`        | 一个字典引用, 用来存储在函数上添加的任意属性                              | 可写   |
| `__closure__`     | 函数中对自由变量绑定的元组                                             | 只读   |
| `__annotations__` | 存储参数注解的字典, 字典的键值为参数名, 如果带有return注解, 则键值为`return`  | 可写  |
| `__kwdefaults__`  | 仅存储关键字参数(keyword-only)的默认值字典                              | 可写  |

```python
# 译者给出的示例:
def f(a, b=1, *c, d=2):
    '''
    function documentation
    '''
    def g():
        print(h)
    h = 3
    return g
f.__doc__ # 'function documentation'
f.__name__ # 'f'
f.__qualname__ # 'f'
f.__module__ # __main__
f.__defaults__ # (1,)
f.__code__ # <code object f at 0x7f3636318660, file "<stdin>", line 1>
f.__globals__ # ...
f.__dict__ # {}
f.__annotations__ #
f.__kwdefaults__ # {'d': 2}
f(0).__closure__[0].cell_contents # 3
```

基本上所有标为可写的属性在进行赋值时都会做类型检查.

函数对象也支持用点标记(dot-notation)获取和设置任意属性, 比如给函数添加元数据信息(metadata). *注意在目前的实现中, 只有用户定义的函数支持设置其他属性, 内建函数不支持(未来的版本可能会支持)这一操作*.

关于函数定义的其他信息可以通过代码对象来获得.

*   **`Instance methods`**: 实例方法涉及到类, 类实例和其他可调用对象(通常是用户定义的函数).

在实例方法仅可读的特殊属性中, `__self__`是类实例, `__func__`是函数对象, `__doc__`是方法的文档字符串(与`__func__.__doc__`相同). `__name__`是方法的名称(与`__func__.__name__`相同), `__module__`是包含该方法定义的模块, 如果没有, 则为`None`.

实例方法也支持访问`__func__`的任意属性.

当我们获取一个类或类实例的属性, 并且该属性恰好是一个用户定义的函数或类方法时, 方法对象就被创建了.

当我们从类实例访问实例方法(instace method)时, 该方法的`__self__`就是该类实例. 并且我们称原函数被绑定了. 实例方法的`__func__`属性为原用户定义的函数对象.

当我们从类或类实例的其他实例方法获取一个实例方法时, 这个方法和普通函数对象的行为是一样的, 除了该方法的`__func__`属性不是原来的方法对象, 而是该方法对象的`__func__`属性.

当我们从类或类实例访问类方法(class method)时, 该方法的`__self__`为类本身. 类方法的`__func__`属性为原函数对象.

```python
# 译者给出的示例
class A:
    def f(self):
        pass
    @classmethod
    def cf(cls):
        pass

# 从类实例访问实例方法
a = A()
a.f()
# 从类实例的其他实例方法创建实例方法
b = A()
b.g = b.f # b.g为新创建的实例方法
b.g.__func__ == b.f.__func__ # True
A.h = A.f # A.g为新创建的实例方法
c = A()
c.h.__func__ = c.f.__func__ # True
# 从类或类实例访问类方法
A.cf == a.cf
A.cf.__self__ == A # True
```

当一个实例方法被调用时, 该实例方法的`__func__`指向的函数对象被执行, 同时将类实例本身(`__self__`)插入到参数列表中第一位. 比如说类`C`定义了一个函数`f`, `x`是类`C`的一个实例, 那么`x.f(1)`等价于`C.f(x, 1)`.

当一个实例方法刚好来自于类方法时, 那么该实例方法的`__self__`则是类本身, 所以`x.f(1)`和`C.f(1)`等价于`f(C, 1)`.

注意当我们每次从实例访问函数对象时, 该函数对象会自动转换成实例方法对象. 在某些情况下, 我们可以通过优化来避免这里的多次转换开销, 比如我们将该实例属性保存成一个局部变量. 同时你也要注意这个转换过程只针对用户定义的方法对象, 其他的可调用以及不可调用对象都不会有这个过程. 还需要注意类实例的函数属性不会转成绑定方法, 只有当该函数对象是类的属性时才会发生转换.

*   **`Generator functions`**: 使用了`yield`语句的函数或者方法被称作生成函数(*generator function*), 这样的函数在被调用时, 总是会返回一个迭代器(iterator)对象. 调用该迭代器的`iterator.__next__()`方法时会执行该函数直到`yield`提供了一个值. 当该函数有`return`语句或已经执行到最后, 此时会产生一个`StopIteration`异常用来表示该迭代器已经没有更多值了.

*   **`Built-in functions`**: 内建函数是对底层C函数的包装, 比如`len()`和`math.sin()`(`math`是内建模块), 参数的数量和类型取决于被包装的C函数, 一些特殊的只读属性包括: `__doc__`是函数的文档字符串, 如果没有, 为`None`, `__name__`是函数的名称, `__self__`为`None`, `__module__`是该函数定义处在的模块, 如果没有, 则为`None`.

*   **`Built-in methods`**: 它也是对底层C函数的包装, 只是该C函数在调用时会隐式地传一个对象参数. 比如`alist.append()`是一个内建方法(假定`alist`是一个list对象). 它的`__self__`属性为该对象参数.

*   **`Classes`**: 类也是可调用的, 它通常用来作为创建类实例的工厂, 当然如果你重载`__new__`方法的话, 你也可以创建其他类的实例. 调用类时的参数会传给`__init__`方法, 在大部分情况下, 我们通过`__init__`方法来初始化该实例.

*   **`Class Instances`**: 如果该类实例有`__call__`方法的话, 那么该类实例也是可以调用的.

```python
# 译者给出的示例
# Generator Functions
def fibo(n):
    a, b = 0, 1
    for i in range(0, n):
        yield a
        a, b = b, a + b
b = fibo(1) # <generator object fibo at 0x7f36333dfc60>
# 有__iter__和__next__方法, 表示遵循iterator协议
b.__iter__ # <method-wrapper '__iter__' of generator object at 0x7f36333dfca8>
b.__next__ # <method-wrapper '__next__' of generator object at 0x7f36333dfca8>
list(b) # [0]
# Built-in Functions
abs # <built-in function abs>
# Built-in Methods
l = [1, 2, 3]
l.append # <built-in method append of list object at 0x7f36333f7e08>
# Classes和Class Instances
class Constructor:
    def __new__(cls):
        return super().__new__(cls)
    def __init__(self):
        self.p1 = 1
        self.p2 = '2'
    def __call__(self):
        return self.p1 * self.p2
c = Constructor() # 像调用方法一样调用类名
c() # 像调用方法一样调用类实例
```

**`Modules`**: 模块是Python代码的基本组织单元, 它通过import系统创建, 比如常规的`import`语句, `importlib.import_module()`方法和内建方法`__import__()`. 模块含有一个字典对象, 用来存储模块的命名空间(该字典对象也是该模块中定义的所有函数的`__globals__`属性). 访问模块的属性都会转成访问模块的字典对象, 因此`m.x`等价于`m.__dict__['x']`. 模块对象不包含有初始化的代码对象(因为模块的初始化只做一次).

模块属性的赋值更新等价于更新模块的字典对象, 因此`m.x = 1`等价于`m.__dict__['x'] = 1`.

模块的`__dict__对象`是只读的.

> CPython的实现细节: 当模块不再能被访问时, 模块的字典对象会被清除(即使该字典还包含有存活对象的引用). 为了避免这种行为, 你可以复制该字典对象或者在你直接访问模块字典对象时始终保存该模块对象.

预定义的一些可写属性: `__name__`是模块名称. `__doc__`是模块的文档字符串, 如果不可用则为`None`. `__file__`是模块加载的文件路径(如果它是从文件加载的话). `__file__`属性对于某些模块来时可能会缺失(missing), 比如静态链进解释器中的C模块. 对于从共享库动态加载进来的扩展模块, `__file__`则会表示该共享库的路径名.

```python
# 译者给出的示例
import math as m
m.__name__ # 'math'
m.__doc__ # 'This module is always available.  It provides access to the\nmathematical functions defined by the C standard.'
m.__file__ # '/usr/lib/python3.4/lib-dynload/math.cpython-34m.so'
import sys as s
s.__file__ # AttributeError: 'module' object has no attribute '__file__'
```

**`Custom classes`**: 自定义类型一般通过类定义来创建. 类有一个通过字典对象来维护的命名空间. 类的属性访问也会转成对该字典对象的查询, 因此`C.x`等价于`C.__dict__['x']`(虽然也有很多钩子函数(hooks)来自定义属性的查找规则). 当属性名未找到时, 查找过程会转向父类继续(使用C3方法, 该方法能正确处理菱形继承结构). 关于C3方法可以查看[https://www.python.org/download/releases/2.3/mro/](https://www.python.org/download/releases/2.3/mro/)(注: 该文译者也进行了翻译, 位于[这里](https://github.com/zhouhaibing089/Blog/blob/master/%5B%E8%AF%91%5DThe%20Python%202.3%20Method%20Resolution%20Order.md)).

当一个类属性访问的是类方法时(class method), 他会转成一个实例方法对象(该方法的`__self__`为类本身). 当一个类属性访问的是静态方法时(static method), 他会转成另外一个包装对象. 另外, 我们可以通过描述符协议来实现另外一种属性访问方式.

类的属性赋值只会更改当前类的字典, 不会影响到父类.

可以通过调用类对象来创建一个类实例.

一些特殊属性: `__name__`为类名, `__module__`是该定义所在的模块, `__dict__`是包含该类名称空间的字典. `__bases__`是一个包含父类的元组(可能为空或只包含一个元素, 元素顺序与声明顺序一致), `__doc__`是类的文档字符串, 如果不可用, 为`None`

```python
# 译者给出的示例
class A:
    '''demo class A'''
    pass
class B(A):
    '''demo class B'''
    pass
A.__name__ # 'A'
A.__module__ # '__main__'
A.__dict__ # mappingproxy({'__module__': '__main__', '__dict__': <attribute '__dict__' of 'A' objects>, '__weakref__': <attribute '__weakref__' of 'A' objects>, '__doc__': 'demo class A'})
A.__bases__ # (<class 'object'>,)
A.__doc__ # 'demo class A'
B.__bases__ # (<class '__main__.A'>,)
```

**`Class instances`**: 类实例通过调用类方法创建. 类实例也包含一个存储命名空间的字典, 该字典总是作为属性访问的首选查询之处, 未找到时, 类的属性会被接着搜索, 如果搜索到的是一个用户自定义函数, 该函数会转成一个实例方法对象(该方法的`__self__`为实例本身). 静态方法与类方法也会进行转换, 同样的, 我们也可以通过实现描述符协议来实现另外的属性访问方式. 如果类属性也没能找到该属性, 那么该类定义的`__getattr__()`方法会被调用.

属性的赋值和删除都是直接在实例的字典上进行操作, 而且永远不会修改类的字典. 如果该类定义了`__setattr__()`或`__delatttr__()`方法, 那么这些方法会被调用, 而不是修改类实例的字典.

类实例可以表现的像数字, 序列或者映射(只要他们定义了对应的方法).

特殊属性: `__dict__`是实例的字典, `__class__`是对应的类.

**`I/O objects(也被称作file objects)`**: 文件对象表示打开的文件. 有许多快捷方式来创建文件, `open()`内建函数, `os.popen()`, `os.fdopen()`, `makefile()`(用来创建套接字对象, 也可以通过扩展模块的方法来实现).

`sys.stdin`, `sys.stdout`和`sys.stderr`是三个文件对象, 分别表示解释器的标准输入, 标准输出和标准错误流. 他们都是以文本模式打开的, 所以也就都遵循`io.TextIOBase`中定义的接口.

**`Internal types`**: 这些类型是解释器暴露给用户的一些内部类型(internally). 它们的定义可能在将来的版本中做些调整, 它们在这被提及仅仅是为了本文档的完整性.

*   **`Code objects`**: 代码对象表示字节编译(*byte-compiled*)的可执行Python代码, 也可称为字节码(*bytecode*). 代码对象和函数对象的区别在于函数对象包含一个显式的指向globals的引用, 而代码对象是不包含上下文的, 另外的, 默认参数值存储在函数对象中, 而不在代码对象中(因为它们代表运行时计算的值). 与函数对象不同的是, 代码对象是不可变的, 而且它也不包含(直接或间接)任何指向可变对象的引用.

一些特殊的只读属性: `co_name`给出函数的名称, `co_argcount`是位置参数(positional arguments)的数量(包括带默认值的参数). `co_nlocals`是函数使用的局部变量的数量(包括参数). `co_varnames`是包含所有局部变量名称的元组. `co_cellvars`是包含所有被内部函数(nested functions)引用的局部变量名称的元组. `co_freevars`是包含自由变量名称的元组. `co_code`是字节码指令的字符串表示. `co_consts`是包含字节码中所使用的所有字面量的元组. `co_names`是字节码中所使用的名称的元组. `co_filename`表示该字节码是从哪个文件编译而来. `co_firstlineno`是函数的第一行编号. `co_lnotab`是一个字符串, 表示字节码偏移到行数的映射. `co_stacksize`表示需要的栈的大小(包括局部变量). `co_flags`是一个整数, 表示关于解释器的一些标记.

下面是一些可以用于`co_flags`的标记位. `0x04`表示该函数使用了`*arguments`语法来接受任意多的位置参数. `0x08`表示该函数使用了`**arguments`语法来接受任意多的关键字参数. `0x20`表示该函数是一个生成器(generator).

Future特征声明(`from __future__ import division`)也使用了`co_flags`. 它用来表示该代码对象是在启用了特定功能的情况下编译的. `0x2000`表示该函数启用了future division. `0x10`和`0x1000`在早期的Python版本中也有用到.

关于`co_flags`的其他位被保留以供内部使用.

如果代码对象表示一个函数, 那么`co_consts`的第一项表示该函数的文档字符串, 如果未定义, 则为`None`.

```python
# 译者给出的示例
def f(a, b=1, *c, d=2, **e):
     '''demo function'''
     g=1
     def h():
         i = g + 1
         return i
    return h
c = f.__code__ # 取得代码对象
c.co_name # 'f'
c.co_argcount # 2
c.co_nlocals # 6
c.co_varnames # ('a', 'b', 'd', 'c', 'e', 'h')
c.co_cellvars # ('g',)
c.co_freevars # ()
c.co_code # b'd\x01\x00\x89\x00\x00\x87\x00\x00f\x01\x00d\x02\x00d\x03\x00\x86\x00\x00}\x05\x00|\x05\x00S'
c.co_consts # ('demo function', 1, <code object h at 0x7f13cfc62d20, file "<stdin>", line 4>, 'f.<locals>.h')
c.co_names # ()
c.co_filename # '<stdin>'
c.co_firstlineno # 1
c.co_lnotab # b'\x00\x02\x06\x01\x12\x03'
c.co_stacksize # 3
c.co_flags # 15
h = f(1)
h.__code__.co_freevars # ('g',)
```

*   **`Frame objects`**: Frame对象表示执行Frame. 它可能出现在Traceback对象中.

特殊的只读属性: `f_back`表示前一个栈帧(stack frame, 指向调用者). 如果已经在栈底, 则为`None`. `f_code`表示当前栈帧中的可执行代码对象. `f_locals`是一个用来查询局部变量的字典.    `f_globals`是用来存储全局变量的. `f_builtins`用来查询内建的名称(built-in names). `f_lasti`给出精确的上一个指令(它是一个索引, 指向代码对象的指令字符串的某个位置).

特殊的可写属性: `f_trace`, 如果不是`None`的话, 则表示在每行源代码执行之前都会调用的一个函数. `f_lineno`表示帧中的当前代码行数, 在trace函数中设置该变量可以使执行跳转到指定的行数. 调试器可以通过设置此变量来实现跳转命令(jump command).

帧对象只支持一个方法: `frame.clear()`, 该方法会清除该帧指向局部变量的所有引用. 同样的, 如果该帧属于一个生成器, 那么该生成器也会被终止. 这个方法对于打破关于帧对象的循环引用非常有帮助(比如捕获异常时还存有它的trackback时). 如果当前帧还在执行时, 调用此方法会抛出`RuntimeError`.(该方法是3.4版本中新加的)

*   **`Traceback objects`**: 该对象表示发生异常时的回溯栈. trackback对象在发生异常时创建. 当搜寻异常处理器unwinds执行栈时, 在每个unwind层级都会在当前traceback之前插入一个新的traceback对象. 当异常处理器执行时, 程序可以获取回溯栈对象(`sys.exc_info`返回的元组里的第三个元素). 当程序并没有恰当的异常处理器时, 回溯栈会被打印到标准错误流中. 当用户使用交互式解释器时, 回溯栈对象也可通过`sys.last_traceback`对象获得.

特殊的只读属性: `tb_next`是回溯栈的下一级(朝向异常发生的帧), 如果没有, 则为`None`. `tb_frame`指向当前级的执行帧. `tb_lineno`表示异常发生所在的行数. `tb_lasti`表示发生异常的指令. 如果异常发生在没有`except`或`finally`的`try`语句中, `tb_lineno`和`tb_lasti`与`tb_frame`对象中的行数可能会不一样.

```python
# 译者给出的示例
import sys
def f():
    try:
        1 / 0
    except ZeroDivisionError:
        raise ValueError
f()
# Traceback (most recent call last):
#   File "<stdin>", line 3, in f
# ZeroDivisionError: division by zero
#
# During handling of the above exception, another exception occurred:

# Traceback (most recent call last):
#   File "<stdin>", line 1, in <module>
#   File "<stdin>", line 5, in f
# ValueError
t1 = sys.last_traceback # the ValueError traceback
t2 = t1.tb_next # the ZeroDivisionError traceback
t1.tb_frame # <frame object at 0x7fbe832efba8>
t2.tb_frame # <frame object at 0x7fbe832eabe8>
t1.tb_lineno # 1
t2.tb_lineno # 5
t1.tb_lasti # 3
t2.tb_lasti # 31
t1.tb_lineno == t1.tb_frame.f_lineno # True
t2.tb_lineno == t2.tb_frame.f_lineno # True
```

*   **`Slice objects`**: slice对象用来表示`__getitem__()`方法中的切片对象. 它们也可以通过内建函数`slice()`来创建.

特殊的只读属性: `start`表示下界, `stop`表示上界, `step`表示步长, 如果未指定, 则为`None`. 这些属性可以为任意类型.

切片对象只支持一个方法: `slice.indices(self, length)`, 这个方法接收一个整数参数`len`, 然后计算出该切片对象应用到长度为`len`的序列时的相关信息. 它返回长度为3的元组, 对应于`start`, `stop`和`step`. 缺失(missing)或者越界(out-of-bounds)与常规使用切片对象的行为一致.

```python
# 译者给出的示例
s = slice(1, 6, 1)
s.start # 1
s.stop # 6
s.step # 1
s.indices(10) # (1, 6, 1)
s.indices(4) # (1, 4, 1)
```

*   **`Static method objects`**: 静态方法对象提供了一种将函数对象转成方法对象的方式. 一个静态方法对象通常是对用户自定义函数的包装. 当我们通过类或者类实例访问静态方法对象时,实际上返回的是一个包装对象. 该对象不会再被做任何转换(transformation). 静态方法对象所包装的对象通常是可调用的, 但其本身是不可调用的. 静态方法对象由内建构造方法`staticmethod()`创建.

```python
# 译者给出的示例
def f(a, b):
    return a + b
sf = staticmethod(f)
sf(1, 2)
# Traceback (most recent call last):
#   File "<stdin>", line 1, in <module>
# TypeError: 'staticmethod' object is not callable
sf # <staticmethod object at 0x7fbe831f0668>
class A:
    pass
A.f = sf
A.f(1, 2) # 3
```

*   **`Class method objects`**: 类方法对象, 和静态方法对象类似, 也是对另外一个对象的包装, 并且当该对象通过类或者类实例访问时, 修改其默认行为. 关于类方法的行为已在上文进行过描述. 类方法对象可以通过内建构造方法`classmethod()`创建.

```python
# 译者给出的示例
def f(cls, a, b):
    return a + b
cf = classmethod(f)
class A:
    pass
A.f = cf
A.f(1, 2) # 3
a = A()
a.f(1, 2) # 3
```

### 特殊方法名称(special method names)

类可以通过实现一些特定的操作来支持一些特殊的语法(比如算术操作(arithmetic operations), 下标索引(subscripting)以及切片(slicing)等). 这是Python实现操作符重载的方式, 它允许类针对语言定义的操作符定义自己的行为. 比如, 如果一个类定义了`__getitem__()`方法, 并且`x`是该类的一个实例, 那么`x[i]`基本等价于`type(x).__getitem__(x, i)`. 除非特别提及, 当没有定义对应的方法却尝试执行某操作时会抛出相关异常(通常是`AttributeError`或`TypeError`).

当实现的类需要模仿内建类型的行为时, 该行为对该类一定是要有意义的, 这一点非常重要. 比如说, 某些序列类型随机访问索引处元素可能是可以工作的, 但通过切片选择可能就没有意义.(在W3C的文档对象模型中`NodeList`就是这么一个例子).

#### 基本的自定义(Basic customization)

```python
object.__new__(cls[, ...])
```

调用该方法创建一个新的类实例, `__new__()`是一个静态方法(因为这是一个特例, 所以你无需特别声明), 它的第一个参数是要创建实例的类对象(ths class of which an instance was requested). 其他的参数是调用类时(the call to the class)传递的其他参数. `__new__()`返回的值是新的对象实例(通常是`cls`的一个实例).

通常的实现方式是先通过调用父类的`__new__()`创建一个实例(`super(currentClass, cls).__new__(cls, ...)`), 然后再对父类创建的实例进行稍许修改.

如果返回值是`cls`的实例, 那么该实例的`__init__()`会被调用, 且其第一个参数为该实例本身, 其他的参数与传给`__new__()`的参数一致.

如果返回值不是`cls`的实例, 那么该实例的`__init__()`方法不会被调用.

`__new__()`主要是用来允许继承不可变类型(如`int`, `str`, `tuple`等). 它也经常出现在需要自定义类创建过程时, 比如自定义一个元类(metaclass).

```python
object.__init__(self[, ...])
```

在`__new__()`创建出新实例后和返回给调用者之前被调用. 参数是传给类的构造表达式中的参数. 如果基类有`__init__()`方法, 那么那些继承自该基类的子类都需要在自己的`__init__()`方法中显式地调用父类的`__init__()`方法. 比如`BaseClass.__init__(self, [args...])`.

因为`__new__()`和`__init__`一起合作来创建新实例(`__new__()`创建类实例, `__init__()`设置好该实例), 因此返回值一定不是`None`, 返回`None`会造成`TypeError`.

```python
object.__del__(self)
```

在对象将被销毁时调用, 因此也被称为类的析构方法(destructor), 如果父类有`__del__()`方法, 那么子类的`__del__()`方法中一定要显式地调用父类的`__del__()`. 值得注意的是, 你可以在`__del__()`方法的实现中增加一个对当前对象的引用, 这样可以起到推迟对象销毁的作用(这样的话你可以在将来某个时机来清除这个新引用). 另外一点, 在解释器退出的时候并不能保证每个还存活的对象都会调用其`__del__()`方法.

> 注意, `del x`并不会直接调用`x.__del__()`——前者是减少对象的引用数(reference count), 而后者只在引用数为0时才会被调用. 下面的一些场景会使引用计数无法成为0: 对象间的循环引用(双向链表, 包含父子节点指针的树等); 在函数中捕获异常时包含有一个指向处在栈上的对象引用(`sys.exc_info()[2]`使得栈帧一直不被回收); 在交互模式下(interactive mode)含有指向未捕获异常函数中的栈帧对象引用(`sys.last_traceback`使得栈帧对象一直不被回收). 第一种情景只能依靠显式地打破循环引用来解决, 第二种场景可以在栈帧对象不再可用时释放该引用, 第三种可以通过设置`sys.last_traceback`为`None`来解决. 默认情况下, 循环引用的垃圾清除会自动检测并执行, 关于这方面的更多材料可以查阅`gc`模块.

> 由于执行`__del__()`方法时的不确定性, 执行过程当中的任何异常都会被忽略, 但是会在标准错误输出上打印出一些警告(warning). 另外, 当一个模块被删除时, `__del__()`方法执行时引用的一些全局变量可能已经不可用了. 因此, `__del__()`方法中应该尽量少地引入外部的不变量. 从1.5版本开始, Python会保证模块中所有以下划线打头的变量会先于模块中其他对象被删除. 如果没有任何指向这些全局变量的引用, 这对于在执行`__del__()`方法时那些引入的模块仍然可用会有一些帮助.

```python
object.__repr__(self)
```

当我们通过内建函数`repr()`来希望获取一个对象的正式字符串表示(official string representation)时被调用. 如果可行的话, 这个字符串应该看起来像是一个合法的Python表达式, 通过这个表达式, 我们可以在特定的环境中重建此对象. 如果不可行, 那么该字符串应该看起来像`<...一些有用的描述...>`. 返回的必须是一个字符串对象, 如果类只定义了`__repr__()`, 没有定义`__str__()`方法, 那么`__repr__()`返回的结果也会作为非正式的字符串表示.

这个方法通常用来辅助调试, 所以返回的字符串信息应该尽可能地包含多一点信息, 以及不要产生歧义.

```python
object.__str__(self)
```

内建构造方法`str(object)`和内建函数`format()`与`print()`会调用对象的这个方法来生成非正式或者适于打印的字符串表示. 返回值必须是一个字符串对象.

与`object.__repr__()`不同的是, 这个方法不被期望返回一个有效的Python表达式, 相反, 它返回的是更方便(convenient)且更准确(concise)的字符串表示.

```python
object.__bytes__(self)
```

内建构造方法`bytes()`会调用对象的这个方法来计算该对象的字节字符串(byte-string)表示. 返回值是一个`bytes`对象.

```python
object.__format__(self, format_spec)
```

内建函数`format()`以及`str`的扩展方法`str.format()`会调用此方法来产生一个格式化的字符串. `format_spec`是一个字符串参数, 它包含了期望的格式化选项的信息. 对`format_spec`参数的解析由实现了`__format__()`方法的类型决定. 大部分实现基本都要不是委托(delegate)给内建类型, 要不就是使用类似的格式化选项语法.

[formatspec](http://www.baidu.com)[还未翻译到对应章节]有关于标准格式化语法的描述.

返回值必须是字符串类型.

> Python3.4中的改动: 如果`object`的`__format__()`方法接收了任意非空字符串, 它会抛出了`TypeError`异常.

```python
object.__lt__(self, other)
object.__le__(self, other)
object.__eq__(self, other)
object.__ne__(self, other)
object.__gt__(self, other)
object.__ge__(self, other)
```

这些方法也被称作"富比较(rich-comparison)方法", 比较符号与方法名称之间的对应关系如下: `x < y`调用`x.__lt__(y)`, `x <= y`调用`x.__le__(y)`, `x == y`调用`x.__eq__(y)`, `x != y`调用`x.__ne__(y)`, `x > y`调用`x.__gt__(y)`, `x >= y`调用`x.__ge__(y)`.

这些富比较方法可能返回`NotImplemented`, 如果这样的话, 表示对应的的操作数, 该比较操作符尚未实现. 默认情况下, 返回`True`或者`False`表示执行了一次成功的比较操作, 不过要是有需要的话, 返回任意值也是允许的, 在需要布尔值的上下文环境下, 该返回值会通过`bool()`转成对应的布尔值.

默认情况下, `__ne__()`会委托给`__eq__()`, 并反转它的返回值(除非它返回`NotImplemented`). 其他操作符之间没有隐含的关系, 比如说`x < y`和`x == y`并不能推出`x <= y`. 如果需要根据某个根操作来自动生成其他操作, 我们可以使用`functools.total_ordering()`.

在我们需要创建一些可哈希的对象并需要支持自定义的比较操作或需要作为字典的键值时, 可以查看关于`__hash__`的段落来获取一些关键的注解.

这些方法没有对换参数的方法版本(即右边参数支持左边参数不支持的操作). 然而, `__lt__()`和`__gt__()`是左右参数的对应版本, `__le__()`和`__ge__()`是左右参数的对应版本, `__eq__()`和`__ne__()`是各自参数的对应版本. 如果左右操作符是不同类型, 并且右操作符是左操作符的子类(直接或者间接). 那么对应的版本具有更高的优先级. 这里不考虑虚子类(virtual subclassing).

```python
object.__hash__(self)
```

内建函数`hash()`以及哈希集合(`set`, `frozenset`)的成员哈希操作和`dict.__hash__()`会调用此方法来生成一个整数. 这边只有唯一一个约束, 当两个对象的相等比较为`True`时, 他们的哈希值也要一样. 我们还建议当我们比较对象时可以考虑使用对象中成员(components)的hash值.

> 注意: `hash()`的返回值会剪切到`Py_ssize_t`的大小, 通常来说这个值在64位版本中为8个字节, 在32位版本中为4个字节. 如果一个对象的`__hash__()`必须能够在不同位的版本中使用, 切记要记得检查各个支持的版本中的长度, 一种简单的做法是`python -c "import sys; print(sys.hash_info.width)"`

如果我们没有为类定义`__eq__()`方法, 那么同样的, 我们也不应该定义`__hash__()`方法. 如果类定义了`__eq__()`但是没有定义`__hash__()`, 那么该类的实例不能作为可哈希集合中的项. 如果一个类定义的是可变对象, 并且也定义了`__eq__()`, 那么它不应该再定义`__hash__()`, 原因是可哈希集合中的元素值都是不可变的(如果他的哈希值改变了的话, 那它就落到不同的桶子(bucket)里了).

用户自定义的类默认就有`__eq__()`和`__hash__()`. 有了这两个方法后, 所有的实例之间的相等性比较都会返回`False`(除非与自身比较). 另外一点, `__hash__()`返回的值能够暗示出如果`x == y`, 则`x is y`, 并且`hash(x) == hash(y)`.

对于那些只定义了`__eq__()`(没有定义`__hash__()`)的类来说, 该类的`__hash__()`会被隐式地设为`None`, 这样当代码尝试获取该类实例的哈希值, 调用方会收获一个`TypeError`异常. 该类的实例在做`isinstance(obj, collections,Hashable)`检查时也会返回`False`.

如果类自定义了`__eq__()`方法, 同时还想沿用父类的`__hash__()`方法. 那么你必须通过设置`__hash__ = <ParentClass>.__hash__`来显式地告诉解释器.

如果类没有定义`__eq__()`方法但是想取消`__hash__()`方法, 那么在类定义中需要指定`__hash__ = None`. 如果类自己定义了`__hash__()`方法, 但是在实现中显式地抛出`TypeError`会导致`isinstance(obj, collections.Hashable)`错误地返回`True`.

> 注意, 字符串, 字节序列和日期类型的`__hash__()`的返回值是不可预测的随机值. 虽然在同一个Python进程中, 这些值会保持不变, 但是在Python的不同进程中, 这些类型的哈希值可能是不一样的.

> 这种设计是为了杜绝拒绝服务攻击(denial-of-service, 可以通过精心挑选对应的输入来是的字典的插入效率降至O(n^2)复杂度), 在[这里](http://www.ocert.org/advisories/ocert-2011-003.html)可以获取更多细节.

> 改变哈希值会影响字典, 集合和其他映射的迭代顺序. Python从来没有保证其中的迭代顺序, 而且通常来说在32位和64位版本中都会不一样.

> 也可以查看 `PYTHONHASHSEED`

> 在3.3中默认启用了的哈希的随机性.

```python
object.__bool__(self)
```

用来实现对象的真值测试以及内建函数`bool()`测试, 应该返回`True`或者`False`. 如果这个方法没有定义, 那么真值测试会检查`__len__()`的返回值. 如果该方法定义了, 那么该对象的真值等同于检查该方法的返回值是否是非零. 如果一个类既没有定义`__len__()`也没有定义`__bool__()`, 那么该类的实例始终表现为`True`.
