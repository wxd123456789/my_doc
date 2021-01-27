@[toc](目录)

# 基础

## 关键字

**static**

static的主要意义是在于创建独立于具体对象的域变量或者方法。**以致于即使没有创建对象，也能使用属性和调用方法**！

static关键字还有一个比较关键的作用就是 **用来形成静态代码块以优化程序性能**。static块可以置于类中的任何地方，类中可以有多个static块。在类初次被加载的时候，会按照static块的顺序来执行每个static块，并且只会执行一次。为什么说static块可以用来优化程序性能，是因为它的特性:只会在类加载的时候执行一次。因此，很多时候会将一些只需要进行一次的初始化操作都放在static代码块中进行。

因为static是被类的实例对象所共享，因此如果**某个成员变量是被所有对象所共享的，那么这个成员变量就应该定义为静态变量**。因此比较常见的static应用场景有：

> 1、修饰成员变量 2、修饰成员方法 3、静态代码块 4、修饰类【只能修饰内部类也就是静态内部类】 5、静态导包

**transient**

表明该字段不支持序列化

**native**

JNI:Java Native Interface为JAVA本地调用，允许java调用其他语言。例如compareAndSwapInt就是借助C来调用CPU底层指令实现的。

## 内部类

在Java中，可以将一个类的定义放在另外一个类的定义内部，这就是**内部类**。内部类本身就是类的一个属性，与其他属性定义方式一致。内部类可以分为四种：**成员内部类、局部内部类、匿名内部类和静态内部类**。

内部类的优点：

- 一个内部类对象可以访问创建它的外部类对象的内容，包括私有数据！
- 内部类不为同一包的其他类所见，具有很好的封装性；
- 内部类有效实现了“多重继承”，优化 java 单继承的缺陷。
- 匿名内部类可以很方便的定义回调。

内部类的应用场景：

1. 一些多算法场合，比如集合类中
2. 解决一些非面向对象的语句块。
3. 适当使用内部类，使得代码更加灵活和富有扩展性。
4. 当某个类除了它的外部类，不再被其他的类使用时。

### 静态内部类

定义在类内部的静态类，就是静态内部类。

```java
public class Outer {

    private static int radius = 1;

    static class StaticInner {
        public void visit() {
            System.out.println("visit outer static  variable:" + radius);
        }
    }
}
```

静态内部类可以访问外部类所有的静态变量，而不可访问外部类的非静态变量；静态内部类的创建方式，`new 外部类.静态内部类()`，如下：

```java
Outer.StaticInner inner = new Outer.StaticInner();
inner.visit();
```

### 成员内部类

定义在类内部，成员位置上的非静态类，就是成员内部类。

```java
public class Outer {

    private static  int radius = 1;
    private int count =2;
    
     class Inner {
        public void visit() {
            System.out.println("visit outer static  variable:" + radius);
            System.out.println("visit outer   variable:" + count);
        }
    }
}

```

成员内部类可以访问外部类所有的变量和方法，包括静态和非静态，私有和公有。成员内部类依赖于外部类的实例，它的创建方式`外部类实例.new 内部类()`，如下：

```java
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();
inner.visit();
```

### 局部内部类

定义在方法中的内部类，就是局部内部类。

```java
public class Outer {

    private  int out_a = 1;
    private static int STATIC_b = 2;

    public void testFunctionClass(){
        int inner_c =3;
        class Inner {
            private void fun(){
                System.out.println(out_a);
                System.out.println(STATIC_b);
                System.out.println(inner_c);
            }
        }
        Inner  inner = new Inner();
        inner.fun();
    }
    public static void testStaticFunctionClass(){
        int d =3;
        class Inner {
            private void fun(){
                // System.out.println(out_a); 编译错误，定义在静态方法中的局部类不可以访问外部类的实例变量
                System.out.println(STATIC_b);
                System.out.println(d);
            }
        }
        Inner  inner = new Inner();
        inner.fun();
    }
}

```

定义在实例方法中的局部类可以访问外部类的所有变量和方法，定义在静态方法中的局部类只能访问外部类的静态变量和方法。局部内部类的创建方式，在对应方法内，`new 内部类()`，如下：

```java
 public static void testStaticFunctionClass(){
    class Inner {
    }
    Inner  inner = new Inner();
 }

```

### 匿名内部类

匿名内部类就是没有名字的内部类，日常开发中使用的比较多。

```java
public class Outer {

    private void test(final int i) {
        new Service() {
            public void method() {
                for (int j = 0; j < i; j++) {
                    System.out.println("匿名内部类" );
                }
            }
        }.method();
    }
 }
 //匿名内部类必须继承或实现一个已有的接口 
 interface Service{
    void method();
}

```

除了没有名字，匿名内部类还有以下特点：

- 匿名内部类必须继承一个抽象类或者实现一个接口。
- 匿名内部类不能定义任何静态成员和静态方法。
- 当所在的方法的形参需要被匿名内部类使用时，必须声明为 final。
- 匿名内部类不能是抽象的，它必须要实现继承的类或者实现的接口的所有抽象方法。

匿名内部类创建方式：

```java
new 类/接口{ 
  //匿名内部类实现部分
}
```

**局部内部类和匿名内部类访问局部变量的时候，为什么变量必须要加上final？局部内部类和匿名内部类访问局部变量的时候，为什么变量必须要加上final呢？它内部原理是什么呢？**

先看这段代码：

```java
public class Outer {

    void outMethod(){
        final int a =10;
        class Inner {
            void innerMethod(){
                System.out.println(a);
            }

        }
    }
}
```

以上例子，为什么要加final呢？是因为**生命周期不一致**， 局部变量直接存储在栈中，当方法执行结束后，非final的局部变量就被销毁。而局部内部类对局部变量的引用依然存在，如果局部内部类要调用局部变量时，就会出错。加了final，可以确保局部内部类使用的变量与外层的局部变量区分开，解决了这个问题。

## 字符串

**String、StringBuffer、StringBuilder：**
String声明的是不可变的对象，每次操作都会生成新的String对象，然后将指针指向新的String对象，而StringBuffer、StringBuilder可以在原有对象的基础上进行操作，所以在经常改变字符串内容的情况下最好不要使用String。
StringBuffer和StringBuilder最大在于，StringBuffer是线程安全的，而StringBuilder是非线程安全的，但StringBuilder的性能却高于StringBuffer，所以在单线程环境下推荐使用StringBuilder，多线程环境下推荐使用StringBuffer。

```java
//jvm会将其分配到常量池中
String str="i"
//jvm会将其分配到堆内存中
String strt = new String("i")
```

**String类的常用方法**

| 函数名        | 作用                                     |
| ------------- | ---------------------------------------- |
| indexOf()     | 返回指定字符的索引。                     |
| charAt()      | 返回指定索引处的字符。                   |
| replace()     | 字符串替换。                             |
| trim()        | 去除字符串两端空白。                     |
| split()       | 分割字符串，返回一个分割后的字符串数组。 |
| getBytes()    | 返回字符串的byte类型数组。               |
| length()      | 返回字符串长度。                         |
| toLowerCase() | 将字符串转成小写字母。                   |
| toUpperCase() | 将字符串转成大写字符。                   |
| substring()   | 截取字符串。                             |
| equals()      | 字符串比较。                             |

**字符串常量池：**

字符串常量池位于堆内存中，专门用来存储字符串常量，可以提高内存的使用率，避免开辟多块空间存储相同的字符串，在创建字符串时 JVM 会首先检查字符串常量池，如果该字符串已经存在池中，则返回它的引用，如果不存在，则实例化一个字符串放到池中，并返回其引用。

Java 中的基本数据类型只有 8 个 ：byte、short、int、long、float、double、char、boolean；除了基本类型（primitive type），剩下的都是引用类型（referencetype），Java 5 以后引入的枚举类型也算是一种比较特殊的引用类型。

**String的特性：**

- 不变性：String 是只读字符串，是一个典型的 immutable 对象，对它进行任何操作，其实都是创建一个新的对象，再把引用指向该对象。不变模式的主要作用在于当一个对象需要被多线程共享并频繁访问时，可以保证数据的一致性。String 底层就是一个 final char 类型的数组，只是使用的时候开发者不需要直接操作底层数组，用更加简便的方式即可完成对字符串的使用。
- 常量池优化：String 对象创建之后，会在字符串常量池中进行缓存，如果下次创建同样的对象时，会直接返回缓存的引用。
- final：使用 final 来定义 String 类，表示 String 类不能被继承，提高了系统的安全性。

## 异常

| 异常类名                         | 备注                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| NullPointerException             | 当应用程序试图访问空对象时，则抛出该异常。                   |
| SQLException                     | 提供关于数据库访问错误或其他错误信息的异常。                 |
| IndexOutOfBoundsException        | 指示某排序索引（例如对数组、字符串或向量的排序）超出范围时抛出。 |
| NumberFormatException            | 当应用程序试图将字符串转换成一种数值类型，但该字符串不能转换为适当格式时，抛出该异常。 |
| FileNotFoundException            | 当试图打开指定路径名表示的文件失败时，抛出此异常。           |
| IOException                      | 当发生某种I/O异常时，抛出此异常。此类是失败或中断的I/O操作生成的异常的通用类。 |
| ClassCastException               | 当试图将对象强制转换为不是实例的子类时，抛出该异常。         |
| ArrayStoreException              | 试图将错误类型的对象存储到一个对象数组时抛出的异常。         |
| IllegalArgumentException         | 抛出的异常表明向方法传递了一个不合法或不正确的参数。         |
| ArithmeticException              | 当出现异常的运算条件时，抛出此异常。例如，一个整数“除以零”时，抛出此类的一个实例。 |
| NegativeArraySizeException       | 如果应用程序试图创建大小为负的数组，则抛出该异常。           |
| NoSuchMethodException            | 无法找到某一特定方法时，抛出该异常。                         |
| SecurityException                | 由安全管理器抛出的异常，指示存在安全侵犯。                   |
| UnsupportedOperationException    | 当不支持请求的操作时，抛出该异常。                           |
| RuntimeExceptionRuntimeException | 是那些可能在Java虚拟机正常运行期间抛出的异常的超类。         |

异常组织架构：

![img](https://img-blog.csdnimg.cn/20200314173417278.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RoaW5rV29u,size_16,color_FFFFFF,t_70) 

## IO

**Files的常用方法：**

- Files. exists()：检测文件路径是否存在。
- Files. createFile()：创建文件。
- Files. createDirectory()：创建文件夹。
- Files. delete()：删除一个文件或目录。
- Files. copy()：复制文件。
- Files. move()：移动文件。
- Files. size()：查看文件个数。
- Files. read()：读取文件。
- Files. write()：写入文件。



## JDBC

jdbc操作数据库流程如下。

```flow
flowchat
st=>start: 开始
e=>end: 结束
op=>operation: 加载JDBC驱动程序
op2=>operation: 创建数据库的连接
op3=>operation: 创建Statement实例
op4=>operation: 通过Statement执行SQL语句
op5=>operation: 处理结果
op6=>operation: 释放资源
st->op->op2->op3->op4->op5->op6->e

```

1、加载JDBC驱动程序
在连接数据库之前，首先要加载想要连接的数据库的驱动到JVM。成功加载后，会将Driver类的实例注册到DriverManager类中。
2、创建数据库的连接
要连接数据库，需要向java.sql.DriverManager请求并获得Connection对象，该对象就代表一个数据库的连接。
3、创建一个Statement实例，Statement实例分为以下3种类型：

- 通过Statement实例实现执行静态SQL语句；
- 通过PreparedStatement实例实现执行动态SQL语句（底层具体啥原理？？？占位符？），可以防止sql注入；
- 通过CallableStatement实例实现执行数据库存储过程。

以上的sql注入和数据库存储过程概念详见链接: [Mysql基础](https://blog.csdn.net/wxd1234567890/article/details/111160003)；

4、执行SQL语句，Statement接口提供了三种执行SQL语句的方法：

- executeQuery返回一个结果集（ResultSet）对象；
- executeUpdate（用于执行INSERT、UPDATE或DELETE语句以及DDL数据库定义语句包括创建表、修改表等）；
- execute用于执行返回多个结果集、多个更新计数或二者组合的语句。

5、处理结果
6、释放JDBC资源，关闭顺序和声明顺序相反：关闭记录集-->关闭声明-->关闭连接对象

```java
class JDBCTest {
    public void test() {
        //加载MySql的驱动类
        try {
            Class.forName("com.mysql.jdbc.Driver");
        } catch (ClassNotFoundException ex) {
            System.out.println("找不到驱动程序类，加载驱动失败！");
            ex.printStackTrace();
        }
        Connection con = null;
        Statement stmt = null;
        ResultSet rs = null;
        try {
            String url = "jdbc:mysql://localhost:3306/test?useSSL=false&characterEncoding=utf8";
            String username = "root";
            String password = "root";
            //创建数据库的连接
            con = DriverManager.getConnection(url, username, password);
            //创建Statement
            stmt = con.createStatement();
            String pstmtSql = "SELECT * FROM user WHERE login=? AND password=?";
            PreparedStatement pstmt = con.prepareStatement(pstmtSql);
            pstmt.setObject(1, "name");
            pstmt.setObject(2, "password");
            CallableStatement cstmt = con.prepareCall("");
            //获取结果集，处理结果
            String sqlQueryString = "SELECT * FROM tablename";
            rs = stmt.executeQuery(sqlQueryString);
            while (rs.next()) {
                String field1 = rs.getString("field1");
                System.out.println(field1);
            }
            String sqlInsertString = "INSERT INTO tablename (field1, field2) VALUES('1', '2')";
            int result = stmt.executeUpdate(sqlInsertString);
            if (result == 0) {
                System.out.println("insert data success");
            }

        } catch (SQLException ex) {
            System.out.println("数据库操作失败！");
            ex.printStackTrace();
        } finally {
            try {
                //释放资源，关闭记录集、声明、连接对象
                if (rs != null) {
                    rs.close();
                }
                if (stmt != null) {
                    stmt.close();
                }
                if (con != null) {
                    con.close();
                }
            } catch (SQLException ex) {
                ex.printStackTrace();
            }
        }
    }
}
```

## 设计模式

### 代理模式

动态代理主要通过反射实现。反射在框架中常用，Java反射的主要功能：查找对象的类；查找类的数据成员、方法、构造器、超类；构造类的对象；调用方法；实现动态代理。

1、**静态代理：**

静态代理模式实现如下
```java
interface IUserDao {
    void save();
}

class UserDao implements IUserDao {

    @Override
    public void save() {
        System.out.println("save data");
    }
}

class UserDaoProxy implements IUserDao {
    private IUserDao iUserDao;

    public UserDaoProxy(IUserDao iUserDao) {
        this.iUserDao = iUserDao;
    }

    @Override
    public void save() {
        System.out.println("transactions start");
        this.iUserDao.save();
        System.out.println("transactions end");
    }
}

class StaticUserProxy {
    public void testStaticProxy() {
        IUserDao iUserDao = new UserDao();
        UserDaoProxy userDaoProxy = new UserDaoProxy(iUserDao);
        userDaoProxy.save();
    }
}
```

**2、动态代理：**

定义：当想要给实现了某个接口的类中的方法，加一些额外的处理。比如说加日志，加事务等。可以给这个类创建一个代理，就是创建一个新的类，这个类不仅包含原来类方法的功能，而且还在原来的基础上添加了额外处理的新类。这个代理类并不是定义好的，是动态生成的。具有解耦意义，灵活，扩展性强。
应用：Spring的AOP加事务、权限判断、打印日志
实现：JDK动态代理、CGLIB代理
JDK的动态代理有一个限制，就是使用动态代理的对象必须实现一个或多个接口。如果想代理不需要实现接口的类，就可以使用CGLIB实现。
CGLIB是一个强大的高性能的代码生成包，它可以在运行期扩展Java类与实现Java接口。它广泛的被许多AOP的框架使用，例如Spring AOP，为他们提供方法的interception拦截。CGLIB包的底层是通过使用一个小而快的字节码处理框架ASM，来转换字节码并生成新的类。CGLIB代理无需实现接口，通过生成类字节码实现代理，比反射稍快，不存在性能问题，但CGLIB会继承目标对象，需要重写方法，所以目标对象不能为final类。
**CGLIB代理与JDK动态代理区别：**
  1. 使用JDK动态代理的对象必须实现一个或多个接口；使用CGLIB代理的对象则无需实现接口，达到代理类无侵入，直接代理类。
  2. 使用CGLIB需要引入CGLIB的jar包。
```java
//JDK动态代理和CGLIB代理实现
interface IUserDao {
    void save();
}

class UserDao implements IUserDao {

    @Override
    public void save() {
        System.out.println("save data");
    }
}
//JDK Dynamic Proxy
class ProxyFactory {
    private Object target;

    public ProxyFactory(Object target) {
        this.target = target;
    }

    public Object getProxyInstance() {
        return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(),
                new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        System.out.println("transactions start");
                        Object returnValue = method.invoke(target, args);
                        System.out.println("transactions end");
                        return null;
                    }
                });
    }
}

class UserDaoCGLIB {
    void save(){
        System.out.println("save data");
    }
}

//CGLIB Dynamic Proxy
class CGLIBProxyFactory implements MethodInterceptor {
    private Object target;

    public CGLIBProxyFactory(Object target) {
        this.target = target;
    }

    public Object getProxyInstance() {
        Enhancer en = new Enhancer();
        en.setSuperclass(target.getClass());
        en.setCallback(this);
        return en.create();
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("transactions start");
        Object returnValue = method.invoke(target, args);
        System.out.println("transactions end");
        return null;
    }
}


class DynamicProxy {
    public void jdkWay() {
        IUserDao iUserDao = new UserDao();
        ProxyFactory proxyFactory = new ProxyFactory(iUserDao);
        IUserDao proxy = (IUserDao) proxyFactory.getProxyInstance();
        proxy.save();
    }

    public void cglibWay() {
        UserDaoCGLIB userDao = new UserDaoCGLIB();
        CGLIBProxyFactory proxyFactory = new CGLIBProxyFactory(userDao);
        UserDaoCGLIB proxy = (UserDaoCGLIB) proxyFactory.getProxyInstance();
        proxy.save();

    }

}

```
3、**静态代理与动态代理：**

静态代理在编译时就已经实现，编译完成后代理类是一个实际的class文件。
动态代理是在运行时动态生成的，即编译完成后没有实际的class文件，而是在运行时动态生成类字节码，并加载到JVM中。相对于静态代理，动态代理消耗系统性能，但更灵活。
参考链接: [Java三种代理模式：静态代理、动态代理和cglib代理](https://segmentfault.com/a/1190000011291179).

## 关注点

### JDK和JRE

JDK：Java Development Kit的简称，java开发工具包，提供了java的开发环境和运行环境。
JRE：JavaRuntimeEnvironment的简称，java运行环境，为java的运行提供了所需环境。
具体来说JDK其实包含了JRE，同时还包含了编译java源码的编译器javac，还包含了很多java程序调试和分析的工具。简单来说：如果你需要运行java程序，只需安装JRE就可以了，如果你需要编写java程序，需要安装JDK。

### equals和==

基本类型（byte、boolean、char、short、int、long、float、double）都是用\=\=判断相等性。对象引用的话，\=\=判断引用所指的对象是否是同一个。equals是Object的成员函数，有些类会覆盖（override）这个方法，用于判断对象的值的等价性。例如String类，两个引用所指向的String都是”abc”，但可能出现他们实际对应的对象并不是同一个（和jvm实现方式有关），因此用==判断他们可能不相等，但用equals判断一定是相等的。

```java
class StringTest{
    public void test(){
        String a= "123";
        String b = "123";
        String c = new String("123");
        System.out.println(a == b); //true
        System.out.println(a == c); //false
        System.out.println(a.equals(c)); //true
    }
}
```

### 重写和重载

override（重写）

1. 存在于父类和子类之间。 
2. 方法名、参数、返回值相同。 
3. 子类方法不能缩小父类方法的访问权限。   
4. 子类方法不能抛出比父类方法更多的异常(但子类方法可以不抛出异常)。
5. 方法被定义为final不能被重写。

overload（重载）

1. 参数类型、个数、顺序至少有一个不相同。 
2. 不能重载只有返回值不同的方法名。
3. 存在于父类和子类、同类中。

### 抽象类

**抽象类和普通类**

1. 普通类不能包含抽象方法，抽象类可以包含抽象方法。 
2. 抽象类不能直接实例化，普通类可以直接实例化。
3. 抽象类能不能使用final修饰。定义抽象类就是让其他类继承的，如果定义为final该类就不能被继承。

**抽象类和接口**

1. 接口是公开的，里面不能有私有的方法或变量，用于让别人使用的，而抽象类是可以有私有方法、私有变量、构造函数。
2. 实现接口的一定要实现接口里定义的所有方法，而实现抽象类可以有选择地重写需要用到的方法，一般的应用里，最顶级的是接口，然后是抽象类实现接口，最后才到具体类实现。
3. 接口可以实现多重继承，多个接口可以通过一个类实现；而一个类只能继承一个抽象超类，但可以通过继承多个接口实现多重继承。
4. 访问修饰符：接口中的方法默认使用public修饰；抽象类中的方法可以是任意访问修饰符。
5. 接口还有标识（里面没有任何方法，如Remote接口）和数据共享（里面的变量全是常量）的作用。

### 序列化

**序列化场景：**

1. 把的内存中的对象保存到一个文件中或者数据库中时候；
2. 用套接字在网络上传送对象的时候，需要将对象序列化，反序列化；
3. 当你想通过RMI传输对象的时候；
4. json序列化。。。

### 对象拷贝

想对一个对象进行处理，又想保留原有的数据进行接下来的操作，就需要使用对象拷贝了。
**实现方式：**
1、实现Cloneable接口并重写Object类中的clone()方法；
2、实现Serializable接口，通过对象的序列化和反序列化实现克隆。
基于序列化和反序列化实现的克隆不仅仅是深度克隆，更重要的是通过泛型限定，可以检查出要克隆的对象是否支持序列化，这项检查是编译器完成的，不是在运行时抛出异常，这种是方案明显优于使用Object类的clone方法克隆对象。让问题在编译的时候暴露出来总是好过把问题留到运行时。

```java
//对象拷贝实现
class Car implements Serializable {
    private static final long serialVersionUID = -5713945027627603702L;

    private String brand;       // 品牌
    private int maxSpeed;       // 最高时速

    public Car(String brand, int maxSpeed) {
        this.brand = brand;
        this.maxSpeed = maxSpeed;
    }

    public String getBrand() {
        return brand;
    }

    public void setBrand(String brand) {
        this.brand = brand;
    }

    public int getMaxSpeed() {
        return maxSpeed;
    }

    public void setMaxSpeed(int maxSpeed) {
        this.maxSpeed = maxSpeed;
    }

    @Override
    public String toString() {
        return "Car [brand=" + brand + ", maxSpeed=" + maxSpeed + "]";
    }

}

class Person implements Serializable {
    private static final long serialVersionUID = -9102017020286042305L;

    private String name;    // 姓名
    private int age;        // 年龄
    private Car car;        // 座驾

    public Person(String name, int age, Car car) {
        this.name = name;
        this.age = age;
        this.car = car;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public Car getCar() {
        return car;
    }

    public void setCar(Car car) {
        this.car = car;
    }

    @Override
    public String toString() {
        return "Person [name=" + name + ", age=" + age + ", car=" + car + "]";
    }

}

class CloneTest {

    public static void main(String[] args) {
        try {
            Person p1 = new Person("郭靖", 33, new Car("Benz", 300));
            // deepy copy
            Person p2 = MyUtil.clone(p1);
            p2.getCar().setBrand("BYD");
            System.out.println(p1);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

class MyUtil {

    private MyUtil() {
        throw new AssertionError();
    }

    @SuppressWarnings("unchecked")
    public static <T extends Serializable> T clone(T obj) throws Exception {
    // key code
        ByteArrayOutputStream bout = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bout);
        oos.writeObject(obj);

        ByteArrayInputStream bin = new ByteArrayInputStream(bout.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bin);
        return (T) ois.readObject();
        // 说明：调用ByteArrayInputStream或ByteArrayOutputStream对象的close方法没有任何意义。 这两个基于内存的流只要垃圾回收器清理对象就能够释放资源，这一点不同于对外部资源（如文件流）的释放
    }
}
```

**深拷贝和浅拷贝：**
浅拷贝只是复制了对象的引用地址，两个对象指向同一个内存地址，所以修改其中任意的值，另一个值都会随之变化，这就是浅拷贝，如赋值引用。深拷贝是将对象及值复制过来，两个对象修改其中任意的值另一个值不会改变，这就是深拷贝（例：JSON.parse()和JSON.stringify()）

### 值传递

首先回顾一下在程序设计语言中有关将参数传递给方法（或函数）的一些专业术语。**按值调用(call by value)表示方法接收的是调用者提供的值，而按引用调用（call by reference)表示方法接收的是调用者提供的变量地址。一个方法可以修改传递引用所对应的变量值，而不能修改传递值调用所对应的变量值。** 它用来描述各种程序设计语言（不只是Java)中方法参数传递方式。**Java程序设计语言总是采用按值调用。也就是说，方法得到的是所有参数值的一个拷贝，也就是说，方法不能修改传递给它的任何参数变量的内容。**

下面再总结一下Java中方法参数的使用情况：

- 一个方法不能修改一个基本数据类型的参数（即数值型或布尔型）
- 一个方法可以改变一个对象参数的状态。
- 一个方法不能让对象参数引用一个新的对象。

值传递和引用传递的区别：

值传递：指的是在方法调用时，传递的参数是按值的拷贝传递，传递的是值的拷贝，也就是说传递后就互不相关了。

引用传递：指的是在方法调用时，传递的参数是按引用进行传递，其实传递的引用的地址，也就是变量所对应的内存空间的地址。传递的是值的引用，也就是说传递前和传递后都指向同一个引用（也就是同一个内存空间）。

以下函数值传递的实例：

**example 1**

```java
public static void main(String[] args) {
    int num1 = 10;
    int num2 = 20;

    swap(num1, num2);

    System.out.println("num1 = " + num1);
    System.out.println("num2 = " + num2);
}

public static void swap(int a, int b) {
    int temp = a;
    a = b;
    b = temp;

    System.out.println("a = " + a);
    System.out.println("b = " + b);
}
```

**结果**：

```
a = 20
b = 10
num1 = 10
num2 = 20
```

**example2**

```java
    public static void main(String[] args) {
        int[] arr = { 1, 2, 3, 4, 5 };
        System.out.println(arr[0]);
        change(arr);
        System.out.println(arr[0]);
    }

    public static void change(int[] array) {
        // 将数组的第一个元素变为0
        array[0] = 0;
    }
```

**结果**：

```
1
0
```

**example 3**

```java
public class Test {

    public static void main(String[] args) {
        // TODO Auto-generated method stub
        Student s1 = new Student("小张");
        Student s2 = new Student("小李");
        Test.swap(s1, s2);
        System.out.println("s1:" + s1.getName());
        System.out.println("s2:" + s2.getName());
    }

    public static void swap(Student x, Student y) {
        Student temp = x;
        x = y;
        y = temp;
        System.out.println("x:" + x.getName());
        System.out.println("y:" + y.getName());
    }
}
```

**结果**：

```
x:小李
y:小张
s1:小张
s2:小李
```

# 集合

## 概况

jdk集合的关系拓扑图如下。
![集合接口类的关系图](https://img-blog.csdnimg.cn/20201214141116959.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d4ZDEyMzQ1Njc4OTA=,size_16,color_FFFFFF,t_70)

List、Set、Map接口详情如下。
![。。。](https://img-blog.csdnimg.cn/20201214141533386.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d4ZDEyMzQ1Njc4OTA=,size_16,color_FFFFFF,t_70)

## 源码原理

### ArrayList

**ArrayList扩容机制：**
在JDK1.8中，如果通过无参构造的话，初始数组容量为0，当真正对数组进行添加时（即添加第一个元素时），才真正分配容量，默认分配容量为10；当容量不足时（容量为size，添加第size+1个元素时），先按照1.5倍的比例扩容能否满足最低容量要求，若能，则以1.5倍扩容，否则以最低容量要求进行扩容。当ArrayList扩容时，根据扩容的大小分配一块新的连续的内存空间，将已经有数组的数据复制到新的存储空间中，并插入新的数据。例如，不指定大小的ArrayList添加第11个元素时，进行扩容到10*1.5=15，添加第16个元素时，扩容到对应的大小。

### HashMap

**实现原理**

HashMap概述：HashMap是基于哈希表的Map接口的非同步实现。此实现提供所有可选的映射操作，并允许使用null值和null键。此类不保证映射的顺序，特别是它不保证该顺序恒久不变。
HashMap的数据结构：HashMap实际上是一个“链表散列”的数据结构，即**数组和链表和红黑树**的结合体。
当我们往Hashmap中put元素时,首先根据key的hashcode重新计算hash值,根据hash值（例如假设hash算法为通过hash值对数组的大小取余）得到这个元素在**数组的位置及下标**，如果该数组在该位置上已经存放了其他元素，那么在这个位置上的元素将以链表的形式存放，新加入的放在链头,最先加入的放入链尾。如果数组中该位置没有元素,就直接将该元素放到数组的该位置上。Java8中对HashMap的实现做了优化，当链表中的节点数据超过八个之后，该链表会转为红黑树来提高查询效率,从原来的O(n)到O(logn)。其中红黑树详见链接: [高阶数据结构](https://blog.csdn.net/wxd1234567890/article/details/111170599).

**扩容机制**

**哈希冲突：**key通过哈希算法得到的哈希值相同，导致数据存储在同一个节点的链表上，当冲突太多链表过长，查询效率就会急剧下降。
**扩容应用实例：**
当键值对的实际大小size大于table的实际大小*负载因子时需要进行扩容。
假设了hash算法就是简单的用key的hash值取余数组的长度，其中的哈希桶数组table的原始size=2，key = 3、7、5的put顺序依次为 5、7、3。在mod取余2以后都冲突在table[1]这里了。这里假设负载因子 loadFactor=1，即当键值对的实际大小size大于table的实际大小就进行扩容。接下来的三个步骤是哈希桶数组resize扩容成4（后续如果不够，继续扩容8,16，，，），然后所有的Node重新哈希映射的过程。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020121415443077.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d4ZDEyMzQ1Njc4OTA=,size_16,color_FFFFFF,t_70)
扩容是一个特别耗性能的操作，所以在使用HashMap的时候，需要估算map的大小，初始化的时候给一个大致的数值，避免map进行频繁的扩容影响性能。
**HashMap的长度必须是2的幂原因：**

hash算法中，为了使元素分布的更加均匀，很多都会使用取模运算，在hashMap中并没有使用hash%n这样进行取模运算，而是使用(n - 1) & hash进行代替，原因是在计算机中，&的效率要远高于%；需要注意的是，只有容量为2的n次幂的时候，(n - 1) & hash 才能等效hash%n，这也是hashMap 初始化初始容量时，无论传入任何值，都会通过tableSizeFor(int cap) 方法转化成2的n次幂的原因

 **源码：**

https://juejin.cn/post/6844903799748821000#heading-2

```java
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
             //如果table尚未初始化，则此处进行初始化数组，并赋值初始容量，重新计算阈值
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            //通过hash找到下标，如果hash值指定的位置数据为空，则直接将数据存放进去
            tab[i] = newNode(hash, key, value, null);
        else {
            //如果通过hash找到的位置有数据，发生碰撞
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                //如果需要插入的key和当前hash值指定下标的key一样，先将e数组中已有的数据
                e = p;
            else if (p instanceof TreeNode)
                //如果此时桶中数据类型为 treeNode，使用红黑树进行插入
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //此时桶中数据类型为链表
                // 进行循环
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        //如果链表中没有最新插入的节点，将新放入的数据放到链表的末尾
                        p.next = newNode(hash, key, value, null);

                        //如果链表过长，达到树化阈值，将链表转化成红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果链表中有新插入的节点位置数据不为空，则此时e 赋值为节点的值，跳出循环
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }

            //经过上面的循环后，如果e不为空，则说明上面插入的值已经存在于当前的hashMap中，那么更新指定位置的键值对
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //如果此时hashMap size大于阈值，则进行扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }

//扩容
final Node<K,V>[] resize() {

        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;

        //1、table已经初始化，且容量 > 0
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                //如果旧的容量已近达到最大值，则不再扩容，阈值直接设置为最大值
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                //如果旧的容量不小于默认的初始容量，则进行扩容，容量扩张为原来的二倍
                newThr = oldThr << 1; // double threshold
        }
        //2、阈值大于0 threshold 使用 threshold 变量暂时保存 initialCapacity 参数的值
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        //3 threshold 和 table 皆未初始化情况，此处即为首次进行初始化
        //也就在此处解释了构造方法中没有对threshold 和 初始容量进行赋值的问题
        else {               // zero initial threshold signifies using defaults
            //如果阈值为零，表示使用默认的初始化值
            //这种情况在调用无参构造的时候会出现，此时使用默认的容量和阈值
            newCap = DEFAULT_INITIAL_CAPACITY;
            //此处阈值即为 threshold=initialCapacity*loadFactor
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // newThr 为 0 时，按阈值计算公式进行计算，容量*负载因子
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }

        //更新阈值
        threshold = newThr;

        //更新数组桶
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;

        //如果之前的数组桶里面已经存在数据，由于table容量发生变化，hash值也会发生变化，需要重新计算下标
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                //如果指定下标下有数据
                if ((e = oldTab[j]) != null) {
                    //1、将指定下标数据置空
                    oldTab[j] = null;
                    //2、指定下标只有一个数据
                    if (e.next == null)
                        //直接将数据存放到新计算的hash值下标下
                        newTab[e.hash & (newCap - 1)] = e;
                    //3、如果是TreeNode数据结构
                    else if (e instanceof TreeNode)

                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //4、对于链表，数据结构
                    else { // preserve order
                        //如果是链表，重新计算hash值，根据新的下标重新分组
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }

//查询
public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {

            //1、根据hash算法找到对应位置的第一个数据，如果是指定的key，则直接返回
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;

            if ((e = first.next) != null) {
                //如果该节点为红黑树，则通过树进行查找
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                //如果该节点是链表，则遍历查找到数据
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

//扰动函数
 static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
//
first = tab[(n - 1) & hash]) 
```



### ConcurrentHashMap

JDK8中ConcurrentHashMap采用的是**CAS+Synchronized锁并且锁粒度是每一个桶**。简单来说JDK7中锁的粒度是Segment，JDK8锁粒度细化到了桶级别。可想而知锁粒度是大大提到了。辅之以代码的优化，JDK8中的ConcurrentHashMap在性能上的表现非常优秀。 

**源码**

https://www.cnblogs.com/hello-shf/p/12183263.html

在ConcurrentHashMap中使用了unSafe方法，通过直接操作内存的方式来保证并发处理的安全性，使用的是硬件的安全机制。

```java
/*
     * 用来返回节点数组的指定位置的节点的原子操作
     */
    @SuppressWarnings("unchecked")
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

    /*
     * cas原子操作，在指定位置设定值
     */
    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
    /*
     * 原子操作，在指定位置设定值
     */
    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```



```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());//hash，对hashcode再散列
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {//迭代桶数组，自旋
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)//懒加载。如果为空，则进行初始化
            tab = initTable();//初始化桶数组
        //(n - 1) & hash)计算下标，取值，为空即无hash碰撞
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))//通过cas插入新值
                break;                   // no lock when adding to empty bin
        }
        //判断是否正在扩容。如果正在扩容，当前线程帮助进行扩容。
        //每个线程只能同时负责一个桶上的数据迁移，并且不影响其它桶的put和get操作。
        //（很牛逼的思路，能这么做建立在更细粒度的锁基础上）
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {//put5,存在hash碰撞
            V oldVal = null;
            //此处，f在上面已经被赋值，f为当前下标桶的首元素。对链表来说是链表头对红黑树来说是红黑树的头元素。
            synchronized (f) {
                //再次检查当前节点是否有变化，有变化进入下一轮自旋
                //为什么再次检查？因为不能保证，当前线程到这里，有没有其他线程对该节点进行修改
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {//当前桶为链表
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {//迭代链表节点
                            K ek;
                            if (e.hash == hash &&//key相同，覆盖（onlyIfAbsent有什么用？）
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            //找到链表尾部，插入新节点。（什么这里不用CAS？因为这在同步代码块里面）
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {//当前桶为红黑树
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {//想红黑树插入新节点
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                //树化。binCount > 8，进行树化，链表转红黑树
                if (binCount >= TREEIFY_THRESHOLD)
                    //如果容量 < 64则直接进行扩容；不转红黑树。
                    //（你想想，假如容量为16，你就插入了9个元素，巧了，都在同一个桶里面，
                    //如果这时进行树化，时间复杂度会增加，性能下降，不如直接进行扩容，空间换时间）
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);//扩容。addCount内部会进行判断要不要扩容
    return null;
}
```

put总结：

1，懒加载，未初始化则初始化table
2，hash，hashcode再散列，并计算下标
3，无碰撞，通过CAS插入
4，有碰撞
　　4.1、如果正在扩容，协助其它线程去扩容
　　4.2、如果是链表，插入链表
　　4.3、如果是红黑树，插入红黑树
　　4.4、如果链表长度超过8，树化
　　4.5，如果key已经存在，覆盖旧值
5，需要扩容，则扩容

扩容函数：

。。。

### HashSet

HashSet底层由HashMap实现，HashSet的值存放于HashMap的key上，HashMap的value统一为PRESENT为空值。

## 关注点
### Collection和Collections
java.util.Collection是一个集合接口，集合类的一个顶级父接口。它提供了对集合对象进行基本操作的通用接口方法。Collection接口在Java类库中有很多具体的实现。Collection接口的意义是为各种具体的集合提供了最大化的统一操作方式，其直接继承接口有List与Set。
Collections则是集合类的一个工具类/帮助类，其中提供了一系列静态方法，用于对集合中元素进行排序、搜索以及线程安全等各种操作。

### Arraylist和LinkedList

ArrrayList底层的数据结构是数组，使用下标访问一个元素，支持随机访问，而LinkedList的底层数据结构是双向循环链表，不支持随机访问。ArrayList的时间复杂度是O(1)，而LinkedList是O(n)。
**Arraylist：**
优点：ArrayList是实现了基于动态数组的数据结构,因为**地址连续**，一旦数据存储好了，查询操作效率会比较高，在内存里是连着放的。
缺点：因为地址连续，ArrayList要移动数据，所以插入和删除操作效率比较低。
**LinkedList：**
优点：LinkedList基于链表的数据结构,地址是任意的，所以在开辟内存空间的时候不需要等一个连续的地址，对于新增和删除操作add和remove，LinedList比较占优势。LinkedList适用于要头尾操作或插入指定位置的场景
缺点：因为LinkedList要移动指针,所以查询操作性能比较低。
**适用场景分析：**
当需要对数据经常查询的场景下选用ArrayList，当需要对数据进行经常插入删除的场景采用LinkedList。

### ArrayList和Vector

ArrayList和Vector底层都是用数组实现的。

**ArrayList与Vector区别：**
- Vector是多线程安全的，而ArrayList不是线程安全的（线程安全就是说多线程修改共享数据时，需求保持同一时刻只有一个修改共享数据，不会产生不确定的结果）。
- 两个都是采用的线性连续的内存空间存储元素，但是当空间不足的时候，两个类的增加方式是不同。ArrayList在内存不够时默认是扩展50% + 1个，Vector是默认扩展1倍或旧容量加增长因子大小。Vector可以设置增长因子，而ArrayList不可以。
- Vector类的源码中的方法很多有synchronized进行修饰，这样就导致了Vector在效率上无法与ArrayList相比，效率较低；

**适用场景分析：**
Vector是线程同步的，所以它也是线程安全的，而ArrayList是线程异步的，是不安全的。如果不考虑到线程的安全因素，一般用ArrayList效率比较高。如果集合中的元素的数目大于目前集合数组的长度时，在集合中使用数据量比较大的数据，用Vector有一定的优势。


### HashMap和Hashtable

1. hashMap去掉了HashTable的contains方法，但是加上了containsValue（）和containsKey（）方法。
2. hashTable同步的，线程安全的，而HashMap是非同步的，效率上比hashTable要高。
3. hashMap允许空键值，而hashTable不允许。

### HashMap和ConcurrentHashMap

ConcurrentHashMap是线程安全的HashMap的实现。
1. ConcurrentHashMap对整个数组进行了分割分段(Segment)，然后在每一个分段上都用lock锁进行保护，相对于HashTable的syn关键字锁的粒度更精细了一些，并发性能更好，而HashMap没有锁机制，不是线程安全的。
2. HashMap的键值对都允许有null，但是ConCurrentHashMap都不允许。

### ConcurrentHashMap和HashTable

HashTable里使用的是synchronized关键字，这其实是对整个Map对象加锁，锁住的都是对象整体，当Hashtable的大小增加到一定的时候，性能会急剧下降，因为迭代时需要被锁定很长的时间。ConcurrentHashMap算是对上述问题的优化。

### HashMap和TreeMap

- TreeMap<K,V>的Key值是要求实现java.lang.Comparable，所以迭代的时候TreeMap默认是按照Key值升序排序的；TreeMap的实现是基于红黑树结构，适用于按升序或自定义顺序遍历键key。
- HashMap<K,V>的Key值实现散列hashCode()，分布是散列的、均匀的，不支持排序；数据结构主要是数组，链表或红黑树。适用于在Map中插入、删除和定位元素。
- 适用场景分析：需要按照key排序使用TreeMap。除此之外，由于HashMap有更好的性能，所以大多不需要排序的时候我们会使用HashMap。


# 并发

## 创建线程
1. 继承Thread类创建线程类
  定义Thread类的子类，并重写该类的run方法，该run方法的方法体就代表了线程要完成的任务。因此把run()方法称为执行体。
  创建Thread子类的实例，即创建了线程对象。
  调用线程对象的start()方法来启动该线程。
2. 通过Runnable接口创建线程类
  定义runnable接口的实现类，并重写该接口的run()方法，该run()方法的方法体同样是该线程的线程执行体。
  创建 Runnable实现类的实例，并依此实例作为Thread的target来创建Thread对象，该Thread对象才是真正的线程对象。
  调用线程对象的start()方法来启动该线程。
3. 通过Callable和Future创建线程
  创建Callable接口的实现类，并实现call()方法，该call()方法将作为线程执行体，并且有返回值。
  创建Callable实现类的实例，使用Future类来包装Callable对象，该Future对象封装了该Callable对象的call()方法的返回值。

```java
// create thread 3 ways
class Thread1 extends Thread {
    @Override
    public void run() {
        System.out.println("Thread1 start");
    }
}

class Thread2 implements Runnable {

    @Override
    public void run() {
        System.out.println("Thread2 start");
    }
}

class Thread3 implements Callable<String> {

    @Override
    public String call() throws Exception {
        System.out.println("Thread3 start");
        return "return value";
    }
}

class ThreadTest {

    public void test() {
        Thread thread1 = new Thread1();
        thread1.start();
        Thread thread2 = new Thread(new Thread2());
        thread2.start();
        try {
            ExecutorService executorService = Executors.newFixedThreadPool(4);
            Callable<String> task = new Thread3();
            Future<String> future = executorService.submit(task);
            //may block
            String result = future.get();
            System.out.println(String.format("thread3 result %s", result));
        } catch (Exception ex) {
        }
    }
}
```

**runnable和callable:**
- Runnable接口中的run()方法的返回值是void，它做的事情只是纯粹地去执行run()方法中的代码而已；
- Callable接口中的call()方法是有返回值的，是一个泛型，和Future、FutureTask配合可以用来获取异步任务的执行结果。

**创建线程方式的对比：**
1. 采用实现Runnable、Callable接口的方式创见多线程
  优势：线程类只是实现了Runnable接口或Callable接口，还可以继承其他类。在这种方式下，多个线程可以共享同一个target对象，所以非常适合多个相同线程来处理同一份资源的情况，从而可以将CPU、代码和数据分开，形成清晰的模型，较好地体现了面向对象的思想。
  劣势：编程稍微复杂，如果要访问当前线程，则必须使用Thread.currentThread()方法。
2. 使用继承Thread类的方式创建多线程
  优势：编写简单，如果需要访问当前线程，则无需使用Thread.currentThread()方法，直接使用this即可获得当前线程。
  劣势：线程类已经继承了Thread类，所以不能再继承其他父类。

## 线程的生命周期

线程通常都有五种状态，创建、就绪、运行、阻塞和死亡。
- 创建状态。在生成线程对象，并没有调用该对象的start方法，这是线程处于创建状态。
- 就绪状态。当调用了线程对象的start方法之后，该线程就进入了就绪状态，但是此时线程调度程序还没有把该线程设置为当前线程，此时处于就绪状态。线程正在就绪队列中排队等候得到CPU资源。在线程运行之后，从等待或者睡眠中回来之后，也会处于就绪状态。
- 运行状态。线程调度程序将处于就绪状态的线程设置为当前线程，此时线程就进入了运行状态，开始运行run函数当中的代码。
- 阻塞状态。线程正在运行的时候，被暂停，通常是为了等待某个时间的发生(比如说某项资源就绪)之后再继续运行。sleep，wait，suspend（被另一个线程所阻塞）等方法都可以导致线程阻塞。
- 死亡状态。如果一个线程的run方法执行结束或者调用stop方法后，该线程就会死亡。对于已经死亡的线程，无法再使用start方法令其进入就绪。

## 守护线程

守护线程（Daemon Thread）是指为其他线程服务的线程。在JVM中，所有非守护线程都执行完毕后，无论有没有守护线程，虚拟机都会自动退出。因此，JVM退出时，不必关心守护线程是否已结束。
在守护线程中，编写代码要注意：守护线程不能持有任何需要关闭的资源，例如打开文件等，因为虚拟机退出时，守护线程没有任何机会来关闭文件，这会导致数据丢失。创建守护线程如下代码。

```java
Thread t = new MyThread();
t.setDaemon(true);
t.start();
```

## 常用方法

1. sleep()方法
  在指定的毫秒数内让当前正在执行的线程休眠（暂停执行），此操作受到系统计时器和调度程序精度和准确性的影响。让其他线程有机会继续执行，**但它并不释放对象锁。也就是如果有synchronized同步块，其他线程仍然不能访问共享数据**。比如有两个线程同时执行(没有synchronized同步块)，一个线程优先级为MAX_PRIORITY，另一个为MIN_PRIORITY，如果没有Sleep()方法，只有高优先级的线程执行完成后，低优先级的线程才能执行；但当高优先级的线程sleep(5000)后，低优先级就有机会执行了。总之，sleep()可以使低优先级的线程得到执行的机会，当然也可以让同优先级、高优先级的线程有执行的机会。sleep方法是线程类Thread的静态方法，让调用线程进入睡眠状态，让出执行机会给其他线程，等到休眠时间结束后，线程进入就绪状态和其他线程一起竞争cpu的执行时间。sleep不能改变对象的锁，当一个synchronized同步块中调用了sleep() 方法，线程虽然进入休眠，但是对象的锁没有被释放，其他线程依然无法访问这个对象。
2. wait()方法
  wait()是Object类的方法，当一个线程执行到wait方法时，它就进入到一个和该对象相关的等待池，同时释放对象的机锁，使得其他线程能够访问同步块，可以通过notify，notifyAll方法来唤醒等待的线程。
3. yield()方法
  yield()方法和sleep()方法类似，也不会释放对象锁，区别在于，它没有入参，即yield()方法只是使当前线程重新回到可执行状态（就绪状态？），所以执行yield()的线程有可能在进入到可执行状态后马上又被执行；另外yield()方法只能使同优先级或者高优先级的线程得到执行机会，这也和sleep()方法不同。
4. join()方法
  如下代码，当main线程对线程对象t调用join()方法时，主线程将等待变量t表示的线程运行结束，即join就是指等待该线程结束，然后才继续往下执行自身线程。所以，上述代码打印顺序可以肯定是main线程先打印start，t线程执行打印，main线程最后再打印end。通过对另一个线程对象调用join()方法可以等待其执行结束，对已经运行结束的线程调用join()方法会立刻返回。
```java
class ThreadTest {
    public void threadFunctionTest(){
        Thread t = new Thread(()->{
            System.out.println("in thread t...");
        });
        try{
            System.out.println("Main thread start");
            t.start();
            t.join();
            System.out.println("Main thread end");
        }catch (InterruptedException ex){
        }
    }
}
```

5. notify和notifyAll
  如果线程调用了对象的 wait()方法，那么线程便会处于该对象的等待池中，等待池中的线程不会去竞争该对象的锁。当有线程调用了对象的 notifyAll()方法（唤醒所有 wait 线程）或 notify()方法（只随机唤醒一个 wait 线程），被唤醒的的线程便会进入该对象的锁池中，锁池中的线程会去竞争该对象锁。也就是说，调用了notify后只要一个线程会由等待池进入锁池，而notifyAll会将该对象等待池内的所有线程移动到锁池中，等待锁竞争。优先级高的线程竞争到对象锁的概率大，假若某线程没有竞争到该对象锁，它还会留在锁池中，唯有线程再次调用 wait()方法，它才会重新回到等待池中。而竞争到对象锁的线程则继续往下执行，直到执行完了 synchronized 代码块，它会释放掉该对象锁，这时锁池中的线程会继续竞争该对象锁。可以通过wait、notify、notifyAll实现多线程协调调用。如下代码。

```java
class TaskQueue{
    Queue<String> queue = new LinkedList<>();

    public synchronized void addTask(String s){
        this.queue.add(s);
        this.notifyAll();
    }

    public synchronized String getTask() throws InterruptedException{
        while (queue.isEmpty()) {
            this.wait();
        }
        return queue.remove();
    }
}

```

## 线程中断
中断一个线程可以在其他线程中对目标线程调用interrupt()方法，目标线程需要反复检测自身状态是否是interrupted状态，如果是，就立刻结束运行。如果调用interrupt()的目标线程中正在调用join()方法等待其它线程完成，join()方法会立刻抛出InterruptedException。
另一个常用的中断线程的方法是设置标志位。如下代码是两种方法的代码示例。

```java
class ThreadI1 extends Thread {

    @Override
    public void run() {
        while (!isInterrupted()) {
            System.out.println("in ThreadI1");
        }
    }
}

class ThreadI2 extends Thread {
    public volatile boolean running = true;

    @Override
    public void run() {
        while (running) {
            System.out.println("in ThreadI2");
        }
    }
}

class ThreadInterruptedTest {
    public void interrupt1() throws InterruptedException {
        Thread t1 = new Thread1();
        Thread.sleep(1);
        t1.interrupt();
        t1.join();
        System.out.println("end");
    }

    public void interrupt2() throws InterruptedException {
        ThreadI2 t2 = new ThreadI2();
        Thread.sleep(1);
        t2.running = false;
        System.out.println("end");
    }
}
```
中断线程的应用场景：假设从网络下载一个100M的文件，如果网速很慢，用户等得不耐烦，就可能在下载过程中点取消，这时，程序就需要中断下载线程的执行。线程池中也需要一些中断线程。

## ThreadLocal

线程局部变量是局限于线程内部的变量，属于线程自身所有，不在多个线程间共享。Java提供ThreadLocal类来支持线程的局部变量，是一种实现线程安全的方式。但是在管理环境下（如 web 服务器）使用线程局部变量的时候要特别小心，在这种情况下，工作线程的生命周期比任何应用变量的生命周期都要长。任何线程局部变量一旦在工作完成后没有释放，Java 应用就存在内存泄露的风险。注意点如下：
- 可以把ThreadLocal看成一个全局Map<Thread, Object>，每个线程获取ThreadLocal变量时，总是使用Thread自身作为key。
- ThreadLocal相当于给每个线程都开辟了一个独立的存储空间，各个线程的ThreadLocal关联的实例互不干扰。
- ThreadLocal一定要在finally中清除，因为当前线程执行完相关代码后，很可能会被重新放入线程池中，如果ThreadLocal没有被清除，该线程执行其他代码时，会把上一次的状态带进去。
- ThreadLocal表示线程的局部变量，它确保每个线程的ThreadLocal变量都是各自独立的；
- ThreadLocal适合在一个线程的处理流程中保持上下文（避免了同一参数在所有方法中传递）；
- 使用ThreadLocal可以使用try ... finally结构，并在finally中清除。

```java
class ThreadLocalTest {
    static class Context {
        private String name;
        private int age;
        //...
    }

    static ThreadLocal<Context> threadLocal = new ThreadLocal<>();

    public void process() {
        try {
            threadLocal.set(new Context());
            checkPermission();
            doWork();
            saveStatus();
            sendResponse();
        } finally {
            threadLocal.remove();
        }
    }

    void checkPermission() {
    }

    void doWork() {
        Context ctx = threadLocal.get();
    }

    void saveStatus() {
        Context ctx = threadLocal.get();
    }

    void sendResponse() {
    }

}
```

## 线程池
### 线程池概念
线程池Thread Pool是一种基于池化思想管理线程的工具，经常出现在多线程服务器中，如MySQL。线程过多会带来额外的开销，其中包括创建销毁线程的开销、调度线程的开销（多个线程的上下文切换）等等。线程池维护多个线程，等待监督管理者分配可并发执行的任务。使用线程池可以带来一系列好处，包括如下：
- 降低资源消耗：通过池化技术重复利用已创建的线程，降低线程创建和销毁造成的损耗。
- 提高响应速度：任务到达时，无需等待线程创建即可立即执行。
- 提高线程的可管理性：线程是稀缺资源，如果无限制创建，不仅会消耗系统资源，还会因为线程的不合理分布导致资源调度失衡，降低系统的稳定性。使用线程池可以进行统一的分配、调优和监控。
- 提供更多更强大的功能：线程池具备可拓展性，允许开发人员向其中增加更多的功能。比如延时定时线程ScheduledThreadPoolExecutor，就允许任务延期执行或定期执行。

基于基于池化思想原理的其它池子包括：
- 内存池(Memory Pooling)：预先申请内存，提升申请内存速度，减少内存碎片。
- 连接池(Connection Pooling)：预先申请数据库连接，提升申请连接的速度，降低系统的开销。
- 实例池(Object Pooling)：循环使用对象，减少资源在初始化和释放时的昂贵损耗。

### 线程池的生命周期
线程池的生命周期有如下状态，以及其转换如下。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201216150610222.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d4ZDEyMzQ1Njc4OTA=,size_16,color_FFFFFF,t_70)![在这里插入图片描述](https://img-blog.csdnimg.cn/2020121615062387.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d4ZDEyMzQ1Njc4OTA=,size_16,color_FFFFFF,t_70)

### 线程池实现的分类
java内置的线程池实现如下。
- newFixedThreadPool(int nThreads)创建一个固定大小的线程池，每当提交一个任务就创建一个线程，直到达到线程池的最大数量，这时线程规模将不再变化，当线程发生未预期的错误而结束时，线程池会补充一个新的线程。

```java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

```
- newCachedThreadPool()创建一个可缓存的线程池，如果线程池的规模超过了处理需求，将自动回收空闲线程，而当需求增加时，则可以自动添加新线程，线程池的规模不存在任何限制。

```java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

- newSingleThreadExecutor()这是一个单线程的Executor，它创建单个工作线程来执行任务，如果这个线程异常结束，会创建一个新的来替代它；它的特点是能确保依照任务在队列中的顺序来串行执行。
- newScheduledThreadPool(int corePoolSize)创建了一个固定长度的线程池，而且以延迟或定时的方式来执行任务，类似于Timer定时器。

如下，其中两个常用的线程池的应用。

```java
class ThreadPoolTest {
    static class Thread1 implements Runnable {
        @Override
        public void run() {
            System.out.println("in t1");
        }
    }

    public void test() {
        //
        ExecutorService exec = Executors.newFixedThreadPool(2);
        exec.submit(new Thread1());
        //
        ExecutorService exec2 = Executors.newCachedThreadPool();
        exec2.submit(new Thread1());
    }
}
```



### ThreadPoolExecutor继承关系
java的原生线程池的核心实现是ThreadPoolExecutor，其继承关系如下图。ThreadPoolExecutor实现的顶层接口是Executor，顶层接口Executor提供了一种思想：将任务提交和任务执行进行解耦。用户无需关注如何创建线程，如何调度线程来执行任务，用户只需提供Runnable对象，将任务的运行逻辑提交到执行器(Executor)中，由Executor框架完成线程的调配和任务的执行部分。ExecutorService接口增加了一些能力：（1）扩充执行任务的能力，补充可以为一个或一批异步任务生成Future的方法；（2）提供了管控线程池的方法，比如停止线程池的运行。AbstractExecutorService则是上层的抽象类，将执行任务的流程串联了起来，保证下层的实现只需关注一个执行任务的方法即可。最下层的实现类ThreadPoolExecutor实现最复杂的运行部分，ThreadPoolExecutor将会一方面维护自身的生命周期，另一方面同时管理线程和任务，使两者良好的结合从而执行并行任务。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201216145034448.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d4ZDEyMzQ1Njc4OTA=,size_16,color_FFFFFF,t_70)

### ThreadPoolExecutor构造器参数
如下图是ThreadPoolExecutor的其中一个构造器。
```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) 
```

| 项目            | Value                                                        |
| --------------- | ------------------------------------------------------------ |
| corePoolSize    | 核心池的大小，这个参数跟后面讲述的线程池的实现原理有非常大的关系。在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了prestartAllCoreThreads()或者prestartCoreThread()方法，从这2个方法的名字就可以看出，是预创建线程的意思，即在没有任务到来之前就创建corePoolSize个线程或者一个线程。默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中； |
| maximumPoolSize | 线程池最大线程数，表示在线程池中最多能创建多少个线程；       |
| keepAliveTime   | 表示线程没有任务执行时最多保持多久时间会终止。默认情况下，只有当线程池中的线程数大于corePoolSize时，keepAliveTime才会起作用，直到线程池中的线程数不大于corePoolSize，即当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize。但是如果调用了allowCoreThreadTimeOut(boolean)方法，在线程池中的线程数不大于corePoolSize时，keepAliveTime参数也会起作用，直到线程池中的线程数为0； |
| unit            | 参数keepAliveTime的时间单位，默认ms                          |
| workQueue       | 工作队列                                                     |
| threadFactory   | 创建线程的工厂                                               |
| handler         | 当线程数达到最大值后的拒绝策略，可自定义，也可使用内置的策略。 |
### 线程池运行机制
线程池的总体工作机制如下图所示。线程池在内部实际上构建了一个生产者消费者模型，将线程和任务两者解耦，并不直接关联，从而良好的缓冲任务，复用线程。**线程池的运行主要分成两部分：任务管理、线程管理**。任务管理部分充当生产者的角色，当任务提交后，线程池会判断该任务后续的流转：（1）直接申请线程执行该任务；（2）缓冲到队列中等待线程执行；（3）拒绝该任务。线程管理部分是消费者，它们被统一维护在线程池内，根据任务请求进行线程的分配，当线程执行完任务后则会继续获取新的任务去执行，最终当线程获取不到任务的时候，线程就会被回收。
![。。。](https://img-blog.csdnimg.cn/20201216150353884.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d4ZDEyMzQ1Njc4OTA=,size_16,color_FFFFFF,t_70)
#### 任务调度
任务调度是线程池的主要入口，当用户提交了一个任务，接下来这个任务将如何执行都是由这个阶段决定的。
首先，所有任务的调度都是由execute方法完成的，这部分完成的工作是：检查现在线程池的运行状态、运行线程数、运行策略，决定接下来执行的流程，是直接申请线程执行，或是缓冲到队列中执行，亦或是直接拒绝该任务。其执行过程如下：
1. 首先检测线程池运行状态，如果不是RUNNING，则直接拒绝，线程池要保证在RUNNING的状态下执行任务。
2. 如果workerCount （工作线程数）< corePoolSize，则创建并启动一个线程来执行新提交的任务。
3. 如果workerCount >= corePoolSize，且线程池内的阻塞队列未满，则将任务添加到该阻塞队列中。
4. 如果workerCount >= corePoolSize && workerCount < maximumPoolSize，且线程池内的阻塞队列已满，则创建并启动一个线程来执行新提交的任务。
5. 如果workerCount >= maximumPoolSize，并且线程池内的阻塞队列已满, 则根据拒绝策略来处理该任务, 默认的处理方式是直接抛异常。
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201216150848528.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d4ZDEyMzQ1Njc4OTA=,size_16,color_FFFFFF,t_70)

#### 任务缓冲
任务缓冲模块是线程池能够管理任务的核心部分。线程池的本质是对任务和线程的管理，而做到这一点最关键的思想就是将任务和线程两者解耦，不让两者直接关联，才可以做后续的分配工作。线程池中是以生产者消费者模式，通过一个阻塞队列来实现的。阻塞队列缓存任务，工作线程从阻塞队列中获取任务。
阻塞队列(BlockingQueue)是一个支持两个附加操作的队列。这两个附加的操作是：在队列为空时，获取元素的线程会等待队列变为非空。当队列满时，存储元素的线程会等待队列可用。阻塞队列常用于生产者和消费者的场景，生产者是往队列里添加元素的线程，消费者是从队列里拿元素的线程。阻塞队列就是生产者存放元素的容器，而消费者也只从队列里拿元素。使用不同的队列可以实现不一样的任务存取策略。jdk提供多种阻塞队列，包括同步队列、有界队列、无界队列、优先级队列。如下表所示。
| 实现类                | 类型       | 说明                                                         |
| --------------------- | ---------- | ------------------------------------------------------------ |
| SynchronousQueue      | 同步队列   | 该队列不存储元素，每个插入操作必须等待另一个线程调用移除操作，否则插入操作会一直阻塞 |
| ArrayBlockingQueue    | 有界队列   | 基于数组的阻塞队列，按照 FIFO 原则对元素进行排序             |
| LinkedBlockingQueue   | 无界队列   | 基于链表的阻塞队列，按照 FIFO 原则对元素进行排序             |
| PriorityBlockingQueue | 优先级队列 | 具有优先级的阻塞队列                                         |

#### 任务申请
任务的执行有两种可能：一种是任务直接由新创建的线程执行。另一种是线程从任务队列中获取任务然后执行，执行完任务的空闲线程会再次去从队列中申请任务再去执行。第一种情况仅出现在线程初始创建的时候，第二种是线程获取任务绝大多数的情况。线程需要从任务缓存队列中不断地取任务执行。这部分策略由getTask方法实现。
#### 任务拒绝
任务拒绝模块是线程池的保护部分，线程池有一个最大的容量，当线程池的任务缓存队列已满，并且线程池中的线程数目达到maximumPoolSize时，就需要拒绝掉该任务，采取任务拒绝策略，保护线程池。拒绝策略是一个接口，用户可以通过实现这个接口去定制拒绝策略，也可以选择JDK提供的四种已有拒绝策略，如下图所示内置的拒绝策略。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201216150124865.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d4ZDEyMzQ1Njc4OTA=,size_16,color_FFFFFF,t_70)

### Worker线程
ThreadPoolExecutor类中内置一个包装类Worker，如下源码。

```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;

        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }
}
```

一个 Worker 代表一个线程。线程池内部用一个 HashSet 集合管理这些所有的工作线程。Worker这个工作线程，实现了Runnable接口，并持有一个线程thread，一个初始化的任务firstTask。thread是在调用构造方法时通过ThreadFactory来创建的线程，可以用来执行任务；firstTask用它来保存传入的第一个任务，这个任务可以有也可以为null。如果这个值是非空的，那么线程就会在启动初期立即执行这个任务，也就对应核心线程创建时的情况；如果这个值是null，那么需要从任务队列（workQueue）取任务。如下图。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020121615271980.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d4ZDEyMzQ1Njc4OTA=,size_16,color_FFFFFF,t_70)

#### worker线程回收
线程回收的工作是在processWorkerExit方法完成的。
线程池需要管理线程的生命周期，需要在线程长时间不运行的时候进行回收。线程池使用一张Hash表去持有线程的引用，这样可以通过添加引用、移除引用这样的操作来控制线程的生命周期。这个时候重要的就是如何判断线程是否在运行。​Worker是通过继承AQS AbstractQueuedSynchronizer，使用AQS来实现独占锁这个功能。没有使用可重入锁ReentrantLock，而是使用AQS，为的就是实现不可重入的特性去查看线程现在的执行状态。
1.lock方法**一旦获取了独占锁，表示当前线程正在执行任务中。** 2.如果正在执行任务，则不应该中断线程。 3.如果该线程现在不是独占锁的状态，也就是空闲的状态，说明它没有在处理任务，这时可以对该线程进行中断。 4.线程池在执行shutdown方法或tryTerminate方法时会调用interruptIdleWorkers方法来中断空闲的线程，interruptIdleWorkers方法会使用tryLock方法来判断线程池中的线程是否是空闲状态；如果线程是空闲状态则可以安全回收。
线程池中线程的销毁依赖JVM自动的回收，线程池做的工作是根据当前线程池的状态维护一定数量的线程引用，防止这部分线程被JVM回收，当线程池决定哪些线程需要回收时，只需要将其引用消除即可。Worker被创建出来后，就会不断地进行轮询，然后获取任务去执行，核心线程可以无限等待获取任务，非核心线程要限时获取任务。当Worker无法获取到任务，也就是获取的任务为空时，循环会结束，Worker会主动消除自身在线程池内的引用。
#### Worker线程增加
增加线程是通过线程池中的addWorker方法，该方法的功能就是增加一个线程，该方法不考虑线程池是在哪个阶段增加的该线程，该步骤仅仅完成增加线程，并使它运行，最后返回是否成功这个结果。addWorker方法有两个参数：firstTask、core。firstTask参数用于指定新增的线程执行的第一个任务，该参数可以为空；core参数为true表示在新增线程时会判断当前活动线程数是否少于corePoolSize，false表示新增线程前需要判断当前活动线程数是否少于maximumPoolSize，
#### Worker线程执行任务
在Worker类中的run方法调用了runWorker方法来执行任务，runWorker方法的执行过程如下：
1. while循环不断地通过getTask()方法获取任务。
 2. getTask()方法从阻塞队列中取任务。
 3. 如果线程池正在停止，那么要保证当前线程是中断状态，否则要保证当前线程不是中断状态。  
 4. 执行任务。 
 5. 如果getTask结果为null则跳出循环，执行processWorkerExit()方法，销毁线程。

### 线程池应用
#### 线程池应用场景
1、快速响应用户。
用户发起的实时请求，服务追求响应时间。比如说用户要查看一个商品的信息，那么我们需要将商品维度的一系列信息如商品的价格、优惠、库存、图片等等聚合起来，展示给用户。使用线程池这种简单的方式，将调用封装成任务并行的执行，缩短总体响应时间。这种情况不设置队列去缓冲并发任务，调高corePoolSize和maxPoolSize去尽可能创造多的线程快速执行任务。
2、快速处理批量任务。
这种场景需要执行大量的任务，任务执行的越快越好。这种情况下，也应该使用多线程策略，并行计算。但与响应速度优先的场景区别在于，这类场景任务量巨大，并不需要瞬时的完成，而是关注如何使用有限的资源，尽可能在单位时间内处理更多的任务，也就是吞吐量优先的问题。所以应该设置队列去缓冲并发任务，调整合适的corePoolSize去设置处理任务的线程数。在这里，设置的线程数过多可能还会引发线程上下文切换频繁的问题，也会降低处理任务的速度，降低吞吐量。

#### 线程池使用问题
1. 线程池的参数并不好配置
  一方面线程池的运行机制不是很好理解，配置合理需要强依赖开发人员的个人经验和知识；另一方面，线程池执行的情况和任务类型相关性较大，IO密集型和CPU密集型的任务运行起来的情况差异非常大，这导致业界并没有一些成熟的经验策略帮助开发人员参考。

2. 线程池隔离
  如果我们很多业务都依赖于同一个线程池,当其中一个业务因为各种不可控的原因消耗了所有的线程，导致线程池全部占满。这样其他的业务也就不能正常运转了，这对系统的打击是巨大的。比如web服务器Tomcat 接受请求的线程池，假设其中一些响应特别慢，线程资源得不到回收释放；线程池慢慢被占满，最坏的情况就是整个应用都不能提供服务。所以需要将线程池进行隔离。通常的做法是按照业务进行划分：比如下单的任务用一个线程池，获取数据的任务用另一个线程池。这样即使其中一个出现问题把线程池耗尽，那也不会影响其他的任务运行。

3. 动态化线程池（解决方案之一）
- 动态调参：支持线程池参数动态调整、界面化操作；在配置中心调整包括修改线程池核心大小、最大核心大小、阻塞队列大小等；参数修改后及时生效。 
- 任务监控：支持应用粒度、线程池粒度、任务粒度的Transaction事务监控；可以看到线程池的任务执行情况、最大任务执行时间、平均任务执行时间等。 
- 负载告警：线程池队列任务积压到一定值的时候会通过告知应用开发负责人；当线程池负载数达到一定阈值的时候会通过大象告知应用开发负责人。 
- 操作监控：创建/修改和删除线程池都会通知到应用的开发负责人。
- 操作日志：可以查看线程池参数的修改记录，谁在什么时候修改了线程池参数、修改前的参数值是什么。 
- 权限校验：只有应用开发负责人才能够修改应用的线程池参数。

#### tomcat中的线程池
首先了解两个概念，任务可以分为cpu密集型任务和io密集型任务。
- cpu 密集型任务： 需要线程长时间进行的复杂的运算，这种类型的任务需要少创建线程，过多的线程将会频繁引起上文切换，降低任务处理处理速度。
- io 密集型任务：由于线程并不是一直在运行，可能大部分时间在等待 IO 读取/写入数据，增加线程数量可以提高并发度，尽可能多处理任务。web应用大部分是io密集型任务，大部分时间在网络io、数据库io、文件io等，实际的cpu计算时间相对较少。

JDK实现线程池功能比较完善，但是比较适合运行 CPU 密集型任务，不适合 IO 密集型的任务。
Tomcat/Jetty作为web常用框架，需要处理大量客户端请求任务，如果采用原生线程池，一旦接受请求数量大于线程池核心线程数，这些请求就会被放入到队列中，等待核心线程处理，用户体验不好。这样做显然降低这些请求总体处理速度，所以两者都没采用 JDK 原生线程池。Jetty直接不用自己写了。Tomcat在原生连接池基础上进行了扩展（具体待研究？？？）。

## 锁机制

### 线程安全
线程安全是指要控制多个线程对某个共享资源的有序访问或修改，而在这些线程之间没有产生冲突。在Java里，线程安全一般体现在两个方面：
1. 多个thread对同一个java实例的访问（read和modify）不会相互干扰，它主要体现在关键字synchronized。如ArrayList和Vector，HashMap和Hashtable（后者每个方法前都有synchronized关键字）。如果你在interator一个List对象时，其它线程remove一个element，问题就出现了。
2. 每个线程都有自己的字段，而不会在多个线程之间共享。它主要体现在ThreadLocal类。

保证多线程安全：
原子性：提供互斥访问，同一时刻只能有一个线程对数据进行操作，（atomic,synchronized）；
可见性：一个线程对主内存的修改可以及时地被其他线程看到，（synchronized,volatile）；
有序性：一个线程观察其他线程中的指令执行顺序，由于指令重排序，该观察结果一般杂乱无序，（happens-before原则）。

### synchronized
synchronized可以保证方法或者代码块在运行时，同一时刻只有一个线程可以执行临界区，同时它还可以保证共享变量的内存可见性。synchronized作用的对象包括：普通同步方法，锁是当前实例对象；静态同步方法，锁是当前类的class对象；同步方法块，锁是括号里面的对象

### volatile
在JVM中，变量的值保存在主内存中，但是，当线程访问变量时，它会先获取一个副本，并保存在自己的工作内存中。如果线程修改了变量的值，虚拟机会在某个时刻把修改后的值回写到主内存，但是，这个时间是不确定的。
这会导致如果一个线程更新了某个变量，另一个线程读取的值可能还是更新前的。例如，主内存的变量a = true，线程1执行a = false时，它在此刻仅仅是在该线程的工作的内存中把变量a的副本变成了false，主内存的变量a还是true，在JVM把修改后的a回写到主内存之前，其他线程读取到的a的值仍然是true，这就造成了多线程之间共享的变量不一致。
因此，volatile关键字的目的是告诉虚拟机：
- 每次访问变量时，总是获取主内存的最新值；
- 每次修改变量后，立刻回写到主内存。

volatile关键字解决的是可见性问题：当一个线程修改了某个共享变量的值，其他线程能够立刻看到修改后的值。

### 常用的Lock
- **ReentrantLock**
- **ReadWriteLock**
- **StampedLock**

java上述的lock的应用代码如下。
```java
/////////////////////////ReentrantLock
class Counter {
    private final Lock lock = new ReentrantLock();
    private int count;

    public void add(int i) throws InterruptedException {
        if (lock.tryLock(1, TimeUnit.SECONDS)) {
            try {
                count += i;
            } finally {
                lock.unlock();
            }
        }
    }
}

/////////////////////////ReentrantLock && Condition 
class TaskQueueUseLock {
    private final Lock lock = new ReentrantLock();
    private final Condition condition = lock.newCondition();
    private Queue<String> queue = new LinkedList<>();

    public void addTask(String s) {
        lock.lock();
        try {
            queue.add(s);
            condition.signalAll();
        } finally {
            lock.unlock();
        }
    }

    public String getTask() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                condition.await();
            }
            return queue.remove();

        } finally {
            lock.unlock();
        }
    }
}

/////////////////////////ReadWriteLock 
class CounterReadWrite {
    private final ReadWriteLock rwlock = new ReentrantReadWriteLock();
    private final Lock rlock = rwlock.readLock();
    private final Lock wlock = rwlock.writeLock();
    private int[] counts = new int[10];

    public void inc(int i) {
        wlock.lock();
        try {
            counts[i] += 1;
        } finally {
            wlock.unlock();
        }
    }

    public int[] get() {
        rlock.lock();
        try {
            return Arrays.copyOf(counts, counts.length);
        } finally {
            rlock.unlock();
        }
    }

}

/////////////////////////StampedLock 
class Point {
    private final StampedLock stampedLock = new StampedLock();
    private double x;
    private double y;

    public void move(double ix, double iy) {
        long stamp = stampedLock.writeLock();
        try {
            x += ix;
            y += iy;
        } finally {
            stampedLock.unlock(stamp);
        }
    }

    public double getDistance() {
        //get Optimistic read lock
        long stamp = stampedLock.tryOptimisticRead();
        double currentX = x;
        double currentY = y;
        //check value modify by other thread
        if (!stampedLock.validate(stamp)) {
            // get pessimistic read lock
            stamp = stampedLock.tryReadLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                stampedLock.unlock(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```
1. ReentrantLock

  **可重入锁：**JVM允许同一个线程重复获取同一个锁，这种能被同一个线程反复获取的锁，就叫做可重入锁。
  由于Java的线程锁是可重入锁，所以，获取锁的时候，不但要判断是否是第一次获取，还要记录这是第几次获取。每获取一次锁，记录+1，每退出synchronized块，记录-1，减到0的时候，才会真正释放锁。

  使用ReentrantLock可重入锁比直接使用synchronized更安全，可以替代synchronized进行线程同步。ReentrantLock和Condition可以配合用来来实现线程的协调任务。Condition提供的await()、signal()、signalAll()原理和synchronized锁对象的wait()、notify()、notifyAll()是一致的，并且其行为也是一样的：
- await()会释放当前锁，进入等待状态；
- signal()会唤醒某个等待线程；
- signalAll()会唤醒所有等待线程， 唤醒线程从await()返回后需要重新获得锁。
  此外，和tryLock()类似，await()可以在等待指定时间后，如果还没有被其他线程通过signal()或signalAll()唤醒，

2. ReadWriteLock
  实际上我们想要的是：允许多个线程同时读，但只要有一个线程在写，其他线程就必须等待（悲观读）。使用ReadWriteLock可以解决这个问题，它保证：只允许一个线程写入（其他线程既不能写入也不能读取）；没有写入时，多个线程允许同时读（提高性能）。把读写操作分别用读锁和写锁来加锁，在读取时，多个线程可以同时获得读锁，这样就大大提高了并发读的执行效率。ReadWriteLock适合读多写少的场景。如场景论坛。

3. StampedLock
  ReadWriteLock存在一个潜在的问题：如果有线程正在读，写线程需要等待读线程释放锁后才能获取写锁，即读的过程中不允许写，这是一种悲观的读锁。要进一步提升并发执行效率，Java 8引入了新的读写锁：StampedLock。**StampedLock和ReadWriteLock相比，改进之处在于：读的过程中也允许获取写锁后写入**。这样一来，我们读的数据就可能不一致，所以，需要一点额外的代码来判断读的过程中是否有写入，这种读锁是一种乐观锁。
  **乐观锁的意思就是乐观地估计读的过程中大概率不会有写入，因此被称为乐观锁。反过来，悲观锁则是读的过程中拒绝有写入，也就是写入必须等待**。显然乐观锁的并发效率更高，但一旦有小概率的写入导致读取的数据不一致，需要能检测出来，再读一遍就行。
  和ReadWriteLock相比，写入的加锁是完全一样的，不同的是读取。上述StampedLock代码中首先我们通过tryOptimisticRead()获取一个乐观读锁，并返回版本号。接着进行读取，读取完成后，我们通过validate()去验证版本号，如果在读取过程中没有写入，版本号不变，验证成功，我们就可以放心地继续后续操作。如果在读取过程中有写入，版本号会发生变化，验证将失败。在失败的时候，我们再通过获取悲观读锁再次读取。由于写入的概率不高，程序在绝大部分情况下可以通过乐观读锁获取数据，极少数情况下使用悲观读锁获取数据。
  可见，StampedLock把读锁细分为乐观读和悲观读，能进一步提升并发效率。但这也是有代价的：一是代码更加复杂，二是StampedLock是不可重入锁，不能在一个线程中反复获取同一个锁。

### 多线程锁升级
JVM优化synchronized的运行机制，当JVM检测到不同的竞争状态时，就会根据需要自动切换到合适的锁，这种切换就是锁的升级。升级是不可逆的，也就是说只能从低到高，也就是偏向-->轻量级-->重量级，不能够降级
Java中锁升级的最佳实例就是synchronized，synchronized把锁信息存放在对象头的MarkWord中。在早期的jdk版本中，synchronized是一个重量级锁，保证线程的安全但是效率很低。后来对synchronized进行了优化，有了一个锁升级的过程：
无锁态-->偏向锁-->轻量级锁（自旋锁）-->重量级锁
锁升级过程详解：
当给一个对象增加synchronized锁之后，相当于上了一个偏向锁。当有一个线程去请求时，就把这个对象MarkWord的ID改为当前线程指针ID（JavaThread），只允许这一个线程去请求对象。
当有其他线程也去请求时，就把锁升级为轻量级锁。每个线程在自己的线程栈中生成LockRecord，用CAS自旋操作将请求对象MarkWord ID改为自己的LockRecord，成功的线程请求到了该对象，未成功的对象继续自旋。
如果竞争加剧，当有线程自旋超过一定次数时，就将轻量级锁升级为重量级锁，线程挂起，进入等待队列，等待操作系统的调度。
？？？

### 死锁
在获取多个锁的时候，不同线程获取多个不同对象的锁可能导致死锁。对于如下释掉的代码注，线程1和线程2如果分别执行add()和dec()方法时：
1. 开始：线程1：进入add()，获得lockA；线程2：进入dec()，获得lockB。
2. 随后：线程1：已经获取lockA，准备获得lockB，失败，等待中；线程2：已经获得lockB，准备获得lockA，失败，等待中。
  此时，**两个线程各自持有不同的锁，然后各自试图获取对方手里的锁，造成了双方无限等待下去，这就是死锁**。

防止死锁：
- 线程获取锁的顺序要一致。优化后的代码正确顺序如下。
- 采用Lock对象锁，获取锁设置了超时时间。
- 。。。

```java
class LockTest {
    private Object lockA = new Object();
    private Object lockB = new Object();
    private int value = 0;
    private int another = 0;

    public void add(int m) {
        synchronized (lockA) { // 获得lockA的锁
            this.value += m;
            synchronized (lockB) { // 获得lockB的锁
                this.another += m;
            } // 释放lockB的锁
        } // 释放lockA的锁
    }

    // may dead lock
    /*    public void dec(int m) {
        synchronized (lockB) { // 获得lockB的锁
            this.another -= m;
            synchronized (lockA) { // 获得lockA的锁
                this.value -= m;
            } // 释放lockA的锁
        } // 释放lockB的锁
    }*/

    //优化后的正确顺序
    public void dec(int m) {
        synchronized (lockA) {
            this.another -= m;
            synchronized (lockB) {
                this.value -= m;
            } // 释放lockA的锁
        } // 释放lockB的锁
    }

}
```

### 悲观锁与乐观锁
乐观锁悲观锁是一种思想。可以用在很多方面。
比如数据库方面。悲观锁就是forupdate（锁定查询的行）乐观锁就是version字段（比较跟上一次的版本号，如果一样则更新，如果失败则要重复读-比较-写的操作。）
JDK方面：悲观锁就是sync乐观锁就是原子类（内部使用CAS实现）
本质来说，就是悲观锁认为总会有人抢我的。乐观锁就认为，基本没人抢。

独占锁是一种悲观锁，synchronized就是一种独占锁，会导致其它所有需要锁的线程挂起，等待持有锁的线程释放锁。而另一个更加有效的锁就是乐观锁。所谓乐观锁就是，每次不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止。乐观锁用到的机制就是CAS，Compare and Swap。 

### CAS
CAS（Compare and Swap）。比如你要操作一个变量，他的值为A，你希望将他修改为B，这期间不会进行加锁，当你在修改的时候，你发现值仍旧是A，然后将它修改为B，如果此时值被其他线程修改了，变成了C，那么将不会进行值B的写入操作，这就是CAS的核心理论。CAS 是 compare and swap 的缩写，即我们所说的比较交换。

CAS是一种基于锁的操作，而且是乐观锁。在 java 中锁分为乐观锁和悲观锁。悲观锁是将资源锁住，等一个之前获得锁的线程释放锁之后，下一个线程才可以访问。而乐观锁采取了一种宽泛的态度，通过某种方式不加锁来处理资源，比如通过给记录加 version 来获取数据，性能较悲观锁有很大的提高。

CAS 操作包含三个操作数 —— 内存位置（V）、预期原值（A）和新值(B)。如果内存地址里面的值和 A 的值是一样的，那么就将内存里面的值更新成 B。CAS是通过无限循环来获取数据的，若果在第一轮循环中，a 线程获取地址里面的值被b 线程修改了，那么 a 线程需要自旋，到下次循环才有可能机会执行。

CAS工作原理：

UnSafe类和自旋锁。
 UnSafe类在jdk的rt.jar下面的一个类，全包名是sun.misc.UnSafe 。这个类大多数方法都是native方法。由于Java不能操作计算机系统，所以设计之初就留了一个UnSafe类。通过UnSafe类，Java就可以操作指定内存地址的数据。调用UnSafe类的CAS，JVM会帮我们实现出汇编指令，从而实现原子操作。

所谓的自旋，其实就是上面getAndAddInt方法中的do while循环操作。当预期值和主内存中的值不等时，就重新获取主内存中的值，这就是自旋。 

java.util.concurrent.atomic 包下的类大多是使用 CAS 操作来实现的(AtomicInteger,AtomicBoolean,AtomicLong)。

CAS乐观锁：

乐观锁是一种思想，即认为读多写少，遇到并发写的可能性比较低，所以采取在写时先读出当前版本号，然后加锁操作（比较跟上一次的版本号，如果一样则更新），如果失败则要重复读-比较-写的操作。
CAS是一种更新的原子操作，比较当前值跟传入值是否一样，一样则更新，否则失败。CAS顶多算是乐观锁写那一步操作的一种实现方式罢了，不用CAS自己加锁也是可以的。

**CAS缺点：**
 **1、循环时间长，开销大。**
 synchronized是加锁，同一时间只能一个线程访问，并发性不好。而CAS并发性提高了，但是由于CAS存在自旋操作，即do while循环，如果CAS失败，会一直进行尝试。如果CAS长时间不成功，会给CPU带来很大的开销。

**2、只能保证一个共享变量的原子性。**
 上面也看到了，getAndAddInt方法的val1是代表当前对象，所以它也就是能保证这一个共享变量的原子性。如果要保证多个，那只能加锁了。

**3、引来的ABA问题。**

假设现在主内存中的值是A，现有t1和t2两个线程去对其进行操作。t1和t2先将A拷贝回自己的工作内存。这个时候t2线程将A改成B，刷回到主内存。此刻主内存和t2的工作内存中的值都是B。接下来还是t2线程抢到执行权，t2又把B改回A，并刷回到主内存。这时t1终于抢到执行权了，自己工作内存中的值的A，主内存也是A，因此它认为没人修改过，就在工作内存中把A改成了X，然后刷回主内存。也就是说，在t1线程执行前，t2将主内存中的值由A改成B再改回A。这便是ABA问题

 解决：

ABA问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么A－B－A 就会变成1A-2B－3A。从Java1.5开始JDK的atomic包里提供了一个类AtomicStampedReference来解决ABA问题。这个类的compareAndSet方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。 



引用：

https://www.cnblogs.com/549294286/p/3766717.html
https://www.jianshu.com/p/8e74009684c7

**https://cloud.tencent.com/developer/article/1098115**

### Atomic

Atomic包中的类基本的特性就是在多线程环境下，当有多个线程同时对单个（包括基本类型及引用类型）变量进行操作时，具有排他性，即当多个线程同时对该变量的值进行更新时，仅有一个线程能成功，而未成功的线程可以向自旋锁一样，继续尝试，一直等到执行成功。
Java的java.util.concurrent包除了提供底层锁、并发集合外，还提供了一组原子操作的封装类，它们位于java.util.concurrent.atomic包。
以AtomicInteger为例，它提供的主要操作有：

- 增加值并返回新值：int addAndGet(int delta)
- 加1后返回新值：int incrementAndGet()
- 获取当前值：int get()
- 用CAS方式设置：int compareAndSet(int expect, int update)

CAS是指，在这个操作中，如果AtomicInteger的当前值是prev，那么就更新为next，返回true。如果AtomicInteger的当前值不是prev，就什么也不干，返回false。通过CAS操作并配合do ... while循环，即使其他线程修改了AtomicInteger的值，最终的结果也是正确的。
Atomic类是通过无锁（lock-free）的方式实现的线程安全（thread-safe）访问。它的主要原理是利用了CAS：Compare and Set。
利用AtomicLong可以编写一个多线程安全的全局唯一ID生成器。原子操作实现了无锁的线程安全；适用于计数器，累加器等。

```java
class AtomicTest {

    public int incrementAndGet(AtomicInteger var) {
        int prev, next;
        do {
            prev = var.get();
            next = prev + 1;
        } while (!var.compareAndSet(prev, next));
        return next;
    }

    AtomicLong var = new AtomicLong(0);

    public long getNextId() {
        return var.incrementAndGet();
    }
}
```

## 关注点

### synchronized和volatile

- volatile本质是在告诉jvm当前变量在寄存器（工作内存）中的值是不确定的，需要从主内存中读取，保证一个共享变量被一个线程修改，其他线程可见； synchronized则是锁定当前变量，只有当前线程可以访问该变量，其他线程被阻塞住。
- volatile仅能使用在变量级别；synchronized则可以使用在变量、方法（常规方法和静态方法）、和类级别的。
- volatile仅能实现变量的修改的可见性，不能保证原子性；而synchronized则可以保证变量的修改可见性和原子性。
- volatile不会造成线程的阻塞；synchronized可能会造成线程的阻塞。
- volatile标记的变量不会被编译器优化；synchronized标记的变量可以被编译器优化。

### synchronized和Lock

- synchronized是java内置关键字，在jvm层面，Lock是个java类；
- synchronized无法判断是否获取锁的状态，Lock可以判断是否获取到锁；
- synchronized会自动释放锁(线程执行完同步代码会释放锁 ；线程执行过程中发生异常会释放锁)，Lock需在finally中手工释放锁（unlock()方法释放锁），否则容易造成线程死锁；
- 用synchronized关键字的两个线程1和线程2，如果当前线程1获得锁，线程2线程等待。如果线程1阻塞，线程2则会一直等待下去，而Lock锁就不一定会等待下去，如果尝试获取不到锁超时，线程可以不用一直等待就结束了；
- synchronized的锁可重入、不可中断、非公平（啥意思？？？），而Lock锁可重入、可判断、可公平（两者皆可）；
- Lock锁适合大量同步的代码的同步问题，synchronized锁适合代码少量的同步问题。

### synchronized和ReentrantLock

1. synchronized是关键字，ReentrantLock是类，这是二者的本质区别。既然ReentrantLock是类，那么它就提供了比synchronized更多更灵活的特性，可以被继承、可以有方法、可以有各种各样的类变量，ReentrantLock可重入锁比synchronized的扩展性体体现如下： 

- ReentrantLock可以对获取锁的等待时间进行设置，这样就避免了死锁 
- ReentrantLock可以获取各种锁的信息
- ReentrantLock可以灵活地实现多路通知？？？

1. 二者的锁机制其实也是不一样的:ReentrantLock底层调用的是Unsafe的park方法加锁，synchronized操作的应该是对象头中mark word。java对象在内存中的存储方式见链接: [JVM基础](https://editor.csdn.net/md/?articleId=111184654).

## 其他

### CountDownLatch

### CyclicBarrier

### Semaphore

### Exchanger



















