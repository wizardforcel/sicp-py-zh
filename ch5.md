# 第五章 序列和协程

> 来源：[Chapter 5: Sequences and Coroutines](http://www-inst.eecs.berkeley.edu/~cs61a/sp12/book/streams.html)

> 译者：[飞龙](https://github.com/wizardforcel)

> 协议：[CC BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/)

## 5.1 引言

在这一章中，我们通过开发新的工具来处理有序数据，继续讨论真实世界中的应用。在四儿张中，我们介绍了序列接口，在 Python 内置的数据类型例如`tuple`和`list`中实现。序列支持两个操作：获取长度和由下标访问元素。第三章中，我们开发了序列接口的用户定义实现，用户表示递归列表的`Rlist`类。序列类型具有高效的代表性，并且可以让我们高效访问大量有序数据集。

但是，使用序列抽象表示有序数据有两个重要限制。第一个是序列的长度`n`要占据比例为`n`的内存总数。于是，序列越长，表示它所占的内存空间就越大。

第二个限制是，序列只能表示已知且长度有限的数据集。许多我们想要表示的有序集合并没有定义好的长度，甚至有些是无限的。两个无线序列的数学示例是正整数和斐波那契数。无限长度的有序数据集也出现在其它计算领域，例如，所有推特状态的序列每秒都在增长，所以并没有固定的长度。与之类似，经过基站发送出的电脑呼叫序列，由计算机用户发出的鼠标动作序列，以及飞机上的传感器产生的加速度测量值序列，都在世界演化过程中无限扩展。

在这一章中，我们介绍了新的构造方式用于处理有序数据，它为容纳未知或无限长度的集合而设计，但仅仅使用有限的内存。我们也会讨论这些工具如何用于一种叫做协程的程序结构，来创建高效、模块化的数据处理流水线。

## 5.2 隐式序列

序列可以使用不将每个元素显式储存在内存中的程序结构来表示，这是让我们高效处理有序数据的核心概念。为了将这个概念用于时间，我们需要构造提供序列中所有元素访问的对象，但是不要实现把所有元素计算出来并储存。

这个概念的一个简单示例就是第二章出现的`range`序列类型。`range`表示连续有界的整数序列。但是，它的每个元素并不显式在内存中表示，当元素从`range`中获取时，才被计算出来。所以，我们可以表示非常大的整数范围。只有范围的结束位置才被储存为`range`对象的一部分，元素都被凭空计算出来。

```py
>>> r = range(10000, 1000000000)
>>> r[45006230]
45016230
```

这个例子中，当单位示例构造时，并不是这个范围内的所有 999,990,000 个整数都被储存。反之，范围对象将第一个元素 10,000 加上下标 45,006,230 来产生第 45,016,230 个元素。计算所求的元素值并不是从现有的表示中获取它们，这是惰性计算的一个例子。计算机科学的规则是，将惰性作为一种重要的计算工具来赞扬。

迭代器是提供底层有序数据集的有序访问的对象。迭代器在许多编程原因中都是内建对象，包括 Python。迭代器抽象拥有两个组成部分：一种获取底层元素序列的下一个元素的机制，以及一种标识元素序列已经到达末尾，没有更多剩余元素的机制。在带有内建对象系统的编程语言中，这个抽象通常相当于可以由类实现的特定接口。Python 的迭代器接口会在下一节中描述。

迭代器的使用性来源于底层数据序列并不能显式在内存中表达的事实。迭代器提供了一种机制，可以依次计算序列中的每个值，但是所有元素不需要连续储存。反之，当下个元素从迭代器获取的时候，这个元素会按照请求计算，而不是从现有的内存来源中获取。

范围可以惰性计算序列中的元素，因为序列的表示是统一的，并且任何元素都可以轻易从范围的起始和结束位置计算出来。迭代器支持更广泛的底层有序数据集的惰性生成，因为它们不需要提供底层序列任意元素的访问。反之，它们仅仅需要计算出序列的下一个元素，按照顺序，在每次其它元素被请求的时候。虽然不像序列可访问任意元素那样灵活（叫做随机访问），有序数据序列的顺序访问通过对于数据处理应用来说已经足够了。

### 5.2.1 Python 迭代器

Python 迭代器接口包含两个消息。`__next__`消息向迭代器获取所表示的底层序列的下一个元素。为了对`__next__`方法调用做出回应，迭代器可以执行任何计算来获取或计算底层数据序列的下一个元素。`__next__`的调用对让迭代器产生变化：它们向前移动迭代器的位置。所以多次调用`__next__`会有序返回底层序列的元素。在`__next__`的调用过程中，Python 通过`StopIteration`异常，来表示底层数据序列已经到达末尾。

下面的`Letters`类迭代了从`a`到`d`字母的底层序列。成员变量`current`储存了当前序列中的字母。`__next__`方法返回这个字母，并且使用它来计算`current`的新值。

```py
>>> class Letters(object):
        def __init__(self):
            self.current = 'a'
        def __next__(self):
            if self.current > 'd':
                raise StopIteration
            result = self.current
            self.current = chr(ord(result)+1)
            return result
        def __iter__(self):
            return self
```

`__iter__`消息是 Python 迭代器所需的第二个消息。它只是简单返回迭代器，它对于提供迭代器和序列的通用接口很有用，在下一节会描述。

使用这个类，我们就可以访问序列中的字母：

```py
>>> letters = Letters()
>>> letters.__next__()
'a'
>>> letters.__next__()
'b'
>>> letters.__next__()
'c'
>>> letters.__next__()
'd'
>>> letters.__next__()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 12, in next
StopIteration
```

`Letters`示例只能迭代一次。一旦`__next__()`方法产生了`StopIteration`异常，它就从此之后一直这样了。除非创建新的实例，否则没有办法来重置它。

迭代器也允许我们表示无限序列，通过实现永远不会产生`StopIteration`异常的`__next__`方法。例如，下面展示的`Positives`类迭代了正整数的无限序列：

```py
>>> class Positives(object):
        def __init__(self):
            self.current = 0;
        def __next__(self):
            result = self.current
            self.current += 1
            return result
        def __iter__(self):
            return self
```

## 5.2.2 `for`语句

Python 中，序列可以通过实现`__iter__`消息用于迭代。如果一个对象表示有序数据，它可以在`for`语句中用作可迭代对象，通过回应`__iter__`消息来返回可迭代对象。这个迭代器应拥有`__next__()`方法，依次返回序列中的每个元素，最后到达序列末尾时产生`StopIteration`异常。

```py
>>> counts = [1, 2, 3]
>>> for item in counts:
        print(item)
1
2
3
```

在上面的实例中，`counts`列表返回了迭代器，作为`__iter__()`方法调用的回应。`for`语句之后反复调用迭代器的`__next__()`方法，并且每次都将返回值赋给`item`。这个过程一直持续，直到迭代器产生了`StopIteration`异常，这时`for`语句就终止了。

使用我们关于迭代器的知识，我们可以拿`while`、赋值和`try`语句实现`for`语句的规则：

```py
>>> i = counts.__iter__()
>>> try:
        while True:
            item = i.__next__()
            print(item)
    except StopIteration:
        pass
1
2
3
```

在上面，由调用`counts`的`__iter__`方法返回的迭代器绑定到了名称`i`上面，便于一次获取每个元素。`StopIteration`异常的处理子句不做任何事情，但是这个异常的处理提供了退出`while`循环的控制机制。

### 5.2.3 生成器和`yield`语句

上面的`Letters`和`Positives`对象需要我们引入一种新的字段，`self.current`，来跟踪序列的处理过程。在上面所示的简单序列中，这可以轻易实现。但对于复杂序列，`__next__()`很难在计算中节省空间。生成器允许我们通过利用 Python 解释器的特性定义更复杂的迭代。

生成器是由一类特殊函数，叫做生成器函数返回的迭代器。生成器函数不同于普通的函数，因为它不在函数体中包含`return`语句，而是使用`yield`语句来返回序列中的元素。

生成器不使用任何对象属性来跟踪序列的处理过程。它们控制生成器函数的执行，每次`__next__`方法调用时，它们执行到下一个`yield`语句。`Letters`迭代可以使用生成器函数更简洁地实现。

```py
>>> def letters_generator():
        current = 'a'
        while current <= 'd':
            yield current
            current = chr(ord(current)+1)
>>> for letter in letters_generator():
        print(letter)
a
b
c
d
```

即使我们永不显式定义`__iter__()`或`__next__()`方法，Python 会理解当我们使用`yield`语句时，我们打算定义生成器函数。调用时，生成器函数并不返回特定的产出值，而是返回一个生成器（一种迭代器），它自己就可以返回产出的值。生成器对象拥有`__iter__`和`__next__`方法，每个对`__next__`的调用都会从上次的地方继续执行生成器函数，直到另一个`yield`语句执行的地方。

`__next__`第一次调用时，程序从`letters_generator`的函数体一直执行到进入`yield`语句。之后，它暂停并返回`current`值。`yield`语句并不破坏新创建的环境，并为之后的使用保留了它。当`__next__`再次调用时，执行在它停下来的地方恢复。`letters_generator`作用域中`current`和任何所绑定名称的值都会在随后的`__next__`调用中保留。

我们可以通过手动调用`__next__()`来遍历生成器：

```py
>>> letters = letters_generator()
>>> type(letters)
<class 'generator'>
>>> letters.__next__()
'a'
>>> letters.__next__()
'b'
>>> letters.__next__()
'c'
>>> letters.__next__()
'd'
>>> letters.__next__()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```

在第一次`__next__()`调用之前，生成器并不会开始执行任何生成器函数体重的语句。

### 5.2.4 可迭代对象

Python 中，迭代只会遍历一次底层序列的元素。在遍历之后，`__next__()`调用时迭代器会产生`StopIteration`异常。许多应用共需要迭代多次元素。例如，我们需要对一个列表迭代多次来枚举所有的元素偶对：


```py
>>> def all_pairs(s):
        for item1 in s:
            for item2 in s:
                yield (item1, item2)
>>> list(all_pairs([1, 2, 3]))
[(1, 1), (1, 2), (1, 3), (2, 1), (2, 2), (2, 3), (3, 1), (3, 2), (3, 3)]
```

序列本身不是迭代器，但是它是可迭代对象。Python 的可迭代接口只包含一个消息，`__iter__`，返回一个迭代器。Python 中内建的序列类型在`__iter__`方法调用时，返回迭代器的新实例。如果一个可迭代对象返回了迭代器的新实例，在每次调用`__iter__`的时候，那么它就能被迭代多次。

新的可迭代类可以通过实现可迭代接口来定义。例如，下面的可迭代对象`LetterIterable`类在每次调用`__iter__`时返回新的迭代器来迭代字母。

```py
>>> class LetterIterable(object):
        def __iter__(self):
            current = 'a'
            while current <= 'd':
                yield current
                current = chr(ord(current)+1)
```

`__iter__`方法是个生产呢过器函数，它返回一个生成器对象，产出从`'a'`到`'d'`的字母。

`Letters`迭代器对象在单次迭代之后就被“用完”了，但是`LetterIterable`对象可被迭代多次。所以，`LetterIterable`示例可以用于`all_pairs`的参数。

```py
>>> letters = LetterIterable()
>>> all_pairs(letters).__next__()
('a', 'a')
```

### 5.2.5 流

## 5.3 协程

这篇文章有大部分专注于将复杂程序解构为小型、模块化组件的技巧。当一个带有复杂行为的函数逻辑划分为几个独立的、本身为函数的步骤时，这些函数叫做辅助函数或者子过程。子过程有主函数调用，主函数负责协调子函数的使用。

![](img/subroutine.png)

这一节中，我们使用写成，引入了一种不同方式来解构复杂的计算。它是一种特别用于有序数据的处理任务的方式。就像子过程那样。写成计算复杂计算的一小步。但是，在使用协程时，没有主函数来协调结果。反之，写成自己链接到一起来组成流水线。可能有一些协程消耗输入数据，并把它发送到其它协程。也可能有一些协程，每个都对发送给它的数据执行简单的处理步骤。最后可能有另外一些协程输出最终结果。

![](img/coroutine.png)

协程和子过程的差异是概念上的：子过程在主函数中位于夏季，但是协程都是平等的，它们协作组成流水线，不带有任何上级的函数来负责以特定顺序调用它们。

这一节中，我们会学到 Python 如何通过`yield`和`send()`语句来支持协程的构建。之后，我们会看到协程在流水线中的不同作用，以及协程如何支持多任务。

### 5.3.1 Python 协程

在之前一节中，我们介绍了生成器函数，它使用`yield`来返回一个值。Python 的生成器函数也可以使用`(yield)`语句来接受一个值。生成器对象上有两个额外的方法：`send()`和`close()`，创建了一个模型使对象可以消耗或产出值。定义了这些对象的生成器函数叫做协程。

协程可以通过`(yield)`语句来消耗值，向像下面这样：

```py
value = (yield)
```

使用这个语法，在对象`send`方法带参数调用之前，执行会停顿在这条语句上。

```py
coroutine.send(data)
```

之后，执行会恢复，`value`会被赋为`data`的值。为了发射计算终止的信号，我们需要使用`close()`方法来关闭协程。这会在协程内部产生`GeneratorExit`异常，它可以由`try/except`子句来捕获。

下面的例子展示了这些概念。它是一个协程，用于打印匹配所提供的模式串的字符串。

```py
>>> def match(pattern):
        print('Looking for ' + pattern)
        try:
            while True:
                s = (yield)
                if pattern in s:
                    print(s)
        except GeneratorExit:
            print("=== Done ===")
```

我们可以使用一个模式串来初始化它，之后调用`__next__()`来开始执行：

```py
>>> m = match("Jabberwock")
>>> m.__next__()
Looking for Jabberwock
```

对`__next__()`的调用会执行函数体，所以`"Looking for jabberwock"`会被打印。执行一直持续，直到进入`line = (yield)`语句。之后，执行会展厅，并且等待一个值发送给`m`。我们可以使用`send`来将值发送给它。

```py
>>> m.send("the Jabberwock with eyes of flame")
the Jabberwock with eyes of flame
>>> m.send("came whiffling through the tulgey wood")
>>> m.send("and burbled as it came")
>>> m.close()
=== Done ===
```

当我们以一个值调用`m.send`时，协程`m`内部的求值，会在`line = (yield)`语句处恢复，这里会把发送的值赋给`line`变量。`m`中的求值还在继续，如果匹配的话会打印出那一行，并继续执行循环，直到再次进入`line = (yield)`。之后，`m`中的求值会暂停，并在`m.send`调用后恢复。

我们可以将使用`send()`和`yield`的函数链到一起来完成复杂的行为。例如，下面的函数将名为`text`的字符串分割为单词，并把每个单词发送给另一个协程。

每个单词都发送给了绑定到`next_coroutine`的协程，使`next_coroutine`开始执行，随后这个函数暂停并等待。它在`next_coroutine`暂停之前会一直等待，随后这个函数恢复执行，发送下一个单词或执行完毕。

如果我们将上面定义的`match`和这个函数链到一起，我们就可以创建出一个程序，只打印出匹配特定单词的单词。

```py
>>> text = 'Commending spending is offending to people pending lending!'
>>> matcher = match('ending')
>>> matcher.__next__()
Looking for ending
>>> read(text, matcher)
Commending
spending
offending
pending
lending!
=== Done ===
```

`read`函数向协程`matcher`发送每个单词，协程打印出任何匹配`pattern`的输入。在`matcher`协程中，`s = (yield)`一行等待每个发送进来的单词，并且在执行到这一行之后将控制流交还给`read`。

![](img/read-match-coroutine.png)

## 5.3.2 生产、过滤和消耗

协程基于如何使用`yield`和`send()`而具有不同的作用：

![](img/produce-filter-consume.png)

+ **生产者**创建序列中的物品，并使用`send()`，而不是`(yield)`。
+ **过滤器**使用`(yield)`来消耗物品并将结果使用`send()`发送给下一个步骤。
+ **消费者**使用`(yield)`来消耗物品，但是从不发送。

上面的`read`函数是一个生产者的例子。它不使用`(yield)`，但是使用`send`来生产数据。函数`match`是个消费者的例子。它不使用`send`发送任何东西，但是使用`(yield)`来消耗数据。我们可以将`match`拆分为过滤器和消费者。过滤器是一个协程，只发送与它的模式相匹配的字符串。

```py
>>> def match_filter(pattern, next_coroutine):
        print('Looking for ' + pattern)
        try:
            while True:
                s = (yield)
                if pattern in s:
                    next_coroutine.send(s)
        except GeneratorExit:
            next_coroutine.close()
```

消费者是一个函数，只打印出发送给它的行：

```py
>>> def print_consumer():
        print('Preparing to print')
        try:
            while True:
                line = (yield)
                print(line)
        except GeneratorExit:
            print("=== Done ===")
```

当过滤器或消费者被构建时，它的`__next__`方法必须调用来开始执行：

```py
>>> printer = print_consumer()
>>> printer.__next__()
Preparing to print
>>> matcher = match_filter('pend', printer)
>>> matcher.__next__()
Looking for pend
>>> read(text, matcher)
spending
pending
=== Done ===
```

即使名称`filter`暗示移除元素，过滤器也可以转换元素。下面的函数是个转换元素的过滤器的示例。它消耗字符串并发送一个字典，包含了每个不同的字母在字符串中的出现次数。

```py
>>> def count_letters(next_coroutine):
        try:
            while True:
                s = (yield)
                counts = {letter:s.count(letter) for letter in set(s)}
                next_coroutine.send(counts)
        except GeneratorExit as e:
            next_coroutine.close()
```

我们可以使用它来计算文本中最常出现的字母，并使用一个消费者，将字典累计来找出最常出现的键。

```py
>>> def sum_dictionaries():
        total = {}
        try:
            while True:
                counts = (yield)
                for letter, count in counts.items():
                    total[letter] = count + total.get(letter, 0)
        except GeneratorExit:
            max_letter = max(total.items(), key=lambda t: t[1])[0]
            print("Most frequent letter: " + max_letter)
```

为了在文件上运行这个谁留下你，我们必须首先按行读取文件。之后，将结果发送给`count_letters`，最后发送给`sum_dictionaries`。我们可以服用`read`协程来读取文件中的行。

```py
>>> s = sum_dictionaries()
>>> s.__next__()
>>> c = count_letters(s)
>>> c.__next__()
>>> read(text, c)
Most frequent letter: n
```

### 5.3.3 多任务

生产者或过滤器并不受限于唯一的下一步。它可以拥有多个协程作为它的下游，并使用`send()`向它们发送数据。例如，下面是`read`的一个版本，向多个下一步发送字符串中的单词：

```py
>>> def read_to_many(text, coroutines):
        for word in text.split():
            for coroutine in coroutines:
                coroutine.send(word)
        for coroutine in coroutines:
            coroutine.close()
```

我们可以使用它来检测多个单词中的相同文本：

```py
>>> m = match("mend")
>>> m.__next__()
Looking for mend
>>> p = match("pe")
>>> p.__next__()
Looking for pe
>>> read_to_many(text, [m, p])
Commending
spending
people
pending
=== Done ===
=== Done ===
```

首先，`read_to_many`在`m`上调用了`send(word)`。这个协程正在等待循环中的`text = (yield)`，之后打印出所发现的匹配，并且等待下一个`send`。之后执行流返回到了`read_to_many`，它向`p`发送相同的行。所以，`text`中的单词会按照顺序打印出来。
