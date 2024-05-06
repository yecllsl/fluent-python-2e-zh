<link href="Styles/Style00.css" rel="stylesheet" type="text/css"> <link href="Styles/Style01.css" rel="stylesheet" type="text/css"> 

# 第十九章。Python 中的并发模型

> 并发是指同时处理大量的事情。
> 
> 并行就是同时做很多事情。
> 
> 不相同，但有联系。
> 
> 一个是关于结构，一个是关于执行。
> 
> 并发性提供了一种构建解决方案的方式，以解决可能(但不一定)可并行化的问题。
> 
> 罗布·派克，围棋语言 [1] 的共同发明人

这一章是关于如何让 Python“一次处理很多事情”这可能涉及并发或并行编程——即使是热衷于行话的学者也不同意如何使用这些术语。在本章的题词中，我将采用 Rob Pike 的非正式定义，但是请注意，我已经找到了一些声称是关于并行计算的论文和书籍，但是大部分都是关于并发性的。 [2]

在 Pike 看来，并行性是并发性的一个特例。所有并行系统都是并发的，但不是所有并发系统都是并行的。在 21 世纪初，我们使用单核机器在 GNU Linux 上同时处理 100 个进程。在正常、随意使用的情况下，一台具有 4 个 CPU 内核的现代笔记本电脑在任何给定时间都会例行运行 200 多个进程。要并行执行 200 个任务，你需要 200 个内核。因此，在实践中，大多数计算是并发的，而不是并行的。操作系统管理数百个进程，确保每个进程都有机会取得进展，即使 CPU 本身不能同时做四件以上的事情。

本章假定没有并发或并行编程的先验知识。在简单的概念介绍之后，我们将研究简单的例子来介绍和比较 Python 用于并发编程的核心包:`threading`、`multiprocessing`和`asyncio`。

本章的最后 30%是对第三方工具、库、应用服务器和分布式任务队列的高级概述，所有这些都可以增强 Python 应用程序的性能和可伸缩性。这些都是重要的主题，但是超出了专注于核心 Python 语言特性的书的范围。尽管如此，我觉得在第二版的 *Fluent Python* 中阐述这些主题是很重要的，因为 Python 对于并发和并行计算的适应性并不局限于标准库所提供的。这就是为什么 YouTube、DropBox、Instagram、Reddit 和其他公司在开始时能够实现网络规模，将 Python 作为他们的主要语言——尽管他们一直声称“Python 没有规模”。

# 本章的新内容

这个章节是*流畅 Python* 第二版新增的。在“一个并发的 Hello World”中的 spinner 例子之前在关于 *asyncio* 的章节中。这里对它们进行了改进，并首次展示了 Python 的三种并发方法:线程、进程和本机协程。

除了最初出现在`concurrent.futures`和 *asyncio* 章节中的几个段落之外，其余内容都是新的。

《多核世界中的 Python》与本书其余部分不同:没有代码示例。我们的目标是提及一些重要的工具，您可能想学习这些工具来实现高性能并发和并行，而这是 Python 标准库所无法实现的。

# 大局

有许多因素使并发编程变得困难，但我想谈谈最基本的因素:启动线程或进程很容易，但如何跟踪它们呢？ [3]

当您调用一个函数时，调用代码被阻塞，直到函数返回。所以你知道什么时候函数完成了，你可以很容易地得到它返回的值。如果函数引发异常，调用代码可以用`try/except`包围调用位置来捕捉错误。

当您启动一个线程或进程时，这些熟悉的选项是不可用的:您不自动知道它何时完成，并且获取结果或错误需要设置一些通信通道，比如消息队列。

此外，启动一个线程或进程并不便宜，所以您不希望只是为了执行一个计算并退出而启动其中的一个。通常，您希望通过将每个线程或进程变成一个“工人”来分摊启动成本，该工人进入一个循环并等待输入进行处理。这使得交流更加复杂，并引入了更多的问题。当你不再需要一个工人时，你如何让他辞职？如何让它退出，而不中途中断作业，留下半生不熟的数据和未释放的资源(如打开的文件)？同样，通常的答案涉及消息和队列。

协程的启动成本很低。如果使用`await`关键字启动一个协程，很容易得到它返回的值，可以安全地取消它，并且有一个清晰的站点来捕捉异常。但是协程通常是由异步框架启动的，这使得它们像线程或进程一样难以监控。

最后，Python 协程和线程不适合 CPU 密集型任务，我们将会看到。

这就是为什么并发编程需要学习新概念和编码模式。让我们首先确保我们在一些核心概念上是一致的。

# 一点术语

这里的是我将在本章剩余部分和接下来的两章中用到的一些术语:

Concurrency

处理多个未决任务的能力，一次或并行(如果可能)处理一个任务，以便每个任务最终成功或失败。如果单核 CPU 运行交错执行未决任务的操作系统调度程序，则单核 CPU 具有并发能力。也称为多任务处理。

Parallelism

同时执行多项计算的能力。这需要一个多核 CPU、多个 CPU、一个 [GPU](https://fpy.li/19-2) ，或者一个集群中的多台计算机。

Execution unit

通用术语指并发执行代码的对象，每个对象都有独立的状态和调用栈。Python 原生支持三种执行单元:*进程*、*线程*和*协程*。

Process

计算机程序运行时的一个实例，使用内存和一部分 CPU 时间。现代桌面操作系统通常同时管理数百个进程，每个进程都被隔离在自己的私有内存空间中。进程通过管道、套接字或内存映射文件进行通信，所有这些都只能携带原始字节。Python 对象必须被序列化(转换)成原始字节，才能从一个进程传递到另一个进程。这是很昂贵的，并且不是所有的 Python 对象都是可序列化的。一个流程可以产生子流程，每个子流程称为一个子流程。这些也是相互隔离的，也是与母体隔离的。进程允许*抢占式多任务*:操作系统调度程序*抢占*——即周期性地暂停——每个正在运行的进程，以允许其他进程运行。这意味着一个冻结的进程不能冻结整个系统——理论上。

Thread

单个进程中的一个执行单元。当一个进程启动时，它使用一个线程:主线程。通过调用操作系统 API，一个进程可以创建更多的线程并发运行。一个进程中的线程共享同一个内存空间，其中保存着活动的 Python 对象。这允许线程之间轻松共享数据，但是当多个线程同时更新同一个对象时，也会导致数据损坏。像进程一样，线程也可以在操作系统调度程序的监督下实现*抢占式多任务处理*。一个线程比做同样工作的进程消耗更少的资源。

Coroutine

一个函数，可以挂起自己，稍后再恢复。在 Python 中，*经典协程*由生成器函数构建，*原生协程*用`async def`定义。“经典协同程序”介绍了这个概念，第 21 章介绍了本机协同程序的使用。Python 协同程序通常在一个单独的线程中运行，受一个*事件循环*的监督，也在同一个线程中。诸如 *asyncio* 、 *Curio* 或 *Trio* 等异步编程框架为非阻塞的、基于协程的 I/O 提供了事件循环和支持库。协程支持*协作多任务处理*:每个协程必须用`yield`或`await`关键字显式地放弃控制，以便另一个协程可以并发进行(但不是并行的)。这意味着协程中的任何阻塞代码都会阻塞事件循环和所有其他协程的执行——与进程和线程支持的*抢占式多任务处理*形成对比。另一方面，每个协程消耗的资源比做同样工作的线程或进程少。

Queue

一个让我们放入和获取项目的数据结构，通常按照 FIFO 的顺序:先进先出。队列允许独立的执行单元交换应用程序数据和控制消息，例如错误代码和终止信号。队列的实现根据底层并发模型的不同而不同:Python 标准库中的`queue`包提供了队列类来支持线程，而`multiprocessing`和`asyncio`包实现了它们自己的队列类。`queue`和`asyncio`包还包括非 FIFO 的队列:`LifoQueue`和`PriorityQueue`。

Lock

一个对象，执行单元可以使用它来同步它们的动作并避免损坏数据。更新共享数据结构时，运行的代码应该持有一个关联的锁。这通知程序的其他部分在访问相同的数据结构之前等待，直到锁被释放。最简单的锁也称为互斥锁(互斥锁)。锁的实现依赖于底层的并发模型。

Contention

对有限资产的争议。当多个执行单元试图访问共享资源(如锁或存储)时，就会发生资源争用。还有 CPU 争用，当计算密集型进程或线程必须等待操作系统调度程序给它们一部分 CPU 时间时。

现在让我们使用一些术语来理解 Python 中的并发支持。

## 进程、线程和 Python 臭名昭著的 GIL

这里的是我们刚刚看到的概念如何应用于 Python 编程的 10 个要点:

1.  Python 解释器的每个实例都是一个进程。您可以使用*多重处理*或 *concurrent.futures* 库启动额外的 Python 进程。Python 的*子进程*库被设计成启动进程来运行外部程序，而不管使用什么语言来编写它们。

2.  Python 解释器使用单线程运行用户程序和内存垃圾收集器。您可以使用*线程*或 *concurrent.futures* 库来启动额外的 Python 线程。

3.  对对象引用计数和其他内部解释器状态的访问由一个锁控制，即全局解释器锁(GIL)。任何时候只有一个 Python 线程可以容纳 GIL。这意味着无论 CPU 内核的数量有多少，任何时候都只有一个线程可以执行 Python 代码。

4.  为了防止 Python 线程无限期地持有 GIL，Python 的字节码解释器默认每隔 5ms 暂停当前 Python 线程， [4] 释放 GIL。该线程然后可以尝试重新获取 GIL，但是如果有其他线程等待它，操作系统调度程序可能会选择其中一个线程继续。

5.  当我们编写 Python 代码时，我们无法控制 GIL。但是内置函数或用 C 语言编写的扩展——或任何在 Python/C API 级别接口的语言——可以在运行耗时的任务时释放 GIL。

6.  每一个进行系统调用 [5] 的 Python 标准库函数都会释放 GIL。这包括执行磁盘 I/O、网络 I/O 和`time.sleep()`的所有功能。NumPy/SciPy 库中的许多 CPU 密集型函数，以及来自`zlib`和`bz2`模块的压缩/解压缩函数，也会释放 GIL。 [6]

7.  在 Python/C API 级别集成的扩展也可以启动其他不受 GIL 影响的非 Python 线程。这样的无 GIL 线程一般不能改变 Python 对象，但是它们可以读写支持[缓冲协议](https://fpy.li/pep3118)的内存底层对象，比如`bytearray`、`array.array`和 *NumPy* 数组。

8.  GIL 对使用 Python 线程进行网络编程的影响相对较小，因为 I/O 函数会释放 GIL，并且与读取和写入内存相比，读取或写入网络总是意味着高延迟。因此，每个单独的线程无论如何都要花费大量时间等待，所以它们的执行可以交错进行，而不会对整体吞吐量产生重大影响。这就是为什么 David Beazley 说:“Python 线程擅长于无所事事。” [7]

9.  对 GIL 的争夺会降低计算密集型 Python 线程的速度。对于此类任务，顺序的单线程代码更简单、更快。

10.  要在多个内核上运行 CPU 密集型 Python 代码，必须使用多个 Python 进程。

下面是来自`threading`模块文档的一个很好的总结: [8]

> **CPython 实现细节**:在 CPython 中，由于全局解释器锁，一次只有一个线程可以执行 Python 代码(尽管某些面向性能的库可能会克服这一限制)。如果你想让你的应用更好的利用多核机器的计算资源，建议你使用`multiprocessing`或者`concurrent.futures.ProcessPoolExecutor`。但是，如果您想要同时运行多个 I/O 绑定的任务，线程仍然是一个合适的模型。

上一段从“CPython 实现细节”开始，因为 GIL 不是 Python 语言定义的一部分。Jython 和 IronPython 实现没有 GIL。不幸的是，两者都落后了——仍然在跟踪 Python 2.7。高性能的 PyPy 解释器也有 2.7 和 3.7 版本的 GIL——最新版本是 2021 年 6 月。

###### 注意

本节没有提到协程，因为默认情况下，它们之间共享同一个 Python 线程，并与异步框架提供的监督事件循环共享，因此 GIL 不会影响它们。在一个异步程序中可以使用多个线程，但是最佳实践是一个线程运行事件循环和所有协同程序，而其他线程执行特定的任务。这将在“将任务委派给执行者”中解释。

现在有足够的概念。让我们看一些代码。

# 一个并发的 Hello World

在关于线程和如何避免 GIL 的讨论中，Python 贡献者 Michele Simionato [发布了一个类似于并发“Hello World”的示例](https://fpy.li/19-10):展示 Python 如何“行走和嚼口香糖”的最简单程序

西米奥纳托的程序使用了`multiprocessing`，但是我对它进行了修改，引入了`threading`和`asyncio`。让我们从`threading`版本开始，如果你学习过 Java 或 c 语言中的线程，这个版本可能看起来很熟悉。

## 带螺纹的纺纱机

接下来几个例子的 思路很简单:启动一个函数，在终端中动画角色时阻塞 3 秒，让用户知道程序在“思考”而不是停滞。

该脚本制作了一个动画旋转器，在同一个屏幕位置显示字符串`"\|/-"`中的每个字符。 [9] 慢速计算结束后，带有微调器的行被清除，结果显示:`Answer: 42`。

图 19-1 显示了旋转示例的两个版本的输出:首先是线程，然后是协程。如果你远离电脑，想象最后一行的`\`在旋转。

![Shell console showing output of two spinner examples.](Images/flpy_1901.png)

###### 图 19-1。脚本 spinner_thread.py 和 spinner_async.py 产生类似的输出:spinner 对象的 repr 和文本“Answer: 42”。截图中 spinner_async.py 还在运行，动画消息“/思考！”被显示；3 秒钟后，该行将被替换为“答案:42”。

我们先来回顾一下 *spinner_thread.py* 脚本。示例 19-1 列出了脚本中的前两个函数，示例 19-2 展示了其余的。

##### 示例 19-1：spinner _ thread . py:`spin`和`slow`函数

```
import itertools
import time
from threading import Thread,Event

defspin(msg:str,done:Event)->None:①
    for charin itertools.cycle(r'\|/-'):②
        status=f'\r{char} {msg}'③
        print(status,end='',flush=True)
        if done.wait(.1):④
            break⑤
    blanks=''*len(status)
    print(f'\r{blanks}\r',end='')⑥
def slow()->int:
    time.sleep(3)⑦
    return 42
```

① 这个函数将在一个单独的线程中运行。`done`参数是`threading.Event`的一个实例，一个简单的同步线程的对象。

② 这是一个无限循环，因为`itertools.cycle`一次产生一个字符，永远循环遍历字符串。

③ 文本模式动画的技巧:用回车 ASCII 控制字符(`'\r'`)将光标移回到行首。

④ 当事件被另一个线程设置时，`Event.wait(timeout=None)`方法返回`True`；如果`timeout`过去，它返回`False`。. 1s 超时将动画的“帧速率”设置为 10 FPS。如果希望微调器运行得更快，请使用较小的超时值。

⑤ 退出无限循环。

⑥ 通过用空格覆盖并将光标移回到开头来清除状态行。

⑦ `slow()`会被主线程调用。假设这是一个缓慢的网络 API 调用。调用`sleep`会阻塞主线程，但是 GIL 被释放，所以旋转线程可以继续。

###### 小费

这个例子的第一个重要观点是`time.sleep()`阻塞了调用线程，但是释放了 GIL，允许其他 Python 线程运行。

`spin`和`slow`功能将同时执行。主线程——程序启动时唯一的线程——会启动一个新线程运行`spin`，然后调用`slow`。按照设计，Python 中没有终止线程的 API。你必须给它发送一条关闭消息。

`threading.Event`类是 Python 协调线程的最简单的信令机制。一个`Event`实例有一个以`False`开始的内部布尔标志。调用`Event.set()`将标志设置为`True`。当标志为假时，如果一个线程调用`Event.wait()`，它将被阻塞，直到另一个线程调用`Event.set()`，此时`Event.wait()`返回`True`。如果给`Event.wait(s)`一个以秒为单位的超时，当超时过去时，这个调用返回`False`，或者一旦`Event.set()`被另一个线程调用，就返回`True`。

在示例 19-2 中列出的`supervisor`功能使用一个`Event`信号通知`spin`功能退出。

##### 示例 19-2：spinner _ thread . py:`supervisor`和`main`功能

```
def supervisor() -> int:  # <1>
    done = Event()  # <2>
    spinner = Thread(target=spin, args=('thinking!', done))  # <3>
    print(f'spinner object: {spinner}')  # <4>
    spinner.start()  # <5>
    result = slow()  # <6>
    done.set()  # <7>
    spinner.join()  # <8>
    return result

def main() -> None:
    result = supervisor()  # <9>
    print(f'Answer: {result}')

if __name__ == '__main__':
    main()
```

① `supervisor`将返回`slow`的结果。

② `threading.Event`实例是协调`main`线程和`spinner`线程活动的关键，下面会进一步解释。

③ 要创建一个新的`Thread`，提供一个函数作为`target`关键字参数，并提供位置参数给`target`作为通过`args`传递的元组。

④ 显示`spinner`对象。输出是`<Thread(Thread-1, initial)>`，其中`initial`是线程的状态——意味着它还没有开始。

⑤ 启动`spinner`螺纹。

⑥ 调用`slow`，阻塞`main`线程。同时，辅助线程正在运行微调器动画。

⑦ 将`Event`标志设置为`True`；这将终止`spin` 功能内的`for`循环。

⑧ 等到`spinner`线程结束。

⑨ 运行`supervisor`功能。我编写了单独的`main`和`supervisor`函数，使这个例子看起来更像例子 19-4 中的`asyncio`版本。

当`main`线程设置`done`事件时，`spinner`线程最终会注意到并干净地退出。

现在让我们看一个使用`multiprocessing`包的类似例子。

## 带流程的旋转器

`multiprocessing`包支持在单独的 Python 进程而不是线程中运行并发任务。当您创建一个`multiprocessing.Process`实例时，一个全新的 Python 解释器作为子进程在后台启动。由于每个 Python 进程都有自己的 GIL，这允许您的程序使用所有可用的 CPU 内核——但这最终取决于操作系统调度程序。我们将在“一个自制的进程池”中看到实际的效果，但是对于这个简单的程序来说，它并没有真正的不同。

本节的重点是介绍`multiprocessing`并展示其 API 模拟了`threading` API，使得简单程序从线程到进程的转换变得容易，如 *spinner_proc.py* ( 示例 19-3 )所示。

##### 示例 19-3： spinner_proc.py:只显示改变的部分；其他都和 spinner_thread.py 一样

```

import itertools
import time
from multiprocessing import Process, Event  # <1>
from multiprocessing import synchronize     # <2>

def spin(msg: str, done: synchronize.Event) -> None:  # <3>
# end::SPINNER_PROC_IMPORTS[]
    for char in itertools.cycle(r'\|/-'):
        status = f'\r{char} {msg}'
        print(status, end='', flush=True)
        if done.wait(.1):
            break
    blanks = ' ' * len(status)
    print(f'\r{blanks}\r', end='')

def slow() -> int:
    time.sleep(3)
    return 42

# tag::SPINNER_PROC_SUPER[]
def supervisor() -> int:
    done = Event()
    spinner = Process(target=spin,               # <4>
                      args=('thinking!', done))
    print(f'spinner object: {spinner}')          # <5>
    spinner.start()
    result = slow()
    done.set()
    spinner.join()
    return result
# end::SPINNER_PROC_SUPER[]

def main() -> None:
    result = supervisor()
    print(f'Answer: {result}')


if __name__ == '__main__':
    main()
```

① 基本的`multiprocessing` API 模仿了`threading` API，但是类型提示和 Mypy 暴露了这种差异:`multiprocessing.Event`是一个函数(不是像`threading.Event`那样的类),它返回一个`synchronize.Event`实例…

② …迫使我们进口`multiprocessing.synchronize` …

③ …来编写这个类型提示。

④ `Process`类的基本用法与`Thread`类似。

⑤ `spinner`对象显示为`<Process name='Process-1' parent=14868 initial>`，其中`14868`是运行 *spinner_proc.py* 的 Python 实例的进程 ID。

`threading`和`multiprocessing`的基本 API 是相似的，但是它们的实现非常不同，`multiprocessing`有一个更大的 API 来处理多进程编程增加的复杂性。例如，当从线程转换到进程时，一个挑战是如何在被操作系统隔离并且不能共享 Python 对象的进程之间进行通信。这意味着跨越进程边界的对象必须被序列化和反序列化，这会产生开销。在示例 19-3 中，唯一跨越进程边界的数据是`Event`状态，它是由位于`multiprocessing`模块下面的 C 代码中的低级操作系统信号量实现的。 [10]

###### 小费

从 Python 3.8 开始，标准库中有了一个 [`multiprocessing.shared_memory`](https://fpy.li/19-12) 包，但是它不支持用户自定义类的实例。除了原始字节，这个包还允许进程共享一个`ShareableList`，一个可变的序列类型，可以保存固定数量的`int`、`float`、`bool`和`None`类型的项，以及每个项高达 10 MB 的`str`和`bytes`。详见 [`ShareableList`](https://fpy.li/19-13) 文档。

现在让我们看看如何用协程而不是线程或进程来实现相同的行为。

## 带协程的旋转器

###### 注意

第 21 章是 完全致力于用协程进行异步编程。这只是一个高级介绍，将这种方法与线程和进程并发模型进行对比。因此，我们会忽略许多细节。

分配 CPU 时间来驱动线程和进程是操作系统调度程序的工作。相比之下，协程由应用级事件循环驱动，该事件循环管理未决协程的队列，逐个驱动它们，监视由协程发起的 I/O 操作触发的事件，并在每个事件发生时将控制传递回相应的协程。事件循环、库协同程序和用户协同程序都在一个线程中执行。因此，花在协程上的任何时间都会降低事件循环的速度——以及所有其他协程的速度。

如果我们从`main`函数开始，然后研究`supervisor`，那么 spinner 程序的协程版本更容易理解。这就是示例 19-4 所示。

##### 示例 19-4：spinner _ async . py:`main`函数和`supervisor`协程

```
def main() -> None:  # <1>
    result = asyncio.run(supervisor())  # <2>
    print(f'Answer: {result}')

async def supervisor() -> int:  # <3>
    spinner = asyncio.create_task(spin('thinking!'))  # <4>
    print(f'spinner object: {spinner}')  # <5>
    result = await slow()  # <6>
    spinner.cancel()  # <7>
    return result

if __name__ == '__main__':
    main()
```

① `main`是这个程序中唯一定义的正则函数——其他的是协程。

② `asyncio.run`函数启动事件循环来驱动协程，这将最终启动其他协程。在`supervisor`返回之前，`main`功能将保持锁定。`supervisor`的返回值将是`asyncio.run`的返回值。

③ 本机协程用`async def`定义。

④ `asyncio.create_task`调度`spin`的最终执行，立即返回`asyncio.Task`的一个实例。

⑤ `spinner`对象的`repr`看起来像`<Task pending name='Task-2' coro=<spin() running at /path/to/spinner_async.py:11>>`。

⑥ `await`关键字调用`slow`，阻塞`supervisor`，直到`slow`返回。`slow`的返回值将被赋给`result`。

⑦ `Task.cancel`方法在`spin`协程中引发了一个`CancelledError`异常，我们将在示例 19-5 中看到。

示例 19-4 展示了运行协程的三种主要方式:

`asyncio.run(coro())`

从一个常规函数调用来驱动一个协程对象，该对象通常是程序中所有异步代码的入口点，就像本例中的`supervisor`。该调用会一直阻塞，直到`coro`的主体返回。`run()`调用的返回值是`coro`主体返回的值。

`asyncio.create_task(coro())`

从一个协程调用以调度另一个协程最终执行。这个调用不会挂起当前的协程。它返回一个`Task`实例，这个对象包装了协程对象，并提供了控制和查询其状态的方法。

`await coro()`

从协程调用，将控制权转移给由`coro()`返回的协程对象。这将挂起当前的协同程序，直到`coro`的主体返回。await 表达式的值是`coro`主体返回的值。

###### 注意

记住:调用协程作为`coro()`会立即返回一个协程对象，但不会运行`coro`函数的主体。驱动协程的主体是事件循环的工作。

现在让我们研究示例 19-5 中的`spin`和`slow`协程。

##### 示例 19-5：spinner _ async . py:`spin`和`slow`协程

```
import asyncio
import itertools

async def spin(msg: str) -> None:  # <1>
    for char in itertools.cycle(r'\|/-'):
        status = f'\r{char} {msg}'
        print(status, flush=True, end='')
        try:
            await asyncio.sleep(.1)  # <2>
        except asyncio.CancelledError:  # <3>
            break
    blanks = ' ' * len(status)
    print(f'\r{blanks}\r', end='')

async def slow() -> int:
    await asyncio.sleep(3)  # <4>
    return 42
```

① 我们不需要用于表示`slow`已经完成其在 *spinner_thread.py* 中的工作的`Event`参数(示例 19-1 )。

② 使用`await asyncio.sleep(.1)`代替`time.sleep(.1)`，暂停而不阻塞其他协程。看这个例子后的实验。

③ 当在控制这个协程的`Task`上调用`cancel`方法时，`asyncio.CancelledError`被引发。是时候退出循环了。

④ `slow`协程也使用`await asyncio.sleep`代替`time.sleep`。

### 实验:打破旋转器获得洞察力

我推荐一个实验来了解 *spinner_async.py* 是如何工作的。导入`time`模块，然后转到`slow`协程，用对`time.sleep(3)`的调用替换行`await asyncio.sleep(3)`，如示例 19-6 所示。

##### 示例 19-6： spinner_async.py:用`time.sleep(3)`替换`await asyncio.sleep(3)`

```
async def slow() -> int:
    time.sleep(3)
    return 42
```

观看这种行为比阅读它更令人难忘。去吧，我等着。

当你运行实验时，你会看到:

1.  显示 spinner 对象，类似于这个:`<Task pending name='Task-2' coro=<spin() running at /path/to/spinner_async.py:12>>`。

2.  旋转器永远不会出现。程序挂起 3 秒钟。

3.  显示`Answer: 42`，程序结束。

为了理解发生了什么，回想一下使用`asyncio`的 Python 代码只有一个执行流，除非您已经显式地启动了额外的线程或进程。这意味着在任何时间点只有一个协程执行。并发是通过控制从一个协程传递到另一个协程来实现的。在示例 19-7 中，让我们关注在提议的实验中`supervisor`和`slow`协程中发生了什么。

##### 示例 19-7：spinner _ async _ experiment . py:`supervisor`和`slow`协程

```
asyncdefslow()->int:time.sleep(3)④return42asyncdefsupervisor()->int:spinner=asyncio.create_task(spin('thinking!'))①print(f'spinner object: {spinner}')②result=awaitslow()③spinner.cancel()⑤returnresult
```

① 创建`spinner`任务，最终驱动`spin`的执行。

② 显示屏显示`Task`处于“待定”状态

③ `await`表达式将控制权转移给`slow`协程。

④ `time.sleep(3)`阻滞 3 秒；程序中不会发生其他事情，因为主线程被阻塞了——而且它是唯一的线程。操作系统将继续其他活动。3 秒钟后，`sleep`解锁，`slow`返回。

⑤ 就在`slow`返回之后，`spinner`任务被取消。控制流从未到达`spin`协程的主体。

*spinner _ async _ experience . py*给出了一个重要的教训，如以下警告所述。

###### 警告

除非你想暂停你的整个程序，否则不要在`asyncio`中使用`time.sleep(…)`。如果一个验尸官需要花一些时间什么也不做，它应该`await asyncio.sleep(DELAY)`。这将控制权交还给`asyncio`事件循环，它可以驱动其他未决的协同事件。

## 并排的主管

*spinner_thread.py* 和 *spinner_async.py* 的行数几乎相同。`supervisor`功能是这些示例的核心。让我们详细比较一下。示例 19-8 仅列出了示例 19-2 中的`supervisor`。

##### 示例 19-8： spinner_thread.py:螺纹`supervisor`功能

```
def supervisor() -> int:
    done = Event()
    spinner = Thread(target=spin,
                     args=('thinking!', done))
    print('spinner object:', spinner)
    spinner.start()
    result = slow()
    done.set()
    spinner.join()
    return result
```

为了比较，示例 19-9 显示了来自示例 19-4 的`supervisor`协程。

##### 示例 19-9： spinner_async.py:异步`supervisor`协程

```
async def supervisor() -> int:
    spinner = asyncio.create_task(spin('thinking!'))
    print('spinner object:', spinner)
    result = await slow()
    spinner.cancel()
    return result
```

下面总结了两个`supervisor`实现之间需要注意的差异和相似之处:

*   一只`asyncio.Task`大致相当于一只`threading.Thread`。

*   一个`Task`驱动一个协程对象，一个`Thread`调用一个 callable。

*   协程使用关键字`await`显式地产生控制。

*   您不需要自己实例化`Task`对象，而是通过向`asyncio.create_task(…)`传递一个协程来获得它们。

*   当`asyncio.create_task(…)`返回一个`Task`对象时，它已经被调度运行，但是一个`Thread`实例必须通过调用它的`start`方法被明确告知运行。

*   在线程化的`supervisor`中，`slow`是一个普通函数，由主线程直接调用。在异步`supervisor`中，`slow`是由`await`驱动的协程。

*   没有从外部终止线程的 API 相反，你必须发送一个信号——就像设置`done` `Event`对象一样。对于任务，有一个`Task.cancel()`实例方法，它在协同程序体当前挂起的`await`表达式处引发`CancelledError`。

*   `supervisor`协程必须从`main` 函数中的`asyncio.run`开始。

这种比较有助于您理解如何使用 *asyncio* 协调并发作业，与您可能更熟悉的`Threading`模块相比。

关于线程与协程的最后一点:如果您使用线程进行过任何重要的编程，您就会知道对程序进行推理是多么具有挑战性，因为调度程序可以在任何时候中断线程。您必须记住使用锁来保护程序的关键部分，以避免在多步操作中被中断——这可能会使数据处于无效状态。

有了协程，默认情况下，您的代码就不会被中断。您必须显式地`await`让程序的其余部分运行。根据定义，协程是“同步的”,而不是持有锁来同步多个线程的操作:任何时候只有一个线程在运行。当您想放弃控制权时，您可以使用`await`将控制权交还给调度程序。这就是为什么可以安全地取消一个协程:根据定义，一个协程只有当它在一个`await`表达式中被挂起时才能被取消，所以你可以通过处理`CancelledError`异常来执行清理。

`time.sleep()`调用阻塞但不做任何事情。现在，我们将尝试一个 CPU 密集型调用，以更好地理解 GIL，以及异步代码中 CPU 密集型函数的影响。

# GIL 的真正影响

在 线程代码(示例 19-1 )中，可以将`slow`函数中的`time.sleep(3)`调用替换为自己喜欢的库的 HTTP 客户端请求，spinner 会一直旋转下去。这是因为一个设计良好的网络库会在等待网络时释放 GIL。

您还可以将`slow`协同程序中的`asyncio.sleep(3)`表达式替换为`await`，以获得来自设计良好的异步网络库的响应，因为这些库提供了协同程序，在等待网络时将控制权交还给事件循环。同时，旋转器将继续旋转。

对于 CPU 密集型代码，情况就不同了。考虑示例 19-10 中的函数`is_prime`，如果参数是质数，则返回`True`，否则返回`False`。

##### 示例 19-10：py:一个易读的素性检查，来自 Python 的 [`ProcessPool​Executor`例子](https://fpy.li/19-19)

```
def is_prime(n: int) -> bool:
    if n < 2:
        return False
    if n == 2:
        return True
    if n % 2 == 0:
        return False

    root = math.isqrt(n)
    for i in range(3, root + 1, 2):
        if n % i == 0:
            return False
    return True
```

在我现在使用的公司笔记本电脑上，通话时间大约为 3.3 秒。 [12]

## 快速测验

鉴于我们目前所看到的，请花时间考虑以下三个部分的问题。答案的一部分很棘手(至少对我来说是这样)。

> 假设`n = 5_000_111_000_222_021`——我的机器需要 3.3 秒来验证的质数，如果你做了以下更改，旋转器动画会发生什么变化:
> 
> 1.  在 *spinner_proc.py* 中，将`time.sleep(3)`替换为对`is_prime(n)`的调用？
>     
>     
> 2.  在 *spinner_thread.py* 中，将`time.sleep(3)`替换为对`is_prime(n)`的调用？
>     
>     
> 3.  在 *spinner_async.py* 中，用对`is_prime(n)`的调用替换`await asyncio.sleep(3)`？

在您运行代码或继续阅读之前，我建议您自己找出答案。然后，您可能想要复制和修改*微调器 _*。py* 示例为建议。

现在回答，从容易到难。

### 1.多重处理的答案

微调器由子进程控制，因此它在父进程计算素性测试时继续旋转。 [13]

### 2.线程的答案

旋转器由第二个线程控制，因此它在主线程计算主要性测试时继续旋转。

一开始我没有得到正确的答案:我预计旋转器会冻结，因为我高估了 GIL 的影响。

在这个特殊的例子中，微调器一直在旋转，因为 Python 每隔 5 毫秒(默认情况下)挂起一次正在运行的线程，使得 GIL 可供其他挂起的线程使用。因此，运行`is_prime`的主线程每隔 5ms 中断一次，允许次线程唤醒并迭代一次`for`循环，直到它调用`done`事件的`wait`方法，此时它将释放 GIL。然后主线程将获取 GIL，并且`is_prime`计算将继续进行 5 毫秒。

这对这个特定示例的运行时间没有明显的影响，因为`spin`函数快速迭代一次，并在等待`done`事件时释放 GIL，所以对 GIL 没有太多争用。运行`is_prime`的主线程大部分时间都会有 GIL。

在这个简单的实验中，我们使用线程完成了一项计算密集型任务，因为只有两个线程:一个占用 CPU，另一个每秒钟只唤醒 10 次来更新微调器。

但是如果你有两个或更多的线程争夺大量的 CPU 时间，你的程序会比顺序代码慢。

### 3.asyncio 的答案

如果调用 *spinner_async.py* 示例的`slow`协程中的`is_prime(5_000_111_000_222_021)`，spinner 将永远不会出现。当我们将`await asyncio.sleep(3)`替换为`time.sleep(3)`时，效果将与示例 19-6 中的相同:完全不旋转。控制流程将从`supervisor`转到`slow`，然后转到`is_prime`。当`is_prime`返回时，`slow`也返回，`supervisor`恢复，在`spinner`任务被执行之前取消它。程序显示冻结大约 3 秒钟，然后显示答案。T32

到目前为止，我们只试验了对 CPU 密集型函数的单次调用。下一节将介绍多个 CPU 密集型调用的并发执行。

# 自主开发的进程池

###### 注意

我 写这一节是为了展示 CPU 密集型任务的多进程使用，以及使用队列分发任务和收集结果的常见模式。第 20 章将展示一种更简单的向流程分配任务的方式:来自`concurrent.futures`包的`ProcessPoolExecutor`，它在内部使用队列。

在这一节中，我们将编写程序来计算 20 个整数样本的素性，从 2 到 9，999，999，999，999，999，999，即 10 个[16]–1，或多于 2 个 [53] 。样本包括小素数和大素数，以及具有小素数因子和大素数因子的合数。

*sequential.py* 程序提供了性能基线。下面是一个运行示例:

```
$ python3 sequential.py
               2  P  0.000001s
 142702110479723  P  0.568328s
 299593572317531  P  0.796773s
3333333333333301  P  2.648625s
3333333333333333     0.000007s
3333335652092209     2.672323s
4444444444444423  P  3.052667s
4444444444444444     0.000001s
4444444488888889     3.061083s
5555553133149889     3.451833s
5555555555555503  P  3.556867s
5555555555555555     0.000007s
6666666666666666     0.000001s
6666666666666719  P  3.781064s
6666667141414921     3.778166s
7777777536340681     4.120069s
7777777777777753  P  4.141530s
7777777777777777     0.000007s
9999999999999917  P  4.678164s
9999999999999999     0.000007s
Total time: 40.31
```

结果显示在三列中:

*   要检查的数字。

*   `P`如果是质数，不为空。

*   检查该特定数字的素性所用的时间。

在本例中，总时间大约是每次检查时间的总和，但它是单独计算的，如示例 19-12 所示。

##### 示例 19-12： sequential.py:小数据集的顺序素性检查

```
#!/usr/bin/env python3"""
sequential.py: baseline for comparing sequential, multiprocessing,
and threading code for CPU-intensive work.
"""fromtimeimportperf_counterfromtypingimportNamedTuplefromprimesimportis_prime,NUMBERSclassResult(NamedTuple):①prime:boolelapsed:floatdefcheck(n:int)->Result:②t0=perf_counter()prime=is_prime(n)returnResult(prime,perf_counter()-t0)defmain()->None:print(f'Checking {len(NUMBERS)} numbers sequentially:')t0=perf_counter()forninNUMBERS:③prime,elapsed=check(n)label='P'ifprimeelse''print(f'{n:16}  {label} {elapsed:9.6f}s')elapsed=perf_counter()-t0④print(f'Total time: {elapsed:.2f}s')if__name__=='__main__':main()
```

① `check`函数(在下一个标注中)返回一个`Result`元组，其中包含`is_prime`调用的布尔值和经过的时间。

② `check(n)`调用`is_prime(n)`并计算运行时间以返回一个`Result`。

③ 对于样本中的每个数字，我们调用`check`并显示结果。

④ 计算并显示总运行时间。

## 基于流程的解决方案

下一个例子是 *procs.py* ，展示了如何使用多个进程在多个 CPU 内核之间分配素性检查。这些是我用 *procs.py* 得到的时间:

```
$ python3 procs.py
Checking 20 numbers with 12 processes:
               2  P  0.000002s
3333333333333333     0.000021s
4444444444444444     0.000002s
5555555555555555     0.000018s
6666666666666666     0.000002s
 142702110479723  P  1.350982s
7777777777777777     0.000009s
 299593572317531  P  1.981411s
9999999999999999     0.000008s
3333333333333301  P  6.328173s
3333335652092209     6.419249s
4444444488888889     7.051267s
4444444444444423  P  7.122004s
5555553133149889     7.412735s
5555555555555503  P  7.603327s
6666666666666719  P  7.934670s
6666667141414921     8.017599s
7777777536340681     8.339623s
7777777777777753  P  8.388859s
9999999999999917  P  8.117313s
20 checks in 9.58s
```

输出的最后一行显示 *procs.py* 比 *sequential.py* 快 4.2 倍。

## 了解运行时间

注意第一栏中的经过时间是用于检查该特定数字。比如`is_prime(7777777777777753)`用了差不多 8.4s 才回`True`。与此同时，其他进程也在并行检查其他数字。

有 20 个数字需要核对。我编写了 *procs.py* 来启动与 CPU 内核数量相等的工作进程，这由`multiprocessing.cpu_count()`决定。

在这种情况下，总时间远小于各个检查所用时间的总和。在启动进程和进程间通信时会有一些开销，所以最终结果是多进程版本只比顺序版本快 4.2 倍。这很好，但考虑到代码启动 12 个进程来使用这台笔记本电脑上的所有内核，这有点令人失望。

###### 注意

在我用来写这一章的 MacBook Pro 上，`multiprocessing.cpu_count()`函数返回`12`。它实际上是一个 6 CPU 的 Core-i7，但由于超线程(一种每核执行 2 个线程的英特尔技术)，操作系统报告了 12 个 CPU。然而，当其中一个线程没有同一个内核中的另一个线程努力工作时，超线程会更好地工作——也许第一个线程在缓存未命中后停止等待数据，而另一个线程正在处理数据。无论如何，没有免费的午餐:这款笔记本电脑的性能就像一台 6 CPU 的机器，用于计算密集型工作，不使用大量内存——就像简单的素性测试一样。

## 多芯质数检验器的代码

当我们将计算委托给线程或进程时，我们的代码不会直接调用 worker 函数，所以我们不能简单地获得一个返回值。相反，工人是由线程或进程库驱动的，它最终产生一个需要存储在某个地方的结果。协调工作人员和收集结果是队列在并发编程中的常见用途，在分布式系统中也是如此。

*procs.py* 中的许多新代码都与设置和使用队列有关。文件的顶部在示例 19-13 中。

###### 警告

`SimpleQueue`在 Python 3.9 中被添加到`multiprocessing`中。如果你使用的是 Python 的早期版本，你可以用示例 19-13 中的`SimpleQueue`替换`Queue`。

##### 示例 19-13：多进程素性检查；导入、类型和函数

```
importsysfromtimeimportperf_counterfromtypingimportNamedTuplefrommultiprocessingimportProcess,SimpleQueue,cpu_count①frommultiprocessingimportqueues②fromprimesimportis_prime,NUMBERSclassPrimeResult(NamedTuple):③n:intprime:boolelapsed:floatJobQueue=queues.SimpleQueue[int]④ResultQueue=queues.SimpleQueue[PrimeResult]⑤defcheck(n:int)->PrimeResult:⑥t0=perf_counter()res=is_prime(n)returnPrimeResult(n,res,perf_counter()-t0)defworker(jobs:JobQueue,results:ResultQueue)->None:⑦whilen:=jobs.get():⑧results.put(check(n))⑨results.put(PrimeResult(0,False,0.0))⑩defstart_jobs(procs:int,jobs:JobQueue,results:ResultQueue⑪)->None:forninNUMBERS:jobs.put(n)⑫for_inrange(procs):proc=Process(target=worker,args=(jobs,results))⑬proc.start()⑭jobs.put(0)⑮
```

① 试图模仿`threading`，`multiprocessing`提供了`multiprocessing.SimpleQueue`，但这是一个绑定到较低级别的`BaseContext`类的预定义实例的方法。我们必须调用这个`SimpleQueue`来构建队列，我们不能在类型提示中使用它。

② `multiprocessing.queues`有我们需要类型提示的`SimpleQueue`类。

③ `PrimeResult`包括检查素性的数字。将`n`与其他结果字段放在一起可以简化以后的结果显示。

④ 这是一个`SimpleQueue`的类型别名，`main`函数(示例 19-14 )将使用它向完成工作的进程发送数字。

⑤ 键入第二个`SimpleQueue`的别名，该别名将在`main`中收集结果。队列中的值将是由要进行素性测试的数字组成的元组，以及一个`Result`元组。

⑥ 这个类似于 *sequential.py* 。

⑦ `worker`获取一个包含待检查数字的队列，另一个用于存放结果。

⑧ 在这段代码中，我使用数字`0`作为*毒丸*:让工人完成的信号。如果`n`不是`0`，继续循环。 [14]

⑨ 调用素性检查并入队`PrimeResult`。

⑩ 发送回一个`PrimeResult(0, False, 0.0)`让主循环知道这个 worker 已经完成了。

⑪ `procs`是并行计算主要检查的进程数。

⑫ 将待检查的数字排入`jobs`中。

⑬ 为每个工作线程派生一个子进程。每个子进程将在自己的`worker`函数实例中运行循环，直到它从`jobs`队列中取出一个`0`。

⑭ 启动每个子进程。

⑮ 为每个进程排队一个`0`，以终止它们。

现在我们来研究示例 19-14 中 *procs.py* 的`main`函数。

##### 示例 19-14：多进程素性检查；`main`功能

```
defmain()->None:iflen(sys.argv)<2:①procs=cpu_count()else:procs=int(sys.argv[1])print(f'Checking {len(NUMBERS)} numbers with {procs} processes:')t0=perf_counter()jobs:JobQueue=SimpleQueue()②results:ResultQueue=SimpleQueue()start_jobs(procs,jobs,results)③checked=report(procs,results)④elapsed=perf_counter()-t0print(f'{checked} checks in {elapsed:.2f}s')⑤defreport(procs:int,results:ResultQueue)->int:⑥checked=0procs_done=0whileprocs_done<procs:⑦n,prime,elapsed=results.get()⑧ifn==0:⑨procs_done+=1else:checked+=1⑩label='P'ifprimeelse''print(f'{n:16}  {label} {elapsed:9.6f}s')returncheckedif__name__=='__main__':main()
```

① 如果没有给出命令行参数，则将进程数设置为 CPU 核心数；否则，创建与第一个参数中给出的一样多的进程。

② `jobs`和`results`为示例 19-13 中描述的队列。

③ 启动`proc`流程消耗`jobs`和过账`results`。

④ 检索结果并显示它们；`report`在⑥中定义。

⑤ 显示检查了多少个数字和总运行时间。

⑥ 参数是`procs`的编号和发布结果的队列。

⑦ 循环，直到所有进程都完成。

⑧ 弄一个`PrimeResult`。在队列块上调用`.get()`,直到队列中有一个项目。也可以使它不阻塞，或者设置一个超时。详见 [`SimpleQueue.get`](https://fpy.li/19-23) 文档。

⑨ 如果`n`为零，则一个进程退出；增加`procs_done`计数。

⑩ 否则，增加`checked`计数(跟踪检查的数字)并显示结果。

结果不会按照提交作业的顺序返回。这就是为什么我必须把`n`放在每个`PrimeResult`元组中。否则，我就没有办法知道哪个结果属于每个数字。

如果主进程在所有子进程完成之前退出，您可能会看到由`multiprocessing`中的内部锁引起的`FileNotFoundError`异常的混乱回溯。调试并发代码总是很难，而调试`multiprocessing`更难，因为线程外观背后的复杂性。幸运的是，我们将在第 20 章中见到的`ProcessPoolExecutor`更容易使用，也更健壮。

###### 注意

感谢读者 Michael Albert，他注意到我在早期版本中发布的代码在示例 19-14 中有一个 [*竞争条件*](https://fpy.li/19-24) 。竞争条件是一种错误，它可能发生也可能不发生，这取决于并发执行单元执行操作的顺序。如果“A”发生在“B”之前，一切正常；但如果 B 先发生，就会出问题。这就是比赛。

如果你很好奇，这个 diff 显示了这个 bug 以及我是如何修复它的:[*example-code-2e/commit/2c 123057*](https://fpy.li/19-25)——但是请注意，我后来重构了这个示例，将部分`main`委托给了`start_jobs`和`report`函数。同一个目录下有一个 [*README.md*](https://fpy.li/19-26) 文件解释了问题和解决方案。

## 尝试更多或更少的流程

您可能想要尝试运行 *procs.py* ，传递参数来设置工作进程的数量。例如，这个命令…

```
$ python3 procs.py 2
```

…将启动两个工作进程，产生结果的速度几乎是 *sequential.py* 的两倍——如果你的机器至少有两个内核，并且不忙于运行其他程序。

我用 1 到 20 个进程运行了 *procs.py* 12 次，总共运行了 240 次。然后，我计算了相同数量流程的所有运行的平均时间，并绘制了图 19-2。

![Median run times for each number of processes from 1 to 20.](Images/flpy_1902.png)

###### 图 19-2。从 1 到 20 的每个流程数的平均运行时间。最高中值时间为 40.81 秒，有 1 个进程。最低的中值时间是 10.39 秒，有 6 个进程，用虚线表示。

在这款 6 核笔记本电脑中，6 个进程的平均时间最低:10.39 秒——由图 19-2 中的虚线标出。由于 CPU 争用，我预计运行时间在 6 个进程后会增加，在 10 个进程时达到了 12.51 秒的本地最大值。我没有预料到，也无法解释为什么在 11 个进程时性能提高了，而在 13 到 20 个进程时性能几乎保持不变，平均时间仅比 6 个进程时的最低平均时间稍高。

## 基于线程的非解决方案

我 还写了 *threads.py* ，一个用`threading`代替`multiprocessing`的 *procs.py* 版本。代码非常相似——在这两个 API 之间转换简单示例时通常都是这样。 ^(16](ch19.xhtml#idm46582390056752)) 由于`is_prime`的 GIL 和计算密集型特性，线程版本比示例 19-12 中的顺序代码要慢，而且随着线程数量的增加，由于 CPU 争用和上下文切换的成本，它会变得更慢。为了切换到新线程，操作系统需要保存 CPU 寄存器并更新程序计数器和堆栈指针，触发昂贵的副作用，如使 CPU 缓存无效，甚至可能交换内存页面。 ^([17)

接下来的两章将更多地介绍 Python 中的并发编程，使用高级的 *concurrent.futures* 库来管理线程和进程(第 20 章)和用于异步编程的 *asyncio* 库(第 21 章)。

本章的其余部分旨在回答这个问题:

> 鉴于目前讨论的局限性，Python 是如何在多核世界中蓬勃发展的？

# 多核世界中的 Python

考虑一下 Herb Sutter 的文章[“免费午餐结束了:软件并发性的根本转变”中的引文:](https://fpy.li/19-29)

> 从 Intel 和 AMD 到 Sparc 和 PowerPC，主要的处理器制造商和体系结构已经用尽了它们提高 CPU 性能的大多数传统方法。他们没有提高时钟速度和直线指令吞吐量，而是集体转向超线程和多核架构。2005 年 3 月。[网上有售]。

Sutter 称之为“免费午餐”的是软件变得更快的趋势，而不需要额外的开发人员努力，因为 CPU 年复一年地执行顺序代码更快。自 2004 年以来，这一点不再正确:时钟速度和执行优化达到了一个稳定的水平，现在性能的任何显著提高都必须来自于利用多核或超线程技术，这些技术的进步只会让为并发执行而编写的代码受益。

Python 的故事始于 20 世纪 90 年代初，当时 CPU 在顺序代码执行方面的速度仍呈指数级增长。当时除了超级计算机之外，没有人谈论多核 CPU。当时，拥有 GIL 的决定是显而易见的。GIL 使解释器在单核上运行时速度更快，实现也更简单。[18]GIL 也使得通过 Python/C API 编写简单的扩展变得更加容易。

###### 注意

我只是写了“简单的扩展”，因为扩展根本不需要处理 GIL。用 C 或 Fortran 编写的函数可能比用 Python 编写的函数快几百倍。 [19] 因此，释放 GIL 以利用多核 CPU 的额外复杂性在许多情况下可能并不需要。因此，我们应该感谢 GIL 为 Python 提供了许多扩展——这当然是这种语言今天如此流行的关键原因之一。

尽管有了 GIL，Python 在需要并发或并行执行的应用程序中仍然蓬勃发展，这要归功于克服了 CPython 局限性的库和软件架构。

现在让我们讨论一下在 2021 年的多核分布式计算世界中，Python 是如何用于系统管理、数据科学和服务器端应用程序开发的。

## 系统管理

Python 广泛用于管理大型服务器、路由器、负载平衡器和网络连接存储(NAS)群。这也是软件定义网络(SDN)和道德黑客的领先选择。主要的云服务提供商通过由提供商自己或其大型 Python 用户社区创作的库和教程来支持 Python。

在这个领域中，Python 脚本通过发出由远程机器执行的命令来自动执行配置任务，因此很少有 CPU 限制的操作要做。线程或协程非常适合这样的工作。特别是，我们将在第 20 章中看到的`concurrent.futures`包可以用来同时在许多远程机器上执行相同的操作，而没有太多的复杂性。

在标准库之外，还有流行的基于 Python 的项目来管理服务器集群:像 [*Ansible*](https://fpy.li/19-30) 和 [*Salt*](https://fpy.li/19-31) 这样的工具，以及像 [*Fabric*](https://fpy.li/19-32) 这样的库。

支持协程和`asyncio`的系统管理库也越来越多。2016 年，脸书的[生产工程团队](https://fpy.li/19-33)报告称:“我们越来越依赖于 Python 3.4 中引入的 AsyncIO，随着我们将代码库从 Python 2 中移出，我们看到了巨大的性能提升。”

## 数据科学

Python 很好地服务于数据科学——包括 人工智能——和科学计算。这些领域的应用是计算密集型的，但是 Python 用户受益于用 C、 C++ 、Fortran、Cython 等语言编写的数值计算库的巨大生态系统。—其中许多能够在异构群集中利用多核机器、GPU 和/或分布式并行计算。

截至 2021 年，Python 的数据科学生态系统包括令人印象深刻的工具，例如:

[Project Jupyter](https://fpy.li/19-34)

两个基于浏览器的界面——Jupyter Notebook 和 Jupyter lab——允许用户运行和记录可能在远程机器上跨网络运行的分析代码。两者都是混合 Python/JavaScript 应用程序，支持用不同语言编写的计算内核，都通过 ZeroMQ 集成，zero MQ 是一个用于分布式应用程序的异步消息库。名字 *Jupyter* 其实来源于 Julia、Python、R，笔记本支持的前三种语言。基于 Jupyter 工具构建的丰富生态系统包括 [Bokeh](https://fpy.li/19-35) ，这是一个强大的交互式可视化库，借助现代 JavaScript 引擎和浏览器的性能，用户可以导航大型数据集或持续更新的流数据并与之交互。

[TensorFlow](https://fpy.li/19-36) and [PyTorch](https://fpy.li/19-37)

根据[O ' Reilly 2021 年 1 月关于 2020 年期间学习资源使用情况的报告](https://fpy.li/19-38)，这是排名前两位的深度学习框架。这两个项目都是用 C++编写的，并且能够利用多核、GPU 和集群。他们也支持其他语言，但是 Python 是他们的主要焦点，并且被他们的大多数用户使用。TensorFlow 由 Google 创建并在内部使用；脸书的 PyTorch。

[Dask](https://fpy.li/dask)

一个并行计算库，可以将工作外包给本地进程或机器集群，“在世界上一些最大的超级计算机上测试过”，正如他们的[主页](https://fpy.li/dask)所述。Dask 提供了接近模拟 NumPy、pandas 和 scikit-learn 的 API，这些 API 是当今数据科学和机器学习领域最流行的库。Dask 可以在 JupyterLab 或 Jupyter Notebook 上使用，并利用 Bokeh 不仅实现数据可视化，还实现了交互式仪表板，以近乎实时的方式显示数据和计算在进程/机器间的流动。Dask 令人印象深刻，我推荐观看这样的视频 [15 分钟的演示](https://fpy.li/19-39)，其中项目维护者 Matthew Rocklin 展示了 Dask 在 AWS 上分布在 8 台 EC2 机器上的 64 个内核上处理数据。

这些只是一些例子，说明数据科学社区如何创建解决方案，利用 Python 的优势并克服 CPython 运行时的限制。

## 服务器端 Web/移动开发

Python 广泛应用于 web 应用和支持移动应用的后端 API。Google、YouTube、Dropbox、Instagram、Quora 和 Reddit 以及其他公司是如何成功构建 Python 服务器端应用程序，为数亿用户提供全天候服务的？同样，答案远远超出了 Python 所提供的“开箱即用”

在我们讨论大规模支持 Python 的工具之前，我必须引用 Thoughtworks *技术雷达*的一条警告:

> **高性能羡慕/网络规模羡慕**
> 
> 我们看到许多团队陷入困境，因为他们选择了复杂的工具、框架或架构，因为他们“可能需要扩展”像 Twitter 和网飞这样的公司需要支持极端负载，因此需要这些架构，但他们也有非常熟练的开发团队能够处理复杂性。大多数情况下不需要这些工程技术；团队应该控制他们的*网络规模嫉妒*,支持仍然能完成工作的更简单的解决方案。 [20]

在 *web scale* ，关键是一个允许水平伸缩的架构。在这一点上，所有的系统都是分布式系统，没有一种编程语言可能是解决方案每个部分的正确选择。

分布式系统是学术研究的一个领域，但幸运的是，一些从业者已经写了基于扎实的研究和实践经验的通俗易懂的书。《设计数据密集型应用程序》的作者 Martin Kleppmann 就是其中之一。

考虑图 19-3 ，这是 Kleppmann 书中许多架构图中的第一个。以下是我在 Python 项目中看到的一些组件，这些组件是我参与开发的，或者我有第一手的知识:

*   应用缓存:[21]*memcached*， *Redis* ， *Varnish*

*   关系数据库: *PostgreSQL* ， *MySQL*

*   文档数据库: *Apache CouchDB* ， *MongoDB*

*   全文索引: *Elasticsearch* ， *Apache Solr*

*   消息队列: *RabbitMQ* ， *Redis*

![Architecture for data system that combining several components](Images/flpy_1903.png)

###### 图 19-3。一个由几个组件组合而成的系统的可能架构。 [22]

在每个类别中都有其他工业级的开源产品。主要的云提供商也提供他们自己专有的替代方案。

Kleppmann 的图表是通用的，与语言无关——就像他的书一样。对于 Python 服务器端应用程序，通常会部署两个特定的组件:

*   一个应用服务器，用于在 Python 应用程序的几个实例之间分配负载。应用服务器将出现在图 19-3 的顶部附近，在客户端请求到达应用程序代码之前对其进行处理。

*   围绕图 19-3 右侧的消息队列构建的任务队列，提供了一个更高级、更易于使用的 API 来将任务分配给在其他机器上运行的进程。

接下来的两节将探讨这些组件，它们是 Python 服务器端部署中推荐的最佳实践。

## WSGI 应用服务器

WSGI—Web 服务器网关接口](https://fpy.li/pep3333)—是 Python 框架或应用程序接收来自 HTTP 服务器的请求并向其发送响应的标准 API。 ^([23) WSGI 应用服务器管理一个或多个运行应用程序的进程，最大限度地利用可用的 CPU。

图 19-4 说明了一个典型的 WSGI 部署。

###### 小费

如果我们想合并前面的一对图，图 19-4 中的虚线矩形的内容将替换图 19-3 顶部的实心“应用代码”矩形。

Python web 项目中最著名的应用服务器是:

*   [*mod_wsgi*](https://fpy.li/19-41)

*   *uWSGI*](https://fpy.li/19-42)^([24)

*   [*古尼康*](https://fpy.li/gunicorn)

*   [*NGINX 单元*](https://fpy.li/19-43)

对于 Apache HTTP 服务器的用户来说， *mod_wsgi* 是最好的选择。它和 WSGI 本身一样古老，但得到了积极的维护，现在提供了一个名为`mod_wsgi-express`的命令行启动器，使其更容易配置，也更适合在 Docker 容器中使用。

![Block diagram showing client connected to HTTP server, connected to application server, connected to four Python processes.](Images/flpy_1904.png)

###### 图 19-4。客户端连接到 HTTP 服务器，该服务器提供静态文件并将其他请求路由到应用服务器，应用服务器利用多个 CPU 内核派生子进程来运行应用程序代码。WSGI API 是应用服务器和 Python 应用代码之间的粘合剂。

uWSGI 和 *Gunicorn* 是我所知道的最近几个项目的首选。两者都经常与 NGINX HTTP 服务器一起使用。uWSGI 提供了许多额外的功能，包括应用程序缓存、任务队列、类似 cron 的周期性任务以及许多其他特性。另一方面， *uWSGI* 比 *Gunicorn* 更难正确配置。 [25]

*NGINX Unit* 发布于 2018 年，是来自知名 *NGINX* HTTP server 和反向代理的制造商的新产品。

*mod_wsgi* 和 *Gunicorn* 只支持 Python web 应用，而 *uWSGI* 和 *NGINX 单元*也支持其他语言。请浏览他们的文档以了解更多信息。

要点:所有这些应用服务器都有可能通过分叉多个 Python 进程来使用服务器上的所有 CPU 内核，以运行传统的 web 应用程序，这些应用程序是在 *Django* 、 *Flask* 、 *Pyramid* 等中用良好的旧顺序代码编写的。这解释了为什么不学习`threading`、`multiprocessing`或`asyncio`模块就可以作为 Python web 开发人员谋生:应用服务器透明地处理并发。

# ASGI—异步服务器网关接口

WSGI 是一个同步 API。它不支持带有`async/await`的协同程序——这是在 Python 中实现 WebSockets 或 HTTP 长轮询的最有效方式。 [ASGI 规范](https://fpy.li/19-46)是 WSGI 的继承者，为异步 Python web 框架设计，如 *aiohttp* 、 *Sanic* 、 *FastAPI* 等。，以及 *Django* 和 *Flask* ，这些都在逐渐加入异步功能。

现在让我们转向绕过 GIL 的另一种方式，以实现服务器端 Python 应用程序的更高性能。

## 分布式任务队列

当 应用服务器向运行您代码的 Python 进程之一提交请求时，您的应用程序需要快速响应:您希望该进程能够尽快处理下一个请求。但是，有些请求需要较长时间的操作，例如发送电子邮件或生成 PDF。这就是分布式任务队列旨在解决的问题。

[*芹菜*](https://fpy.li/19-47) 和 [*RQ*](https://fpy.li/19-48) 是最著名的使用 Python APIs 的开源任务队列。云提供商也提供他们自己专有的任务队列。

这些产品包装了一个消息队列，并提供了一个高级 API，用于将任务委托给工作人员，这些工作人员可能运行在不同的机器上。

###### 注意

在任务队列的上下文中，使用词语*生产者*和*消费者*来代替传统的客户机/服务器术语。例如，一个 *Django* 视图处理程序*产生*作业请求，这些请求被放入队列，由一个或多个 PDF 渲染进程使用*。*

直接引用*芹菜*的[常见问题](https://fpy.li/19-49)，以下是一些典型的使用案例:

> *   在后台运行一些东西。例如，尽快完成 web 请求，然后增量更新用户页面。这给用户留下了良好的性能和“敏捷”的印象，尽管实际工作可能需要一些时间。
>     
>     
> *   在 web 请求完成后运行一些东西。
>     
>     
> *   通过异步执行和使用重试来确保事情已经完成。
>     
>     
> *   安排周期性工作。

除了解决这些紧迫的问题，任务队列还支持水平可伸缩性。生产者和消费者是分离的:生产者不调用消费者，而是将请求放入队列中。消费者不需要知道生产者的任何信息(但是如果需要确认，请求可能包括生产者的信息)。至关重要的是，随着需求的增长，您可以轻松地添加更多的工人来完成任务。这就是为什么*芹菜*和 *RQ* 被称为分布式任务队列。

回想一下，我们简单的 *procs.py* ( 示例 19-13 )使用了两个队列:一个用于作业请求，另一个用于收集结果。 *Celery* 和 *RQ* 的分布式架构使用了类似的模式。两者都支持使用 [*Redis*](https://fpy.li/19-50) NoSQL 数据库作为消息队列和结果存储。 *Celery* 还支持其他消息队列，如 *RabbitMQ* 或*亚马逊 SQS* ，以及其他用于存储结果的数据库。

这就结束了我们对 Python 并发性的介绍。接下来的两章将继续这个主题，重点关注标准库的`concurrent.futures`和`asyncio`包。

# 章节摘要

在了解了一些理论之后，本章介绍了在 Python 的三种本地并发编程模型中实现的 spinner 脚本:

*   线头，带`threading`包

*   流程，用`multiprocessing`

*   带`asyncio`的异步协同程序

然后，我们通过一个实验探索了 GIL 的实际影响:更改旋转器示例来计算大整数的素性，并观察结果行为。这形象地证明了在`asyncio`中必须避免 CPU 密集型函数，因为它们会阻塞事件循环。尽管存在 GIL，但实验的线程版本仍然有效，因为 Python 会周期性地中断线程，并且该示例仅使用了两个线程:一个执行计算密集型工作，另一个每秒仅驱动动画 10 次。`multiprocessing`变种在 GIL 周围工作，为动画启动一个新的进程，而主进程进行素性检查。

下一个例子，计算几个素数，强调了`multiprocessing`和`threading`的区别，证明了只有进程才能让 Python 从多核 CPU 中获益。对于繁重的计算，Python 的 GIL 使得线程比顺序代码更糟糕。

GIL 主导了关于 Python 中并发和并行计算的讨论，但是我们不应该高估它的影响。这就是“多核世界中的 Python”的观点。例如，GIL 不会影响系统管理中的许多 Python 用例。另一方面，数据科学和服务器端开发社区已经围绕 GIL 开展工作，提供符合其特定需求的工业级解决方案。最后两节提到了大规模支持 Python 服务器端应用程序的两个常见元素:WSGI 应用服务器和分布式任务队列。

# 进一步阅读

这一章有一个广泛的阅读清单，所以我把它分成几个小节。

## 线程和进程的并发性

第 20 章中的*concurrent . futures*库使用了线程、进程、锁和队列，但是你不会看到它们的单个实例；它们由一个`ThreadPoolExecutor`和一个`ProcessPoolExecutor`的高层抽象捆绑和管理。如果你想了解更多关于那些底层对象的并发编程实践，Jim Anderson 的“Python 线程介绍”是一个不错的读物。Doug Hellmann 在他的[网站](https://fpy.li/19-52)和书[*The Python 3 Standard Library by Example*](https://fpy.li/19-53)(Addison-Wesley)上有一章题为“并发处理、线程和协同程序”。

布雷特·斯拉特金的 [*有效 Python*](https://fpy.li/effectpy) ，第 2 版。(Addison-Wesley)，David Beazley 的 *Python 基本参考*，第 4 版。(Addison-Wesley)，和 Martelli 等人，*Python in the null*，第 3 版。(O'Reilly)是其他一般性的 Python 参考资料，主要涉及`threading`和`multiprocessing`。大量的官方文档在其[“编程指南”部分](https://fpy.li/19-54)中包含了有用的建议。

Jesse Noller 和 Richard Oudkerk 贡献了`multiprocessing`包，在 [PEP 371 中介绍——将多处理包添加到标准库](https://fpy.li/pep371)。这个包的官方文档是一个 93 KB 的文件。rst 文件——大约 63 页——是 Python 标准库中最长的章节之一。

在 [*高性能 Python* ，第二版。作者 Micha Gorelick 和 Ian Ozsvald 在关于`multiprocessing`的一章中提供了一个关于使用不同于 *procs.py* 示例的策略检查素数的示例。对于每个数字，他们将可能因素的范围——从 2 到`sqrt(n)`——分成子范围，并让每个工人迭代其中一个子范围。他们的分而治之方法是典型的科学计算应用，其中数据集非常庞大，工作站(或集群)的 CPU 内核比用户多。在处理来自许多用户的请求的服务器端系统上，让每个进程从头到尾处理一个计算会更简单、更高效，从而减少进程间的通信和协调开销。除了`multiprocessing`，Gorelick 和 Ozsvald 还展示了许多其他利用多核、GPU、集群、分析器和编译器(如 Cython 和 Numba)开发和部署高性能数据科学应用的方法。他们的最后一章“来自现场的经验教训”是由 Python 中高性能计算的其他从业者贡献的简短案例研究的有价值的集合。](https://fpy.li/19-56)

[*马修·威尔克斯(Apress)著的《高级 Python 开发*](https://fpy.li/19-57) 》是一本罕见的书，其中包括简短的例子来解释概念，同时还展示了如何构建一个可供生产的现实应用:一个数据聚合器，类似于 DevOps 监控系统或分布式传感器的物联网数据收集器。*高级 Python 开发*中的两章讲述了使用`threading`和`asyncio`进行并发编程。

Jan Palach 的《用 Python 进行 的 [*并行编程》(Packt，2014)解释了并发和并行背后的核心概念，涵盖了 Python 的标准库以及*芹菜*。*](https://fpy.li/19-58)

“线程的真相”是 Caleb Hattingh (O'Reilly)在 Python 中使用 Asyncio 的 [*中第二章的标题。这一章涵盖了线程化的优点和缺点——引用了几个权威来源的令人信服的引文——阐明了线程的根本挑战与 Python 或 GIL 无关。使用 Python 中的 Asyncio 逐字引用*第 14 页的内容*:*](https://fpy.li/hattingh)

> 这些主题贯穿始终:
> 
> *   线程化使得代码难以推理。
>     
>     
> *   对于大规模并发(成千上万的并发任务)，线程是一种低效的模型。

如果你想知道线程和锁的推理有多困难——而不冒丢掉工作的风险——试试艾伦·唐尼的工作手册中的练习，[](https://fpy.li/19-59)*(绿茶出版社)。唐尼书中的练习从简单到非常难到无法解决，但即使是简单的练习也令人大开眼界。*

 *## GIL

如果你对 GIL 感兴趣，记住我们无法从 Python 代码中控制它，所以规范参考在 C-API 文档中: [*线程状态和全局解释器锁*](https://fpy.li/19-60) 。 *Python 库和扩展 FAQ* 解答: [*“我们不能摆脱全局解释器锁吗？”*](https://fpy.li/19-61) 。同样值得一读的是吉多·范·罗苏姆和杰西·诺勒(`multiprocessing`包的贡献者)的帖子，分别是:[“移除 GIL 并不容易”](https://fpy.li/19-62)和[“Python 线程和全局解释器锁”](https://fpy.li/19-63)。

[*安东尼·肖(Real Python)的 CPython Internals*](https://fpy.li/19-64) 讲解了 C 编程层面的 CPython 3 解释器的实现。Shaw 最长的章节是“并行性和并发性”:深入探究 Python 对线程和进程的原生支持，包括使用 C/Python API 管理扩展的 GIL。

最后，David Beazley 在“了解 Python GIL”](https://fpy.li/19-65)中进行了详细的探索。 ^([27) 在[演示文稿](https://fpy.li/19-66)的第 54 张幻灯片中，Beazley 报告了在 Python 3.2 中引入新的 GIL 算法后，特定基准的处理时间有所增加。根据实施新 GIL 算法的 Antoine Pitrou 在 Beazley 提交的错误报告中的评论，这个问题在实际工作负载中并不严重。

## 超越标准库的并发性

*Fluent Python* 专注于标准库的核心语言特性和核心部分。 [*全栈 Python*](https://fpy.li/19-69) 是对这本书的一个很好的补充:它是关于 Python 的生态系统的，有标题为“开发环境”、“数据”、“Web 开发”和“DevOps”等章节。

我已经提到了两本书，它们涵盖了使用 Python 标准库的并发性，也包含了关于第三方库和工具的重要内容: [*高性能 Python* ，第二版。](https://fpy.li/19-56)和 [*用 Python 进行并行编程*](https://fpy.li/19-58) 。Francesco Pierfederici 的 [*使用 Python*](https://fpy.li/19-72) (Packt)的分布式计算涵盖了标准库以及云提供商和 HPC(高性能计算)集群的使用。

[“Python、性能和 GPU”](https://fpy.li/19-73)作者 Matthew Rocklin 是“使用 Python 的 GPU 加速器的状态更新”，发布于 2019 年 6 月。

“Instagram 目前拥有全球最大的 Django web 框架部署，该框架完全用 Python 编写。”这是由 Instagram 软件工程师闵妮写的博文[“用 Python 提高 Instagram 的 Web 服务效率”](https://fpy.li/19-74)的开篇句子。这篇文章描述了 Instagram 用来优化其 Python 代码库效率的指标和工具，以及在“每天 30-50 次”部署其后端时检测和诊断性能退化。

[*Python 架构模式:支持测试驱动开发、领域驱动设计和事件驱动微服务*](https://fpy.li/19-75)Harry PERC ival 和 Bob Gregory (O'Reilly)提出了 Python 服务器端应用的架构模式。作者们还在 cosmicpython.com[](https://fpy.li/19-76)*网站上免费提供这本书。*

 *joo s . o . Bueno 的 [*lelo*](https://fpy.li/19-77) 和 Nat Pryce 的[*python-parallelise*](https://fpy.li/19-78)是两个优雅且易于使用的跨进程任务并行库。 *lelo* 包定义了一个`@parallel`装饰器，你可以将它应用于任何函数，神奇地使它解除阻塞:当你调用被装饰的函数时，它的执行在另一个进程中开始。Nat Pryce 的 *python 并行化*包提供了一个`parallelize`生成器，它将一个`for`循环的执行分布在多个 CPU 上。这两个包都是基于*多处理*库构建的。

Python 核心开发人员埃里克·斯诺维护着一个[多核 Python](https://fpy.li/19-79) wiki，其中记录了他和其他人为改善 Python 对并行执行的支持所做的努力。斯诺是 [PEP 554 的作者——Stdlib](https://fpy.li/pep554)中的多个解释器。如果获得批准和实施，PEP 554 为未来的增强奠定了基础，最终可能允许 Python 使用多个内核，而没有*多处理*的开销。最大的障碍之一是假设单个解释器的多个活动子解释器和扩展之间的复杂交互。

mark Shannon——也是 Python 的维护者——创建了一个[有用的表格](https://fpy.li/19-80),比较 Python 中的并发模型，在他、埃里克·斯诺和其他开发者在 [python-dev](https://fpy.li/19-81) 邮件列表上关于子解释器的讨论中被引用。在香农的表格中，“理想 CSP”一栏指的是东尼·霍尔在 1978 年提出的理论上的[沟通顺序过程](https://fpy.li/19-82)模型。Go 还允许共享对象，这违反了 CSP 的一个基本约束:执行单元应该通过通道传递消息来进行通信。

[*无栈 Python*](https://fpy.li/19-83) (又名*无栈*)是 CPython 实现微线程的一个分支，微线程是应用级的轻量级线程——与 OS 线程相反。大型多人在线游戏 [*EVE Online*](https://fpy.li/19-84) 是建立在 *Stackless* 之上的，游戏公司 [CCP](https://fpy.li/19-85) 雇佣的工程师有一段时间是 *Stackless* 的[维护者。*无栈*的一些特性在](https://fpy.li/19-86)[](https://fpy.li/19-87)*Pypy 解释器和 [*greenlet*](https://fpy.li/19-14) 包 [*gevent*](https://fpy.li/19-17) 联网库的核心技术中被重新实现，而后者又是 [*Gunicorn*](https://fpy.li/gunicorn) 应用服务器的基础。*

 *并发编程的参与者模型是高度可伸缩的 Erlang 和 Elixir 语言的核心，也是 Scala 和 Java 的 Akka 框架的模型。如果你想尝试 Python 中的演员模型，可以查看一下 [*演员*](https://fpy.li/19-90) 和 [*Pykka*](https://fpy.li/19-91) 库。

我剩下的建议很少或根本没有提到 Python，但仍然与对本章主题感兴趣的读者相关。**  **## 超越 Python 的并发性和可伸缩性

阿尔瓦罗·维德拉(Alvaro Videla)和杰森·j·w·威廉姆斯(Jason J. W. Williams，Manning)撰写的《RabbitMQ 在行动中》(rabbit MQ in Action)(T3)是一篇非常好的介绍 *RabbitMQ* 和高级消息队列协议(Advanced Message Queuing Protocol，AMQP)标准的文章，并附有 Python、PHP 和 Ruby 的例子。不考虑您的其他技术，即使您计划使用 *Celery* 和 *RabbitMQ* ，我也推荐这本书，因为它涵盖了分布式消息队列的概念、动机和模式，以及大规模操作和调优 *RabbitMQ* 。

我通过阅读保罗·布切(实用主义书架)的《七周七个并发模型》(7 周 14 个并发模型，15 个并发模型)，以及《当线程解开时》(17 个并发模型)的雄辩副标题(16 个并发模型)，学到了很多。本书的第一章介绍了 Java 中线程和锁编程的核心概念和挑战。本书剩余的六章专门讨论作者认为的并发和并行编程的更好的替代方案，由不同的语言、工具和库支持。示例使用了 Java、Clojure、Elixir 和 C(关于使用 [OpenCL 框架](https://fpy.li/19-94)进行并行编程的章节)。CSP 模型以 Clojure 代码为例，尽管 Go 语言因推广了这种方法而值得称赞。Elixir 是说明 actor 模型的示例语言。一个免费提供的关于演员的额外章节使用了 Scala 和 Akka 框架。除非你已经了解 Scala，否则 Elixir 是一种更容易理解的语言，可以用来学习和实验 actor 模型和 Erlang/OTP 分布式系统平台。

Thoughtworks 的 Unmesh Joshi 为 Martin Fowler 的博客[贡献了几页记录“分布式系统模式”的文章。第](https://fpy.li/19-96)[页](https://fpy.li/19-97)是这个主题的一个很好的介绍，带有到单个模式的链接。Joshi 正在增量地添加模式，但是已经存在的模式提取了任务关键型系统中多年来辛苦获得的经验。

Martin Kleppmann 的 [*设计数据密集型应用*](https://fpy.li/19-98) (O'Reilly)是一本由具有深厚行业经验和高级学术背景的从业者撰写的不可多得的书。作者在成为剑桥大学分布式系统研究员之前，曾在 LinkedIn 和两家初创公司从事大规模数据基础设施工作。Kleppmann 书中的每一章都以广泛的参考文献列表结尾，包括最近的研究成果。这本书还包括许多启发性的图表和美丽的概念图。

我有幸成为 2016 年 OSCON 上 Francesco Cesarini 关于可靠分布式系统架构的杰出研讨会的观众:“使用 Erlang/OTP 设计和构建可扩展性”([视频](https://fpy.li/19-99)在 O'Reilly 学习平台)。尽管标题如此，在视频的第 9:35 分，Cesarini 解释道:

> 我要说的很少是针对 Erlang 的[…]。事实是，Erlang 将消除许多意外的困难，使系统具有弹性、永不失败、可伸缩。因此，如果您使用 Erlang 或者运行在 Erlang 虚拟机上的语言，事情会简单得多。

该研讨会基于 Francesco Cesarini 和 Steve Vinoski (O'Reilly)的《使用 Erlang/OTP 设计可伸缩性的[》的最后四章。](https://fpy.li/19-100)

编写分布式系统具有挑战性并且令人兴奋，但是要小心 [*网络规模的嫉妒*](https://fpy.li/19-40) 。[亲理](https://fpy.li/19-102)保持扎实的工程建议。

查看论文[“可扩展性！但代价是什么？”弗兰克·麦克雪莉、迈克尔·伊萨德和德里克·g·默里。作者确定了学术研讨会上展示的并行图形处理系统，这些系统需要数百个内核才能胜过“合格的单线程实现”他们还发现系统“在他们报告的所有配置中表现不如一个线程”](https://fpy.li/19-103)

这些发现让我想起了一句经典的黑客妙语:

> 我的 Perl 脚本比你的 Hadoop 集群快。***  **[1] 第八讲的幻灯片[“并发不是并行”](https://fpy.li/19-1)。

我曾和西蒙教授一起学习和工作，他喜欢说科学中有两大原罪:用不同的词来表示同一件事和用一个词来表示不同的事情。imre Simon(1943–2009)是巴西计算机科学的先驱，他对自动机理论做出了开创性的贡献，开创了热带数学领域。他也是自由软件和自由文化的倡导者。

这个部分是我的朋友布鲁斯·埃凯尔建议的，他写过一些关于 Kotlin、Scala、Java 和 C++的书。

[4] 调用 [`sys.getswitchinterval()`](https://fpy.li/19-3) 得到音程；用 [`sys.setswitchinterval(s)`](https://fpy.li/19-4) 换一下。

[5] 系统调用是用户代码对操作系统内核函数的调用。I/O、定时器和锁是一些可以通过系统调用获得的内核服务。要了解更多信息，请阅读维基百科[“系统调用”文章](https://fpy.li/19-5)。

Antoine Pitrou 在一篇 [python-dev 消息中特别提到了`zlib`和`bz2`模块，他为 Python 3.2 贡献了时间分片 GIL 逻辑。](https://fpy.li/19-6)

[7] 来源:Beazley[《发电机:最后的边疆》教程](https://fpy.li/19-7)第 106 张幻灯片。

[8] 来源:[“线程对象”部分最后一段](https://fpy.li/19-8)。

[9] Unicode 有很多对简单动画有用的字符，例如[盲文图案](https://fpy.li/19-11)。为了保持例子简单，我使用了 ASCII 字符`"\|/-"`。

[10] 信号量是可以用来实现其他同步机制的基本构件。Python 为线程、进程和协程提供了不同的信号量类。我们将在“使用 asyncio.as_completed 和一个线程” ( 第 21 章)中看到`asyncio.Semaphore`。

[11] 感谢科技评测师 Caleb Hattingh 和 Jürgen Gmach，他们没有让我忽略 *greenlet* 和 *gevent* 。

[12] 这是一款 15 英寸 MacBook Pro 2018，采用 6 核、2.2 GHz 英特尔酷睿 i7 CPU。

这在今天是真实的，因为你可能正在使用一个带有抢占式多任务处理的现代操作系统。NT 时代之前的 Windows 和 OSX 时代之前的 macOS 都不是“抢占式”的，因此任何进程都可能占用 100%的 CPU 并冻结整个系统。我们今天还没有完全摆脱这种问题，但是相信这个灰胡子:这在 1990 年代困扰着每个用户，硬重置是唯一的解决办法。

[14] 在这个例子中，`0`是一个方便的哨兵。`None`也常用于此。使用`0`简化了`PrimeResult`的类型提示和`worker`的代码。

[15] 生存连载而不失身份是一个相当不错的人生目标。

[16] 参见 [*中*](https://fpy.li/code)*[*19-concurrency/primes/threads . py*](https://fpy.li/19-27)流畅 Python* 代码库。

[17] 了解更多，请参见英文维基百科中的[【上下文切换】](https://fpy.li/19-28)。

这些可能也是促使 Ruby 语言的创造者松本幸宏在他的解释器中使用 GIL 的原因。

作为大学里的一项练习，我必须用 c 语言实现 LZW 压缩算法，但首先我用 Python 写了它，以检查我对规范的理解。C 版本大约快了 900 倍。

[20] 来源:Thoughtworks 技术顾问委员会、 [*技术雷达*—2015 年 11 月](https://fpy.li/19-40)。

^(21](ch19.xhtml#idm46582389974960-marker)) 将应用程序缓存(由您的应用程序代码直接使用)与 HTTP 缓存进行对比，HTTP 缓存将放置在[图 19-3 的顶部边缘，以提供静态资产，如图像、CSS 和 JS 文件。内容交付网络(cdn)提供了另一种类型的 HTTP 缓存，部署在离应用程序最终用户更近的数据中心。

[22] 图表改编自图 1-1，*设计数据密集型应用*作者 Martin Kleppmann (O'Reilly)。

一些发言者拼出了 WSGI 的首字母缩写词，而另一些人则将其读作与“威士忌”押韵的一个单词。

[25] 彭博工程师 Peter Sperl 和 Ben Green 撰写了[“为生产部署配置 uw SGI”](https://fpy.li/19-44)，解释了 *uWSGI* 中有多少默认设置不适合很多常见的部署场景。Sperl 在[europpython 2019](https://fpy.li/19-45)上展示了他们的建议摘要。强烈推荐给 *uWSGI* 的用户。

[26] 凯勒是本版*流畅巨蟒*的技术评论家之一。

[27] 感谢卢卡斯·布鲁尼拉蒂给我发来了这篇演讲的链接。

[28] Python 的`threading`和`concurrent.futures`API 受 Java 标准库的影响很大。

[29] 二郎群体用“过程”这个词来形容行动者。在 Erlang 中，每个进程都是其自身循环中的一个函数，因此它们非常轻量级，在一台机器上同时激活数百万个进程是可行的——与我们在本章其他地方讨论的重量级操作系统进程没有关系。因此，我们这里有西蒙教授描述的两种罪恶的例子:用不同的词来表示同一件事，用一个词来表示不同的事。**