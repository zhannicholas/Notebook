# Nested Classes

Java允许我们在一个类中定义另一个类，后者被称为嵌套类（nested class）。嵌套类可以分为两种：
1. 静态内部类（static nested class）
2. 内部类（inner class）

其中内部类又可以分为三种：
1. 非静态成员类（non-static member class）
2. 局部内部类（local class）
3. 匿名内部类（anonymous class）

四种嵌套类的定义如下：
```Java
class OuterClass {
    static class NestedClass{}  // 静态嵌套类
    class MemberClass{}         // 非静态成员类
    void m1() {
        class LocalClass{}      // 局部内部类
        new Thread(new Runnable() { // 匿名内部类
            @Override
            public void run() {
                // nothing
            }
        });
    }
}
```

## 静态内部类
静态内部类指被声明为`static`的内部类，它可以单独实例化，其它的内部类需要在外部类实例化之后才能实例化。静态内部类不能和外部类同名，不能访问外部类的实例变量，只能访问外部类的类变量和类方法（包括被`private`修饰的类变量和类方法）。

## 非静态成员类
非静态成员类不被`static`修饰。它可以自由的访问外部类的属性和方法，无论这些属性和方法是静态的还是非静态的。但是成员类是和外部类实例相绑定的，因此不可以定义静态变量和静态方法。只有在外部类实例化之后，这个成员类才能被实例化。

## 局部内部类
局部内部类是指定义在一个代码块内的类，它的作用范围只限于其所在的代码块。局部内部类和局部变量类似，不能被`private`、`public`、`protected`和`static`关键字修饰，并且只能访问方法中定义为`final`的局部变量。局部类的其它限制和成员类基本相同。

## 匿名内部类
匿名内部类没有类名。不能使用关键字`class`、`extends`、`implements`，没有构造函数。一个匿名类一定是在`new`的后面，并且它必须继承其它类或实现一个接口。成员类的限制也适用于匿名类。