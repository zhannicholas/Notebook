# HTTP认证
Web连接着不计其数的资源，但并不是所有的资源都是可以随意访问的。某些资源只对部分特定的用户开放，为了达到这个目的，需要对用户进行**认证（authentication）**。通过认证，服务器可以知道用户是谁，进而判定用户可以访问哪些资源。简而言之，认证就是向服务器证明你是谁，而**授权（authorization）**就是服务器授予用户访问特定资源的权限。

HTTP1.1使用的认证方式如下：
* BASIC认证（基本认证）
* DIGEST认证（摘要认证）
* SSL客户端认证
* FormBase认证（表单认证）
此外，还有NTLM、Kerberos、Windows Live ID等认证方式。

## HTTP访问认证框架
HTTP提供了一个简单的**质询-响应（challenge-response）**认证机制，服务器可以用它来对客户端请求进行质询，要求用户提供认证信息。下图展示了HTTP的认证模型：

![Simplified challenge/response authentication](images/simplified-challenge-response-authentication.png)

HTTP认证模型由4个步骤组成：
1. **请求（Request）**：客户端发送请求。
2. **质询（Challenge）**：当客户端请求的资源需要认证时，服务器会以`401 Unauthorized`状态码拒绝请求，同时在响应报文中添加`WWW-Authenticate`首部字段。`WWW-Authenticate`首部字段会对保护区域进行描述并指定认证算法。
3. **授权（Authorization）**：客户端重新发起请求，但这次会给请求报文附加一个`Authorization`首部。`Authorization`首部说明了认证算法、用户名和密码。
4. **成功（Success）**：如果授权成功，服务器就会返回客户端所请求的内容。有些授权算法会在`Authentication-Info`首部中返回一些与授权会话相关的附加信息。

HTTP允许服务器给不同的资源指定不同的访问权限，这是通过安全域来实现的。Web服务器会将受保护的文档组织成一个**安全域（security realm）**，每个安全域都可以有不同的授权用户。

## BASIC认证
BASIC认证早在HTTP/1.0规范中就被提出来了，后来被移到了RFC2617中。现在仍然有一部分网站使用这种传统的认证方式。BASIC认证中会使用`WWW-Authenticate`和`Authorization`两个首部，但并不会使用`Authentication-Info`首部：
* 质询（服务器发往客户端）：网站的不同部分可能使用不同的密码，`Realm`是一个字符串，告诉用户该使用哪个账号和密码。首部格式为：`WWW-Authenticate: Basic realm=quoted-realm`。
* 响应（客户端发往服务端）：客户端用`:`将用户名和密码连接起来，然后用Base-64进行编码。首部格式为：`Authorization: Basic base64-username-and-password`。

### BASIC认证的步骤
下面是BASIC认证的一个例子：

![Basic authentication example](images/basic-authentication-example.png)

BASIC认证的步骤如下：
1. 客户端请求的资源需要BASIC认证的资源。
2. 服务器会随状态码`401 Authorization Required`返回带`WWW-Authenticate`首部字段的响应，该字段内包含了认证的方式（`BASIC`）和请求资源所属安全域（`realm="xxx"`）。
3. 客户端向用户询问用户名和密码，然后用`:`连接二者，再进行Base-64编码处理。编码结果会放在`Authorization`首部中回送给服务器。
4. 服务器对用户名和密码进行解码，验证它们的正确性。如果验证通过，则返回一条包含请求资源的响应。


### 代理认证
位于客户端和服务器之间的代理也可以实现认证功能。代理认证的步骤和BASIC认证的身份验证步骤相同，但首部和状态码都有所不同。下表列出了Web服务器和代理在认证中使用的状态码和首部的差异：

| Web Server                    | Proxy Server                  |
| ----------------------------- | ----------------------------- |
| Unauthorized status code: 401 | Unauthorized status code: 407 |
| WWW-Authenticate              | Proxy-Authenticate            |
| Authorization                 | Proxy-Authorization           |
| Authentication-Info           | Proxy-Authentication-Info     |

### BASIC认证的缺陷
BASIC认证虽然简单便捷，但并不安全。只能用它来防止非恶意用户的无意间访问，或配合SSL等加密技术一起使用。BASIC认证存在一些缺陷：
* 通过网络发送用户名和密码。虽然用户名和密码经过了Base-64编码，但可以轻松解码。
* 攻击者可以嗅探到用户名和密码，然后实施重放攻击。
* 缺乏针对代理或中间人等中间节点的防护措施。
* 服务器很容易被假冒。

## DIGEST认证
BASIC认证虽然简单，但极不安全：用户名和密码是明文传输的，报文也容易被篡改，保证安全的唯一方式就是配合SSL进行BASIC认证。由于BASIC认证无法达到大多数Web网站期望的安全等级，它并不常用。为了弥补BASIC认证的不足，HTTP/1.1引入了DIGEST认证。DIGEST认证同样采用了质询/响应的方式，但不会发送明文密码，因此更加安全。

### DIGEST认证步骤

由于DIGEST认证和BASIC认证都采用了HTTP的质询/响应框架，所以DIGEST认证的步骤与BASIC认证的步骤基本相同。下图展示了二者再语法上的差异：

![Basic authentication vesus digest authentication](images/basic-authentication-vesus-digest-authentication.png)

DIGEST认证的步骤如下：
1. 客户端请求受保护的资源。
2.  服务器会随状态码`401 Authorization Required`返回带`WWW-Authenticate`首部字段的响应，该字段内包含了认证的方式（`DIGEST`）、对应的保护域（`realm`）、临时质询码（`nonce`，一个随机数）等。`realm`和`nonce`这两个字段是必须要有的，客户端需要依靠向服务端回送这两个值进行认证。
3. 客户端再`Authorization`首部中返回DIGEST认证所必须的信息，包括`username`、`realm`、`nonce`、`uri`、`response`的字段信息等。其中`realm`和`nonce`就是上一步中从服务器响应中收到的字段。
4. 服务器对客户端提供的信息进行验证。若验证通过，则返回受保护的资源。


## 参考资料
1. David Gourley, Brian Totty. HTTP: The Definitive Guide. O'Reilly Media, 2002.
2. [RFC2617](https://tools.ietf.org/html/rfc2617).
