# 将Tomcat源码导入IDEA
> 这里记录了将Tomcat源码导入IDEA并启动容器的过程。

## 准备环境
为了将Tomcat的源码导入IDEA，我们需要先准备好JDK、Intellij IDEA和Apache Ant。不同版本的Tomcat对JDK和Ant的版本要求有所不同，具体的版本要求可以在Tomcat官网的文档里找到，例如：[Tomcat 9对工具的要求](https://tomcat.apache.org/tomcat-9.0-doc/building.html)。

在安装完JDK和Ant之后，我们还需要设置两个环境变量：
* JAVA_HOME：指向JDK的安装目录。
* ANT_HOME：指向Ant的安装目录。

为了使用`java`和`ant`命令，我还需要将`%JAVA_HOME%\bin`和`%ANT_HOME%\bin`添加到Windows系统的PATH环境变量里去，或者将`${JAVA_HOME}/bin`和`${ANT_HOME}/bin`添加到Linux系统的PATH环境变量里去。

## 获取Tomcat源码
我所用的源码是直接从[Github](https://github.com/apache/tomcat)下载的，对应的是master分支。此外，还可以到[Tomcat官网](https://tomcat.apache.org/)下载特定版本的源码。
 
## 构建Tomcat

### 配置构建参数
将Tomcat源代码目录中的`build.properties.default`复制一份为`build.properties`并根据实际情况进行修改。在构建Tomcat的过程中，会有很多的依赖库被下载。默认情况下，这些依赖库将会被下载到：`${user.home}/tomcat-build-libs`中，这个位置可以通过修改配置项`base.path`的值来改变。我没有对这个文件进行修改，一切都是用的默认值。

### 构建Tomcat
首先切换到Tomcat源代码的根目录。我们可以以默认的方式构建Tomcat，只需要执行`ant`命令即可。但是这样构建完之后直接导入IDEA是会有问题的。为了将代码导入IDEA，我们需要执行`ant -buildfile build.xml ide-intellij`。这告诉Ant将`ide-intellij`作为此次构建的目标。我们不妨来看一下相关内容：
```xml
<target name="ide-intellij"
        depends="download-compile, download-test-compile"
        description="Creates project directory .idea for IntelliJ IDEA">

    <copy todir="${tomcat.home}/.idea">
      <fileset dir="${tomcat.home}/res/ide-support/idea"/>
    </copy>

    <echo>IntelliJ IDEA project directory created. Please create PATH VARIABLES for

      ANT_HOME          = ${ant.home}
      TOMCAT_BUILD_LIBS = ${base.path}
    </echo>
</target>
```

命令执行完之后，终端会提醒我们创建`ANT_HOME`和`TOMCAT_BUILD_LIBS`这两个PATH变量。

### 将源码导入IDEA
按照上面的步骤构建完Tomcat之后，我们就可以直接将Tomcat源码导入IDEA了。

若不设置`ANT_HOME`和`TOMCAT_BUILD_LIBS`，直接构建工程的结果就是各种找不到依赖。打开Project Structure -> Project Settings -> Modules -> tomcat -> Dependencies你会发现这里罗列的依赖项都是红色的(IDEA找不到这些依赖)。

打开File -> Settings -> Appearance & Behavior -> Path Variables。添加`ANT_HOME`和`TOMCAT_BUILD_LIBS`两条变量，值分别设置为`ant.home`和`base.path`的实际值。然后重新构建就OK了。

### 启动Tomcat
Tomcat的启动类为`org.apache.catalina.startup.Bootstrap`。直接运行是会报错的。

其实在执行`ant`命令之后，源代码根目录下面的`output`的文件夹下会出现一个名为`build`的文件夹。它的路径就是运行 **`Bootstrap`** 类所需要的`catalina.home`参数。例如，我的机器上`build`的路径为`C:/Code/Apache/tomcat/output/build`。所以将`-Dcatalina.home="C:/Code/Apache/tomcat/output/build"`添加到VM options中之后，就可以正常启动Tomcat了，然后使用浏览器访问`http://127.0.0.1:8080`就能看到那只猫了。

不过你会发现，后台会出现一堆乱码。解决思路可以参考：[编译Tomcat9源码及tomcat乱码问题解决](https://www.shuzhiduo.com/A/pRdBP8XPJn/)。他使用`-Dfile.encoding=UTF8 -Duser.language=en -Duser.region=US`这三个参数绕过了乱码的问题。

