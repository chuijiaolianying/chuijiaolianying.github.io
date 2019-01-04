---
layout: post
title: 【译】可迭代对象 vs. 迭代器 vs. 生成器
---

[原文：https://nvie.com/posts/iterators-vs-generators/](https://nvie.com/posts/iterators-vs-generators/)

-------

有时我会混淆以下几个 Python 中相关的概念：

- 容器
- 可迭代对象
- 迭代器
- 生成器
- 生成器表达式
- 列表(/集合/字典)推导式

因此我写下这篇文章用作以后的参考。

![relationships]({{ site.baseurl }}/images/relationships.png)



##  >> 容器

容器是一种包含元素，并且支持检测元素是否存在于容器中(`in`, `not in`)的数据结构。容器和容器中的元素都寄存在内存中，在 Python 中，常见的容器有：

- `list`, `deque`, ...
- `set`, `frozensets`, ...
- `dict`, `defaultdict`, `OrderedDict`, `Counter`, ...
- `tuple`, `namedtuple`, ...
- `str`

容器很容易理解，你可以把它想象成生活中真实的容器，比如一个盒子、一个柜子、一幢房子、一艘船…

从技术的角度来说，只要我们可以检查一个对象是否包含特定的元素，就称之为容器。比如我们可以对列表、集合或者元组进行元素检查：

```python
>>> assert 1 in [1, 2, 3] # 列表
>>> assert 4 not in [1, 2, 3]
>>> assert 1 in {1, 2, 3} # 集合
>>> assert 4 not in {1, 2, 3}
>>> assert 1 in (1, 2, 3) # 元组
>>> assert 4 not in (1, 2, 3)
```

字典则是检查键值：

```python
>>> d = {1: 'foo', 2: 'bar', 3: 'qux'}
>>> assert 1 in d
>>> assert 4 not in d
>>> assert 'foo' not in d
```

对于字符串，你可以检查是否包含一个子字符串

```python
>>> s = 'foobar'
>>> assert 'b' in s
>>> assert 'x' not in s
>>> assert 'foo' in s # 字符串容器「包含」了它所有的子字符串
```

最后一个例子看起来有点奇怪，但它展示了容器接口是如何隐藏容器对象的具体实现。一个字符串显然不会在内存中保存它所有子字符串的副本，但你仍然可以这样使用。

##### Notes:

- 尽管大多数容器都提供了可以生产出他们所包含元素的方法，但这并不使他们称为容器，而是使他们称为迭代器。

- 并不是所有的容器都是可迭代的，比如 [Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter)。我们可以检查元素是否存在于这样的概率数据结构中，但我们没办法得到它们中的每一个元素。

-------

## >> 可迭代的(对象)

如上所说，大多数容器都是可迭代的。同时也有许多其他可迭代的对象，比如打开的文件对象、`socket` 对象等等。

相对于容器代表了有限元素的集合，可迭代对象也可以是一个无限的数据源。

一个可迭代对象是可以返回迭代器（返回特定元素的迭代器）的对象，不一定是数据结构。这听起来有点奇怪，但可迭代对象和迭代器有一个很重要的区别。让我们来看看这个例子：

```python
>>> x = [1, 2, 3]
>>> y = iter(x)
>>> z = iter(x)
>>> next(y)
1
>>> next(y)
2
>>> next(z)
1
>>> type(x)
<class 'list'>
>>> type(y)
<class 'list_iterator'>
```

在这个例子中，`x` 是可迭代的，同时 `y` 和 `z` 是两个独立的迭代器，用于从可迭代对象 `x` 中产出值，并且迭代器 `x` 和 `y` 分别保持自己的状态。在本例中，`x` 是一个数据结构（一个列表），但这并不是必须的

##### Note：

- 出于实用的原因，通常一个可迭代的类会同时实现 `__iter__()` 和 `__next__()`，并且在 `__iter__()` 返回本身 —— 这使得类既是可迭代的，也是自己的迭代器。但返回一个不同的对象作为迭代器也是完全可以的。

最后，当你这样写时：

```python
x = [1, 2, 3]
for elem in x:
    ...
```

下图展示了 `for` 循环如何迭代可迭代对象：

![iterable-vs-iterator]({{ site.baseurl }}/images/iterable-vs-iterator.png)

当你反编译  Python  代码，你会发现对 `GET_ITER` 的显式调用 —— 本质上等同于调用 `iter(x)`。`FOR_ITER` 所做的相当于重复的执行 `next()` 来获取每一个元素（因为针对解释器所做的优化，这一部分没有在字节码中显示出来）。

```python
>>> import dis
>>> x = [1, 2, 3]
>>> dis.dis('for _ in x: pass')
  1           0 SETUP_LOOP              14 (to 17)
              3 LOAD_NAME                0 (x)
              6 GET_ITER
        >>    7 FOR_ITER                 6 (to 16)
             10 STORE_NAME               1 (_)
             13 JUMP_ABSOLUTE            7
        >>   16 POP_BLOCK
        >>   17 LOAD_CONST               0 (None)
             20 RETURN_VALUE
```

-------

## >> 迭代器

那么迭代器到底是什么呢？迭代器是一个有状态的辅助对象，当你对它调用 `next()` 时会生产下一个值。因此，任何一个实现了 `__next__()` 方法的对象都可以称为迭代器，而它具体是如何产生值的细节无关紧要。

因此迭代器实际上是一个生产值的工厂。每次你向它请求下一个值的时候，它会知道如何计算得到这个值，因为它始终保持着一个内部状态。

迭代器的例子数不胜数，所有 `itertools` 里的函数都返回一个迭代器，有些甚至返回一个无限的序列：

```python
>>> from itertools import count
>>> counter = count(start=13)
>>> next(counter)
13
>>> next(counter)
14
```

有些可以从一个有限序列中生产一个无限序列

```python
>>> from itertools import cycle
>>> colors = cycle(['red', 'white', 'blue'])
>>> next(colors)
'red'
>>> next(colors)
'white'
>>> next(colors)
'blue'
>>> next(colors)
'red'
```

有些可以从一个有限序列中生产一个无限序列：

```python
>>> from itertools import islice
>>> colors = cycle(['red', 'white', 'blue'])
>>> limited = islice(colors, 0, 4)
>>> for x in limited:
...     print(x)
red
white
blue
red
```

为了能够更好的理解迭代器，让我们来创建一个生产斐波那契数的迭代器。

```python
>>> class fib:
...     def __init__(self):
... 		self.prev = 0
... 		self.curr = 1
...
... 	def __iter__(self):
... 	return self
...
... 	def __next__(self):
... 		value = self.curr
... 		self.curr += self.prev
... 		self.prev = value
... 	return value
...
>>> f = fib()
>>> list(islice(f, 0, 10))
[1, 1, 2, 3, 5, 8, 13, 21, 34, 55]
```

注意，这个类既是可迭代的（因为它实现了 `__iter__()` 方法），也是它自己的迭代器（因为它也实现了 `__next__()`方法）

这个迭代器的状态完全保存在它自己的类变量 `prev` 和 `curr` 中，用于对迭代器的后续调用。每次调用 `next()` 方法会做两件重要的事：

1. 为下一次 `next()` 调用修改状态
2. 生产出本次调用的结果

> 核心思想：惰性计算
> 从外部来看，迭代器就像一个懒惰的工厂，只有在请求一个值的时候才会进行生产，而生产完成后又会变得空闲

-------

## >> 生成器

我们终于到达了旅途的终点！生成器绝对是我最喜欢的 Python 特性，它是一种特殊的迭代器 —— 最优雅的一种。

生成器允许你像上述斐波那契数列那样去写迭代器，但用一种更优雅的语法可以避免去写一个必须包含 `__iter__()` 和 `__next__()` 方法的类。

让我们明确一下：

1. 生成器都是迭代器（反过来则不一定如此）
2. 生成器都是惰性求值

这里有一个用生成器表达的斐波那契数列：

```python
>>> def fib():
... 	prev, curr = 0, 1
... 	while True:
... 		yield curr
... 		prev, curr = curr, prev + curr
...
>>> f = fib()
>>> list(islice(f, 0, 10))
[1, 1, 2, 3, 5, 8, 13, 21, 34, 55]
```

这岂不是很优雅？注意这里有一个魔术般的关键字是这种美妙表达的关键：

```python
	yield
```

让我们分析一下这里具体发生了什么：首先注意，`fib` 被定义为一个普通的 Python 函数。但在这个函数中不包含 `return` 关键字，这个函数的返回值将会是一个生成器（一个迭代器，一个生成值的工厂，一个保持状态的类）

当 `f = fib()` 被调用，生成器会被实例化并返回给变量 `f`。此时没有发生任何计算：生成器处于空闲状态，`prev, curr = 0, 1` 这一行还没有被执行。

然后生成器被传递给 `islice()` 函数，它本身也是一个生成器。到目前为止仍然只是初始化。

然后生成器又被传递给 `list()`，这个函数会用所有传递给它的参数构建一个列表。因此，它会对 `islice()` 调用 `next()` ，然后`islice()` 又会对 `f` 调用 `next()`。

当第一次调用发生时，代码开始运行：`prev, curr = 0, 1` 被执行，然后进去 `while True` 循环，当进行到 `yield curr` 语句时，它会生产出当前的 `curr` 值，并再次空闲。

这个值被传递给 `islice()` 并生产出来（因为还没有传递够 10 次），`list()` 则会将这个值加入到要构建的列表中。

接着，`list()` 会向 `islice()` 请求下一个值，继而向 `f` 请求下一个值，这时工厂会被唤醒，从语句 `prev, curr = curr, prev + curr` 开始执行，然后重新回到 `while` 循环中，直到 `yield curr` 语句被执行，返回此时 `curr` 的值。

这样的过程不断重复直到 `list()` 请求第 11 个值时，`islice()` 会抛出 `StopIteration` 异常，表明已经到达终点，这时 `list()` 会返回包含 10 个元素（也就是斐波那契数列的前 10 个数）的列表。注意，生成器并没有接收第 11 次 `next()` 调用，事实上，它将永远不会被调用，然后被垃圾回收。

### >>> 生成器的种类

在 Python 中有两种生成器：生成器函数和生成器推导式。如上述例子所示，任何一个在函数体内包含了 `yield` 关键字的函数都可以成为生成器函数。

生成器推导式类似于列表推导式，它的语法非常优雅。

假设你正在使用如下语法构建列表：

```python
>>> numbers = [1, 2, 3, 4, 5, 6]
>>> [x * x for x in numbers]
[1, 4, 9, 16, 25, 36]
```

你可以用相同的语法构建集合推导：

```python
>>> {x * x for x in numbers}
{1, 4, 9, 16, 25, 36}
```

同样你可以用类似的语法构建生成器推导式（而不是元组推导）

```python
>>> lazy_squares = (x * x for x in numbers)
>>> lazy_squares
<generator object <genexpr> at 0x10d1f5510>
>>> next(lazy_squares)
1
>>> list(lazy_squares)
[4, 9, 16, 25, 36]
```

注意，因为我们对 `lazy_sqaures` 使用 `next()` 来读取第一个值，它的状态因此指向了第二个元素，所以当我们使用 `list()` 获取列表时，只能得到剩余元素的列表。这与上面是示例一样，是一个生成器（也是一个迭代器）。

-------

## >> 总结

生成器是一种非常强大的编程手段。它允许你使用更少的中间变量和数据结构来编写流式代码。同时，它也具有更高的内存和 CPU 使用效率。最后，它一般只需要几行代码而已。

开始使用生成器的提示：在你的代码中寻找类似的结构：

```python
def somethong():
	result = []
	for ... in ...:
		result.append(x)
	return result
```

用下边的代码替换它：

```python
def iter_someting():
	for ... in ...:
		yield x
# def something(): # only if you really need a list structure
# return list(iter_something())
```
