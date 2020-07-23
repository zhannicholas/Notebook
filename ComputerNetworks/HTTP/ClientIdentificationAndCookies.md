# 客户端身份识别与Cookie
HTTP最初是一个匿名、无状态的请求/响应协议。服务器接收客户端请求，处理并回送响应，Web服务器几乎没有什么信息可以用来判断请求来自于哪个用户。通常情况下，Web服务器可能同时和上千个客户端进行通信。现代的Web站点希望能够为不同用户提供个性化体验，因此服务器需要通过某种方式识别出当前对话的用户。常见的用户识别机制有：
* 使用承载有用户身份信息的HTTP首部。
* 追踪客户端的IP地址。
* 通过登录认证来识别用户。
* 使用Fat URL，在URL中嵌入识别信息。
* 使用cookie。

## 承载有用户信息的HTTP首部
常见的用来承载用户信息的HTTP首部有：

| 首部名称      | 首部类型            | 描述                                 |
| ------------- | ------------------- | ------------------------------------ |
| From          | Request             | 用户的Email地址                      |
| User-Agent    | Request             | 用户的浏览器                         |
| Referer       | Request             | 用户是从这个页面上面的链接跳转过来的 |
| Authorization | Request             | 用户名和密码                         |
| Client-ip     | Extension (Request) | 客户端IP地址                         |
| X-Forward-For | Extension (Request) | 客户端IP地址                         |
| Cookie        | Extension (Request) | 服务端生成的ID标签                   |

`From`首部包含了用户的E-mail地址，这在理想情况下可以用来识别用户。但某些服务器可能会收集这些E-mail地址，用于散发垃圾邮件，所以很少有浏览器会发送`From`首部。`From`首部一般是由向机器人或蜘蛛这样的自动化程序发送的。

`User-Agent`可以将用户浏览器的相关信息发送给服务器。这些信息包括程序的名称、版本，甚至操作系统的相关信息等。例如：
```
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/84.0.4147.89 Safari/537.36
```

`Referer`首部可以告诉服务器用户是从哪个页面过来的。服务器可以通过它更好地理解用户的浏览行为。

`From`、`User-Agent`、`Referer`首部均不足以实现可靠的用户识别。

## 通过客户端IP地址识别用户
早期的Web服务器尝试将客户端IP地址作为一种识别用户的方式。如果每个用户都有不同的IP地址，并且IP地址很少变化的话，这种方案还是可行的。虽然`Client-IP`不是HTTP标准的一部分，但很多代理都会添加这个首部，Web服务器可以通过它找到承载HTTP请求的TCP连接另一端的IP地址。

虽然如此，使用客户端IP地址来识别用户还是存在诸多缺点：
* 客户端IP地址描述的是用户所用机器的IP，而不是用户。当多个用户共享同一台计算机时，就无法区分用户了。
* 很多ISP都会为用户动态分配IP地址。用户的IP地址不固定，识别失效。
* 为了提高安全性和管理稀缺的IP地址资源，很多用户都是通过NAT技术来浏览网络的。这个时候用户的真实IP就被NAT隐藏了，客户端IP成为了NAT设备的IP。
* HTTP代理和网关通常会打开一些新的连接到服务器的TCP连接，这个时候服务器看到的就是代理或网关的地址。有些代理或网关会在特殊的`Client-IP`或`X-Forwarded-For`扩展首部中存放原始的IP地址，但并不是所有的代理都会这么做。

## 通过登录识别用户
Web服务器除了被动地根据IP来猜测用户身份外，还可以要求用户进行认证来显式地询问用户是谁。

为了帮助简化Web站点的登录，HTTP包含了内置了一种将用户信息传送给Web站点的机制，即使用`WWW-Authenticate`和`Authorization`首部。一旦登录，浏览器就会在接下来访问该站点的每个请求中携带登录信息。

如果服务器希望在访问站点内容之前先登录，它会先给浏览器回送一个`401 Login Required`响应状态码并添加`WWW-Authenticate`首部到响应中。浏览器接着弹出一个登录对话框要求用户输入用户名和密码。用户输入用户名和密码之后，浏览器就会添加一个`Authorization`首部（携带加密后的用户名和密码）并重发原来的请求，服务器将利用它们对用户进行认证。通常，浏览器会保存用户输入的信息，并在往后需要使用用户名和密码的时候自动发送出去。这样一来，只需要登录一次，就可以在整个会话期间维持用户的身份信息了。下图展示了这个过程：

![Registering username using HTTP authentication headers](images/registering-username-using-HTTP-authentication-headers.png)

## Fat URL
有些Web站点会为每个用户生成特定版本的URL，从而追踪用户的身份。通常的做法是：扩展原来的URL，往其中加入一些状态信息。当用户浏览站点时，Web服务器会动态生成一些超链接，继续维护URL中的状态信息。这些扩展后的包含了用户状态信息的URL就被称为**Fat URL**。

虽然Fat URL能够识别用户的身份，但它也存在不少缺点，例如：
* URL可能会变得又长又丑陋，可能会给用户带来困惑。
* 当包含了用户状态信息的URL被分享给别人时，用户的个人信息也被分享出去了。
* 生成URL可能会给服务器带来额外的负担。

## Cookie
Cookie是一种识别用户，实现持久会话的方式，使用非常广泛，也非常重要。

### Cookie的类型
Cookie可以粗略地分为两类：会话cookie和持久cookie：
* 会话cookie是一种临时cookie，他记录了用户访问站点时的设置和偏好，会在用户退出浏览器时被删除。
* 持久cookie的生存时间比较长，因为它们存储在磁盘上。即使浏览器退出，计算机被重启也依然存在。

会话cookie和持久cookie之间的唯一区别就是它们的过期时间。

### Cookie的工作原理
当用户首次访问Web站点时，Web服务器对用户一无所知。如果Web服务器希望用户再次回来的话，它就会通过`Set-Cookie`首部给用户返回一个独有的cookie。只要用户在以后的访问中带上这个cookie，服务器就可以识别出用户了。cooKie中包含了一个由`name=value`信息构成的任意列表。Cookie中可以包含任何信息，浏览器会记住服务器返回报文首部中`Set-Cookie`的内容并将cookie保存在浏览器的cookie数据库中。当用户在之后再次访问某个站点时，浏览器就会选出该站点分配给用户的cookie并放在`Cookie`首部中传给服务端。

Cookie的基本思想就是让浏览器维护一组服务器特有的信息，每次访问服务器时都将这些信息提供给它。因为浏览器需要负责cookie的存储，所以cookie也称**客户端状态（client-side state）**，而cookie在HTTP规范中的正式名称则是**HTTP state management mechanism**。

### Cookie的属性

#### Domain
产生cookie的服务器可以向`Set-Cookie`首部中添加一个`Domain`属性来告诉浏览器这个cookie是给哪些站点使用的。例如，`Set-Cookie: user="Tom"; domain="example.com"`告诉浏览器将`Cookie: use="Tom"`发给`example.com`下的所有站点。

#### Path
`Path`标识了主机下的那些路径可以接受该Cookie（该路径必须出现在请求URL中）。例如当`Path=/docs`时，`/docs/`、`/docs/Web/`、`/docs/web/HTTP`都会被匹配。

## 参考资料
1. David Gourley, Brian Totty. HTTP: The Definitive Guide. O'Reilly Media, 2002.
2. [HTTP Cookies](http://developer.mozilla.org/zh-CN/docs/Web/HTTP/Cookies).