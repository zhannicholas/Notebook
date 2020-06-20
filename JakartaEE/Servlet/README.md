# 什么是Servlet
> A servlet is a Java™ technology-based Web component, managed by a container, that generates dynamic content.

和其它基于Java的组件一样，Servlet也由Java类组成，这些Java类(`.class`文件)会被编译成字节码，字节码随后会被动态地加载到Web服务器中并运行。Servlet通过Servlet容器实现的**request/response模型**来与Web客户端进行交互。

# 什么是Servlet容器
> The servlet container is a part of a Web server or application server that provides the network services over which requests and responses are sent, decodes MIME-based requests, and formats MIME-based responses. A servlet container also contains and manages servlets through their lifecycle.

**容器(Container)**，又称**Servlet引擎(Servlet engine)**。它可以是Web服务器内置的，也可以是Web服务器中的插件。对于所有的Servlet容器而言，除了必须支持的HTTP协议以外，还可能支持向HTTPS这种的基于request/response模型的协议。容器必须实现的HTTP协议版本包括HTTP/1.1和HTTP/2，当支持HTTP/2时，容器必须支持`h2`和`h2c`两种协议标识符，即所有的Servlet容器必须支持**ALPN**。

此外，Servlet容器还可能会在Servlet的执行环境中加入安全限制。例如，一些应用服务器可能会限制对**`Thread`**对象的创建，以避免对容器中的其它组件造成负面影响。

# 一个例子
下面是一个典型的Servlet事件序列：
1. 客户端(比如浏览器)访问Web服务器并发起一个HTTP请求。
2. Web服务器接收该HTTP请求并将其转交给Servlet容器。
3. Servlet容器先根据配置信息决定该要调用哪一个Servlet，然后将表示request和response的对象传递给该Servlet并调用它。
4. Servlet从request对象中获取到远程用户的身份、请求参数和其它相关数据，然后执行处理逻辑，最后生成响应数据(通过response对象)并发回给客户端。
5. 一旦Servlet完成整个请求处理过程，Servlet容器就会刷新响应，然后将控制权归还给Web服务器。

# 参考资料
1. Shing Wai, Chan Ed Burns. <i>Java™ Servlet Specification, Version 4.0</i>. Oracle Corporation, 2017.
