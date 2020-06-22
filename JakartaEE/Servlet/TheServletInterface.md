# Servlet接口
**`Servlet`**接口是Java Servlet API的核心抽象，所有的Servlet都直接或间接地实现它。**`GenericServlet`**和 **`HttpServlet`**就是Java Servlet API中的两个 **`Servlet`**实现类。很多时候，开发人员只需要继承 **`HttpServlet`**并实现他们自己的服务就行了。

## 请求处理方法
**`Servlet`**接口中定义的`service`方法就是用来处理客户端请求的，Servlet容器每将一个请求路由到一个Servlet实例之后，都会调用这个`service`方法。为了处理并发请求，开发者编写的Servlet通常要能够处理在特定时间内`service`方法中有多个线程执行的情况。通常情况下，Web容器就是通过在不同线程上并发地执行`service`方法来将并发请求交给同一个Servlet处理的。

### HTTP请求的处理方法
为了简化HTTP请求的处理过程，抽象类 **`HttpServlet`**在 **`Servlet`**的基础上添加了一些额外的方法。这些方法可以自动的被`service`方法调用，它们是：
* 处理`GET`请求的`doGet`方法；
* 处理`POST`请求的`doPost`方法；
* 处理`PUT`请求的`doPut`方法；
* 处理`DELETE`请求的`doDelete`方法；
* 处理`HEAD`请求的`doHead`方法；
* 处理`OPTIONS`请求的`doOptions`方法；
* 处理`TRACE`请求的`doTrace`方法；

在开发基于HTTP的Servlet时，开发者一般只需要关心`doGet`和`doPost`方法即可。

## Servlet生命周期
Servlet有一个定义非常良好的生命周期，它定义了Servlet是如何被加载并被实例化、如何初始化、如何处理客户端请求以及如何结束服务的。这个生命周期是通过 **`Servlet`**接口的`init`、`service`和`destroy`方法来定义的，所有的Servlet都必须直接或间接实现这几个方法。

### 加载与实例化
Servlet的加载与实例化是由Servlet容器负责的。Servlet的加载与初始化可以发生在Servlet容器启动时，也可以延迟到需要处理请求时才发生。Servlet容器使用Java的类加载机制来加载Servlet，被加载的Servlet可能来自本地文件系统，也可能来自远程文件系统或其它网络服务。当Servlet被加载成功之后，容器就会实例化它以备将来使用。

### 初始化
当Servlet对象被实例化之后，容器必须在它开始处理客户端请求之前将它初始化。初始化过程可以用来从数据库内读取配置数据，执行一次性活动等等。容器通过 **`Servlet`**接口的`init`方法来初始化一个Servlet，`init`方法 接收一个 **`ServletConfig`**接口的实现类，它允许Servlet从Web应用的配置信息中读取name-value形式的初始化参数。此外，**`Servlet`**还可以通过 **`ServletConfig`**来访问 **`ServletConfig`**（它描述了当前Servlet的运行环境）。

在初始化过程中，Servlet实例可能会抛出一个 **`UnavailableException`**或 **`ServletException`**。这时，这个Servlet实例就不能被放到现役服务中，Servlet容器必须把它释放掉。这个时候，`destroy`方法并不会被调用，因为这种情况下Servlet的初始化是不成功的。初始化失败的Servlet在之后的某个时间点还可能再次被初始化。

### 请求处理
在Servlet被正确地初始化之后，Servlet容器就可以用它来处理客户端请求了。每个请求都被表示为一个 **`ServletRequest`**对象，而请求的响应会被表示成一个 **`ServletResponse`**对象。这两个对象都会被传递给 **`Servlet`**接口的`service`方法作为入参。当处理HTTP请求时，容器会使用 **`HttpServletRequest`**和 **`HttpServletResponse`**类型的对象。

当然，容器内处于服务状态的Servlet在整个生存期中一个请求也不处理也是有可能的。

#### 多线程问题
Servlet容器可能将并发的请求移交给Servlet的`service`方法。为了处理这些请求，Servlet开发者必须仔细处理好`service`方法内的多个线程。**`SingleThreadModel`**在Servlet 4.0中已被弃用。

#### 请求处理中的异常
在处理请求时，Servlet可能会抛出 **`ServletException`**或 **`UnavailableException`**。**`ServletException`**表示处理过程中发生了一些错误，此时容器需要采取一些操作来清除请求。而 **`UnavailableException`**表示该Servlet临时或永久不可用。如果不可用是永久的，容器必须将这个Servlet移除，调用它的`destroy`方法并释放它，容器会为所有因此而拒绝的请求返回`SC_NOT_FOUND(404)`。如果不可用是临时的，容器会在一定时间内不再路由相关的请求到这个Servlet上，容器会为所有在这段时间内因此而拒绝的请求返回`SC_SERVICE_UNAVAILABLE(503)`。

#### 异步处理
**`AsyncContext`**表示的是 **`ServletRequest`**创建的异步执行上下文。

待完善。

### 终止服务
当容器决定要移除一个Servlet时，它会调用这个Servlet的`destroy`方法。Servlet可以在这个方法中释放当前占用的资源、保存当前状态到数据库等等。在调用`destroy`方法之前，容器会等待`service`方法内的线程执行结束。当等待时间达到某个阈值时，`destroy`方法也可能被直接调用。在`destroy`方法完成后，容器必须释放这个Servlet，以便GC将其回收。
