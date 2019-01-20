[Original Link](https://www.python.org/download/releases/2.3/mro)

### 概要

这篇文档描述了在Python2.3中方法解析顺序(Method Resolution Order)的方式, 它也有一个名字: C3, 虽然对于新手来说阅读此文档可能会有一点点困难, 但是相信我, 我会通过很多清晰易懂的例子来说明它是如何工作的。由于我至今还不知道有其他人在这个领域发表过相关的解释性文档, 因此这篇文档应该是能帮助到一些人的。

### 声明

我将此文档以Python 2.3 License的形式捐献给Python软件委员会, 像通常一样, 我需要告诉读者, 接下来的内容理论上来说应该是正确的, 但是我不会做任何保证, 由于使用此文档而造成的所有问题请读者自行解决。

### 感谢

感谢所有在python邮件列表中给我提供支持的各位同仁。Paul Foley指出了文档中一些描述模糊的地方, 同时也督促我加上了关于local precedence ordering方面的内容。David Goodger帮助我将文档转成rst格式, David Mertz帮助我进行编辑, Joan G. Stark帮助我做了一些好看的图表, 最后也十分感谢Guido van Rossum很热心的将这篇文档链在了python2.3的主页上。

### 开始

> Felix qui potuie rerum cognoscere causas -- Virgilius

事情要从Sameuele Pedroni在python开发者邮件列表中发起的一个主题说起。在那个主题中Sameuele指出在python2.2中的方法解析顺序(Method Resolution Order)不是单调的, 并且建议用C3方法解析顺序来取代原先的机制。Guido同意了他的论断, 因此C3出现在了python2.3中。事实上, C3本身与python并没有任何关联, 它是由一伙Dylan工作人员发明的, 还顺带发表了一片论文。为了向python爱好者解释C3, 我希望本文的讨论能够起到一些帮助。 

首先我需要说明一下, 我接下来将要描述的只针对在python2.2中提出的新式类类型(new style classes), 传统类类型(classic style classes)仍然使用它们原先的方式(从左到右, 深度优先)。这种隔离能够不影响原有的代码, 虽然可能会影响到python2.2中的新式类类型(注: C3出现在python2.3中), 但是实际上C3与原有的方式会产生差异的情况是十分少见的, 因此:

> Don't be scared!

更进一步讲, 除非你大量使用多继承并且你的代码中有一些十分复杂的继承关系, 否则你无须对C3了解太多, 你可以直接略过本文, 但是如果你真的非常想知道多继承是如何工作的, 那么此文将非常适合你。并且, C3也不会如你想像中那么复杂。

我们从一些基本定义开始:

1.  给定一个类C, 它出现在一个非常复杂的多继承结构当中, 要指出方法被重写的顺序不是那么简单的一件事情, 比如: 指出C的父类的顺序。
2.  类C的父类列表, 包括类C本身, 以由近到远的方式进行组织, 我们称之为类的优先级列表或者说叫类C的线性化表示(*linearization*)。
3.  方法解析顺序(Method Resolution Order: MRO)描述的是构建类C线性化表示时的一个规则集合, 在python的文化中, 我们说"The MRO of C"也就是指类C的线性化表示。
4.  举例来说, 在一个单继承结构中, 如果C是C1的子类, C1是C2的子类, 那么C的线性化表示就是`[C, C1, C2]`, 但是在多继承的情况下, 构建C的线性化表示就要更复杂一些, 因为还要兼顾*local precedence ordering*和*monotonicity*(单调性)
5.  关于local precedence ordering我们需要稍后再讲, 因为它要更复杂一些, 单调性则简单些, 它指的是如果在C的线性化表示中, C1出现在C2的前面, 那么在C的任何子类的线性化表示中, C1也要出现在C2的前面。如果单调性得不到保证的话, 我们进行一个简单的继承操作, 就有可能改变方法的解析顺序, 从而造成一些很奇怪的问题(subtle bugs), 在下面我会用例子说明这点。
6.  不是所有的类都能进行线性化表示。在一些非常复杂的继承结构中, 有可能不能得出一个类的线性化表示。

这里我给出一个不能得出线性化表示的例子, 考虑下面这种继承结构:

```python
O = object
class X(O): pass
class Y(O): pass
class A(X, Y): pass
class B(Y, X): pass
# 试图从A和B继承出C是不可以的
class C(A, B): pass
'''
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: Cannot create a consistent method resolution
order (MRO) for bases X, Y
'''
```

此情况下,  A的线性化表示中, X出现在Y前面(也就是说A的所有子类的线性化表示中, X也要出现在Y前面), 而在B的线性化表示中, Y出现在X前面(同理, B的所有子类的线性化表示中, Y也要出现在X的前面), 这样从A和B继承出C就导致了矛盾。

在python2.3中针对这种继承行为会抛出异常, 在python2.2中老式的方法解析顺序中不会出现问题, 根据从左到右深度优先的规则, 我们可以很容易算出顺序为: `CABXYO`

### C3方法解析顺序(The C3 Method Resolution Order)

为了方便接下来的讨论, 我们定义一些符号以及相关的运算:

```
C1C2...CN
```

表示C1到CN类的集合: `[C1 C2 ... CN]`.

```
head = C1
```

`head`表示第一个类

```
tail = C2...CN
```

`tail`表示除第一个类外的其他所有类(注: init和last是另外一对)

我也会用下面这种方式来表示两个列表的相加操作:

```
C + (C1C2...CB) = CC1C2...CN
```

现在我可以解释python2.3中MRO是如何工作的了。

我们假设在一个多继承结构中, 类C继承自B1,B2...,BN, 我们想要计算出类C的线性化表示`L[C]`,只要遵照下面的规则就可以:

> C的线性化表示是C加上其父类的线性化表示及其父类的合并

用符号可以表示为:

```
L[C(B1 .. BN)] = C + merge(L[B1] ... L[BN], B1 ... BN)
```

特别的, 如果C是object, 由于object不存在父类, 它的线性化表示为:

```
L[object] = object
```

通常来说, 我们可以按照下面的说明(prescription, 我记得好像是处方的意思)来进行`merge`操作:

> 首先取得`head`元素, 比如说L[B1][0], 它是一个类, 如果这个类不出现在后面任何一个列表的`tail`中, 我们就将其加到C的线性化表示中并且从待merge的列表中删除, 否则, 我们探查下一个列表的`head`元素, 按这种方式进行下去, 直到我们移除了所有的类或者我们没办法进行下一步, 当没法进行下一步时, 即表示我们没有办法构建此线性化表示, python2.3会拒绝创建此类并抛出一个异常

这份说明确保了`merge`操作能够保持顺序, 换句话说, 如果类的顺序不能保证, 那么merge操作将没法进行计算。

如果C只有一个父类, 那么merge的计算是非常浅显的:

```
L[C(B)] = C + merge(L[B], B) = C + L[B]
```

但在多继承情况下, 如果我不举些例子, 大概我也没法期望你能理解它。

### 例子

第一个示例:

```python
O = object
class F(O): pass
class E(O): pass
class D(O): pass
class C(D, F): pass
class B(D, E): pass
class A(B, C): pass
```

O, D, E和F的线性化表示都是非常明显的:

```
L[O] = O
L[D] = D O
L[E] = E O
L[F] = F O
```

C的线性化表示可以计算为:

```
L[B] = B + merge(L[D], L[E], DE)
     = B + merge(DO, EO, DE)
```

我们发现D没有出现在其他任何列表(EO, DE)的tail(O, E)当中, 于是我们将其加到线性列表中, 并从待merge的列表中删除:

```
= BD + merge(O, EO, E)
```

现在O不满足条件, 因为O出现在EO的tail(O)中了, 于是我们切换到下一个列表EO, 我们发现E也不出现在其他任何列表(O, E)的tail(都是空)当中, 于是我们将其加到线性列表当中, 并从待merge的列表当中删除:

```
= BDE + merge(O, O)
```

最后一步就很简单了, 同理O不出现在其他任何列表的tail当中, 加到结果中, 并从待merge列表中删除:

```
= BDEO + merge() = BDEO
```

于是我们得到了最终的结果.

现在我们也可以计算A的线性表示了:

```
L[A] = A + merge(BDEO, CDFO, BC)
     = A + B + merge(DEO, CDFO, C)
     = A + B + C + merge(DEO, DFO)
     = A + B + C + D + merge(EO, FO)
     = A + B + C + D + E + merge(O, FO)
     = A + B + C + D + E + F + merge(O, O)
     = A + B + C + D + E + F + O
```

在这个例子中, 我们得到的线性表示的顺序与继承层级十分匹配, 即越低层级(More Specialized)的类在列表中出现的越前, 然后这种情形并不常见。

下面这个继承结构我想留给读者作为练习:

```python
O = object
class F(O): pass
class E(O): pass
class D(O): pass
class C(D, F): pass
class B(E, D): pass
class A(B, C): pass
```

这个例子与之前那个例子的唯一区别就是`B(D, E)`换成了`B(E, D)`, 但就算是这么一点点的变化却完全改变了继承结构的顺序。

在python2.2中, 你可以直接获取一个类的MRO, 并且得到的结果与python2.3中的线性化表示是一致的。

```python
>>> A.mro()
(<class '__main__.A'>, <class '__main__.B'>, <class '__main__.E'>, <class '__main__.C'>, <class '__main__.D'>, <class '__main__.F'>, <type 'object'>)
# 注: 在python3.4中测试最后为<class 'object'>
```

最后, 我们来讨论一下在第一节中提出的例子, 计算O, X, Y, A和B的线性化表示是很明显的:

```
L[O] = O
L[X] = X O
L[Y] = Y O
L[A] = A X Y O
L[B] = B Y X O
```

然而计算类C(继承于A和B)的线性化表示却是难以做到的.

```
L[C] = C + merge(AXYO, BYXO, AB)
     = C + A + merge(XYO, BYXO, B)
     = C + A + B + merge(XYO, YXO)
```

现在我们要合并`XYO`和`YXO`是做不到的, 因为X出现在`YXO`的tail中, Y也出现在`XYO`的tail中, 所以C3算法无法继续, 于是python抛出异常并拒绝创建类C。

### 不好的方法解析顺序(Bad Method Resolution Order)

如果一个方法解析顺序(MRO)不能保持local precedence ordering和monotonocity, 我们就称之为不好的方法解析顺序(Bad Method Resolution Order), 接下来我会就python2.2中传统类类型(classic classes)和新式类类型(new style classes)中可能出现不好的方法解析顺序来举些例子。

我们从无法保持local precedence ordering开始说起, 考虑下面这个例子:

```python
F = type('Food', (), {'remember2buy': 'spam'})
E = type('Food', (F,), {'remember2buy': 'eggs'})
G = type('GoodFood', (F, E), {}) # Python2.3中这是个错误
# 注: type方法的签名为:
# type(name, bases, dict)
```

我们看到G类继承自F和E(F在E前面), 因此我们会期望`G.remember2buy`访问的是`F.remember2buy`, 然而在python2.2中, 我们可以看到:

```python
>>> G.remember2buy
'eggs'
```

这里就违反了local precedence ordering, 因为G的父类的顺序没有得到保持, 在python2.2中:

```
L[G] = G E F  # F出现在E的后面
```

有人可能会辩解说F出现在E的后面是因为F比E更泛化(less specialized, F是E的父类), 然后这种破坏local precedence ordering的做法是非常违反直觉且更容易导致错误的。在旧式类类型中能得到不同结果也说明了这一点:

```python
>>> class F: remember2buy = 'spam'
>>> class E(F): remember2buy = 'eggs'
>>> class G(F, E): pass
>>> G.remember2buy
'spam'
```

在这种情况下, MRO为GFEF, 因此local precedence ordering得到了保持。

作为了一个通用规则, 像上面这种继承结构是我们应该尽量避免的. 就拿上面这个例子来说, 我们很难说清楚是F覆盖了E中的方法还是E覆盖了F中的方法. Python2.3会通过抛出异常并拒绝创建G类来解决这种歧义, 我们也可以很轻松地发现, 在C3中我们是无法完成下面这个merge的:

```
merge(FO, EFO, FE)
```

要真正解决上面的歧义, 我们应该从E和F来继承出G, 这样我们才能确定出G的MRO为GEF.

Python2.3强制程序员写出好的继承结构, 或者至少可以说是不产生歧义的继承结构。

作为一个相关的标注, Python2.3还能智能地帮助我们发现一些错误, 比如声明的父类列表中有重复的类:

```python
>>> class A(object): pass
>>> class C(A, A): pass # error
Traceback (most recent call last):
  File "<stdin>", line 1, in ?
TypeError: duplicate base class A
```

但在python2.2中, 不管是新式类还是旧式类都不会有什么问题.

最后, 我想指出我们从这个例子中得到的两个经验:

1.  MRO不仅仅决定了方法(methods)的解析顺序, 也包括属性(attributes)的解析顺序.
2.  Python爱好者总是喜欢spam!(我想你已经知道这点了)

在讨论完local precedence ordering之后, 现在我们可以考虑单调性(monotonicity)方面的内容了, 我的目标是想说明python2.2中不管是新式类还是旧式类, 他们的MRO都不能保持单调性.

我们用一个简单的菱形继承结构就足以证明:

```python
class C: pass
class A(C): pass
class B(C): pass
class D(A, B): pass
```

我们只要分别计算一下B和D的MRO就会发现问题:

```
L[B, P21] = B C       # B在C前面: B的方法覆盖C的方法
L[D, P21] = D A C B C # B在C后面: C的方法覆盖B的方法
# 注: P21指的是python版本
```

python2.2和python2.3中都不会出现问题, 对于B的线性表示, 两个版本都能给出:

```
L[D] = D A B C
``` 

Guido在他的文章中指出传统的MRO方式在实际中并没有那么坏, 因为在旧式类中用户可以很轻松的避免菱形继承结构。但是在新式类中, 所有的类都继承自object, 因此菱形的继承结构是无法避免的, 我们上面讨论的这种不一致性也就会出现在任何一个多继承中。

Python2.2中的MRO方式使得打破单调性变得困难很多, 但却并非不可能, 下面这个最初由Samuele Pedroni提供的一个例子就展示了非单调的情况:

```python
>>> class A(object): pass
>>> class B(object): pass
>>> class C(object): pass
>>> class D(object): pass
>>> class E(object): pass
>>> class K1(A, B, C): pass
>>> class K2(D, B, E): pass
>>> class K3(D, A): pass
>>> class Z(K1, K2, K3): pass
```

下面是根据C3计算出来的各个类的MRO(读者应该试着去验证它):

```
L[A] = A O
L[B] = B O
L[C] = C O
L[D] = D O
L[E] = E O
L[K1] = K1 A B C O
L[K2] = K2 D B E O
L[K3] = K3 D A O
L[Z] = Z K1 K2 K3 D A B C E O
```

Python2.2中关于A, B, C, D, E, K1, K2和K3的线性化表示与上面是一样的, 但是关于Z却有所不同:

```
L[Z, P22] = Z K1 K3 A K2 D B C O
```

很明显, 这种线性化表示是错误的, 因为A出现在D的前面(K3的线性表示中, A是出现在D的后面的, 换句话说, K3类中从D继承的方法会覆盖A中的方法, 但是在Z中, Z继承自K3, 但是在该类从A继承的方法却覆盖了D中的方法, 这就造成了单调性的不一致), 更进一步, 我们也发现在python2.2版本下Z的线性化表示也没有保持local precedence ordering, Z的local precedence ordering本应是`[K1, K2, K3]`, 但在Z的线性化表示中, K3却出现在K2的前面. 这些问题就解释了我们为什么要废弃原先的MRO方式而转而使用C3.

### 最后

偷懒的读者可能直接跳到了最后, 偷懒的程序员可能也不想锻炼他们的大脑, 那些有些傲慢的程序员可能也会直接跳到这里, 为了奖励这三种"美德", 他们应该要有所回报: 这段python2.2的脚本可以让你不费力地把本文提到的例子基本都过一遍, 最后你可以修改最后一行来尝试他们。

```python
# <mro.py>
"""Samuele Pedroni的C3算法(我稍微改了改以增强其可读性)"""
class __meta_class__(type):
    "为了更好地输出"
    __repr__ = lambda cls: cls.__name__

class ex_2:
    "关于顺序问题的严重不一致" # 来自Guido
    class O: pass
    class X(O): pass
    class Y(O): pass
    class A(X, Y): pass
    class B(Y, X): pass
    try:
        class Z(A, B): pass # 在Python2.2中没有问题, 我们创建了类Z
    except TypeError:
        pass # Python2.3中我们无法创建类Z

class ex_5:
    "我的第一个例子"
    class O: pass
    class F(O): pass
    class E(O): pass
    class D(O): pass
    class C(D, F): pass
    class B(D, E): pass
    class A(B, C): pass

class ex_6:
    "我的第二个例子"
    class O: pass
    class F(O): pass
    class E(O): pass
    class D(O): pass
    class C(D, F): pass
    class B(E, D): pass
    class A(B, C): pass

class ex_9:
    "Python2.2中的MRO和C3产生不一致的情况" # 来自 Samuele
    class O: pass
    class A(O): pass
    class B(O): pass
    class C(O): pass
    class D(O): pass
    class E(O): pass
    class K1(A, C, C): pass
    class K2(D, B, E): pass
    class K3(D, A): pass
    class Z(K1, K2, K3): pass

def merge(seqs):
    print '\n\nCPL[%s]=%s' % (seqs[0][0],seqs),
    res = []; i=0
    while 1:
        nonemptyseqs=[seq for seq in seqs if seq]
        if not nonemptyseqs: return res
        i+=1; print '\n',i,'round: candidates...',
        for seq in nonemptyseqs: # 从序列头部找到待合并的类
            cand = seq[0]; print ' ',cand,
            nothead=[s for s in nonemptyseqs if cand in s[1:]]
            if nothead: cand=None # 该类不满足条件
            else: break
        if not cand: raise "不一致的层级结构"
        res.append(cand)
        for seq in nonemptyseqs: # remove cand
            if seq[0] == cand: del seq[0]

def mro(C):
    "根据C3计算C的类优先级顺序"
    return merge([[C]]+map(mro,C.__bases__)+[list(C.__bases__)])

def print_mro(C):
    print '\nMRO[%s]=%s' % (C,mro(C))
    print '\nP22 MRO[%s]=%s' % (C,C.mro())

print_mro(ex_9.Z)

#</mro.py>
```

文完。

> enjoy!
