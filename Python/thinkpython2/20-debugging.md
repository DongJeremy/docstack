# 第二十章：调试

在调试时，你应该区别不同类别的错误，才能更快地追踪定位：

-  语法错误是 Python 将源代码翻译成字节代码的时候产生的，说明程序的结构有一些错误。例如：省略了 ``def`` 语句后面的冒号会产生看上去有点重复的错误信息 ``SyntaxError: invalid syntax`` 。

-  运行时错误是当程序在运行时出错，解释器所产生的错误。大多数运行时错误会包含诸如错误在哪里产生和正在执行哪个函数等信息。例如：一个无限递归最终会造成 ``maximum recursion depth exceeded`` （“超过递归最大深度”）的运行时错误。

-  语义错误是指一个程序并没有抛出错误信息，但是没有做正确的事情。例如：一个表达式可能因为没有按照你预期的顺序执行，因此产生了错误的结果。

调试的第一步是弄清楚你正在处理哪种错误。虽然下面的各节是按照错误类型来组织的，有些技巧实际上适用于多种情形。

## 语法错误

通常一旦找出是哪种语法错误，就容易修正。不幸的是，抛出的错误消息通常没什么帮助。最常见的错误消息是 ``SyntaxError: invalid syntax`` 和 ``SyntaxError: invalid token`` ，都没有提供很多信息。

另一方面，这些错误消息会告诉你程序的哪里出现了错误。实际上，它告诉你 Python 是在哪里发现的问题，但这并一定就是出错的地方。有时，错误出现在错误消息出现的位置之前，通常就在前一行。

如果你是一点一点地增量式地写的代码，你应该能够知道错误在哪里。一般就在你最后添加的那行代码里。

如果你是从书上复制的代码，那请仔细地从头和书中的代码对照。一个一个字母地比照。同时，记住也可能是书上就错了，所以如果你发现看上去像语法错误的地方，那可能就是了。

下面是避免大部分常见语法错误的一些方法：

1. 确保你没有使用 Python 的关键字作为变量名称。

2. 检查你在每个复合语句首行的末尾都加了冒号，包括for，while，if，和 def 语句。
   
3. 确保代码中的字符串都有匹配地引号。确保所有的引号都是“直引号”，而不是“花引号”。

4. 如果你有带三重引号的多行字符串，确保你正确地结束了字符串。一个没有结束的字符串会在   程序的末尾产生 ``invalid token`` 错误，或者它会把剩下的程序看作字符串的一部分，直到遇到下一个字符串。第二种情况下，可能根本不会产生错误！

5. 一个没有关闭的操作符（ ``(``， ``{`` 以及 ``[``）使得 Python 把下一行继续看作当前语句的一部分。通常下一行会马上提示错误消息。

6. 检查条件语句里面的 ``==`` 是不是写成了 ``=`` 。

7. 确保每行的缩进是符合要求。Python 能够处理空格和制表符，但是如果混用则会出错。避免该问题的最好方法是使用一个了解 Python 语法、能够产生一致缩进的纯文本编辑器。

8. 如果代码中包含有非ASCII字符串（包括字符串和注释），可能会出错，尽管 Python 3 一般能处理非ASCII字符串。从网页或其他源粘贴文本时，要特别注意。

如果上面的方法都不想，请接着看下一节...

### 我不断地改代码，但似乎一点用都没有。

如果解释器说有一个错误但是你怎么也看不出来，可能是因为你和解释器看的不是同一个代码。检查你的编码环境，确保你正在编辑的就是Python试图要运行的程序。

如果你不确定，试着在程序开始时制造一些明显、故意的语法错误。再运行一次。如果解释器没有提示新错误，说明你没有运行新修改的代码。

有可能是以下原因：

-  你编辑了文件，但是忘记了在运行之前保存。有一些编程环境会在运行前自动保存，有些则不会。
   
-  你更改了文件的名称，但是你仍然在运行旧名称的文件。

-  开发环境的配置不正确。

-  如果你在编写一个模块，使用了 ``import`` 语句，确保你没有使用标准 Python 模块的名称作为模块名。

-  如果你使用 ``import`` 来载入一个模块，记住你必须重启解释器或者使用 ``reload`` 才能重新载入一个修改了的文件。如果你导入一个模块两次，第二次是无效的。

如果你依然解决不了问题，不知道究竟是怎么回事，有一种办法是从一个类似“Hello, World!”这样的程序重头开始，确保你能运行一个已知的程序。然后逐渐地把原来程序的代码粘贴到新的程序中。

## 运行时错误

一旦你的程序语法正确，Python 就能够编译它，至少可以正常运行它。接下来，可能会出现哪些错误？

### 我的程序什么也没有做。

在文件由函数和类组成，但并没有实际调用函数执行时，这个问题是最常见的。你也可能是故意这么做的，因为你只打算导入该模块，用于提供类和函数。

如果你不是故意的，确保你调用了一个函数来开始执行，请确保执行流能够走到函数调用处（参见下面“执行流”一节）。

### 我的程序挂死了。

如果一个程序停止了，看起来什么都没有做，这就是“挂死”了。通常这意味着它陷入了无限循环或者是无限递归。

-  如果你怀疑问题出在某个循环，在该循环之前添加一个打印语句，输出“进入循环”，在循环之后添加一个打印“退出循环”的语句。

   运行程序。如果打印了第一条，但没有打印第二条，那就是进入了无线循环。跳到下面“无限循环”一节。

-  大多数情况下，无限递归会造成程序运行一会儿之后输出“RuntimeError: Maximum recursion depth exceeded”错误。如果发生了这个错误，跳到下面“无限递归”一节。
   
如果没有出现这个错误，但你怀疑某个递归方法或函数有问题，你仍可以使用“无线递归”一节中的技巧。
   
-  如果上面两种方法都没用，开始测试其他的循环和递归函数或方法是否存在问题。

-  如果这也没有用，那有可能你没有弄懂程序的执行流。跳到下面“执行流”一节。

#### 无限循环

如果你认为程序中有一个无限循环，并且知道是哪一个循环，在循环的最后添加一个打印语句，打印条件中各个变量的值，以及该条件的值。

例如：

```python
while x > 0 and y < 0 :
    # do something to x
    # do something to y

    print('x: ', x)
    print('y: ', y)
    print("condition: ", (x > 0 and y < 0))
```

现在，当你运行程序时，你可以看到每次循环都有3行输出。最后一次循环时，循环条件应该
是 ``False`` 。如果循环继续走下去，你能够看到 x 和 y 的值，这时你或许能弄清楚到为什么它们的值没有被正确地更新。

#### 无限递归

大多数情况，无限递归会造成程序运行一会儿之后输出“RuntimeError: Maximum recursion depth exceeded”错误。

如果你怀疑一个函数造成了无限递归，确保函数有一个基础情形。也就是存在某种条件能够让函数直接返回值，而不会再递归调用下去。如果没有，你需要重新思考算法，找到一个初始条件。

如果有了基础情形了但是程序还是没有到达它，在函数的开头加入一个打印语句来打印参数。现在当你运行程序时，每次递归调用你都能看到几行输出，你可以看到参数的值。如果参数没有趋于基础情形，你会大致明白其背后的原因。

#### 执行流

如果你不确定程序执行的过程，在每个函数的开始处添加打印语句，打印类似“进入函数foo”这样的信息，foo是你的函数名。

现在运行程序时，就会打印出每个函数调用的轨迹。

### 运行程序时产生了异常。

如果在运行时出现了问题，Python会打印出一些信息，包括异常的名称、产生异常的行号和一个回溯（traceback）。

回溯会指出正在运行的函数、调用它的上层函数以及上上层函数等等。换言之，它追踪进行到目前函数调用所调用过的函数，包括每次函数的调用所在的行号。

第一步是检查程序中发生错误的位置，看你能不能找出问题所在。下面是一些常见的运行时错误：

命名错误（NameError）：

    你正在使用当前环境中不存在的变量名。检查下名称是否拼写正确，或者名称前后是否一致。还要记住局部变量是局部的。你不能在定义它们的函数的外面引用它们。

类型错误（TypeError）：

    有几种可能的原因：
    
    -  值的使用方法不对。例如：使用除整数以外的东西作为字符串、列表或元组的索引下标。
    
    -  格式化字符串中的项与传入用于转换的项之间不匹配。如果项的数量不同或是调用了无效的转换，都会出现这个问题。
       
    -  传递给函数的参数数量不对。如果是方法，查看方法定义是不是以 ``self`` 作为第一个参数。然后检查方法调用；确保你在一个正确的类型的对象上调用方法，并且正确地提供了其它参数。

键错误（KeyError）：

   你尝试用字典没有的键来访问字典的元素。如果键是字符串，记住它是区分大小写的。

属性错误（AttributeError）：

    你尝试访问一个不存在的属性或方法。检查一下拼写！你可以使用内建函数 ``dir`` 来列出存在的属性。
    
    如果一个属性错误表明一个对象是 ``NoneType`` ，那意味着它就是 ``None`` 。因此问题不在于属性名，而在于对象本身。
    
    对象是 ``None`` 的一个可能原因，是你忘记从函数返回一个值；如果程序执行到函数的末尾没有碰到 ``return`` 语句，它就会返回 ``None`` 。另一个常见的原因是使用了列表方法的结果，如 ``sort`` ，这种方法返回的是 ``None`` 。

索引错误（IndexError）：

    用来访问列表、字符串或元组的索引要大于访问对象长度减一。在错误之处的前面加上一个打印语句，打印出索引的值和数组的长度。数组的大小是否正确？索引值是否正确？

Python 调试器
(pdb)有助于追踪异常，因为它可以让你检查程序出现错误之前的状态。你可以阅读 https://docs.python.org/3/library/pdb.html 了解更多关于 pdb 的细节。

我加入了太多的打印语句以至于输出刷屏。
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

使用打印语句来调试的一个问题，是你可能会被泛滥的输出所埋没。有两种途径来处理：简化输出或者是简化程序。

为了简化输出，你可以移除或注释掉不再需要的打印语句，或者合并它们，或者格式化输出便于理解。

为了简化程序，有几件事情可以做的。首先，缩减当前求解问题的规模。例如，如果你在检索一个列表，使用一个 **小** 列表来检索。如果程序从用户获得输入，给一个会造成问题的最简单的输入。

其次，清理程序。移除死代码，并且重新组织程序使其易于理解。例如，如果你怀疑问题来自程序深度嵌套的部分，尝试使用简单的结构重写它。如果你怀疑是一个大函数的问题，尝试分解它为小函数并分别测试。

通常，寻找最小化测试用例的过程能够引出bug。如果你发现一个程序在一种条件下运行正
确，在另外的条件下运行不正确，这能够给你提供一些解决问题的线索。

类似的，重写代码能够让你发现难找的bug。如果你做了一处改变，认为不会影响程序但是却事实证明相反，这也可以给你线索。

语义错误
---------------

在某些程度上，语义错误是最难调试的，因为解释器不能提供错误的信息。只有你知道程序本来应该是怎么样做的。

第一步是在程序代码和你看到的表现之间建立连接。你需要首先假设程序实际上干了什么事情。这种调试的难处之一，是电脑运行的太快了。

你会经常希望程序能够慢下来好让你能跟上它的速度，通过一些调试器(debugger)就能做到这点。但是有时候，插入一些安排好位置的打印语句所需的时间，要比你设置好调试器、插入和移除断点，然后“步进”程序到发生错误的地方要短。

我的程序不能工作。
~~~~~~~~~~~~~~~~~~~~~~~~

你应该问自己下面这些问题：

-  是不是有你希望程序完成的但是并没有出现的东西？找到执行这个功能的代码，确保它是按照你认为的方式工作的。

-  是不是有些本不该执行的代码却运行了？找到程序中执行这个功能的代码，然后看看它是不是本不应该执行却执行了。

-  是不是有一些代码的效果和你预期的不一样？确保你理解了那部分的代码，特别是当它涉及调用其它模块的函数或者方法。阅读你调用的函数的文档。尝试写一些简单的测试用例，来测试他们是不是得到了正确的结果。

在编程之前，你需要先建立程序是怎样工作的思维模型。如果你写出来的代码并非按照你预期的工作，问题经常不是在程序本身，而是你的思维模型。

纠正思维模型最好的方，是把程序切分成组件（就是通常的函数和方法），然后单独测试每个组件。
一旦你找到了模型和现实的不符之处，你就能解决问题了。

当然，你应该在写代码的过程中就编写和测试组件。如果你遇到了一个问题，那只能是刚写的一小段代码才有可能出问题。

我写了一个超大的密密麻麻的表达式，结果它运行得不正确。
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

写复杂的表达式是没有问题的，前提是可读，但是它们很难调试。通常把复杂的表达式打散成一系列临时变量的赋值语句，是一个好做法。

例如：

::

    self.hands[i].addCard(self.hands[self.findNeighbor(i)].popCard())

这可以重写成：

::

    neighbor = self.findNeighbor(i)
    pickedCard = self.hands[neighbor].popCard()
    self.hands[i].addCard(pickedCard)

显示的版本更容易读，因为变量名提供了额外的信息，也更容易调试，因为你可以检查中间变量的类型和值。

超长表达式的另外一个问题是，计算顺序可能和你想得不一样。例如如果你把\ :math:`\frac{x}{2 \pi}`\ 翻译成 Python 代码，你可能会写成：

::

    y = x / 2 * math.pi

这就不正确了，因为乘法和除法具有相同的优先级，所以它们从左到右进行计算。所以表达式计算的是\ :math:`x \pi / 2`\ 。

调试表达式的一个好办法，是添加括号来显式地指定计算顺序：

::

     y = x / (2 * math.pi)

只要你不太确定计算的顺序，就用括号。这样不仅能确保程序正确（按照你认为的方式工
作），而且对于那些记不住优先级的人来说更加易读。

有一个函数没有返回我期望的结果。
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果你的 ``return`` 语句是一个复杂的表达式，你没有机会在返回之前打印出计算的结果。
不过，你可以用一个临时变量。例如，与其这样写：

::

    return self.hands[i].removeMatches()

不如写成：

::

    count = self.hands[i].removeMatches()
    return count

现在，你就有机会在返回之前显示 ``count`` 的值了。

我真的是没办法了，我需要帮助。
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

首先，离开电脑几分钟吧。电脑发出的辐射会影响大脑，容易造成以下症状：

-  焦躁易怒

-  迷信(“电脑就是和我作对”)和幻想(“只有我反着带帽子程序才会正常工作”)。

-  随机漫步编程（试图编写所有可能的程序，选择做了正确的事情的那个程序）。

如果你发现你自己出现上述的症状，起身走动走动。当你冷静之后，再想想程序。它在做什么？它异常表现的一些可能的原因是什么？上次代码正确运行时什么时候，你接下来做了什么？

有时，找到一个bug就是需要花很长的时间。我经常都是在远离电脑、让我的思绪飞扬时才找到bug的。
一些寻找bug的绝佳地点是火车上、洗澡时、入睡之前在床上。


我不干了，我真的需要帮助。
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

这个经常发生。就算是最好的程序员也偶尔被难住。
有时你在一个程序上工作的时间太长了，以至于你看不到错误。那你该是休息一下双眼了。

当你拉某人来帮忙之前，确保你已经准备好了。你的程序应该尽量简单，你应该应对造成错误的最小输入。你应该在合适的地方添加打印语句（打印输出应该容易理解）。你应该对程序足够理解，能够简洁地对其进行描述。

当你拉某人来帮忙时，确保提供他们需要的信息：

-  如果有错误信息，它是什么以及它指出程序的错误在哪里？

-  在这个错误发生之前你最后做的事情是什么？你写的最后一行代码是什么，或者失败的新
   的测试样例是怎样的？

-  你至今都尝试了哪些方法，你了解到了什么？

你找到了bug之后，想想你要怎样才能更快的找到它。下次你看到相似的情况时，你就可以更快的找到bug了。

记住，最终目标不是让程序工作，而是学习如何让程序正确工作。