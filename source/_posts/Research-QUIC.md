---
title: 「研」 HTTP 和 QUIC
category_bar: true
date: 2023-06-14 23:11:02
tags:
categories: 理论研究
banner_img:
---

QUIC（Quick UDP Internet Connections）是一种基于 UDP 的传输协议，旨在加速 HTTP 通信，同时使其变得更加安全，其最终目的是在 web 上代替 TCP 和 TLS 协议。

<!--more-->

## HTTP 发展史

Hyper Text Transfer Protocol（超文本传输协议），是用于从万维网（WWW：World Wide Web）服务器传输超文本到本地浏览器的传送协议。是互联网上应用最为广泛的一种网络协议。

![HTTP protocol](3.png)

### HTTP/0.9

HTTP 是基于 TCP/IP 协议的应用层协议。它不涉及数据包（packet）传输，主要规定了客户端和服务器之间的通信格式，默认使用 80 端口。 最早版本是 1991 年发布的 0.9 版。该版本极其简单，只有一个命令：GET。

TCP 连接建立后，客户端向服务器请求（request）网页 index.html。协议规定，服务器只能回应 HTML 格式的字符串，不能回应别的格式。服务器发送完毕，就关闭 TCP 连接。

### HTTP/1.0

1996 年 5 月，HTTP/1.0 版本发布，内容大大增加：
* 任何格式的内容都可以发送。这使得互联网不仅可以传输文字，还能传输图像、视频、二进制文件。
* 除了 GET 命令，还引入了 POST 命令和 HEAD 命令，丰富了浏览器与服务器的互动手段。
* HTTP 请求和回应的格式也变了。除了数据部分，每次通信都必须包括头信息（HTTP header），用来描述一些元数据。
* 其他的新增功能还包括状态码（status code）、多字符集支持、多部分发送（multi-part type）、权限（authorization）、缓存（cache）、内容编码（content encoding）等。

HTTP/1.0 的主要缺点：
* 每个 TCP 连接只能发送一个请求。发送数据完毕，连接就关闭，如果还要请求其他资源，就必须再新建一个连接。
* TCP 连接的新建成本很高，因为需要客户端和服务器三次握手，并且开始时发送速率较慢（Slow start）。

### HTTP/1.1

1997 年 1 月，HTTP/1.1 版本发布，只比 1.0 版本晚了半年。它进一步完善了 HTTP 协议：
* **长连接**：HTTP/1.1 支持长连接（PersistentConnection）和请求的流水线（Pipelining）处理，在一个 TCP 连接上可以传送多个 HTTP 请求和响应，减少了建立和关闭连接的消耗和延迟。
* **缓存处理**：HTTP/1.1 引入了更多的缓存控制策略。
* **带宽优化及网络连接的使用**：HTTP/1.0 中存在一些浪费带宽的现象，例如客户端只是需要某个对象的一部分，而服务器却将整个对象送过来了，并且不支持断点续传功能；HTTP/1.1 则在请求头引入了 range 头域，它允许只请求资源的某个部分，这样就方便了开发者自由的选择以便于充分利用带宽和连接。
* **错误通知的管理**：新增了 24 个错误状态响应码，如 409（Conflict）表示请求的资源与资源的当前状态发生冲突；410（Gone）表示服务器上的某个资源被永久性的删除。
* **Host 头处理**：在 HTTP/1.0 中认为每台服务器都绑定一个唯一的 IP 地址，因此，请求消息中的 URL 并没有传递主机名（hostname）。但随着虚拟主机技术的发展，在一台物理服务器上可以存在多个虚拟主机，并且它们共享一个 IP 地址。HTTP/1.1 的请求消息和响应消息都应支持 Host 头域，且请求消息中如果没有 Host 头域会报告一个错误（400 Bad Request）。

HTTP/1.1 的主要缺点：
* 同一个 TCP 连接里面，所有的数据通信是按次序进行的。服务器只有处理完一个回应，才会进行下一个回应。要是前面的回应特别慢，后面就会有许多请求排队等着。这称为**队头堵塞**（Head-of-line blocking）。
* HTTP/1.x 在传输数据时，所有传输的内容都是明文，客户端和服务器端都无法验证对方的身份，这在一定程度上无法保证数据的安全性。
* HTTP/1.x 在使用时，header 里携带的内容过大，在一定程度上增加了传输的成本，并且每次请求 header 基本不怎么变化，尤其在移动端增加用户流量。
* 虽然 HTTP/1.x 支持了 keep-alive，来弥补多次创建连接产生的延迟，但是 keep-alive 使用多了同样会给服务端带来大量的性能压力，因为它在文件被请求之后还保持了不必要的连接很长时间。

### SPDY 协议

2009 年，Google 公开了自行研发的 SPDY 协议，主要解决 HTTP/1.1 效率不高的问题。 这个协议在 Chrome 浏览器上证明可行以后，就被当作 HTTP/2 的基础，主要特性都在 HTTP/2 之中得到继承：

* **降低延迟**：针对 HTTP 高延迟的问题，SPDY 采取了**多路复用**（multiplexing），通过多个请求 stream 共享一个 tcp 连接的方式，降低延迟同时提高了带宽的利用率。
* **请求优先级**。多路复用带来一个新的问题是，在连接共享的基础之上有可能会导致关键请求被阻塞。SPDY 允许给每个 request 设置优先级，这样重要的请求就会优先得到响应。
* **header 压缩**：前面提到 HTTP/1.x 的 header 很多时候都是重复多余的。选择合适的压缩算法可以减小包的大小和数量。
* **加密传输**：基于 HTTPS 的加密协议传输，大大提高了传输数据的可靠性。
* **服务端推送**：采用了 SPDY 的网页，例如网页有一个 sytle.css 的请求，在客户端收到 sytle.css 数据的同时，服务端会将 sytle.js 的文件推送给客户端，当客户端再次尝试获取 sytle.js 时就可以直接从缓存中获取到，不用再发请求了。

![SPDY](1.png)

### HTTP/2.0

HTTP/2 是基于 HTTPS 的，可以说是 SPDY 的升级版（其实原本也是基于 SPDY 设计的），但是 HTTP/2 跟 SPDY 仍有不同的地方：

* HTTP/2 支持明文 HTTP 传输，而 SPDY 强制使用 HTTPS。
* HTTP/2 消息头的压缩算法采用 HPACK，而非 SPDY 采用的 DEFLATE。

![multiplexing](2.png)

在 Internet 和 HTTP 的发展过程中，HTTP 的底层传输机制基本没有变化。然而，随着移动设备技术的大规模普及，实时应用需求的增加，互联网流量不断增长，HTTP/2 的缺点也越来越明显。

### HTTP/3.0

HTTP/2 的最大特性就是多路复用，但是每个 HTTP 连接都是由 TCP 进行连接建立和传输的，TCP 协议在处理包时有严格的顺序要求。这也就是说，当某个包切分的 stream 由于某些原因丢失后，服务器不会处理其他 stream，而会优先等待客户端发送丢失的 stream。

HTTP/3 使用了一种名为 QUIC 的协议，该协议运行在 UDP 协议之上，而不是 TCP。UDP 允许消息的多向广播，这一特性有助于解决数据包级别的**队头阻塞**问题。

此外，QUIC 重新设计了客户端和服务器之间握手的方式，减少了与建立重复连接相关的延迟。

![重复连接时的 handshake](5.png)

HTTP/3 在语法和语义上与 HTTP/2 相似，遵循着相同的请求和响应消息交换顺序，其数据格式包含方法、报头、状态码和正文。HTTP/3 的显着差异在于 UDP 之上协议层的堆叠顺序，如下图所示。

![比较](6.png)

## TCP 的局限性

* **TCP 的队头阻塞**

    如果序列号较低的数据段尚未到达/接收，即使序列号较高的段已经到达/接收，TCP 的接收器滑动窗口也不会前进。这会导致 TCP 流暂时挂起甚至关闭，这个问题称为 TCP 流的**队头阻塞**。队头阻塞主要是 TCP 协议的可靠性机制引入的。TCP 使用序列号来标识数据的顺序，数据必须按照顺序处理，如果前面的数据丢失，后面的数据就算到达了也不会通知应用层来处理。

![HoL](7.png)

* **TCP 不支持流级多路复用**

    虽然 TCP 允许与应用层之间的多个逻辑连接，但它不允许在单个 TCP 流中多路复用数据包。使用 HTTP/2，浏览器只能打开一个与服务器的 TCP 连接。它使用同一个连接来请求多个对象，例如 CSS、JavaScript 和其他文件。在接收这些对象时，TCP将同一流中的所有对象序列化。因此，它不知道 TCP 段的对象级分区。

* **TCP 握手导致连接成本**

    在进行连接握手时，TCP 交换一系列消息，如果与已知主机建立连接，会有冗余的消息交换序列。对于很多短连接场景，这样的握手延迟影响很大，且无法消除。

## QUIC

在 TCP 中，TCP 为了保证数据的可靠性，使用了序号 + 确认号机制来实现，一旦带有 synchronize sequence number 的包发送到服务器，服务器都会在一定时间内进行响应，如果过了这段时间没有响应，客户端就会重传这个包，直到服务器收到数据包并作出响应为止。

虽然 QUIC 没有使用 TCP 协议，但是它也保证了可靠性，QUIC 实现可靠性的机制是使用了 Packet Number，这个序列号可以认为是 syn 的替代者，这个序列号也是递增的。与 syn 所不同的是，不管服务器有没有接收到数据包，这个 Packet Number 都会 +1，而 syn 是只有服务器发送 ack 响应之后，syn 才会 +1。

![序列号](9.png)

QUIC 引入了一个 stream offset 的概念，一个 stream 可以传输多个 stream offset，每个 stream offset 其实就是一个 PN 标识的数据，即使某个 PN 标识的数据丢失，PN+1 后，它重传的仍旧是 PN 所标识的数据，等到所有 PN 标识的数据发送到服务器，就会进行重组，以此来保证数据可靠性。到达服务器的 stream offset 会按照顺序进行组装，这同时也保证了数据的顺序性。

![stream offset](10.png)

QUIC 相比于 HTTP/2 具有下面这些优势：

* **选择 UDP 作为底层传输层协议**

    如果还在TCP 之上的建立新的传输机制，将仍继承前面所说的TCP限制。QUIC 是在用户级别构建的，它不需要在每次协议升级时更改内核，从而更易采用。

* **流复用和流量控制**

    QUIC 引入了在单个连接上多路复用流的概念。QUIC 实现了单独的、逐流的流量控制。UDP 本身没有建立连接这个概念，QUIC 使用的 stream 之间是相互隔离的，不会阻塞其他 stream 数据的处理，所以使用 UDP 并不会造成队头阻塞。

![使用 UDP，没有包级的队头阻塞](8.png)

* **灵活的拥塞控制**

    TCP 有严格的拥塞控制机制。每次 TCP 协议检测到拥塞时，它都会将拥塞窗口的大小减半。QUIC 的拥塞控制更加灵活，从内核空间到用户空间，可以更有效地利用可用网络带宽，从而获得更好的流量吞吐量。

* **更好的错误处理**

    QUIC 提议使用增强的丢失恢复机制和前向纠错来处理错误的数据包，尤其是对于在传输中容易出现高错误率的缓慢的无线网络。

* **更快地握手**

    QUIC 使用与 HTTP/2 相同的 TLS 模块来实现安全连接。然而，与 TCP 不同的是，QUIC 的握手机制经过优化，可以避免在两个已知对等点相互建立通信时进行冗余协议交换。使用 TCP 和 TLS 建立安全连接至少需要两次往返时间（RTT），增加了延迟开销。使用 QUIC，建立第一个加密连接是 1 个 RTT，当会话恢复时，有效负载数据与第一个数据包一起发送，RTT 最少为零。

![QUIC](11.png)

* **语法和语义**

    通过在 QUIC 之上构建基于 HTTP/3 的应用层，可以获得增强传输机制的所有优势，同时保留与 HTTP/2 相同的语法和语义。但是 HTTP/2 不能直接与 QUIC 集成，因为从应用到传输的底层帧映射是不兼容的。因此，IETF 的 HTTP 工作组建议将 HTTP/3 作为新的 HTTP 版本，并根据 QUIC 协议的帧格式要求修改其帧映射。

* **压缩**

    HTTP/3 还使用了一种称为 QPACK 的新报头压缩机制，是对 HTTP/2 中使用的 HPACK 的修改。在 QPACK 下，HTTP 报头可以在不同的 QUIC 流中乱序到达。QPACK 使用了一种查找表机制来对报头进行编码和解码。

## 总结

 QUIC 作为一个新的传输层协议，它在设计上针对 TCP 的不足进行了很多优化。它提供的多路传输、快速握手等新特性使得它和 TCP 相比在理论上可以获得更低的数据传输延时。现有测量工作表明，QUIC 在大部分情况下的确能比 TCP 达到更低的传输延时，但是仍然有部分情况下 QUIC 的表现不如 TCP。这些 QUIC 性能表现较差的场景往往是拥塞算法的选择、服务器部署等外部因素造成的，而非 QUIC 本身的设计缺陷。因此，QUIC 的软件实现仍然有很大的进步空间。

 另外关于队头阻塞的问题，本文描述过于简略，有兴趣可以学习[这篇文章](https://www.cnblogs.com/sexintercourse/p/16839900.html)。