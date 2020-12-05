# 对象克隆

通过**克隆（Clone）**，我们可以快速构建出一个已有对象的副本。

## 浅克隆 VS 深克隆
**浅克隆（Shadow Clone）**或**浅复制（Shallow Copy）**把原对象中成员变量为值类型的属性都复制给克隆对象，把原对象中成员变量为引用类型的引用地址也复制给克隆对象。当原对象中存在引用类型的成员变量时，该变量的地址会被原对象和克隆对象共享。

![Shallow Copy](images/shallow_copy.png)

**深克隆（Deep Clone）**或**深复制（Deep Copy）**将原对象中的所有类型的成员变量（无论是值类型还是引用类型）都复制一份给克隆对象。

![Deep Copy](images/deep_copy.png)

简单来说，浅克隆和深克隆的区别就在于对引用类型的成员变量的复制：前者复制的是引用对象的地址，而后者复制的是引用对象本身。

## Cloneable 接口
在 Java 中，为了实现对象的克隆，我们需要让类实现 `Cloneable` 接口并重写 `Object` 类的 `clone()` 方法。例如：
```Java
@Data
@AllArgsConstructor
public class CloneableName implements Cloneable {
    private String firstName;
    private String lastName;

    @Override
    public CloneableName clone() throws CloneNotSupportedException {
        return (CloneableName) super.clone();
    }
}

@Data
@AllArgsConstructor
public class CloneableUser implements Cloneable {
    private String id;
    private CloneableName name;
    private int age;

    @Override
    public CloneableUser clone() throws CloneNotSupportedException {
        return (CloneableUser) super.clone();
    }
}
```
对于一个类而言，如果我们只重写 `clone()` 方法而不实现 `Cloneable` 接口，调用 `clone()` 方法时就会抛出 `CloneNotSupportedException`。

Java 中的数组本身也是可克隆的，根据 JLS11 数组对象既是可克隆的，又是可序列化的。数组可以看成下面这个类：
```Java
class A<T> implements Cloneable, java.io.Serializable {
    public final int length = X;
    public T[] clone() {
        try {
            return (T[])super.clone();
        } catch (CloneNotSupportedException e) {
            throw new InternalError(e.getMessage());
        }
    }
}
```

## clone() 方法
`clone()` 是一个 native 方法，它在 `Object` 类中定，其详细内容如下：
```Java
/**
     * Creates and returns a copy of this object.  The precise meaning
     * of "copy" may depend on the class of the object. The general
     * intent is that, for any object {@code x}, the expression:
     * <blockquote>
     * <pre>
     * x.clone() != x</pre></blockquote>
     * will be true, and that the expression:
     * <blockquote>
     * <pre>
     * x.clone().getClass() == x.getClass()</pre></blockquote>
     * will be {@code true}, but these are not absolute requirements.
     * While it is typically the case that:
     * <blockquote>
     * <pre>
     * x.clone().equals(x)</pre></blockquote>
     * will be {@code true}, this is not an absolute requirement.
     * <p>
     * By convention, the returned object should be obtained by calling
     * {@code super.clone}.  If a class and all of its superclasses (except
     * {@code Object}) obey this convention, it will be the case that
     * {@code x.clone().getClass() == x.getClass()}.
     * <p>
     * By convention, the object returned by this method should be independent
     * of this object (which is being cloned).  To achieve this independence,
     * it may be necessary to modify one or more fields of the object returned
     * by {@code super.clone} before returning it.  Typically, this means
     * copying any mutable objects that comprise the internal "deep structure"
     * of the object being cloned and replacing the references to these
     * objects with references to the copies.  If a class contains only
     * primitive fields or references to immutable objects, then it is usually
     * the case that no fields in the object returned by {@code super.clone}
     * need to be modified.
     * <p>
     * The method {@code clone} for class {@code Object} performs a
     * specific cloning operation. First, if the class of this object does
     * not implement the interface {@code Cloneable}, then a
     * {@code CloneNotSupportedException} is thrown. Note that all arrays
     * are considered to implement the interface {@code Cloneable} and that
     * the return type of the {@code clone} method of an array type {@code T[]}
     * is {@code T[]} where T is any reference or primitive type.
     * Otherwise, this method creates a new instance of the class of this
     * object and initializes all its fields with exactly the contents of
     * the corresponding fields of this object, as if by assignment; the
     * contents of the fields are not themselves cloned. Thus, this method
     * performs a "shallow copy" of this object, not a "deep copy" operation.
     * <p>
     * The class {@code Object} does not itself implement the interface
     * {@code Cloneable}, so calling the {@code clone} method on an object
     * whose class is {@code Object} will result in throwing an
     * exception at run time.
     *
     * @return     a clone of this instance.
     * @throws  CloneNotSupportedException  if the object's class does not
     *               support the {@code Cloneable} interface. Subclasses
     *               that override the {@code clone} method can also
     *               throw this exception to indicate that an instance cannot
     *               be cloned.
     * @see java.lang.Cloneable
     */
    @HotSpotIntrinsicCandidate
    protected native Object clone() throws CloneNotSupportedException;
```
从源码中可以看出：`Object` 对 `clone()` 有三条约定：
* 对于所有对象来说，表达式 `x.clone() != x` 的值为 `true` ，即克隆对象与原对象不应该同一个对象。
* 对于所有对象来说，表达式 `x.clone().getClass() == x.getClass()` 的值应该为 `true` ，即克隆对象与原对象的类型应该是一样的。
* 对于所有对象来说，表达式 `x.clone().equals(x)` 的值应该为 `true`，即克隆对象与原对象是相等（equal）的。


默认情况下，调用 `Object.clone()` 进行的是浅克隆。例如：
```Java
@Test
public void shallowCopyTest() throws CloneNotSupportedException {
    CloneableUser user1 = new CloneableUser("id1", new CloneableName("fn", "ln"), 20);
    CloneableUser user2 =  user1.clone();
    assertNotSame(user1, user2);
    // shallow copy
    assertSame(user1.getName(), user2.getName());
    user2.getName().setFirstName("nfn");
    assertSame(user1.getName().getFirstName(), user2.getName().getFirstName());

    user2.setId("id2");
    assertNotSame(user1.getId(), user2.getId());
}
```

## 实现深克隆
在Java中，实现深克隆的方式有很多，例如：
* 为所有引用类型都实现克隆方法。
* 使用复制构造函数。
* 通过序列化与反序列化。
* 通过第三方工具实现深克隆，例如 Apache Commons Lang 、 Jackson 、 Gson 、FastJson 等。

### 为引用类型实现克隆方法
这种方法需要为每种类型都实现克隆方法，并且这些类型都要实现 `Cloneable` 接口。例如：
```Java
@Test
public void deepCopyTest_implementsCloneable() throws CloneNotSupportedException {
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    class Name implements Cloneable {
        private String firstName;
        private String lastName;

        @Override
        protected Name clone() throws CloneNotSupportedException {
            return (Name) super.clone();
        }
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    class User implements Cloneable {
        private String id;
        private Name name;
        private int age;

        @Override
        protected User clone() throws CloneNotSupportedException {
            User user = (User) super.clone();
            user.setName(name.clone());
            return user;
        }
    }

    Name newName = new Name("fn", "ln");
    User user = new User("id1", newName, 20);
    User user2 = user.clone();
    user2.getName().setFirstName("nfn");
    assertNotSame(user.getName().getFirstName(), user2.getName().getFirstName());
}
```

### 使用复制构造函数
这种方法不需要实现 `Cloneable` 接口，但是需要为所有的引用类型都定义一个复制构造函数。具体来说，如果构造器的参数为基本数据类型或字符串类型（不可变类型）就直接赋值，否则需要重新 `new` 一个对象。例如：
```Java
@Test
public void deepCopyTest_copyConstructor() {
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    class Name {
        private String firstName;
        private String lastName;

        public Name (Name name) {
            this.firstName = name.getFirstName();
            this.lastName = name.getLastName();
        }
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    class User {
        private String id;
        private Name name;
        private int age;

        public User (User user) {
            this.id = user.getId();
            this.name = new Name(user.getName());
            this.age = user.age;
        }
    }

    Name newName = new Name("fn", "ln");
    User user = new User("id1", newName, 20);
    User user2 = new User(user);
    user2.getName().setFirstName("nfn");
    assertNotSame(user.getName().getFirstName(), user2.getName().getFirstName());
}
```

### 序列化与反序列化
这种方法要求引用类型实现 `Serializable` 接口。思想为：先将要原对象写入到字节流中，然后再从这个字节流中读出刚刚存储的信息，来作为一个新的对象返回，新对象和原对象就不存在任何地址上的共享，这样就实现了深克隆。例如：
```Java
@Test
public void deepCopyTest_serDe() {
    SerializableUser user1 = new SerializableUser("id1", new SerializableName("fn", "ln"), 20);
    try {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(user1);
        oos.close();

        ByteArrayInputStream bais = new ByteArrayInputStream(baos.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bais);
        SerializableUser user2 = (SerializableUser) ois.readObject();
        ois.close();

        assertEquals(user1, user2);
        assertNotSame(user1, user2);
        user2.getName().setFirstName("nfn");
        assertNotSame(user1.getName().getFirstName(), user2.getName().getFirstName());
    } catch (IOException | ClassNotFoundException e) {
        e.printStackTrace();
    }
}
```

### 使用三方工具
Apache Commons Lang 工具包中的 `SerializationUtils` 提供了一个 `clone()` 方法，可以实现对象的深克隆。由于方法内部也是通过序列化与反序列化实现的，因此需要被序列化的类实现 `Serialiable` 接口。例如：
```Java
@Test
public void deepCopy_apacheCommonsLang() {
    SerializableUser user1 = new SerializableUser("id1", new SerializableName("fn", "ln"), 20);
    SerializableUser user2 = SerializationUtils.clone(user1);

    assertEquals(user1, user2);
    assertNotSame(user1, user2);
    user2.getName().setFirstName("nfn");
    assertNotSame(user1.getName().getFirstName(), user2.getName().getFirstName());
}
```

很多 JSON 工具库也可以实现对象的深克隆，它们的核心思想是先将对象转储为 JSON ，然后用 JSON 重新创建出一个全新的对象，新老对象的地址不同，但内容却是相同的。例如 Google 的 Gson 库：
```Java
@Test
public void deepCopy_json() {
    User user1 = new User("id1", new Name("fn", "ln"), 20);
    Gson gson = new Gson();
    User user2 = gson.fromJson(gson.toJson(user1), User.class);

    assertEquals(user1, user2);
    assertNotSame(user1, user2);
    user2.getName().setFirstName("nfn");
    assertNotSame(user1.getName().getFirstName(), user2.getName().getFirstName());
}
```

## 相关代码
相关代码位于 [Github](https://github.com/zhannicholas/java-demos/blob/master/java-lang/src/test/java/ml/zhannicholas/javalang/clone/CloneTest.java)。

## 参考资料
1. [The Java® Language Specification (Java SE 11 Edition)](https://docs.oracle.com/javase/specs/jls/se11/html/jls-10.html#jls-10.7).