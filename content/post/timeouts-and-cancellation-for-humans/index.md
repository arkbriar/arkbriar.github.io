---
title: "代码杂谈：超时与取消（转载）"
date: 2022-05-07T16:14:00+08:00
slug: timeouts-and-cancellation-for-humans
draft: false
image: img/bailan1.jpeg
toc: true
categories: [ "Programming" ]
tags: [ "timeout", "cancellation token" ]
---

转载自 Nathaniel J. Smith 的一篇博客，原文是 [Timeouts and cancellation for humans](https://vorpus.org/blog/timeouts-and-cancellation-for-humans/)，发表于 2018 年 1 月 11 日（所以注意文中的链接和内容的时效性），本文仅仅是对原文的一个翻译。

> 注：原来计划写一篇文章讲讲在网络和异步编程中如何进行超时和取消，但 NJS 的文章珠玉在前，我也没有能力再写一篇更好的文章了，因此选择了直接进行翻译。
> 
> 另外 DeepL 承担了 80% 的翻译工作，而我只是进行了一些简单的校对和润色，在此对它的卓越贡献表示感谢 🤣。

{{< figure src="img/bailan1.jpeg" height="200" width="200" title="摆烂" >}}

## 原文 -- 为人类设计的超时与取消

你的代码可能是完美的，永远也不会失败，但不幸的是，外面的世界往往并不那么可靠。有时，别人的代码崩溃或是卡死了，网络瘫痪了，或是打印机着火了（注：lp0 on fire，这是[一个古老的 UNIX 错误代码](https://en.wikipedia.org/wiki/Lp0_on_fire)）。你的代码需要对此做好准备：每当你从网络上读取信息时，例如获取一把进程间的锁，或是发送一个 HTTP 请求时，你至少需要考虑以下三种可能的情况：

+ 它可能成功；
+ 它可能失败；
+ 它可能会永远挂在那里，从未成功或失败：日子一天天过去，树叶落下了，冬天来了，但我们的请求仍然在等待，渴望着那永远不会到来的回应。

前两种情况的处理方式是很直接的。但是，为了处理最后一种情况，你需要超时机制。几乎在你的程序与另一个程序、人或系统交互的每一个地方，它都需要一个超时。如果你没有的话，那就是一个潜在的缺陷。

说实话，如果你和大多数开发者一样，你的代码可能有大量的由超时引起的错误，当然我的代码也会有。这很奇怪 -- 因为这种需求是如此普遍，而且是正确进行 I/O 操作的基础，你会认为每个编程环境都提供了简单而强大的方法来为任何操作设置一个超时。但是......它们并没有。事实上，大多数超时 API 是如此的繁琐和容易出错，以至于对于开发者来说，要可靠地完成这个任务是不现实的。所以不要感到难过 -- 你的代码有那些超时的错误不是你的错，而是那些 I/O 库的错！

但现在，我正在[写一个 I/O 的库](https://trio.readthedocs.io/)。这不是一个普通的 I/O 库，而是一个以易于使用为核心卖点的库。所以我想确保在我的库（Trio）中，你可以轻松可靠地将超时应用于任意的 I/O 操作。但设计一个用户友好的超时 API 是一项令人惊讶的棘手任务，所以在这篇博文中，我将深入研究可能的设计，特别是启发我的许多先驱者，然后解释我想到的东西，以及为什么我认为它是对旧的最先进技术的真正改进。最后，我将讨论如何更广泛地应用 Trio 的思想，特别是将展示一个用于老式的同步的 Python 代码的原型实现。

那么，超时处理到底难在哪里？

### 简单的超时不支持抽象 😢

处理超时的最简单的和最明显的方法是，在你的 API 中找到每个潜在的阻塞的函数，并为它们加上一个超时 `timeout` 参数。在 Python 标准库中，你会在诸如 `threading.Lock.acquire` 的 API 中看到这一点：

```python
lock = threading.Lock()

# Wait at most 10 seconds for the lock to become available
lock.acquire(timeout=10)
```

如果你使用 `socket` 模块来进行网络操作，它的工作方式和上面是一样的 -- 除了超时参数 `timeout` 是设置在 socket 对象上的，而不是作为参数传递给每个调用。

```python
sock = socket.socket()

# Set the timeout once
sock.settimeout(10)
# Wait at most 10 seconds to establish a connection to the remote host
sock.connect(...)
# Wait at most 10 seconds for data to arrive from the remote host
sock.recv(...)
```

这比每次都要记得传入明确的超时要方便一些（我们将在下文中更多地讨论方便问题），但重要的是要理解这只是一个纯粹的外观变化。其语义与我们在 `threading.Lock` 中看到的一样：每个方法的调用都有自己独立的 10 秒超时。

那么这有什么问题呢？这似乎是一个很显然的方式。如果我们总是直接使用这些低层的 API 来编写代码，那么这可能就足够了。但是，编程是关于抽象的。比如我们想从 [S3](https://en.wikipedia.org/wiki/Amazon_S3) 上获取一个文件，我们可以使用 boto3 库中的这个接口 [S3.Client.get_object](https://botocore.readthedocs.io/en/latest/reference/services/s3.html#S3.Client.get_object) 来完成。`S3.Client.get_object` 做了什么呢？它向 S3 的服务器发起了一系列的 HTTP 请求，每一个请求都要调用 [requests](http://python-requests.org/) 库。而对 `requests` 的每一次调用都会在内部转化成对 `sockets` 模块的一系列的调用，从而完成真正的网络通信[[1]](https://vorpus.org/blog/timeouts-and-cancellation-for-humans/#id6)。

从用户的视角来看，这是用来从远程服务获取数据的三个不用的 API：

```python
s3client.get_object(...)
requests.get("https://...")
sock.recv(...)
```

当然它们处于不同的抽象层次，但抽象的核心思想就是屏蔽细节，因为用户不必关系细节。因此，如果我们的计划是在所有地方使用 `timeout=` 参数，那么我们应该期望这些函数都能接受一个 `timeout=` 参数。

```python
s3client.get_object(..., timeout=10)
requests.get("https://...", timeout=10)
sock.recv(..., timeout=10)
```

现在问题来了：如果我们是这样做的，那么实际实现这些功能就是一个很麻烦的事。为什么呢？让我们来看一个简化的例子。当处理 HTTP 响应时，有一点是我们已经看到了 `Content-Length` 头，那么现在我们就需要读取这么多的字节来获取实际的响应体。因此在 `requests` 库中的某个地方存在一个类似这样的循环：

```python
def read_body(sock, content_length):
    body = bytearray()
    while len(body) < content_length:
        max_to_receive = content_length - len(body)
        body += sock.recv(max_to_receive)
    assert len(body) == content_length
    return body
```

现在我们要修改这个循环，增加对超时的支持。我们希望能够说：“我愿意最多等待 10 秒来读取响应体”。但是我们不能只是把超时参数传递给 `recv`，因为想象一下这种情况，假设第一次调用 `recv` 需要 6 秒 -- 为了使我们的整体操作能在 10 秒内完成，第二次调用 `recv` 必须被赋予 4 秒的超时。使用 `timeout=` 的方法，每次我们在抽象层之间传递时都需要写一些烦人的垃圾代码来重新计算超时：

```python
  def read_body(sock, content_length, timeout):
+     read_body_deadline = timeout + time.monotonic()
      body = bytearray()
      while len(body) < content_length:
          max_to_receive = content_length - len(body)
+         recv_timeout = read_body_deadline - time.monotonic()
-         body += sock.recv(max_to_receive)
+         body += sock.recv(max_to_receive, timeout=recv_timeout)
      assert len(body) == content_length
      return body
```

（即使是这样，实际上也是简化的，因为我们假装 `sock.recv` 需要一个超时参数 -- 如果你想真正做到这一点，你必须在每个 socket 方法之前调用 `settimeout`，并且可能需要使用一些 `try`/`finally` 的东西来将它设置回去，否则就有可能让你程序的其他地方出错。）

在实践中，没有人会这么做 -- 我所知道的所有接受 `timeout=` 参数的高层 Python 库都只是将它们原封不动地传递给下层，而这破坏了抽象性。例如，这里有两个你今天可能会用到的流行的 Python API，它们看起来都接受类似的 `timeout=` 参数：

```python
import threading
lock = threading.Lock()
lock.acquire(timeout=10)

import requests
requests.get("https://...", timeout=10)
```

但事实上，这两个 `timeout=` 参数的意思完全不同。第一个意味着“尝试获取锁，但 10 秒以后放弃”。第二个意味着“尝试获取给定的 URL，但如果在任何时候任何单独的低价套接字操作超过 10 秒就放弃”。也许你使用 `requests` 的全部原因是你不想考虑低级套接字，但对不起，无论如何你都必须这样做。事实上，目前不可能保证 `requests.get` 会在任何有限的时间内返回：如果一个恶意的或行为错误的服务端每 10 秒至少发送一个字节，那么我们上面的 `requests` 调用将不断地重置其超时，并且永远不会返回。

我并不是想在这里挑剔 `requests` 这个库，因为这类问题在 Python 的 API 中随处可见。我用 `requests` 作为例子是因为 Kenneth Reitz（注：`requests` 库作者）因其对使 API 尽可能明显和直观的执着而闻名，而这是他少有的失败的地方之一。我想这是 `requests` 的 API 中唯一一个[在文档中用大方框警告你它是反直觉的部分](http://docs.python-requests.org/en/master/user/quickstart/#timeouts)。所以......如果连 Kenneth Reitz 都搞不清楚这个问题，我想我们可以得出这样的结论：只是在上面添加一个 `timeout=` 参数并不能带来符合人类的直觉的 API。

### 绝对的 deadline 是可行的（但它用起来有点烦人😡）

如果 `timeout=` 参数不起作用，我们应该怎么做呢？好吧，这里有一个一些人提倡的选择。注意到在我们上面的 `read_body` 的例子中，我们把传入的相对超时（“从我调用这个函数的那一刻起 10 秒”）转换为了绝对期限（“当时钟读到 12:01:34:851 时”），然后在每次调用 socket 之前转换回来。如果我们用 `deadline=` 参数来写整个 API，而不是用 `timeout=` 参数来写，这段代码会变得更简单。这对库的实现者来说是很简单的，因为你可以直接把 deadline 传到你的抽象栈中：

```python
def read_body(sock, content_length, deadline):
    body = bytearray()
    while len(body) < content_length:
        max_to_receive = content_length - len(body)
        body += sock.recv(max_to_receive, deadline=deadline)
    assert len(body) == content_length
    return body

 # Wait 10 seconds total for the response body to be downloaded
 deadline = time.monotonic() + 10
 read_body(sock, content_length, deadline)
```

（一个像这样工作的著名的 API 是 [Go 的 socket 层](https://pkg.go.dev/net#Conn)）。

但这种方法也有一个缺点：它成功地将烦人的部分从库的内部移出，而将其放在使用 API 的人身上。在设置超时策略的最外层，你的库的用户可能想说“10秒之后放弃”之类的话，如果你只接受一个 `deadline=` 参数，那么他们每次都必须手工进行转换。或者你可以让每个函数都同时接受 `timeout=` 和 `deadline=` 参数，但是你需要在每个函数中加入一些模板代码来规范它们，在两个参数都指定的时候抛出一个错误，等等。最后期限是对原始超时的一种改进，但感觉上去这里仍然缺少了一些抽象。

### 取消令牌（cancel tokens）

#### 取消令牌封装了取消状态

这里有一个缺失的抽象：与其支持两个不同的参数：

```python
# What users do:
requests.get(..., timeout=...)
# What libraries do:
requests.get(..., deadline=...)
# How we implement it:
def get(..., deadline=None, timeout=None):
    deadline = normalize_deadline(deadline, timeout)
    ...
```

我们可以用一个方便的构造函数将超时信息封装到一个对象中：

```python
class Deadline:
    def __init__(self, deadline):
        self.deadline = deadline

def after(timeout):
    return Deadline(time.monotonic() + timeout)

# Wait 10 seconds total for the URL to be fetched
requests.get("https://...", deadline=after(10))
```

这对用户来说看起来很自然，但由于它在内部使用了一个绝对的 deadline，所以对库的实现者来说也很容易。

一旦我们走到这一步，我们不妨让它变得更抽象一些。毕竟，超时并不是你想放弃某些阻塞操作的唯一原因。“10 秒后放弃”只是“在<某个任意条件为真>后放弃”的一个特例。假设你正在使用 `requests` 库实现一个网络浏览器，你会希望能够说“开始获取这个 URL，但在‘停止’按钮被按下时放弃”。在任何情况下，库都会把这个 `Deadline` 对象视为完全不透明的 -- 它们只是把它传递给低层的调用，并相信最终一些底层的操作原语会适当地解释它。因此，与其认为这个对象封装了一个最后期限，我们不如开始认为它封装了一个任意的“我们现在应该放弃吗”的检查。为了体现它更抽象的性质，与其将它称作 `Deadline`，让我们称这个新东西 为 `CancelToken`。

```python
# This library is only hypothetical, sorry
from cancel_tokens import cancel_after, cancel_on_callback

# Returns an opaque CancelToken object that enters the "cancelled"
# state after 10 seconds.
cancel_token = cancel_after(10)
# So this request gives up after 10 seconds
requests.get("https://...", cancel_token=cancel_token)

# Returns an opaque CancelToken object that enters the "cancelled"
# state when the given callback is called.
cancel_callback, cancel_token = cancel_on_callback()
# Arrange for the callback to be called if someone clicks "stop"
stop_button.on_press = cancel_callback
# So this request gives up if someone clicks 'stop'
requests.get("https://...", cancel_token=cancel_token)
```

将取消条件提升为一级对象（注：first-class object）使我们的超时 API 更容易使用，同时也使其功能大大增强：现在我们不仅可以处理超时，还可以处理任意的取消，这是编写并发代码时非常常见的需求（例如，它可以让我们表达这样的东西：并行发起冗余的两个请求，一旦其中一个请求完成，就取消另一个）。这是一个非常棒的主意。据我所知，它最初来自于 Joe Duffy 在 C# 中提出的 [取消令牌](https://blogs.msdn.microsoft.com/pfxteam/2009/05/22/net-4-cancellation-framework/)，在 Go 语言中的 [context 对象](https://pkg.go.dev/context) 本质上也是同样的想法。这些人（注：API 的设计者）都非常聪明！事实上，取消令牌还解决了其他一些在传统取消系统中出现的问题。

#### 取消令牌是水平触发的，可以根据你的程序需要来确定范围

在我们对超时和取消的 API 的小小考察中，我们是从超时开始的。如果你是从取消开始的，那么你将会看到另一个在很多系统中常见的模式：一个方法可以让你取消另一个线程（或任务，或任何你的框架使用的线程等同物），通过唤醒它并抛出某种异常。例子包括 `asyncio` 的 [Task.cancel](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.cancel)、`Curio` 的 [Task.cancel](https://curio.readthedocs.io/en/latest/reference.html#Task.cancel)、Java 的 [Thread.interrupt](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#interrupt--)、C# 的 [Thread.interrupt](https://msdn.microsoft.com/en-us/library/system.threading.thread.interrupt(v=vs.110).aspx)，等等。为了体现它们的性质，我把这称为“线程中断”的取消方法。

在线程中断方法中，取消是一个针对固定规模实体的时间点事件：一个调用 → 一个线程/任务中的异常。这里有两个问题。

首先，扩展性的问题是很明显的：如果你有一个正常调用的单一函数，但你可能需要取消它，那么你就必须为它生成一个新的线程/任务/其他相应的实体：

```python
http_thread = spawn_new_thread(requests.get, "https://...")
# Arrange that http_thread.interrupt() will be called if someone
# clicks the stop button
stop_button.on_click = http_thread.interrupt
try:
    http_response = http_thread.wait_for_result()
except Interrupted:
    ...
```

在这里，线程并没有被用于并发，它只是一种让你限定取消作用域的方式，这相当的笨拙。

或者，如果你有一个大的复杂的工作想取消 -- 例如，在内部产生了多个工作线程的东西？在我们上面的例子中，如果 `requests.get` 催生了一些额外的后台线程，当我们取消第一个线程时，它们保持被挂起的状态一直留在那。正确处理这个问题需要一些复杂而微妙的记录。

取消令牌解决了这个问题：它们取消的工作是“令牌被传递到的任何东西”，这可能是一个单一的函数，或者是一个复杂的多层线程池集合，或者是介于两者之间的任何东西。

线程中断方法的另一个问题更为微妙：它将取消视为一个事件。相对的，取消令牌的模型则是将取消作为一种状态：它们从未取消的状态开始，最终转变到已取消的状态。

这很微妙，但它使取消令牌不那么容易出错。一种思考这个问题的方式是考虑[边缘触发和水平触发的区别](https://lwn.net/Articles/25137/)：线程中断 API 提供边缘触发的取消通知，相比之下，取消令牌提供的则是水平触发的。边缘触发的 API 在使用上是出了名的棘手。你可以在 Python 的 [threading.Event](https://docs.python.org/3/library/threading.html#threading.Event) 中看到一个例子：尽管它被称为“事件”，但它实际上有一个内部的布尔状态，而取消一个取消令牌则像是设置一个事件。

这些都是相当抽象的，所以让我们把它们变得更具体一点。思考一下，`try`/`finally` 的常见的使用模式是确保连接能够被正确地关闭。这里有一个相当人性化的例子，该函数建立了一个 Websocket 连接，发送了一条消息，然后无论 `send_message` 是否抛出了异常都确保关闭它：

```python
def send_websocket_messages(url, messages):
    open_websocket_connection(url)
    try:
        for message in messages:
            ws.send_message(message)
    finally:
        ws.close()
```

现在，假设我们开始运行这个函数，但在某一个时刻另一方退出了网络，此时我们的 `send_message` 调用将永远挂起。最终，我们厌倦了等待，并取消了它。

通过一个线程中断式的边缘触发 API，这将导致 `send_message` 调用立刻引发一个异常，然后我们的连接清理代码自动运行，到目前为止看起来还不错。但关于 websocket 协议，有一个有趣的事实：它有[一个“关闭”消息](https://tools.ietf.org/html/rfc6455#section-5.5.1)，你应该在关闭连接之前发送。一般来说，这是件好事：它可以使关闭更干净。因此，当我们调用 `ws.close()` 时，它将尝试发送这个消息。但是，在这种情况下，我们试图关闭连接的原因是我们已经放弃了另一方接受任何新的消息。所以现在 `ws.close()` 也将永远挂起。

如果我们使用一个取消令牌，这种情况就不会发生：

```python
def send_websocket_messages(url, messages, cancel_token):
    open_websocket_connection(url, cancel_token=cancel_token)
    try:
        for message in messages:
            ws.send_message(message, cancel_token=cancel_token)
    finally:
        ws.close(cancel_token=cancel_token)
```

一旦取消令牌被触发，那么未来对该令牌的所有操作都会被取消，所以对 `ws.close` 的调用就不会被卡住。这是一个更不容易出错的范式。

有意思的是，这么多老的 API 都会犯这个错误。如果你按照我们在这篇博文中的做法，从考虑一个由多个阻塞调用组成的复杂操作应用超时开始，那么很明显，如果第一个调用用完了整个超时预算，那么以后的调用应该立即失败。超时是很自然地水平触发的。然后当我们从超时扩展到任意的取消时，这个洞察依然保持。但是如果你只考虑原始操作的超时，那么这一点就不会出现；或者如果你从一个通用的取消 API 开始，然后用它来实现超时（比如 Twisted 和 asyncio 做的），那么水平触发的取消的优势就很容易被忽略。

#### 取消令牌在实践中是不可靠的，因为人都是懒惰的

所以，取消令牌有着非常好的语义，显然比原始超时或 deadline 要好的多，但它们仍然有一个可用性问题：要写一个支持取消的函数，你必须接受这个模板参数，然后确保把它传递给你调用的每个子程序。记住，一个正确的、健壮的程序必须在堆栈的任何地方，在每一个做 I/O 的函数中支持取消。如果你偷懒，把它漏掉了，或者忘记把它传递给任何特定的子程序调用，那么你就有一个潜在的缺陷。

人类很不擅长这种模板。我的意思是，不是说你，我相信你是一个非常勤奋的程序员，确保在每个函数中实现正确的取消支持，而且每天都会使用牙线（注：flosses every day，不是很懂啥梗）。但是......也许你的一些同事就没有这么勤奋了？或者你依赖别人写的一些库 -- 你有多信任你的第三方供应商能做好这件事？随着你的工作栈的增长，每个人都能做到这一点的可能性趋近于零。

我可以用任何真实的例子来证明这一点吗？好吧，考虑一下：在 C# 和 Go 这两种最著名的语言中，使用这种方法并且已经提倡了很多年，底层的网络原语仍然没有支持取消令牌。这听上去就像...这些基本操作可能会因为你无法控制的原因而挂起，你需要准备好超时或取消，但是......我猜他们只是还没来得及实现它？相反，他们的 socket 层支持一种旧的机制来设置 socket 对象的[超时](https://msdn.microsoft.com/en-us/library/system.net.sockets.socket.receivetimeout(v=vs.110).aspx)或[最后期限](https://golang.org/pkg/net/#IPConn.SetDeadline)，如果你想使用取消令牌，你必须自己想办法在这两个不同的系统之间建立桥梁。

Go 标准库确实提供了一个如何做到这一点的例子：他们建立网络连接的函数（基本相当于 Python 的 `socket.connect`）确实接受取消令牌。实现这个功能需要 [40 行代码](https://github.com/golang/go/blob/bf0f69220255941196c684f235727fd6dc747b5c/src/net/fd_unix.go#L99-L141)和一个后台任务，并且第一次尝试[导致了一个花了一年才在生产环境排查出来的资源竞争问题](https://github.com/golang/go/issues/16523)。所以...在 Go 中，如果你想使用取消令牌（或 `Context`，用 Go 的说法），那么我想这就是你每次使用任何 socket 操作时需要实现的东西？祝你好运？

我并不是要取笑你，这东西是很难搞。但是 C# 和 Go 都是由高水平的全职开发者团队维护的巨大项目，并由世界 50 强公司支持。如果他们都做不好，谁又能做到呢？反正不是我。我只是一个试图在 Python 中重新发明 I/O 操作的人类。我搞不定那么复杂的东西。

### 取消作用域（cancel scopes）：（Trio 对超时和取消对人性化解决方案 👍）

还记得在这篇文章的开头，我们注意到 Python 的 socket 方法不接受单独的超时参数，而是让你在 socket 上设置一次超时，这样它就隐含地传递给你调用的每个方法？而在上面的部分，我们注意到 C# 和 Go 也做了差不多的事情？我想他们是有道理的。也许我们应该承认，当你有一些数据必须传递给你调用的每个函数时，这是计算机应该处理的事情，而不是让不稳定的人类来做这些工作 -- 但要以一种通用的方式支持复杂的抽象，而不仅仅是 sockets。

#### 取消作用域是怎么工作的

以下是如何在 Trio 中对 HTTP 请求施加 10 秒的超时：

```python
# The primitive API:
with trio.open_cancel_scope() as cancel_scope:
    cancel_scope.deadline = trio.current_time() + 10
    await request.get("https://...")
```

当然，通常你会使用一个[方便的封装器](https://trio.readthedocs.io/en/latest/reference-core.html#trio.move_on_after)，像这样：

```python
# An equivalent but more idiomatic formulation:
with trio.move_on_after(10):
    await requests.get("https://...")
```

但由于这篇文章是关于底层设计的，所以我们将专注于原始的版本。（致谢：使用 `with` 块进行超时的想法是我在 Dave Beazley 的 Curio 中第一次看到的，虽然我改了许多。我会把细节藏在一个脚注里：[[4]](https://vorpus.org/blog/timeouts-and-cancellation-for-humans/#id9)）。

你应该认为 `open_cancel_scope()` 是在创建一个取消令牌，但它实际上并没有公开暴露任何 `CancelToken` 对象。相反的，取消令牌被压进了一个不可见的内部堆栈中，并自动应用于 `with` 块内调用的任何阻塞操作。因此，`requests` 不需要做任何事情来传递它 -- 当它最终通过网络发送和接收数据时，这些原始调用将自动应用其中的最后期限。

`cancel_scope` 对象让我们可以控制取消状态：你可以改变最后期限，或是通过调用 `cancel_scope.cancel()` 发起一个明确的取消，[等等](https://trio.readthedocs.io/en/latest/reference-core.html#trio.The%20cancel%20scope%20interface)。如果你了解 C#，它就类似于一个 [CancellationTokenSource](https://msdn.microsoft.com/en-us/library/system.threading.cancellationtokensource(v=vs.110).aspx)。一个它允许的有用的技巧是，在更原始的取消作用域回卷语义上，实现人们[习惯的那种“当超时发生时抛出一个错误”的 API](https://github.com/python-trio/trio/blob/07d144e701ae8ad46d393f6ca1d1294ea8fc2012/trio/_timeouts.py#L96-L118)。

当一个操作被取消时，它会抛出一个 `Cancelled` 异常，这个异常被用来将堆栈回卷到合适的 `open_cancel_scope` 块中。取消作用域可以是嵌套的，`Cancelled` 异常知道它们是在哪个作用域触发的，并将持续传播直到到达相应的 `with` 块中（因此，你应该总是让 Trio 运行时负责引发和捕捉取消异常，这样它就能正确地跟踪这些关系）。

支持嵌套很重要，因为有些操作可能想在内部使用超时作为实现细节。例如，当你要求 Trio 与一个有多个 IP 地址关联的主机名建立 TCP 连接时，它使用了一种叫“happy eyeballs”的算法，[以交错的方式并行运行多个连接尝试](https://trio.readthedocs.io/en/latest/reference-io.html#trio.open_tcp_stream)，这需要一个内部超时来决定何时启动下一次连接尝试。但是，用户不应该关心这个问题！如果你想说“尝试连接到 example.com:443，但 10 秒后放弃”，那就这么写：

```python
with trio.move_on_after(10):
    tcp_stream = await trio.open_tcp_stream("example.com", 443)
```

感谢取消作用域的嵌套规则，一切都能正常工作。事实证明，`open_tcp_stream` 可以正确地处理这个问题，不需要额外的代码。

#### 在哪里检查取消状态？

面对取消操作，想要正确地编写代码是比较棘手的。如果一个 `Cancelled` 异常突然出现在用户没有准备好的地方 -- 也许是在他们的代码正在操作一些微妙的数据结构的时候 -- 它可能会破坏内部状态并导致难以追踪的缺陷。另一方面，如果你不能相对及时地注意到取消的情况，那么超时和取消系统也没有什么用处。因此，对任何系统来说，一个重要的挑战是首先选择一个“[金发姑娘原则](https://jamesclear.com/goldilocks-rule)”（注：源自童话故事《金发姑娘和三只熊》，比喻做力所能及、难易适中的事情），检查的频率要足够高，但又不能太频繁，然后以某种方式将这个规则传达给用户，使他们能够确保他们的代码已经准备好了。

在 Trio 的例子中，这是很简单的。由于其他一些原因，我们已经使用 Python 的 async/await 语法来注释阻塞函数。它的主要作用是让你在看到任何函数的文本时，可以立刻看到哪些点可能会阻塞，直到某些事情发生。比如这个例子：

```python
async def user_defined_function():
    print("Hello!")
    await trio.sleep(1)
    print("Goodbyte!")
```

在这里我们可以看到，对 `trio.sleep` 对调用时阻塞的，因为它有一个特殊的 `await` 关键字。如果不使用这个关键字，你就不能调用 `trio.sleep` 或任何其他 Trio 的内置阻塞原语，因为它们被标记成同步函数。并且 Python 强制规定，如果你想使用 `await` 关键字，那么你必须要调用方的函数也标记成 async，这意味着所有调用 `user_defined_function` 的人也将使用 `await` 关键字（注：这是 async/await 语法受到诟病的一点，语法的传播导致了以往的阻塞代码无法复用）。这是有道理的，因为如果 `user_defined_function` 调用了一个阻塞的函数，那它也就成了一个阻塞的函数。在许多其他系统中，一个函数是否会阻塞，你只能通过检查它的所有潜在的被调用者，以及它们的所有被调用者等来确定；async/await 采用了这种全局的运行时属性，使它在源代码中一目了然。

Trio 的取消作用域在这个系统上搭了一个便车：我们声明，无论在哪你看到了一个 `await`，那就是一个你可能需要处理 `Cancelled` 异常的地方 -- 要么是因为它调用了 Trio 的直接检查取消的原语，要么是因为它调用了一个间接调用这些原语的函数，从而可能看到一个 `Cancelled` 异常涌现出来。这导致了几个很好的特性。首先，它非常容易向用户解释。它涵盖了所有你绝对需要超时/取消支持以避免无限挂起的函数 -- 只有那些阻塞的函数才会被永远阻塞。这意味着任何定期做 I/O 操作的函数也会定期自动检查取消状态，所以大多数时候你不需要担心这个问题（尽管对于偶尔的长期运行的纯计算，你可能想通过调用 `await trio.sleep(0)` 来增加一些显式的取消检查 -- 无论如何你必须这样做才能让调度器工作！）。阻塞函数往往有[大量的失败模式](https://docs.python.org/3/library/exceptions.html#os-exceptions)，因此在许多情况下，处理 `Cancelled` 异常所需的清理工作将与处理例如一个行为不端的网络对等体（注：peer）所需的清理工作是共享的。而且 Trio 的合作多任务系统也使用等待点来标记调度器可能切换到另一个任务的地方，所以你已经要小心在 `await` 过程中让数据结构处于不一致的状态。取消和 async/await 就像花生酱和巧克力一样（注：比喻最佳的组合）。

#### 一个逃生舱口

虽然检查所有阻塞原始调用的取消状态是一个非常好的默认选择，但在一些非常罕见的情况下，你希望禁用这个功能并对取消的行为进行明确的控制。这些情况非常罕见，所以我没有一个简单的例子可以在这里使用（尽管在 Trio 源代码中有一些神秘的例子，如果你真的很好奇，你可以去找找看）。为了提供这样一个逃生舱口，你可以设置一个取消作用域来“屏蔽”它的内容，使其不受外界取消的影响。它看起来像这样：

```python
with trio.move_on_after(10):
    with trio.open_cancel_scope() as inner_scope:
        inner_scope.shield = True
        # Sleeps for 20 seconds, ignoring the overall 10 second
        # timeout
        await trio.sleep(20)
```

为了支持组合，屏蔽对取消作用域的栈是敏感的：它只阻止外部取消作用域的应用，而对内部作用域没有影响。在我们上面的例子中，我们的屏蔽对任何可能在 `trio.sleep` 中使用的取消作用域没有任何影响 -- 这些作用域仍然表现正常。这很好，因为无论 `trio.sleep` 在内部做什么都是它自己的私有实现细节。事实上，`trio.sleep` 在内部确实[使用了一个取消作用域](https://github.com/python-trio/trio/blob/07d144e701ae8ad46d393f6ca1d1294ea8fc2012/trio/_timeouts.py#L65-L66)。

`shield` 是取消作用域的一个属性，而不是有一个特殊的“屏蔽作用域”，其原因之一是它使得实现这种嵌套很方便，因为我们可以重新使用取消作用域的现有堆栈结构。另一个原因是，在你禁用外部超时的任何地方，你都需要考虑你要做什么来代替，以确保它不会永远挂起。而有一个取消作用域在那里就可以很容易地在局部代码的控制下应用一个新的超时时间：

```python
# Demonstrating that the shielding scope can be used to avoid hangs
# after disabling outside timeouts:
with trio.move_on_after(10) as outer_scope:
    with trio.move_on_after(15) as inner_scope:
        inner_scope.shield = True
        # Returns after 15 seconds, when the shielding scope expires:
        await trio.sleep(1000000)
```

现在，如果你是一个 Trio 的用户，请忘记你读过这一节。如果你认为你需要使用屏蔽能力，那么你几乎肯定应该重新考虑你要做的事情。但如果你是一个 I/O 运行时的实现者，希望增加取消作用域的支持，那么这就是一个重要的功能。

#### 取消作用域和并发

最后，这里还应该提到 Trio 的一个特性。到目前为止，在这篇文章中，我根本没有过多地讨论并发。超时和取消在很大程度上是独立的，上面的一切甚至适用于直接的单线程同步代码。但我们确实做了一些看似微不足道的假设：如果你在一个 `with` 块内调用一个函数，那么（a）执行将时机发生在 `with` 块内，以及（b）它抛出的任何异常都将被传播回 `with` 块，以便它能够捕获它们。不幸的是，许多线程和并发库违反了这一点，特别是在一些工作被生成或调度的情况下：

```python
# This looks innocent enough:
with move_on_after(10):
    do_the_thing()

# But it isn't:
def do_the_thing():
    # Using some made-up API similar to what most systems use:
    start_task_in_background(some_worker_that_will_actually_do_the_thing)
```

如果我们只看 `with` 块，这似乎是完全无辜的。但是当我们看看 `do_the_thing` 是如何实现的，我们就会意识到，我们和可能在后台任务完成之前退出 `with` 块，所以就会出现一些模糊的地方：超时是否应该适用于后台任务？如果它确实适用，那么我们应该如何处理 `Cancelled` 异常？对于大多数系统来说，后台线程/任务中未处理的异常被简单地丢弃了。

然而，这些问题在 Trio 中不会出现，因为它有独特的方法来处理并发性。Trio 的[托儿所系统](https://trio.readthedocs.io/en/latest/reference-core.html#tasks-let-you-do-multiple-things-at-once)（注：nursery system，用于实现结构化并发的系统）意味着子任务总是被整合到调用堆栈中，着实际上成为一个调用树。具体来说，Trio 没有全局的 `start_task_in_background` 原语。相反，如果你想生成一个子任务，你必须先打开一个“nursery”块（[让孩子住在里面](http://www.dictionary.com/browse/nursery)，明白吗？），然后其中创建的子任务的生命周期就和创建托儿所的块绑定了起来：

```python
with move_on_after(10):
    await do_the_thing()

async def do_the_thing():
    async with trio.open_nursery() as nursery:
        nursery.start_soon(some_worker_that_will_actually_do_the_thing)
        # Now the 'async with' block won't complete until the
        # child task has finished, and if the child has an unhandled
        # exception then it will be re-raised here in the parent.
        # Which makes this example pretty silly -- the "background
        # task" acts just like a function call. Which is the point :-)
```

这个系统有很多优点，但与此相关的一点是，它保留了取消作用域所依赖的关键假设。任何给定托儿所要么在取消作用域内，要么在取消作用域外 -- 我们可以通过检查 `with open_cancel_scope` 块是否包围了 `async with open_nursery` 块来判断。然后我们可以直接说，如果一个托儿所在取消作用域内，那么该范围应该适用于该托儿所内的所有孩子（注：child，指子任务）。这意味着，如果我们给一个函数应用一个超时，它就不能通过生成一个子任务来“逃逸” -- 超时也适用于子任务。（例外的情况是，如果你向函数传递来一个外部托儿所，那么它可以向该托儿所派生任务，这样就可以逃脱超时。但这对调用者来说是显而易见的，因为他们必须提供托儿所 -- 关键是要让人明白发生了什么，而不是让人无法生成后台任务）。

#### 总结

回到我们最初的例子：我一直在做一些最初的工作来将 `requests` 移植到 Trio 上（[你可以来帮忙！](https://github.com/python-trio/urllib3/issues/1)），到目前为止，看起来 Trio 版本不仅能比传统的同步版本更好地处理超时问题，而且它能用零行代码来做到这一点 -- 所有你想检查取消的地方，Trio 都会自动这样做，而所有你需要特别注意处理所产生的异常的地方，`requests` 都准备处理由于其他原因产生的任何异常。

天下没有免费的午餐，取消的处理仍然是缺陷的一个来源，在编写代码时需要小心。但是 Trio 的取消作用域比我发现的任何其他系统都更容易使用，因此也更可靠。希望我们能使超时错误成为例外，而不是常规情况。

### 还有谁能从取消作用域中受益？

那么......如果你在使用 Trio，那就太好了！这是只在 Trio 的环境下工作的东西，还是一个更普遍的东西？如果要在其他环境中使用，需要做什么样的调整？

如果你想实现取消作用域，那么你就需要：

+ 某种隐含的上下文本地存储来跟踪取消作用域栈。如果你使用的是线程，那么线程本地存储是可行的；如果你使用的是更奇特的东西，那么你就需要在你的系统中找到相应的东西。（例如，在 Go 中你需要 goroutine-local 的存储，而这是著名的[不存在的](https://stackoverflow.com/questions/31932945/does-go-have-something-like-threadlocal-from-java)）。这可能有点棘手，例如在 Python 中，我们需要类似 [PEP 568](https://www.python.org/dev/peps/pep-0568/) 来解决[取消作用域和生成器之间的一些不良交互](https://github.com/python-trio/trio/issues/264)。

+ 一种限定取消作用域边界的方法。Python 的 `with` 工作得很好，其他的选项包括专门的语法，或者将取消作用域限制在单独的函数调用中，比如 `with_timeout(10, some_fn, arg1, arg2)`（尽管这可能会迫使人们进行别扭的重构，而且你需要想出一种办法来暴露取消作用域对象）。

+ 一个在超时/取消发生后，能够将栈回卷到适当的取消作用域的策略。在这点上，异常能够很好地工作，只要你有办法在取消作用域的边界上捕捉它们 -- 这也是 Python 的 `with` 块在这方面工作得如此出色的另一个原因。但是如果你的语言使用，比如说，错误码返回而不是异常，那么我确信你可以从中建立一种堆栈回卷的约束。

+ 一个关于如何将取消作用域和你的并发 API 结合的故事。当然，最理想的是像 Trio 的 `nursery` 系统那样的东西（它也有很多其他的优点，但那是另一篇博客的内容了）。但即使没有这个，你也可以认为在一个取消作用域内产生的任何新任务都会继承该取消作用域，无论他们何时完成（除非它们选择使用类似上述的屏蔽功能来进行退出）。

+ 一些用于确定哪些操作是可以取消的，并且将其传达给用户。如上所述，async/await 可以完美地实现这一点，但如果你没有使用 async/await，那么其他的约定当然也是可以的。具有丰富的静态类型系统的语言可能会以某种方式利用它们。对最坏的情况你可以小心地将它们写在每个函数的文档中。

+ 集成取消作用域到所有你关心的阻塞 I/O 原语中。如果你正在从头开始构建一个系统，那这个是相当直接的。异步系统在这方面有优势，因为将所有东西整合到事件循环中已经迫使你以某种统一的方式来实现所有的 I/O 原语，这是添加统一的取消处理的一个绝佳时机。

#### 同步、单线程的 Python 代码

我们最初动机里的例子涉及到了 `requests`，一个普通的同步库。而上面几乎所有的东西都同样适用于同步或并发的代码。所以我认为尝试在这些经典的同步 Python 代码中应用这些想法是非常有意思的。也许我们可以修复 `requests` 库，这样它就不必为它的 `timeout` 参数说抱歉了。

然而，有一些限制是我们必须接受的：

+ 它会是无处不在的 -- 库必须确保它们只适用“支持超时范围”的阻塞操作。也许从长远来看，我们可以想象这将成为标准库的一部分，并集成到所有的标准原语中。但即使如此，仍然会有第三方扩展库不通过标准库来完成自己的 I/O 操作。另一方面，像 `requests` 这样的库可以谨慎地只适用支持超时范围的库，然后在文档中将自己标记成支持超时范围的（这也许是像 Trio 这样的异步库在超时和取消方面的最大优势：异步本身并没有什么区别，但异步库被迫重新实现了所有的 I/O 原语，以便它们整合到 I/O 循环中。如果你无论如何都要重新实现一切，那么应该很容易能实现一个一致的对取消操作的支持）。

+ 没有像 await 那样的标记来显示哪些操作是可以取消的。这意味着用户将不得不更加小心，并检查每个函数的文档 -- 但这仍然比目前使超时正常工作的工作量要小。

+ Python 的底层同步原语通常只支持由于超时而取消，而不是任意时间，所以我们可能无法提供 `cancel_scope.cancel()` 操作。但这个限制似乎并不太苛刻，因为如果你有一个单线程的同步程序，而这个但线程被卡在某个阻塞调用中，那么谁会去调用 `cancel()` 呢？总结一下：它不可能像 Trio 提供的那样好，但它仍然是非常有用的，而且肯定比我们现在的要好。

如果你觉得这听起来很有趣，[可以看看我实现的概念验证代码](https://github.com/njsmith/deadline-scopes)。

#### asyncio

这篇博文最初的动机之一是于 Yury 讨论我们是否可以将 Trio 的一些改进应用到 asyncio 之上。通过上面的分析来看 asyncio，有几件事让我们眼前一亮：

+ 在隐含的有状态的任意规模的取消令牌的取消作用域模型和 asyncio 目前的面向任务的边缘触发的取消之间存在一些阻抗不匹配（注：impedence mismatch）（然后 `Future` 层又有一个稍微不用的取消模型），所以我们需要一些故事来说明如何将这些融合到一起。或者，也许我们有可能将任务迁移到一个有状态的取消模型？

+ 如果没有托儿所系统，就没有可靠的方法在任务间传播取消，而且有很多不同的操作，有点像产生一个任务，但是在不用的抽象层次上（例如 `loop.call_soon`）。你可以制定一个规则，任何新的任务总是继承他们的 `spawners` 的取消作用域，但我不确定这是否是一个好主意 -- 这需要一些思考。

+ 如果没有一个通用的机制将异常传回堆栈，就没有办法将 `Cancelled` 异常可靠地传回原始作用域。一般来说，asyncio 只是打印和丢弃来自 `Task` 的未处理的异常。也许这样就可以了？

不幸的是，asyncio 处于一个优点棘手的位置，因为它是建立在 Python 中过去十年的异步 I/O 经验基础之上的......然后在这个架构被锁定之后，它又给 Python 添加了新的语法，使所有这些经验都失效了。但希望还是有可能适应其中的一些经验 -- 至少要做出一些妥协。

#### 其他语言

如果你在工作中使用了其他语言，我很想听听取消作用域是如何适应这些语言的 -- 如果有的话。例如，对于那些不使用异常的语言，或者缺少 Python `with` 块提供的那种用户可扩展语法的语言，它肯定需要一些调整。

### 现在去修复你的超时缺陷吧！

或者如果你想阅读更多关于 Trio 的信息，[我们有一个人们似乎很喜欢的、友好的教程](https://trio.readthedocs.io/)。

## 译者注

Nathaniel J. Smith 对编程语言领域的很多问题都有非常深刻的理解和洞察，他提出并实践了结构化并发，深刻影响了我的编程理念。本文就 I/O、异步系统中的超时和取消的机制进行了探讨，从相对、绝对超时一步步引申出现代的理念取消令牌，并提出了符合人类直觉的取消作用域。

我认为取消作用域是一个相当棒的抽象，但可惜的是在许多现代编程语言中没有提供相应的机制来帮助实现取消作用域，例如 C++/Rust/Java 等，甚至于在 Java 中目前应用取消令牌也不太现实。另外，虽然文中也提到了 Go 语言，但在 Go 中因为没有 goroutine-local 存储（官方也不打算支持），我们也无法实现取消作用域。

就我个人理解，取消令牌在小心地使用下（遵循一些约束，例如最简单的是到处传递它），已经大大降低了程序的复杂度、提升了鲁棒性。在 Go 比较新设计的接口中，涉及到异步任务、I/O 操作的函数/方法基本都会接受一个 `Context` 对象来进行操作的取消，你最好也这么做🤓。

当然上文提到的问题还是存在的，以 Go 为例子，[保持早期版本兼容性导致了 SPDY 代码对取消处理不好引发了 goroutine 泄露问题](https://github.com/kubernetes/kubernetes/pull/103177)。

另外，像在 Rust 的 async/await 下，由于 Rust 异步原理的限制，取消带来的问题更加的致命（可能会导致非预期的对象析构，典型的例子是 async-iouring）。现在在 async Rust 中应用取消会更为麻烦，你可能需要小心地处理某些任务中取消后的状态。作为一个例子，我在 risinglight 中[基于 `CancellationToken` 实现了可随时取消的查询](https://github.com/risinglightdb/risinglight/pull/467)，其中对于取消后的修改和查询都需要小心地处理事务的状态。

NJS 文章的讲解让我对超时和取消有了更深刻的理解，也加深了我对设计简洁、人类友好 API、代码的精神的向往，希望你也能有所收获！

关于原文有任何探讨或想法，欢迎在评论区留言。