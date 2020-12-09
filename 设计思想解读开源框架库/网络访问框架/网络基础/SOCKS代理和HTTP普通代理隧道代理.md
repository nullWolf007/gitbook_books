[TOC]

## SOCKS代理和HTTP普通代理隧道代理

#### 转载

* [网络通信必备基础之Socket通信原则（四）](https://www.cnblogs.com/Android-Alvin/p/12102793.html)

### 一、SOCKS代理

* 全能代理，就像有很多[跳线](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E8%B7%B3%E7%BA%BF%2F101287)的转接板，它只是简单地将一端的系统连接到另外一端。支持多种协议，包括http、ftp请求及其它类型的请求。它分socks 4 和socks 5两种类型，socks 4只支持TCP协议而socks 5支持[TCP/UDP协议](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2FTCP%2FUDP%E5%8D%8F%E8%AE%AE)，还支持各种[身份验证](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E8%BA%AB%E4%BB%BD%E9%AA%8C%E8%AF%81%2F2193233)机制等协议。其标准端口为1080。

#### 1.1 原理

* socks代理相应的采用socks协议的[代理服务器](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E4%BB%A3%E7%90%86%E6%9C%8D%E5%8A%A1%E5%99%A8)就是SOCKS服务器，是一种通用的代理服务器。Socks是个电路级的底层[网关](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E7%BD%91%E5%85%B3)，是DavidKoblas在1990年开发的，此后就一直作为Internet RFC标准的开放标准。Socks不要求应用程序遵循特定的操作系统平台，Socks 代理与[应用层](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E5%BA%94%E7%94%A8%E5%B1%82)代理、 HTTP 层代理不同，Socks代理只是简单地传递数据包，而不必关心是何种应用协议（比如FTP、HTTP和NNTP请求）。所以，Socks代理比其他应用层代理要快得多。它通常绑定在代理服务器的[1080端口](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F1080%E7%AB%AF%E5%8F%A3)上。如果您在[企业网](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E4%BC%81%E4%B8%9A%E7%BD%91)或校园网上，需要透过[防火墙](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E9%98%B2%E7%81%AB%E5%A2%99)或通过代理服务器访问Internet就可能需要使用SOCKS。一般情况下，对于拨号上网用户都不需要使用它。注意，浏览网页时常用的[代理服务器](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E4%BB%A3%E7%90%86%E6%9C%8D%E5%8A%A1%E5%99%A8)通常是专门的[http代理](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2Fhttp%E4%BB%A3%E7%90%86)，它和SOCKS是不同的。因此，您能浏览网页不等于您一定可以通过SOCKS访问Internet。 常用的防火墙，或代理软件都支持SOCKS，但需要其管理员打开这一功能。如果您不确信您是否需要SOCKS或是否有SOCKS可用，请与您的网络管理员联系。为了使用socks，您需要了解一下内容：
  * SOCKS服务器的IP地址
  * SOCKS服务所在的端口
  * 这个SOCKS服务是否需要[用户认证](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E7%94%A8%E6%88%B7%E8%AE%A4%E8%AF%81)？如果需要，您要向您的网络管理员申请一个用户和口令

* 知道了上述信息，您就可以把这些信息填入“网络配置”中，或者在第一次登记时填入，您就可以使用socks代理了。

#### 1.2 实际应用

* 在实际应用中SOCKS代理可以用作为：电子邮件、[新闻组](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E6%96%B0%E9%97%BB%E7%BB%84)软件、网络传呼ICQ、网络聊天MIRC和使用[代理服务器](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E4%BB%A3%E7%90%86%E6%9C%8D%E5%8A%A1%E5%99%A8)打游戏等等各种应用软件当中。

### 二、HTTP普通代理&隧道代理

#### 2.1 普通代理

* 第一种 Web 代理原理特别简单：

* HTTP 客户端向代理发送请求报文，代理服务器需要正确地处理请求和连接（例如正确处理 Connection: keep-alive），同时向服务器发送请求，并将收到的响应转发给客户端。

* 下面这张图片来自于《HTTP 权威指南》，直观地展示了上述行为：

![19956127-fb9cf1346270c2ab.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xOTk1NjEyNy1mYjljZjEzNDYyNzBjMmFiLnBuZw?x-oss-process=image/format,png)

* 假如我通过代理访问 A 网站，对于 A 来说，它会把代理当做客户端，完全察觉不到真正客户端的存在，这实现了隐藏客户端 IP 的目的。当然代理也可以修改 HTTP 请求头部，通过 `X-Forwarded-IP` 这样的自定义头部告诉服务端真正的客户端 IP。但服务器无法验证这个自定义头部真的是由代理添加，还是客户端修改了请求头，所以从 HTTP 头部字段获取 IP 时，需要格外小心。这部分内容可以参考我之前的《[HTTP 请求头中的 X-Forwarded-For](https://links.jianshu.com/go?to=https%3A%2F%2Fimququ.com%2Fpost%2Fx-forwarded-for-header-in-http.html)》这篇文章。

* 给浏览器显式的指定代理，需要手动修改浏览器或操作系统相关设置，或者指定 PAC（Proxy Auto-Configuration，自动配置代理）文件自动设置，还有些浏览器支持 WPAD（Web Proxy Autodiscovery Protocol，Web 代理自动发现协议）。显式指定浏览器代理这种方式一般称之为正向代理，浏览器启用正向代理后，会对 HTTP 请求报文做一些修改，来规避老旧代理服务器的一些问题，这部分内容可以参考我之前的《[Http 请求头中的 Proxy-Connection](https://links.jianshu.com/go?to=https%3A%2F%2Fimququ.com%2Fpost%2Fthe-proxy-connection-header-in-http-request.html)》这篇文章。

* 还有一种情况是访问 A 网站时，实际上访问的是代理，代理收到请求报文后，再向真正提供服务的服务器发起请求，并将响应转发给浏览器。这种情况一般被称之为反向代理，它可以用来隐藏服务器 IP 及端口。一般使用反向代理后，需要通过修改 DNS 让域名解析到代理服务器 IP，这时浏览器无法察觉到真正服务器的存在，当然也就不需要修改配置了。反向代理是 Web 系统最为常见的一种部署方式，例如本博客就是使用 Nginx 的 `proxy_pass` 功能将浏览器请求转发到背后的 Node.js 服务。

* 了解完第一种代理的基本原理后，我们用 Node.js 实现一下它。只包含核心逻辑的代码如下：

```js
var http = require('http');
var net = require('net');
var url = require('url');

function request(cReq, cRes) {
    var u = url.parse(cReq.url);

    var options = {
        hostname : u.hostname, 
        port     : u.port || 80,
        path     : u.path,       
        method     : cReq.method,
        headers     : cReq.headers
    };

    var pReq = http.request(options, function(pRes) {
        cRes.writeHead(pRes.statusCode, pRes.headers);
        pRes.pipe(cRes);
    }).on('error', function(e) {
        cRes.end();
    });

    cReq.pipe(pReq);
}

http.createServer().on('request', request).listen(8888, '0.0.0.0');
```

* 以上代码运行后，会在本地 `8888` 端口开启 HTTP 代理服务，这个服务从请求报文中解析出请求 URL 和其他必要参数，新建到服务端的请求，并把代理收到的请求转发给新建的请求，最后再把服务端响应返回给浏览器。修改浏览器的 HTTP 代理为 `127.0.0.1:8888` 后再访问 HTTP 网站，代理可以正常工作。

* 但是，使用我们这个代理服务后，HTTPS 网站完全无法访问，这是为什么呢？答案很简单，这个代理提供的是 HTTP 服务，根本没办法承载 HTTPS 服务。那么是否把这个代理改为 HTTPS 就可以了呢？显然也不可以，因为这种代理的本质是中间人，而 HTTPS 网站的证书认证机制是中间人劫持的克星。普通的 HTTPS 服务中，服务端不验证客户端的证书，中间人可以作为客户端与服务端成功完成 TLS 握手；但是中间人没有证书私钥，无论如何也无法伪造成服务端跟客户端建立 TLS 连接。当然如果你拥有证书私钥，代理证书对应的 HTTPS 网站当然就没问题了。

* HTTP 抓包神器 Fiddler 的工作原理也是在本地开启 HTTP 代理服务，通过让浏览器流量走这个代理，从而实现显示和修改 HTTP 包的功能。如果要让 Fiddler 解密 HTTPS 包的内容，需要先将它自带的根证书导入到系统受信任的根证书列表中。一旦完成这一步，浏览器就会信任 Fiddler 后续的「伪造证书」，从而在浏览器和 Fiddler、Fiddler 和服务端之间都能成功建立 TLS 连接。而对于 Fiddler 这个节点来说，两端的 TLS 流量都是可以解密的。

* 如果我们不导入根证书，Fiddler 的 HTTP 代理还能代理 HTTPS 流量么？实践证明，不导入根证书，Fiddler 只是无法解密 HTTPS 流量，HTTPS 网站还是可以正常访问。这是如何做到的，这些 HTTPS 流量是否安全呢？这些问题将在下一节揭晓。

#### 2.2 隧道代理

* 第二种 Web 代理的原理也很简单：

* HTTP 客户端通过 CONNECT 方法请求隧道代理创建一条到达任意目的服务器和端口的 TCP 连接，并对客户端和服务器之间的后继数据进行盲转发。

* 下面这张图片同样来自于《HTTP 权威指南》，直观地展示了上述行为：

![19956127-f5db16899e57b88e.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xOTk1NjEyNy1mNWRiMTY4OTllNTdiODhlLnBuZw?x-oss-process=image/format,png)

* 假如我通过代理访问 A 网站，浏览器首先通过 CONNECT 请求，让代理创建一条到 A 网站的 TCP 连接；一旦 TCP 连接建好，代理无脑转发后续流量即可。所以这种代理，理论上适用于任意基于 TCP 的应用层协议，HTTPS 网站使用的 TLS 协议当然也可以。这也是这种代理为什么被称为隧道的原因。对于 HTTPS 来说，客户端透过代理直接跟服务端进行 TLS 握手协商密钥，所以依然是安全的，下图中的抓包信息显示了这种场景：

![19956127-c607d1e9b730236e.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xOTk1NjEyNy1jNjA3ZDFlOWI3MzAyMzZlLnBuZw?x-oss-process=image/format,png)

* 可以看到，浏览器与代理进行 TCP 握手之后，发起了 CONNECT 请求，报文起始行如下：

  * CONNECT imququ.com:443 HTTP/1.1
    对于 CONNECT 请求来说，只是用来让代理创建 TCP 连接，所以只需要提供服务器域名及端口即可，并不需要具体的资源路径。代理收到这样的请求后，需要与服务端建立 TCP 连接，并响应给浏览器这样一个 HTTP 报文：

  * HTTP/1.1 200 Connection Established
    浏览器收到了这个响应报文，就可以认为到服务端的 TCP 连接已经打通，后续直接往这个 TCP 连接写协议数据即可。通过 Wireshark 的 Follow TCP Steam 功能，可以清楚地看到浏览器和代理之间的数据传递：

![19956127-43e1e8ab91c7ab28.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xOTk1NjEyNy00M2UxZThhYjkxYzdhYjI4LnBuZw?x-oss-process=image/format,png)

* 可以看到，浏览器建立到服务端 TCP 连接产生的 HTTP 往返，完全是明文，这也是为什么 CONNECT 请求只需要提供域名和端口：如果发送了完整 URL、Cookie 等信息，会被中间人一览无余，降低了 HTTPS 的安全性。HTTP 代理承载的 HTTPS 流量，应用数据要等到 TLS 握手成功之后通过 Application Data 协议传输，中间节点无法得知用于流量加密的 master-secret，无法解密数据。而 CONNECT 暴露的域名和端口，对于普通的 HTTPS 请求来说，中间人一样可以拿到（IP 和端口很容易拿到，请求的域名可以通过 DNS Query 或者 TLS Client Hello 中的 Server Name Indication 拿到），所以这种方式并没有增加不安全性。

* 了解完原理后，再用 Node.js 实现一个支持 CONNECT 的代理也很简单。核心代码如下：

```js
var http = require('http');
var net = require('net');
var url = require('url');

function connect(cReq, cSock) {
    var u = url.parse('http://' + cReq.url);

    var pSock = net.connect(u.port, u.hostname, function() {
        cSock.write('HTTP/1.1 200 Connection Established\r\n\r\n');
        pSock.pipe(cSock);
    }).on('error', function(e) {
        cSock.end();
    });

    cSock.pipe(pSock);
}

http.createServer().on('connect', connect).listen(8888, '0.0.0.0');
```

* 以上代码运行后，会在本地 8888 端口开启 HTTP 代理服务，这个服务从 CONNECT 请求报文中解析出域名和端口，创建到服务端的 TCP 连接，并和 CONNECT 请求中的 TCP 连接串起来，最后再响应一个 Connection Established 响应。修改浏览器的 HTTP 代理为 127.0.0.1:8888 后再访问 HTTPS 网站，代理可以正常工作。

* 最后，将两种代理的实现代码合二为一，就可以得到全功能的 Proxy 程序了，全部代码在 50 行以内（当然异常什么的基本没考虑，这是我博客代码的一贯风格）：

```js
var http = require('http');
var net = require('net');
var url = require('url');

function request(cReq, cRes) {
    var u = url.parse(cReq.url);

    var options = {
        hostname : u.hostname, 
        port     : u.port || 80,
        path     : u.path,       
        method     : cReq.method,
        headers     : cReq.headers
    };

    var pReq = http.request(options, function(pRes) {
        cRes.writeHead(pRes.statusCode, pRes.headers);
        pRes.pipe(cRes);
    }).on('error', function(e) {
        cRes.end();
    });

    cReq.pipe(pReq);
}

function connect(cReq, cSock) {
    var u = url.parse('http://' + cReq.url);

    var pSock = net.connect(u.port, u.hostname, function() {
        cSock.write('HTTP/1.1 200 Connection Established\r\n\r\n');
        pSock.pipe(cSock);
    }).on('error', function(e) {
        cSock.end();
    });

    cSock.pipe(pSock);
}

http.createServer()
    .on('request', request)
    .on('connect', connect)
    .listen(8888, '0.0.0.0');
```

* 需要注意的是，大部分浏览器显式配置了代理之后，只会让 HTTPS 网站走隧道代理，这是因为建立隧道需要耗费一次往返，能不用就尽量不用。但这并不代表 HTTP 请求不能走隧道代理，我们用 Node.js 写段程序验证下（先运行前面的代理服务）：

```js
var http = require('http');

var options = {
    hostname : '127.0.0.1',
    port     : 8888,
    path     : 'imququ.com:80',
    method     : 'CONNECT'
};

var req = http.request(options);

req.on('connect', function(res, socket) {
    socket.write('GET / HTTP/1.1\r\n' +
                 'Host: imququ.com\r\n' +
                 'Connection: Close\r\n' +
                 '\r\n');

    socket.on('data', function(chunk) {
        console.log(chunk.toString());
    });

    socket.on('end', function() {
        console.log('socket end.');
    });
});

req.end();
```

* 这段代码运行完，结果如下：

```js
HTTP/1.1 301 Moved Permanently
Server: nginx
Date: Thu, 19 Nov 2015 15:57:47 GMT
Content-Type: text/html
Content-Length: 178
Connection: close
Location: https://imququ.com/

<html>
<head><title>301 Moved Permanently</title></head>
<body bgcolor="white">
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx</center>
</body>
</html>

socket end.
```

* 可以看到，通过 CONNECT 让代理打开到目标服务器的 TCP 连接，用来承载 HTTP 流量也是完全没问题的。

* 最后，HTTP 的认证机制可以跟代理配合使用，使得必须输入正确的用户名和密码才能使用代理，这部分内容比较简单，这里略过。