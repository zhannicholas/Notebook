# 反射的基本使用

## 获取`Class`对象
Java中共有两种类型：**引用类型**和 **基本类型**。引用类型包括：类、数组(类和数组都继承自 **`java.lang.Object`**)和接口。基本类型包括：`boolean`、`byte`、`short`、`char`、`int`、`long`、`float`、`double`。
当Java程序运行时，JVM会为每种类型实例化一个<u>不可变的</u> **`java.lang.Class`**的实例，实例的很多数据都来自对应的 **`.class`**文件。**`java.lang.Class`**是反射API的入口，它不仅提供了获取及修改对象内部信息方法，还提供了世袭化对象的途径。有三种方式可以获取到 **`Class`**实例的引用：直接获取、使用 **`Object`**类的`getClass()`方法和使用 **`Class`**类的静态`forName(String className)`方法。

### 直接获取

对于 **`A`**类，我们可以直接使用 **`A.class`**来获取它对应的 **`Class`**对象，这也适用于基本类型。对于包装类，其内部的静态`final`变量 **`TYPE`**与对应基本类型的 **`Class`**对象是同一个。例如：
```Java
Class<Object> objectClass = Object.class;
System.out.println(int.class == Integer.TYPE);  // true
System.out.println(int.class == Integer.class); // false
System.out.println(void.class == Void.TYPE);    // true
```

### 使用 **`Object`**类的`getClass()`方法
**`Object`**类有一个`getClass()`方法，它返回对象实例引用的 **`Class`**对象：
```Java
Class<?> objectClass = new Object().getClass();
```

### 使用 **`Class`**类的静态`forName(String className)`方法
**`Class`**类的静态`forName(String className)`方法会尝试返回虚拟机内`className`对应的 **`Class`**类对象的引用。例如：
```Java
Class<?> objectClass = Class.forName("java.lang.Object");
```

不管采取何种方式获取一个给定类型的 **`Class`**对象引用，返回的引用(如果存在的话)都是同一个。例如：
```Java
System.out.println(Object.class == new Object().getClass());    // true
System.out.println(Object.class == Class.forName("java.lang.Object"));  // true
System.out.println(new Object().getClass() == Class.forName("java.lang.Object"));   // true
```

## 获取类信息
我们可以使用反射来获取类的信息，例如包名、类名、类型、访问修饰符等。

现在有一个 **`User`**类，其定义如下：
```Java
package ml.zhannicholas.reflection;

import java.io.Serializable;
import java.util.UUID;

public class User implements Cloneable, Serializable {
    protected String uuid;
    protected String name;
    protected String email;

    public class Address {}

    public User() {}

    public User(String name) {
        this(name, null);
    }


    private User(String name, String email) {
        this.uuid = UUID.randomUUID().toString();
        this.name = name;
        this.email = email;
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }

    // getter and setter methods
}

```
**`ActiveUser`**是 **`User`**类的一个子类：
```Java
public final class ActiveUser extends User {
    public String approach;

    // getter and setter methods
}
```
后面会使用反射获取这些类的信息。

### 获取包信息
简单的获取包名：
```Java
String packageName = userClass.getPackageName();    // ml.zhannicholas.reflection
```
获取 **`Package`**对象：
```Java
Package package = userClass.getPackage();
```
**`Package`**对象包含了包的元数据，比如包名、版本号等。

### 获取类名
**`Class`**类提供了四个获取类名的方法，不同方法的返回值稍有差别：
获取类的`simple name`(不含包名):
```Java
public String getSimpleName();
```
获取类的`name`(含包名):
```Java
public String getName();
```
获取类的规范名(`canonical name`)：
```Java
public String getCanonicalName();
```
获取对应类型的名字(`type name`)：
```Java
public String getTypeName();
```

#### 基本类型和`void`
对于原始类型和`void`，四种方法返回的结果相同：
```Java
int.class.getSimpleName();  // int
int.class.getName();        // int
int.class.getTypeName();    // int
int.class.getCanonicalName();   // int
void.class.getSimpleName();  // void
void.class.getName();        // void
void.class.getTypeName();    // void
void.class.getCanonicalName();   // void
```

#### 对象类型
对于普通的Java对象，`getSimpleName()`返回简单的类名，其它三个方法的返回值和类的规范名相同：
```Java
User.class.getSimpleName(); // User
User.class.getName();   // ml.zhannicholas.reflection.User
User.class.getTypeName();  // ml.zhannicholas.reflection.User  
User.class.getCanonicalName();  // ml.zhannicholas.reflection.User
```

#### 内部类
对于四种内部类，以上几种方法的返回值略有差异。

##### 静态内部类和非静态成员类
对于静态内部类(`nested classes`)和非静态成员类(`inner class`)，`getSimpleName()`返回简单的类名，`getCanonicalName()`返回类的规范名，而`getName()`和`getTypeName()`返回的结果相同，为`$`连接的外部类的规范名和内部类的简单名：
```Java
User.Address.class.getSimpleName(); // Address
User.Address.class.getName();   // ml.zhannicholas.reflection.User$Address
User.Address.class.getTypeName();  // ml.zhannicholas.reflection.User$Address  
User.Address.class.getCanonicalName();  // ml.zhannicholas.reflection.User.Address
```

##### 匿名类
对于匿名类(`anonymouse classes`)，`getCanonicalName()`方法会返回`null`，`getSimpleName()`返回一个空字符串。`getName()`和`getTypeName()`返回的结果由三部分组成，从左到右分别是：外部类的规范名、$和该匿名类在所有匿名类中出现的位置(从1开始)：
```Java
package ml.zhannicholas.reflecton;

public class ReflectionTest {
    public static void main(String[] args) {
        new User(){}.getClass().getSimpleName();    // 空字符串
        new User(){}.getClass().getName();  // ml.zhannicholas.reflection.ReflectionTest$2
        new User(){}.getClass().getTypeName();  // ml.zhannicholas.reflection.ReflectionTest$3
        new User(){}.getClass().getCanonicalName()  // null
    }
}
```

##### 局部类
对于局部类(`local classes`)，`getCanonicalName()`方法会返回`null`，`getSimpleName()`返回简单的类名。`getName()`和`getTypeName()`返回的结果由四部分组成，从左到右分别是：外部类的规范名、$、该局部类在所有同名局部类中出现的位置(从1开始)和局部类的简单名：
```Java
package ml.zhannicholas.reflection;

public class ReflectionTest {
    public static void main(String[] args) {
        testLocalClasses2();
        testLocalClasses1();
    }

    private static void testLocalClasses1() {
        System.out.println("local classes test1: ");
        class LocalClass {}
        System.out.println(LocalClass.class.getName());
        System.out.println(LocalClass.class.getTypeName());
        System.out.println(LocalClass.class.getCanonicalName());
        System.out.println(LocalClass.class.getSimpleName());
    }

    private static void testLocalClasses2() {
        System.out.println("local classes test2: ");
        class LocalClass {}
        System.out.println(LocalClass.class.getName());
        System.out.println(LocalClass.class.getTypeName());
        System.out.println(LocalClass.class.getCanonicalName());
        System.out.println(LocalClass.class.getSimpleName());
    }
}
```
上面这段代码的运行结果为：
```txt
local classes test2: 
ml.zhannicholas.reflection.ReflectionTest$2LocalClass
ml.zhannicholas.reflection.ReflectionTest$2LocalClass
null
LocalClass
local classes test1: 
ml.zhannicholas.reflection.ReflectionTest$1LocalClass
ml.zhannicholas.reflection.ReflectionTest$1LocalClass
null
LocalClass
```

#### 数组
对于数组，`getSimpleName()`返回数组元素类型对应的`simple name`加上数组维度数量的`[]`,`getTypeName()`返回的是数组元素类型对应的`type name`加上数组维度数量的`[]`，
`getCanonicalName()`返回的是数组元素类型对应的`canonical name`加上数组维度数量的`[]`，`getName()`返回的是数组维度数量的`[`加上数组元素类型对应的编码:
```Java
User[].class.getSimpleName();   // User[]
User[].class.getName(); // [Lml.zhannicholas.reflection.User
User[].class.getTypeName();   // ml.zhannicholas.reflection.User[]
User[].class.getCanonicalName();    // ml.zhannicholas.reflection.User[]
```

#### 全限定名与规范名
全限定名(fully qualified name)与规范名(canonical name)的定义和区别见[JLS11](https://docs.oracle.com/javase/specs/jls/se11/html/jls-6.html#jls-6.7)。

### 获取修饰符
通过`getModifiers()`方法可以获取一个类或接口的修饰符，返回的结果会被编码成一个`int`类型的整数，整数的每一位都代表不同的访问修饰符。我们可以 **`Modifier`**类的静态方法`toString()`获得解码后的访问修饰符。
```Java
int modifier = User.class.getModifiers();
Modifier.toString(modifier);    // public
Modifier.toString(Serializable.class.getModifiers());   // public abstract interface
```
通过这里我们可以明确的知道接口类型是`abstract`的。

值得注意的是，这里的访问修饰符指的是<u>JVM</u>中的访问修饰符，即[JVMS11](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-4.html#jvms-4.1)中定义的`access_flags`，共9种，而[JLS11](https://docs.oracle.com/javase/specs/jls/se11/html/jls-8.html#jls-8.1.1)定义的类修饰符只有7种。

### 获取父类
通过`getSuperClass()`方法可以获得一个类的父类对应的 **`Class`**实例。对于基本类型、`void`、接口、**`Object`**类本身，调用这个方法会返回`null`。对于数组，方法会返回 **`Object`**类对应的 **`Class`**实例：
```Java
System.out.println(User.class.getSuperclass()); // class java.lang.Object
System.out.println(ActiveUser[].class.getSuperclass()); // class java.lang.Object
System.out.println(Object.class.getSuperclass());   // null
System.out.println(int.class.getSuperclass());  // null
System.out.println(void.class.getSuperclass()); // null
System.out.println(Serializable.class.getSuperclass()); // null
```

### 获取类直接实现的接口
通过调用`getInterfaces()`方法，我们可以得到类直接实现的接口，或接口直接继承的接口。对于基本类型，方法返回一个长度为0的数组。对于数组类型，方法会返回一个长度为2的数组，对应的元素分别为 **`java.lang.Cloneable`**和 **`java.io.Serializable`**：
```Java
System.out.println(Arrays.toString(User.class.getInterfaces()));    // [interface java.lang.Cloneable, interface java.io.Serializable]
System.out.println(Arrays.toString(ActiveUser.class.getInterfaces()));  // []
System.out.println(Arrays.toString(int.class.getInterfaces())); // []
System.out.println(Arrays.toString(List.class.getInterfaces()));    // [interface java.util.Collection]
System.out.println(Arrays.toString(User[].class.getInterfaces()));  // [interface java.lang.Cloneable, interface java.io.Serializable]
```

### 获取注解
可以通过以下方式获取一个类或接口上的注解：
```Java
Annotation[] annotations = Deprecated.class.getAnnotations();
```

### 获取泛型参数
可以通过`getTypeParameters()`方法获取泛型参数，例如：
```Java
TypeVariable[] typeVariables = Map.class.getTypeParameters();
for (TypeVariable typeVariable: typeVariables) {
        System.out.print(typeVariable.getName() + " ");
}
```
以上代码获取了 **`java.util.Map`**接口内的泛型参数名字，输出结果为：
```txt
K V 
```

### 获取构造函数信息
**`Class`**类提供了4种可以获取构造函数信息的方法：
```Java
public Constructor<?>[] getConstructors();
public Constructor<T> getConstructor(Class<?>... parameterTypes);
public Constructor<?>[] getDeclaredConstructors();
public Constructor<T> getDeclaredConstructor(Class<?>... parameterTypes);
```
`getConstructors()`方法返回当前类中的所有<u>公有</u>构造函数(对于接口类型、基本类型、`void`类型和数组类型，返回的数组长度为0)，`getDeclaredConstructors()`方法返回当前类中所有声明的所有构造函数，如果当前类有默认构造函数，那么默认构造函数也会被添加到返回结果中。
`getConstructor()`和`getDeclaredConstructor()`可以返回具有特定参数列表的构造函数，如果找不到特定参数列表的构造函数，经抛出一个 **`NoSuchMethodException`**异常。
```Java
private static void printConstructor(Constructor<?> constructor) {
     System.out.println(constructor.toGenericString());
}

private static void printConstructors(Constructor<?>[] constructors) {
    for (Constructor<?> constructor: constructors) {
        printConstructor(constructor);
    }
    System.out.println();
}

private static void getConstructors() throws NoSuchMethodException {
    System.out.println("Constructors of ActiveUser.class:");
    printConstructors(ActiveUser.class.getConstructors());
    System.out.println("Constructors of User.class:");
    printConstructors(User.class.getConstructors());
    System.out.println("Declared constructors of ActiveUser.class:");
    printConstructors(ActiveUser.class.getDeclaredConstructors());
    System.out.println("Declared constructors of User.class:");
    printConstructors(User.class.getDeclaredConstructors());

    System.out.println("Declared constructors of User.class with parameters list (String.class):");
    printConstructor(User.class.getDeclaredConstructor(String.class));
}
```
`getConstructors()`方法会尝试去获取 **`User`**和 **`ActiveUser`**的构造函数并输出，运行方法，将会得到以下结果：
```txt
Constructors of ActiveUser.class:
public ml.zhannicholas.reflection.ActiveUser()

Constructors of User.class:
public ml.zhannicholas.reflection.User(java.lang.String)
public ml.zhannicholas.reflection.User()

Declared constructors of ActiveUser.class:
public ml.zhannicholas.reflection.ActiveUser()

Declared constructors of User.class:
private ml.zhannicholas.reflection.User(java.lang.String,java.lang.String)
public ml.zhannicholas.reflection.User(java.lang.String)
public ml.zhannicholas.reflection.User()

Declared constructors of User.class with parameters list (String.class):
public ml.zhannicholas.reflection.User(java.lang.String)
```

### 获取字段信息
**`Class`**类提供了4种可以获取字段信息的方法：
```Java
public Field[] getFields();
public Field getField(String name);
public Field[] getDeclaredFields();
public Field getDeclaredField(String name);
```
`getFields()`方法返回类中所有的<u>公有</u>字段(包括继承来的<u>公有</u>字段)，`getDeclaredFields()`返回类中声明的<u>所有</u>字段(但不包括从继承来的字段)，`getField()`和`getDeclaredField()`会返回具有指定`name`的字段，若不存在`name`对应的字段，将抛出一个 **`NoSuchFieldException`**。
```Java
private static void printField(Field field) {
    System.out.println(field.toGenericString());
}

private static void printFields(Field[] fields) {
    for (Field field: fields) {
        printField(field);
    }
    System.out.println();
}

private static void getFields() throws NoSuchFieldException {
    System.out.println("Fields of User.class:");
    printFields(User.class.getFields());
    System.out.println("Fields of ActiveUser.class");
    printFields(ActiveUser.class.getFields());
    System.out.println("Declared fields of User.class:");
    printFields(User.class.getDeclaredFields());
    System.out.println("Declared fields of ActiveUser.class:");
    printFields(ActiveUser.class.getDeclaredFields());
    System.out.println("Filed [approach] of ActiveUser.class:");
    printField(ActiveUser.class.getField("approach"));
    System.out.println("Declared field [uuid] of User.class:");
    printField(User.class.getDeclaredField("uuid"));
}
```
`getFields()`方法会尝试获取 **`User`**和 **`ActiveUser`**类中满足要求的字段并输出，输出结果如下：
```txt
Fields of User.class:

Fields of ActiveUser.class
public java.lang.String ml.zhannicholas.reflection.ActiveUser.approach

Declared fields of User.class:
protected java.lang.String ml.zhannicholas.reflection.User.uuid
protected java.lang.String ml.zhannicholas.reflection.User.name
protected java.lang.String ml.zhannicholas.reflection.User.email

Declared fields of ActiveUser.class:
public java.lang.String ml.zhannicholas.reflection.ActiveUser.approach

Filed [approach] of ActiveUser.class:
public java.lang.String ml.zhannicholas.reflection.ActiveUser.approach
Declared field [uuid] of User.class:
protected java.lang.String ml.zhannicholas.reflection.User.uuid
```

### 获取方法信息
**`Class`**类提供了4种可以获取方法信息的方法：
```Java
public Method[] getMethods();
public Method getMethod(String name, Class<?>... parameterTypes);
public Method[] getDeclaredMethods();
public Method getDeclaredMethod(String name, Class<?>... parameterTypes);
```
`getMethods()`方法返回类中所有<u>公有</u>方法(包括继承来的)，`getDeclaredMethods()`方法返回类中所有声明的方法(不包括继承来的),`getMethod()`和`getDeclaredMethod()`返回满足指定参数列表并且方法名匹配的方法，可能抛出一个 **`NoSuchMethodException`**。这四个方法不会获取类中的构造函数。
```Java
private static void printMethod(Method method) {
    System.out.println(method.toGenericString());
}

private static void printMethods(Method[] methods) {
    for (Method method: methods) {
        printMethod(method);
    }
    System.out.println();
}

private static void getMethods() throws NoSuchMethodException {
    System.out.println("Methods of User.class:");
    printMethods(User.class.getMethods());
    System.out.println("Declared method of User.class:");
    printMethods(User.class.getDeclaredMethods());
    System.out.println("Method [equals] without parameter list in User.class:");
    printMethod(User.class.getMethod("equals", Object.class));
    System.out.println("Declared method [getEmail] without parameter list in User.class:");
    printMethod(User.class.getDeclaredMethod("getEmail"));
}
```
这段代码会尝试获取 **`User`**类中满足要求的方法并输出，运行结果为：
```txt
Methods of User.class:
public java.lang.String ml.zhannicholas.reflection.User.getName()
public void ml.zhannicholas.reflection.User.setName(java.lang.String)
public java.lang.String ml.zhannicholas.reflection.User.getEmail()
public void ml.zhannicholas.reflection.User.setEmail(java.lang.String)
public final native void java.lang.Object.wait(long) throws java.lang.InterruptedException
public final void java.lang.Object.wait(long,int) throws java.lang.InterruptedException
public final void java.lang.Object.wait() throws java.lang.InterruptedException
public boolean java.lang.Object.equals(java.lang.Object)
public java.lang.String java.lang.Object.toString()
public native int java.lang.Object.hashCode()
public final native java.lang.Class<?> java.lang.Object.getClass()
public final native void java.lang.Object.notify()
public final native void java.lang.Object.notifyAll()

Declared method of User.class:
protected java.lang.Object ml.zhannicholas.reflection.User.clone() throws java.lang.CloneNotSupportedException
public java.lang.String ml.zhannicholas.reflection.User.getName()
public void ml.zhannicholas.reflection.User.setName(java.lang.String)
public java.lang.String ml.zhannicholas.reflection.User.getEmail()
public void ml.zhannicholas.reflection.User.setEmail(java.lang.String)

Method [equals] without parameter list in User.class:
public boolean java.lang.Object.equals(java.lang.Object)
Declared method [getEmail] without parameter list in User.class:
public java.lang.String ml.zhannicholas.reflection.User.getEmail()
```

## 创建新实例
通过反射，有两种方式可以创建新的实例：
1. 使用`Class.newInstance()`。
2. 使用`Constructor.newInstance()`。

前者在JDK9中已被弃用，因为它会抛出构造函数中的任何异常(不管是受检异常还是非受检异常)，而后者会使用 **`InvocationTargetException`**来包裹构造函数抛出的任何异常。此外，和后者相比，前者只支持无参构造函数并要求这个构造函数是可见的，而后者能够调用所有的构造函数(包括被`private`修饰的构造函数)。前者可以使用`clazz.getDeclaredConstructor().newInstance()`代替。
```Java
System.out.println("Invoke default constructor:");
System.out.println(User.class.getDeclaredConstructor().newInstance());
System.out.println("Invoke public constructor:");
User user1 = User.class.getDeclaredConstructor(String.class).newInstance("Nicholas");
System.out.println("name=" + user1.name + ", email=" + user1.getEmail());
System.out.println("Invoke private constructor:");
Constructor<User> userConstructor = User.class.getDeclaredConstructor(String.class, String.class);
userConstructor.setAccessible(true);
User user2 = userConstructor.newInstance("Nicholas", "nicholas@zhannicholas.ml");
System.out.println("name=" + user2.name + ", email=" + user2.getEmail());
```
上面的代码演示了如何使用反射来调用默认构造函数，公有构造函数以及私有构造函数。在调用私有构造函数时，需要先是该构造函数**可见**，否则会遭遇一个 **`IllegalAccessException`**。运行结果如下：
```txt
nvoke default constructor:
ml.zhannicholas.reflection.User@5594a1b5
Invoke public constructor:
name=Nicholas, email=null
Invoke private constructor:
name=Nicholas, email=nicholas@zhannicholas.ml
```

## 操作数组
**`Class`**类并没有直接提供操作数组的方法，但我们可以通过它提供的`isArray()`方法来判断某个 **`Class`**实例是否代表一个数组。要操作数组，我们可以使用 **`java.lang.reflect.Array`**类，它提供了动态创建和访问数组的静态方法。

**`Array`**类提供了两个创建数组的方法：
```Java
public static Object newInstance(Class<?> componentType, int length);
public static Object newInstance(Class<?> componentType, int... dimensions);
```
第一个方法用来创建一维数组，第二个方法用来创建多维数组。

此外，**`Array`**还提供了获取数组维度(`getLength()`)，设置(`setXXX()`)和读取(`getXXX()`)指定下标值的方法：
```Java
int[] nums = (int[]) Array.newInstance(int.class, 2);
System.out.println(Arrays.toString(nums));
Array.set(nums, 0, 1);
Array.set(nums, 1, 2);
System.out.println("[" + Array.get(nums, 0) + ", " + Array.get(nums, 1) + "]");
int[][] matrix = (int[][]) Array.newInstance(int.class, 2, 2);
System.out.println("Dimension of matrix: " + Array.getLength(matrix));
System.out.println("[" + Arrays.toString(matrix[0]) + ", " + Arrays.toString(matrix[1]) + "]");
Array.set(matrix, 0, new int[]{1,2});
Array.set(matrix, 1, new int[]{3,4});
System.out.println(Arrays.toString((int[]) Array.get(matrix, 0)));
System.out.println(Arrays.toString((int[]) Array.get(matrix, 1)));
```
上面这段代码演示了使用 **`Array`**类进行数组的创建、维度及内容读取，运行结果如下：
```txt
[0, 0]
[1, 2]
Dimension of matrix: 2
[[0, 0], [0, 0]]
[1, 2]
[3, 4]
```

## 调用对象方法
在获取到类的方法信息之后，我们可以通过 **`java.lang.reflect.Method`**的```public Object invoke(Object obj, Object... args);```方法来调用`obj`的中参数列表为`Object... args`的方法并得到方法调用结果。对于静态方法，给第一个参数传递`null`即可。
```Java
public class Calculator {
    public static int add(int a, int b) {return a + b;}
    public int minus(int a, int b) {return a - b;}
}

public class InvokeCalculator {
    public static void main(String[] args) {
        Method add = Calculator.class.getDeclaredMethod("add", int.class, int.class);
        System.out.println(add.invoke(null, 1, 2));
        Method minus = Calculator.class.getDeclaredMethod("minus", int.class, int.class);
        System.out.println(minus.invoke(new Calculator(), 1, 2));
    }
}
```
上面的代码通过反射获取到了 **`Calculator`**的静态方法`add()`和实例方法`minus`，然后通过 **`Method`**类的的`invoke()`方法进行调用，输出结果如下：
```txt
3
-1
```

## 修改字段值
在获取到类的字段信息后，我们可以通过 **`java.lang.reflect.Field`**类提供的`getXXX()`方法来获取实例字段的值，也可以使用`setXXX()`方法来改变实例字段的值：
```Java
User user = new User("Nicholas");
Field name = User.class.getDeclaredField("name");
System.out.println(name.get(user));
name.set(user, "Sarkar");
System.out.println(name.get(user));
```
这段代码先实例化了一个 **`User`**对象，然后通过反射读取`name`字段的值，之后又修改了`name`的值，运行结果如下：
```txt
Nicholas
Sarkar
```

## 参考文献
1. 《The Java Language Specification, Java SE 11 Edition》.
2. 《The Java Virtual Machine Specification, Java SE 11 Edition》.
3. [Trail: The Reflection API](https://docs.oracle.com/javase/tutorial/reflect/).

