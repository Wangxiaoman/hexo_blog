title: Socket与webSocket
date: 2015-11-10 19:25:49
tags: [Socket]
category: 技术
---


## Socket与webSocket

### 1、概述
webSocket是为了满足web端日益增加的实时通信功能而产生的，传统的方式是采用http进行不停的轮询（这种方式既浪费带宽，又消耗服务器的资源）。

WebSocket是HTML5开始提供的一种浏览器与服务器间进行全双工通讯的网络技术。 WebSocket通信协议于2011年被IETF定为标准RFC 6455 ，WebSocketAPI被W3C定为标准。 在WebSocket API中， 浏览器和服务器只需要要做一个握手的动作，然后，浏览器和服务器之间就形成了一条快速通道。 两者之间就直接可以数据互相传送。

<!--more-->

### 2、OSI 和 TCP/IP
TCP/IP 协议和 OSI 模型的基本内容(需要复习一下)。
在这里，我们只需要知道，HTTP、WebSocket 等协议都是处于 OSI 模型的最高层：应用层。
而IP协议工作在网络层（第3层），TCP协议工作在传输层（第4层）。

### 3、WebSocket、HTTP与TCP
HTTP和WebSocket等应用层协议都是基于TCP来传输数据的，都是对TCP的封装。那么大家的链接和断开都需要遵守TCP的握手协议。对于WebSocket来说，它依赖HTTP进行了一次握手，成功之后，就直接通过TCP进行数据传输，与HTTP无关了。

HTTP的生命周期：
HTTP的生命周期用Request来界定，一个Request就有一个Response，在HTTP1.0中代表着一次请求结束。
在HTTP1.1中，增加了keep-alive的状态，在一个连接中，可以发送多个Request，也可以接受多个Response，但是Request和Response是一一对应的。

WebSocket是基于HTTP的，利用了HTTP的协议完成了一部分握手。

看个典型的Websocket握手

	GET /chat HTTP/1.1
	Host: server.example.com

	Upgrade: websocket
	Connection: Upgrade
	Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
	Sec-WebSocket-Protocol: chat, superchat
	Sec-WebSocket-Version: 13

	Origin: http://example.com
中间部分的几项标识出这个协议是WebSocket协议，不是HTTP。
Sec-WebSocket-key是浏览器随机生成的，Base64的encode值；
Sec-WebSocket-Protocol是用户自定义的字符串，区分不同的服务需要的协议；
Sec-WebSocket-Version就表式版本号。

服务器接受到请求会返回如下信息，表式成功建立连接。

	HTTP/1.1 101 Switching Protocols
	Upgrade: websocket
	Connection: Upgrade
	Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
	Sec-WebSocket-Protocol: chat

Sec-WebSocket-Accept是指服务器确认，并且加密之后Sec-WebSocket-Key；
Sec-WebSocket-Protocol是指最终使用的协议。

他解决了HTTP的这几个难题。
首先，被动性，当服务器完成协议升级后（HTTP->Websocket），服务端就可以主动推送信息给客户端啦。所以上面的情景可以做如下修改。

客户端：啦啦啦，我要建立Websocket协议，需要的服务：chat，Websocket协议版本：17（HTTP Request）
服务端：ok，确认，已升级为Websocket协议（HTTP Protocols Switched）
客户端：麻烦你有信息的时候推送给我噢。。
服务端：ok，有的时候会告诉你的。
服务端：balabalabalabala
服务端：balabalabalabala
服务端：哈哈哈哈哈啊哈哈哈哈服务端：笑死我了哈哈哈哈哈哈哈

就变成了这样，只需要经过一次HTTP请求，就可以做到源源不断的信息传送了。（在程序设计中，这种设计叫做回调，即：你有信息了再来通知我，而不是我傻乎乎的每次跑来问你）这样的协议解决了上面同步有延迟，而且还非常消耗资源的这种情况。

那么为什么他会解决服务器上消耗资源的问题呢？
其实我们所用的程序是要经过两层代理的，即HTTP协议在Nginx等服务器的解析下，然后再传送给相应的Handler（tomcat等）来处理。简单地说，我们有一个非常快速的接线员（Nginx），他负责把问题转交给相应的客服（Handler）。本身接线员基本上速度是足够的，但是每次都卡在客服（Handler）了，老有客服处理速度太慢，导致客服不够。
Websocket就解决了这样一个难题，建立后，可以直接跟接线员建立持久连接，有信息的时候客服想办法通知接线员，然后接线员在统一转交给客户。这样就可以解决客服处理速度过慢的问题了。


### 4、Socket与WebSocket
Socket其实并不是一个协议，他工作在OSI会话层（第五层），他其实是一个抽象层（一组为了调用TCP、UDP的一组接口）。

Socket是应用层与TCP/IP协议族通信的中间软件抽象层，它是一组接口。在设计模式中，Socket其实就是一个门面模式，它把复杂的TCP/IP协议族隐藏在Socket接口后面，对用户来说，一组简单的接口就是全部，让Socket去组织数据，以符合指定的协议。

而WebSocket则不同，它是一个完整的应用层协议，包含一套标准的API 。

### 5、HTML5 与 WebSocket
WebSocket API是HTML5标准的一部分， 但这并不代表WebSocket一定要用在HTML中，或者只能在基于浏览器的应用程序中使用。

实际上，许多语言、框架和服务器都提供了 WebSocket 支持，例如：

基于 C 的 libwebsocket.org
基于 Node.js 的 Socket.io
基于 Python 的 ws4py
基于 C++ 的 WebSocket++
Apache 对 WebSocket 的支持： Apache Module mod_proxy_wstunnel
Nginx 对 WebSockets 的支持： NGINX as a WebSockets Proxy 、 NGINX Announces Support for WebSocket Protocol 、WebSocket proxying


### 6、参考：
http://zengrong.net/post/2199.htm
http://www.zhihu.com/question/20215561









