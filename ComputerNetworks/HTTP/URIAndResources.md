# URI与资源

## Web资源
**Web资源（Web resource）**，或**资源（resource）**是一个非常宽泛的概念，它代表一个**可以被识别**的东西。[WikiPedia](https://en.wikipedia.org/wiki/Web_resource)是这么描述它的：
> A web resource, or simply resource, is any identifiable thing, whether digital, physical, or abstract.

在早期的Web中，资源就是可寻址的静态文档（documents）或文件（files）。随着Web的发展，资源变得越来越宽泛和抽象，现在的资源包括一切**可以被识别**的东西或实体。

Web资源保存在Web服务器上，通过**统一资源标识符（Uniform Resource Identifier, URI）**进行识别。

## URI
> A **Uniform Resource Identifier (URI)** is a string of characters that unambiguously identifies a particular resource. 

[RFC2396](https://tools.ietf.org/html/rfc2396#section-1.1)对组成了URI的三个单词进行了定义：
* Uniform: 规定统一的格式可方便处理多种不同类型的资源，而不用根据上下文 环境来识别资源指定的访问方式。另外，加入新增的协议方案（如 http: 或 ftp:）也更容易。
* Resource: 资源的定义是“可标识的任何东西”。除了文档文件、图像或服务（例 如当天的天气预报）等能够区别于其他类型的，全都可作为资源。另 外，资源不仅可以是单一的，也可以是多数的集合体。
* Identifier: 表示可标识的对象。也称为标识符。

简而言之，URI可以唯一标识并定位Web上的资源，它有URL和URN两种形式。

### URI的语法
为了保证**一致性（uniformity）**，所有的URI都遵循了一个预定义的规则集。虽然如此，URI也可以通过*命名方案（naming scheme）*进行扩展。

URI的通用语法定义如下，它由5部分组成：
```txt
URI = scheme:[//authority]path[?query][#fragment]
authority = [userinfo@]host[:port]
```
上面的语法定义也可以用语法树来表示：

![URI syntax diagram](images/URI-syntax-diagram.png)

URI包括：
* 一个非空的**方案（scheme）**（后面跟着`:`）：scheme描述的是客户端在访问服务器并获取资源时需要使用哪种协议。常见的scheme如：`http`、`ftp`、`mailto`、`https`等。scheme对大小写不敏感，因此`HTTP`与`http`指的是同一个scheme。
* 一个可选的**权限（authority）**部分（以`//`开始），它包括：
    * 一个可选的**用户信息（userinfo）**部分，userinfo包括一个*用户名（username）*和一个可选的*密码（password）*。用户名与密码之间用`:`分隔，即`username:password`，由于安全原因，这一部分通常被弃用。userinfo后面会跟着一个`@`。
    * 一个**主机（host）**部分：表示宿主服务器的主机名或IP地址（IPv4地址必须采用`点分十进制`形式，IPv6地址则必须用`[]`括起来）。
    * 一个可选的**端口（port）**部分，以`:`开头：表示宿主服务器的监听端口，很多scheme都有默认的端口号。
* 一个**路径（path）**部分：表示服务器上资源的本地名，path可能由很多`/`分隔的片段组成。
* 一个可选的**查询（query）**部分：以`?`开头，query的语法并没有明确定义，但它通常是很多键-值对组成，这些键-值对会被某个分隔符分隔。例如：`key1=value1&key2=value2`或`key1=value1;key2=value2`中分别采用了`&`和`;`作为分隔符。
* 一个可选的**片段（fragment）**部分：以`#`开始，通常包括一个fragment标识符。有些资源（比如HTML）会在内部作进一步划分，fragment就可以用来引用部分资源或资源的一个片段。当客户端向服务器请求内容时，fragment并不会被发送给服务器，因为fragment是给客户端内部使用的。例如：浏览器从服务器拿到整个资源后，可以将滚动页面指定的fragment部分。

#### 例子
下面是一些URI的例子：
```txt
ftp://ftp.is.co.za/rfc/rfc1808.txt
https://tools.ietf.org/html/rfc3986#section-3.5
ldap://[2001:db8::7]/c=GB?objectClass?one
mailto:John.Doe@example.com
news:comp.infosystems.www.servers.unix
tel:+1-816-555-1212
telnet://192.0.2.16:80/
urn:oasis:names:specification:docbook:dtd:xml:4.1.2
```

### URI的使用
当应用引用一个URI时，它们并不总是使用URI语法规则中所定义的完整形式（绝对URL）。很多网络协议和媒体类型都允许使用URI的缩略形式，例如相对URI。一些浏览器还支持URL的“自动扩展”，即用户输入URL的一部分，浏览器会填充URL的剩余部分。

#### URI引用
资源标识符最常见的使用方式就是**URI引用（URI Reference）**。URI引用不是一个URI就是一个相对引用。

##### 相对引用
如果一个URI引用的前缀匹配不上URI语法中定义的scheme（以`:`结尾），那么它就是一个**相对引用（relative reference）**。相对URI是不完整的，要想从相对URI中获得访问资源所需的全部信息，就必须在**基URI（base URI）**的基础上进行解析。相对引用所指向的URI又称**目标URI（target URI）**。

相对引用有几种不同的形式：
* 以两个斜线（`//`）开头的相对引用是被称为*网络路径引用（network-path reference）*，这一类引用很少被使用。
* 以单个斜线（`/`）开头的相对引用被称为*绝对路径应用（absolute-path reference）*。
* 不以斜线开头的相对引用被称为*相对路径应用（relative-path reference）*。若某个路径片段中包含`:`（例如`this:that`），那么这个路径片段不能直接用作相对路径引用中的第一部分，因为它会被误认为是scheme名。这种路径片段前面必须加上`./`（例如`./this:that`）以消除歧义。

##### 绝对URI
和相对引用不同，**绝对URI（Absolute URI）**包含访问资源所必需的所有信息。绝对URI的开头包括访问资源所采用的scheme。

#### 引用解析
为了访问相对引用指向的资源，我们需要先解析去它对应的绝对URI。解析过程见：[Reference Resolution](https://tools.ietf.org/html/rfc3986#section-5).

### URL
> A **Uniform Resource Locator (URL)**, colloquially termed a web address, is a reference to a web resource that specifies its location on a computer network and a mechanism for retrieving it. 

**统一资源定位符（Uniform Resource Locator, URL）**是资源标识符最常见的形式，描述了特定服务器上特定资源的位置。现在，几乎所有的URI都是URL。

### URN
> A **Uniform Resource Name (URN)** is a Uniform Resource Identifier (URI) that uses the urn scheme. URNs are globally unique persistent identifiers assigned within defined namespaces so they will be available for a long period of time, even after the resource which they identify ceases to exist or becomes unavailable.

**统一资源名（Uniform Resource Name, URN）**是URI的另一种形式，作为特定内容的唯一名字使用，与资源目前的所在地无关。和URL不同，URN是与位置无关的，我们可以四处移动资源，但依然可以通过URN来访问它。例如，不论RFC 2141位于何处（甚至是被复制到多个地方），我们都可以用`urn:ietf:rfc:2141`这个URN来访问它。

## 参考资料
1. [Web resource](https://en.wikipedia.org/wiki/Web_resource).
2. [Uniform Resource Identifier](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier).
3. [Uniform Resource Locator](https://en.wikipedia.org/wiki/URL).
4. [Uniform Resource Name](https://en.wikipedia.org/wiki/Uniform_Resource_Name).
5. [RFC 3986](https://tools.ietf.org/html/rfc3986).
6. David Gourley, Brian Totty. HTTP: The Definitive Guide. O'Reilly Media, 2002.
7. 【日】上野 宣, 图解HTTP, 人民邮电出版社, 2014.