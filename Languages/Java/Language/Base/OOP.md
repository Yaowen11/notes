### 面向对象编程

#### 结构

##### 对象

###### 对象特性

* 对象的行为（behavior）

  可以对对象进行的操作

* 对象的状态（state）

  每个对象都保存着描述当前特征的信息。行为改变状态

* 对象的标识（identity）

  每个对象都有一个唯一的身份

* 变量作用域

  实例变量和静态变量的作用域是整个类，如果一个局部变量和一个类变量具有相同的名字，那么局部变量优先，而同名的类变量将被隐藏

* 在类的内部，变量定义的先后顺序决定了初始化的顺序。即使变量定义散布于方法定义之间，仍会在任何方法（包括构造器）被调用之前得到初始化

###### 实例化

实例化流程：

1. 加载器加载类的字节码
2. 如果类存在基类，对它继续进行加载
3. 基类中的 static 初始化会被执行，然后是下一个导出类
4. 类加载完毕，开始创建对象
5. 对象中所有的基本类型都会被设为默认值，对象引用被设为 null（通过将对象内存设为二进制 0 值）
6. 基类的构造器会被调用（基类构造器以相同的顺序来经历相同的过程）
7. 基类构造器完成后，实例变量按其次序被初始化
8. 构造器的其余部分被执行

构造器应该尽可能简单，尽量避免调用其他方法，在构造器内唯一能够安全调用的那些方法是基类中的 final 方法

使用 new 实例化对象时，每次都会在堆上分配内存，对于装箱类型的基本数据建议使用 `value.of()` 创建装箱对象（会使用内部的缓存机制，默认为 -128 ～ 127（使用 `--xx:AutoBoxCacheMax=N` 来调整最大缓存值））

new 是强类型校验，可以调用任何构造方法，在使用 new 操作时，如果类未加载，会先加载类

使用反射实例化时 newInstance 方法是弱类型，只能调用无参数构造方法，如果没有默认构造方法，就抛出 *InstantiationException* 异常；如果此构造方法没有权限访问，则抛出 *IllegalAccessException* 异常。

###### 对象析构与 finalize 方法

finalize 被设计成在对象被垃圾收集前调用，finalize 的执行是和垃圾收集关联在一起的，一旦实现了非空的 finalize 方法，就会导致相应对象回收呈现数量级上的变慢。finalize 会生吞资源回收时的出错信息，推荐资源用完即显式释放，或者利用资源池来重用

不支持析构器。可以为任何一个类添加 finalize 方法。finalize 方法将在垃圾回收器清除对象之前调用。在实际应用中，不要依赖于使用 finalize 方法回收任何短缺的资源，因为很难知道这个方法什么时候才能够调用。调用 *System*.runFinalizersOnExit(true) 能够确保 finalizer 方法在虚拟机关闭前被调用。不过这个方法并不安全，替代的方法是使用方法 *Runtime*.addShutdownHook 添加关闭钩（shutdown hook）。从 9 开始 finalize 方法标注为 @Deprecated

Java 平台目前在逐步使用 `java.lang.ref.Cleaner` 来替换掉原有的 finalize 实现。Cleaner 实现利用了幻象引用。利用幻象引用和引用队列，可以保证对象被彻底销毁前做一些类似资源回收的工作（关闭文件描述符等），它比 finalize 更加轻量，更加可靠，每个 Cleaner 的操作都是独立的，它有自己的运行线程，可以避免意外死锁等问题，实践中可以为模块构建一个 Cleaner，然后实现相应的清理逻辑

```java
public class CleaningExample implements AutoCloseable {
    // cleaner 实现
    private static final Cleaner cleaner = Cleaner.create();
    static class State implements Runnable {
        @Override
        public void run() {
            // 清理逻辑，最多执行一次
        }
    }
    private final Cleaner.Cleanable cleanable;
    private final State state;
    public CleaningExample() {
        this.state = new State();
        this.cleanable = cleaner.register(this, state);
    }
    @Override
    public void close() throws Exception {
        cleanable.clean();
    }
}
```

从可预测性的角度来判断，Cleaner 或者幻象引用改善的程度仍然有限，如果由于其他原因导致幻象引用堆积，同样会出现问题，Cleaner 适合作为一种最后的保证手段，而不是完全依赖 Cleaner 进行资源回收

###### 修饰符

即使一个类只具有包访问权限，其 public main 仍然可访问

*对象成员修饰符*

|    修饰符    |  类   | 构造方法 | 实例方法 | 实例属性 | 静态块 | 静态方法 | 静态属性 |
| :----------: | :---: | :------: | :------: | :------: | :----: | :------: | :------: |
|   default    | true  |   true   |   true   |   true   |        |   true   |   true   |
|    public    | true  |   true   |   true   |   true   |        |   true   |   true   |
|  protected   | inner |   true   |   true   |   true   |        |   true   |   true   |
|   private    | inner |   true   |   true   |   true   |        |   true   |   true   |
|    static    | inner |          |   true   |   true   |  true  |   true   |   true   |
|    final     | true  |          |   true   |   true   |        |          |   true   |
|   abstract   | true  |          |   true   |          |        |          |          |
|    native    |       |          |   true   |          |        |   true   |          |
| synchronized |       |          |   true   |          |        |   true   |          |
|   strictfp   | true  |          |   true   |          |        |   true   |          |
|  transient   |       |          |          |   true   |        |          |   true   |

###### 方法调用

虚拟机调用过程：

1. 获取所有可能被调用的候选方法

   编译器查看对象的声明类型和方法名，编译器将会一一列举所有类中同名方法和其超类中访问属性为 protected 的同名方法

2. 重载解析（overloading resolution）

   查看调用方法提供的参数类型，从候选列表中选择一个与提供的参数类型完全匹配的方法。如果编译器没有找到与参数类型匹配的方法，或者发现经过类型转换后有多个方法与之匹配，就会报告一个错误。

   虚拟机预先为每个类创建了一个方法表（method table），其中列出了所有方法的签名和实际调用的方法。在真正调用方法的时候，虚拟机仅查找方法表

* 前期绑定

  在程序执行前进行绑定（由编译器和连接器实现），为面向过程的语言中不需要选择就默认的绑定方式

* 静态绑定

  private 方法，static 方法，final 方法或者构造器，那么编译器将可以准确地知道应该调用那个方法，编译时确定如何调用

* 动态绑定（多态）

  调用的方法依赖于隐式参数的实际类型，并且在运行时实现动态绑定（指向父类或接口的引用，可以指向子类或实现类）

##### 类

###### 结构

 * 静态成员
   
   类所有实例对象共享静态成员，静态方法不能访问类实例成员，静态方法只能访问类静态成员，静态变量将变量值存储在一个公共的内存地址，如果某一个对象修改了静态变量的值，那么同一个类的所有对象都会受到影响。

 * 代码块

    * 静态初始化块

      如果对类的静态域进行初始化的代码比较复杂，那么可以使用静态的初始化块，在类第一次加载的时候，将会进行静态域的初始化。所有的静态初始化语句以及静态初始化块都将按照类定义的顺序执行，但先于 main 函数执行

    * 实例化初始化块

      非 static 代码块，会在对象实例化时执行，先于构造函数执行，不推荐使用

  * 包结构

    嵌套的包之间没有任何关系，一个类可以使用所属包中的所有类，以及其他包中的公有类。导入包中类的关键字是 import，定义包的关键字是 package，如果没有在源文件中定义包，这个源文件的类就被放置在一个默认包中，如果类在一个包里，包的名字也会作为类名的一部分。

###### 类路径

类存储在文件系统的子目录中，类的路径必须与包名匹配，否则可以编译，但不能运行，虚拟机找不到类。类路径是所有包含类文件的路径的集合：

* UNIX 环境，类路径中不同项目之间采用冒号 : 分隔

  ```shell
  java -classpath .:/path/to/thrid.jar Class
  // SE 6 开始，可以在 jar 目录使用 * 通配符
  java -classpath .:/path/to/'*' Class
  ```

* Windows 环境

  类路径中不同项目之间采用冒号 ; 分隔

为了使类能够被多个程序共享，需要做到下面几点：

1）把类放到一个目录中，这个目录是包树状结构的基目录。

2）将 jar 文件放在一个目录中

3）设置类路径

  * 首选使用 -classpath 或 -cp 选项指定类路径

    ```shell
    java -Dfile.encoding=UTF-8 -classpath .:/path/to/jar/'*' jdbc.mysql.CityCurd
    ```

  * 设置 CLASSPATH 环境变量设置类路径

      * UNIX

        ```shell
        export CLASSPATH=.:/path/to/classdir/'*'
        ```

      * Windows

        ```powershell
        set CLASSPATH=.;c:\path\to\classdir\*
        ```

  * 将 jar 或类文件放在虚拟机加载路径

编译器总会在当前的目录中查找文件，但虚拟机仅在类路径中有 . 目录的时候才查看当前目录

###### 类的关系

* 依赖（uses-a）

  如果一个类的方法操纵另一个类的对象，则一个类依赖于另一个类

* 聚合（has-a）

  一个类对象包含一些其他类的对象，即关联

* 继承（is-a）

  用于表示特殊与一般关系的。

* 类关系的 UML 符号

    |   关系   |  UML 连接符  |
    | :------: | :----------: |
    |   继承   | 实线连空三角 |
    | 接口实现 | 虚线连空三角 |
    |   依赖   |  虚线连三角  |
    |   聚合   |  菱形连实线  |
    |   关联   |     实线     |
    | 直接关联 |  实现连三角  |

###### super 关键字

关键字 super 指父类，可用于调用父类中的普通方法和构造方法。

调用父类方法时 super 必须出现在子类方法第一行，这是显式调用父类方法的唯一方法

###### final 关键字

一个即是 static 又是 final 的域只占据一段不能改变的存储空间，类常量使用大写蛇形表示，在定义时，必须对其进行赋值

* final 用于对象引用时，final 使引用恒定不变，一旦引用被初始化指向一个对象，就无法再把它改为指向另一个对象，对象其自身是可以被修改的（Java 并未提供使任何对象恒定不变的途径）
* 对于基本类型，final 使值恒定不变

空白 final 

允许生成空白 final（被声明为 final 但又未给定初值的域），无论什么情况，编译器都确保空白 final 在使用前必须被初始化（必须在域的定义处或构造器中用表达式对 final 进行赋值）

使用 final 关键字修饰以阻止继承：

* 使用 final 修饰类，表示不允许扩展该类，其中方法自动成为 final，但不包括域
* 类中所有 private 方法都隐式指定为 final 的，对 private 方法添加 final 修饰符没有意义
* 域声明为 final 时，赋值之后不容许修改

###### 内部类

内部类是定义在另一个类中的类，内部类是一种非常有用的特性，它允许把一些逻辑相关的类组织在一起，并控制位于内部的类的可视性，内部类与组合是完全不同的概念。内部类只是编译器的概念，对于虚拟机不会区分内部类，每个内部类最后都会编译成一个独立的类，生成一个独立的字节码文件

* 静态内部类

  ```java
  Outer.Inner = new Outer.Inner();
  ```

  可以访问外部类的静态成员。

* 成员内部类

  ```java
  Outer out = new Outer;
  Outer.Inner inner = out.new Inner();
  ```

  可以访问外部类的实例和静态成员，可以通过 `外部类.this.` 引用实例成员（重名时使用）。成员内部类对象与外部类对象绑定。不能定义静态成员（除了 final 常量）。如果内部类需要静态成员，可以定义在外部类中

* 方法内部类

  可以访问方法的参数和方法中的局部变量，这些变量必须为 final。通过在构造方法中传递参数来实现，方法内部类操作的并不是外部变量，而是内部类实例变量，只是这些变量的值和外部一样，对这些变量赋值，并不会改变外部的值，需要改变外部值时，可以将变量该为数组，修改数组元素的值

* 匿名内部类

  创建对象的同时定义类，与 new 关联。匿名内部类只能被使用一次，用来创建一个对象。没有名字和构造方法，可以根据参数列表，调用对应的父类构造方法，可以定义实例变量和方法，可以定义初始化代码块（模拟构造函数）。匿名内部类也被生成一个独立类（类名以外部类加数字编号）

  ```java
  new ParentClassName(int param1){};
  new ParentInterfaceName(int param2){};
  ```

###### 接口内部类

静态内部类可以作为接口的一部分，放到接口中的任何类都自动地是 public 和 static 的。只是将嵌套类置于接口的命名空间内，可以在嵌套类中提供外围接口的实现

###### 闭包与回调

闭包（closure）是一个可调用的对象，它记录了一些信息，这些信息来自于创建它的作用域（内部类是面向对象的闭包，它不仅包含外围类对象（创建内部类的作用域）的信息，还自动拥有一个指向外围类对象的引用，在此作用域内，内部类有权操作所有的成员，包括 private 成员）

#### 类关系

##### 继承

###### 继承关系

* 继承（extends）

  is-a 关系是继承的一个明显特征，关键字  extends 表示继承。在子类中可以增加域、增加方法或者覆盖超类的方法，然而绝对不能删除继承的任何域或方法。所有类都继承自 Object 类，如果定义一个类时，没有指定继承性，那么这个类的父类就被默认为 Object

* 继承链（inheritance hierarchy）

  继承并不仅限于一个层次。由一个公共超类派生出来的所有类的集合被称为继承层次，在继承层次中，从某个特定的类到其祖先的路径为该类的继承链（inheritance chain）

###### Object 始祖类

Object 类中的 equals 方法用于检测一个对象是否等于另外一个对象。在 Object 类中，这个方法将判断两个对象是否具有相同的引用。如果两个对象具有相同的引用，它们一定是相等的。在子类中定义 equals 方法时，首先调用超类的 equals。如果检查失败，则不可能相等。语言规范要求 equals 方法具有下面的特性：

* 自反性

  对于任何非空引用 x，x.equals(x) 应该返回 true

* 对称性

  对于任何引用 x 和 y，y.equals(x) 返回 true，x.equals(y) 也应该返回 true

* 传递性

  对于任何引用 x、y、z，如果 x.equlas(y) 返回 true，y.equals(z) 返回 true，x.equals(z) 也应该返回 true

* 一致性

  如果 x 和 y 引用的对象没有发生变化，反复调用 x.equals(y) 应该返回同样的结果

如果重写定义 equals 方法，就必须重新定义 hashCode 方法

###### 方法覆写

使用 @Override 注解覆写父类或接口方法，要覆写一个方法，需要在子类中使用和父类一样的签名以及一样的返回值类型来对该方法进行定义

* 仅当实例方法是可访问时，它才能被覆盖。因为私有方法在它的类本身以外是不能访问的，所以它不能被覆盖。如果子类中定义的方法在父类中是私有的，那么这两个方法完全没有关系
* 静态方法也能被继承，但是静态方法不能被覆盖。如果父类中定义的静态方法在子类中被重新定义，那么在父类中定义的静态方法将被隐藏，可使用 SuperClass.staticMethod 调用隐藏的静态方法
* 方法覆写（具有同样的签名和返回值类型）发生在通过继承或实现接口而相关的不同类中
* 方法重载（具有同样的名字，但是不同的参数列表）可以发生在同一类中，也可以发生在由于继承而相关的不同类中；

###### 协变返回类型

SE 5 中添加了协变返回类型，在导出类中的被覆盖方法可以返回基类方法的返回类型的某种导出类型

###### 对象转换和 instanceof 运算符

对象的引用类型可以转换为对另外一种对象的引用，使用 instanceof 测试对象类型，使用转换操作符转换

* 向上转型总能成功
* 在尝试转换之前确保该对象是目标类型的实例，如果 x 为 null，x instanceof C 不会产生异常，只是返回 false。因为 null 没有引用任何对象
* 基本类型值转换返回一个新值，而转换一个对象引用不会创建一个新的对象

###### 抽象类

* 抽象类不可以用于创建对象，抽象类可以包含抽象方法，这些方法将在具体的子类中实现
* 抽象方法不能包含在非抽象类中，如果抽象父类的子类不能实现所有的抽象方法，那么子类也必须定义为抽象的，在抽象类扩展的非抽象子类中，必须实现所有的抽象方法。抽象方法是非静态的。
* 抽象类不能使用 new 操作符来初始化。但是，仍然可以定义它的构造方法，这个构造方法在它的子类的构造方法中调用。
* 包含抽象方法的类必须是抽象的。但是，可以定义一个不包含抽象方法的抽象类，在这种情况下，不能使用 new 操作符创建该类的实例，这种类是用来定义新子类的基类
* 即使子类的父类是具体的，这个子类也可以是抽象的

##### 接口

编译器会为编译的接口创建一个 class 文件

###### 特性

接口不是类，是对类的一组需求描述，这些类要遵从接口描述的统一格式进行定义

* 接口中所有方法自动属于 public abstract，在接口中声明方法时，不必提供关键字 public abstract
* 接口可以被扩展，可以嵌套在类或其他接口中。类中嵌套的 private 接口，只能够被实现为一个 private 内部类。当实现某个接口时，不需要实现嵌套在其内部的任何接口，private 接口不能在定义它的类之外被实现
* 接口中的域自动是 static 和 final 的，接口中定义的域不能是空 final，可以被非常量表达式初始化（值被存储在该接口的静态存储区域内），一个接口中不允许有名称相同的字段，但接口可能从其父接口继承多个具有相同名称的字段
* SE 8 中，允许在接口中增加静态方法和默认方法

对于创建类，几乎在任何时刻，都可以替代为创建一个接口和一个工厂，任何抽象性都应该是应真正的需求而产生的，当必要时，应该重构接口而不是到处添加额外级别的间接性，并由此带来的额外复杂性。恰当的原则应该是优先选择类而不是接口，从类开始，如果接口的必需性变得非常明确，就进行重构，接口是一种重要的工具，但是它们容易被滥用

###### 默认方法

可以为接口方法提供一个默认实现。必须用 default 修饰符标记这样的的方法

如果在接口中将一个方法定义为默认方法，然后又在超类或另一个接口中定义了同样的方法。对于此种二义性：

1. 超类优先

   如果超类提供了一个具体方法，接口的同名而且有相同参数类型的默认方法会被忽略

2. 接口冲突

   如果一个超接口提供了一个默认方法。另一个接口提供了一个同名而且参数类型（不论是否是默认参数）相同的方法，必须类中覆写这个方法来解决冲突

对于只有一个抽象方法的接口，需要这种情况的对象时，可以提供一个 lambda 表达式。这种接口即函数式接口

Java 支持默认方法的主要原因是为了向后兼容

###### 静态方法

8 及更高版本中，可以在接口中定义静态方法，接口中的静态方法在默认情况下是 public 的

###### 私有方法

9 开始，可以在接口中声明私有和私有静态方法

#### 核心类

##### 字符串

###### *String*

java 字符串就是 Unicode 字符序列（一个 Unicode 字符对应 Unicode 编码表中码点，可能需要1个或 2 个代码单元表示）。使用 + 拼接字符串。*String* 类实例不可变（immutable），空串长度为 0 和内容为空。

为引用变量赋一个字符串字面值与使用 new 关键字的方式不同，使用 new 关键字 JVM 将始终创建一个新的 *String* 实例，使用字符串字面值，可能会使用 string interning 缓存（String 在 Java 6 以后提供了 intern() 方法，提示 JVM 把相应字符串缓存起来，以备重复使用，对于 Java 6 的历史版本，被缓存的字符串是存在 PermGen 里即永久代，容易造成 OOM，后续版本该缓存被放置在堆中，使用 `-XX:+PrintStringTableStatistics`（打印缓存大小），`-XX:StringTableSize=N` 手动调整参数大小））Intern 是一种显式地排重机制，需要在代码中明确的调用，8u20 之后，G1 GC 下的字符串排重（通过将相同数据的字符串指向同一份数据来做到，是 JVM 底层的改变，该功能默认关闭，使用 G1 GC 时，使用 `-XX:+UseStringDeduplication` 开启）

在运行时，字符串的一些基础操作会直接利用 JVM 内部的 Intrinsic（利用 native 方式 hard-coded 逻辑，特别的内联）机制，运行的是特殊优化的本地代码非字节码。

字符串字面值以双引号开始和结束，在右双引号之前换行会产生编译错误

使用索引访问用 String 的 split 方法得到的数组时，需做最后一个分隔符后有无内容的检查，否则会有抛 *IndexOutOfBoundsException* 的风险

在 Java 9 中，引入了 Compact Strings 设计，将字符串存储从 char 数组改变为 byte 数组加上一个标识编码 coder（优势是更小的内存占用，更快的操作速度）

`getBytes(); String(byte[] bytes)` 都是隐含使用平台默认编码

###### *StringBuilder*、*StringBuffer*

如果创建 *StringBuilder* 对象时没有指定容量，该对象的容量将为 16 个字符，会自动扩容。底层都是利用可修改的 char（JDK 9 之后是 byte）数字，都继承了 *AbstractStringBuilder* 里面包含了基本操作，*StringBuffer* 在每个方法上添加了 synchronized。

非静态的拼接逻辑在 JDK 8 中会自动被 javac 转换为 *StringBuilder* 操作，在 JDK 9 中，利用 InvokeDynamic，将字符串拼接的优化与 javac 生成的字节码解耦（假设未来 JVM 增强相关运行时实现，将不需要依赖 javac 的任何修改）

###### *Formatter*

所有新的格式化功能都由 *java.util.Formatter* 类处理，它将格式化字符串与数据翻译成需要的结果

*类型转换字符*

| 字符 |                             含义                             |
| :--: | :----------------------------------------------------------: |
|  d   |                          十进制整型                          |
|  e   |                      浮点数（科学计数）                      |
|  c   |                         Unicode 字符                         |
|  x   |                         十六进制整型                         |
|  b   | Boolean 值（对于 bool 和 Boolean 结果为对应值，其他类型不为 NULL，则始终 true） |
|  h   |                        十六进制散列码                        |
|  s   |                            String                            |
|  %   |                            字符 %                            |
|  f   |                         十进制浮点数                         |

###### String 的正则操作

在 java 中双反斜线表示正则中的反斜线，即普通的反斜线为 `\\\\` ，换行符和制表符只需使用单反斜线 `\n\t`

* *String* 提供了 matches()、replaceFirst()、replaceAll()、split() 方法来进行正则操作。

* *java.util.regex.* 包操作正则
  1. 编译正则
  2. 生成匹配对象

##### 数值操作

###### BigDecimal

`BigDecimal(double)` 构造函数存在精度损失风险，优先推荐入参数 String 的构造方法，或使用 `BigDecimal.valueOf()` 重载方法

使用 *BigDecimal* 来进行精确浮点数计算

###### BigInteger

###### Random/ThreadLocalRandom

java 7 之前，使用 *java.util.Random* 类或 `java.lang.Math.random()` 方法生成随机数，非线程安全

java 7 之后添加了一个新类 `java.util.concurrent.ThreadLocalRandom`，它是线程安全的

```java
ThreadLocalRandom threadLocalRandom = ThreadLocalRandom.current();
int random = threadLocalRandom.current().nextInt();
```

##### 运行时信息

###### class

Class 对象表示运行时的类型信息，每个类都有一个 Class 对象，JVM 每次创建一个对象时，会同时创建一个 `java.lang.Class` 对象来描述对象的类型。同一个类的所有实例共享同一个 Class 对象。

* 对一个已加载的类，使用 Class.forName 再进行加载时不会出错，Class.forName() 会立即进行初始化。

类字面常量（.class）也可以生成对 Class 对象的引用，它在编译时就会受到检查，不需要置于 try 语句块中，也不需要对 forName 方法进行调用，当使用类字面常量来创建 Class 对象的引用时，不会自动地初始化 Class 对象，初始化被延迟到对静态方法或者非常数静态域（对于编译期常量，那么这个值不需要对类进行初始化就可以被读取，如果一个 static 域不是 final，那么在对它访问时，总是要求在它被读取之前，为这个域分配存储空间和初始化该存储空间）进行首次引用时才执行。

* Class<?> 优于 Class 类型，即便它们是等价的

* 5 添加了用于 Class 引用的转型语法，即 cast() 方法，cast() 方法接受参数对象，并将其转型为 Class 引用类型，对于无法使用普通转型的情况非常有用，在编写泛型时，如果存储了 Class 引用，并希望以后通过这个引用来执行转型。

  ```java
  class Building {}
  class House extends Buiding {}
  public class ClassCasts {
  	Building b = new House();
  	Class<House> houseType = House.class; 
      House h = houseType.cast(b); //  等价	h = (House) b;
  }
  ```

###### System

*System* 包含静态的 out（*PrintStream*）、in（*InputStream*）、err（*PrintStream*） 字段映射输出、输入、错误流

*Java的系统属性*

|           系统属性            |           描述            |
| :---------------------------: | :-----------------------: |
|         java.version          |    Java 运行时环境版本    |
|          java.vendor          |   Java 运行时环境供应商   |
|        java.vendor.url        |       Java 厂商网址       |
|           java.home           |       Java 安装目录       |
| java.vm.specification.version |    Java 虚拟机规范版本    |
| java.vm.specification.vendor  |   Java 虚拟机规范供应商   |
|  java.vm.specification.name   |    Java虚拟机规范名称     |
|        java.vm.version        |    Java虚拟机实现版本     |
|        java.vm.vendor         |   Java虚拟机实现供应商    |
|         java.vm.name          |     Java虚拟机实现名      |
|  java.specification.version   |  Java运行时环境规范版本   |
|   java.specification.vendor   | Java 运行时环境规范供应商 |
|    java.specification.name    |  Java运行时环境规范名称   |
|      java.class.version       |     Java类格式版本号      |
|        java.class.path        |        Java类路径         |
|       java.library.path       | 加载库时要搜索的路径列表  |
|        java.io.tmpdir         |     默认临时文件路径      |
|         java.compiler         |  要使用的 JIT 编译器名称  |
|         java.ext.dirs         |   扩展目录或目录的路径    |
|            os.name            |       操作系统名称        |
|            os.arch            |      操作系统的架构       |
|          os.version           |       操作系统版本        |
|        file.separator         |        文件分隔符         |
|        path.separator         |        路径分隔符         |
|        line.separator         |         行分隔符          |
|           user.name           |      用户的账户名称       |
|           user.home           |       用户的主目录        |
|           user.dir            |     用户当前工作目录      |

##### 数组

虚拟机确保数组会被初始化，而且不能越界访问。创建数组对象时，实际上就是创建一个引用数组，并且每个引用都会自动被初始化为一个特定值，当数组元素引用未指向某个对象时为 null，在使用数组元素引用前，必须为其指定一个对象或基本类型，否则会 *NullPointException*

* 基本类型的数组初始化为 0
* boolean 数组会初始化为 false
* 引用、数组元素会初始化为 null
* 可以向导出类型的数组赋予基类型的数组引用。数组对象可以保留有关它们包含的对象类型的规则

```java
// 将数组传递给方法时必须单独实例化数组
int avg = average({1,2,3,10}); // 非法
int avg = average(new int[]{1,2,3,10}); // 合法
```

如果没有给 main 传递参数，字符串数组 args 将是 empty，而不是 null

数组的 length 属性表示数组的大小，而不是实际保存的元素个数。

数组与泛型不能很好的结合，不能实例化具有参数化类型的数组，擦除会移除参数类型信息，而数组必须知道它们所持有的确切类型，以强制保证类型安全

如果复制对象数组，只是复制了对象的引用，而不是对象本身的拷贝

###### *Arrays*

##### Enum 类

对于有限集合的变量取值，可以自定义枚举类型，枚举类型包括有限个命名的值，枚举只能存储声明的枚举值或 null 值。

```java
// 枚举值区分大小写，惯例均为大写，两个枚举用逗号分隔，值可以写在一行或多行上
enum Size {SMALL, MEDIUM, LARGE, EXTRE_LARGE};
// 声明枚举变量
Size s = Size.MEDIUM;
```

###### *Enum* 

* values() 方法返回 enum 实例的数组，而且该数组中的元素严格保持其在 enum 中声明的顺序，values() 是由编译器添加的 static 方法
* 创建 enum 时，编译器会生成一个相关的类，这个类继承自 *java.lang.Enum*，可以使用 == 来比较 enum 实例，编译器会自动为 enum 提供 equal() 和 hashCode()，*Enum* 实现了 Comparable 和 Serializable 接口
* ordinal() 方法返回 enum 实例在声明时的次序，从 0 开始
* name() 方法返回 enum 实例声明时的名字，与 toString() 方法效果一样。
* valueOf() 根据给定的名字返回相应的 enum 实例
* 如果打算定义 enum 实例定义方法，那么必须在 enum 实例序列的最后添加一个分号，enum 实例之间用逗号分隔，必须先定义 enum 实例，如果在定义 enum 实例之前定义了任何方法或属性，会编译错误
* 只能在 enum 内部使用其构造器创建 enum 实例，一旦 enum 的定义结束，编译器就不允许使用构造器创建任何实例

可以在接口的内部，创建实现该接口的枚举，以此将元素进行分组。

可以为 enum 实例编写方法，从而为每个 enum 实例赋予各自不同的行为，需要为 *Enum* 添加一个或多个 abstract 方法，然后为每个 enum 实例实现该方法

###### EnumSet

SE 5 引入了 EnumSet，是为了通过 enum 创建一种替代品，以替代传统的基于 int 的 『位标志』，这种标志可以用来表示某种『开关』信息。

EnumSet 中的元素必须来自一个 enum。EnumSet 的基础是 long，一个 enum 实例只需一位 bit 表示其是否存在，在不超过一个 long 的表达能力的情况下，EnumSet 可以应用于最多不超过 64 个元素的 enum，超过之后性能会下降。

enum 实例定义时的次序决定了其在 EnumSet 中的顺序

###### EnumMap

EnumMap 是一种特殊的 Map，它要求其中的键必须来自一个 enum，EnumMap 内部由数组实现，性能很高。可以使用 enum 实例在 EnumMap 中进行查找操作，只能将 enum 的实例作为键来调用 put()

enum 实例定义时的次序决定了其在 EnumMap 中的顺序

##### Optional

为了处理 *NullPointerException*，Java 8  的 *java.util* 添加了 *Optional*，*OptionalInt*，*OptionalLong*，*OptionalDouble* 类，可以配合 Lambda 实现简洁的代码。

*Optional* 是存放值的容器，该值可能为 Null，将每个可能为 null 的字段包装为 *Optional*

*Optional类常用方法*

|    方法    |                             说明                             |
| :--------: | :----------------------------------------------------------: |
|   empty    |                    返回一个空的 Optional                     |
|   filter   | 如果存在一个值且与给的谓词匹配，返回该值的 Optional，否则返回空 Optional |
|  flatMap   | 如果预先设置了一个值，则对该值应用指定的映射函数，并返回映射结果的 Optional，如果值不存在，返回空 Optional |
|    get     | 如果值存在，返回该值，否则抛出 *NoSuchElementException* 异常 |
| ifPresent  |           如果值存在，使用该值调用给定的 Consumer            |
| isPresent  |           如果值存在，返回 true，否则，返回 false            |
|    map     | 如果值存在，对它应用给定的映射函数；如果结果不为 null，返回一个描述结果的 Optional |
|     of     |              返回描述给定非 null 值的 Optional               |
| ofNullable | 如果给定值非 null，返回描述该值的 Optional；如果值为 null，返回空 Optional |
|   orElse   |            如果值存在，返回该值；否则返回指定的值            |

