## 类和接口

### 使类和成员的可访问性最小化

#### 尽可能的使每个类或者成员不被外界访问

* 对于顶层（非嵌套类）类和接口，只有两种可能的访问级别：**包级私有的**（package-private）和**公有的**（public）。如果类或者接口能够被做成包级私有的，它就应该被做成包级私有的。通过把类或者接口做成保级私有，它实际上成了这个包的实现的一部分，而不是该包导出的 API 的一部分
* 如果一个包级私有的顶层类（或者接口）只是在某一个类的内部被用到，就应该考虑使它成为唯一使用它的那个类的私有嵌套类。这样可以将它的可访问范围从包的所有类缩小到使用它的那个类。然而降低不必要公有类的可访问性，比降低包级私有的顶层类的可访问性重要的多：因为公有类是包的 API 的一部分，而包级私有的顶层类则是包实现的一部分。私有成员和包级私有成员都是一个类的实现中的一部分，一般不会影响导出的 API。然而，如果这个类实现了 `Serializable` 接口，这些就有可能被泄漏到导出的 API 中
* 对于公有类的成员，当访问级别从包级私有变成保护级别时，会大大增强可访问性。受保护的成员是类的导出的 API 的一部分，必须永远得到支持，导出的类的受保护成员也代表了该类对于某个实现细节的公开承诺，

#### 公有类的实例域决不能是公有的

如果实例域是非 final 的，或者是一个执行可变对象的 final 引用，那么一旦使这个域成为公有的，就等于放弃了对存储在这个域中的值进行限制的能力；放弃了强制这个域不可变的能力。同时，当这个域被修改的时候，也失去了对它采取任何行动的能力，因此，**包含公有可变域的类通常并不是线程安全的**。即使域是 final 的，并且引用不可变的对象，但当把这个域变成公有的时候，也就放弃了“切换到一种新的内部数据表示法”的灵活性。

静态域也不能是公有的，但假设常量构成了类提供的整个抽象中的一部分，可以通过公有的静态 final 域来暴露这些常量。按照惯例，这种域的名称由大写字母组成，单词之间用下划线隔开。这些域要么包含基本类型的值，要么包含指向不可变对象的引用。如果 final 域包含可变对象的引用，它便具有非 final 域的所有缺点。虽然引用本身不能修改，但是它所引用的对象却可以被修改，这会导致灾难性的后果

长度非零的数组总是可变的，所以**让类具有公有的静态 final 数组域，或者返回这种域的访问方法，这是错误的**。如果类具有这样的域或者访问方法，客户端将能够修改数组中的内容，这是安全漏洞的一个常见根源。

#### Java 9 中的模块化

从 Java 9 开始，又新增了两种隐式访问级别，作为**模块系统**（module system）的一部分。一个模块就是一组包，就像包就是一组类一样。模块可以通过其**模块声明**（module declaration）中的**导出声明**（export declaration）显式地导出它的一部分包（按照惯例，这包含在名为 `module-info.java` 的源文件中）。模块中未被导出的包在模块之外是不可访问的；在模块内部，可访问性不受导出声明的影响。使用模块系统可以在模块内部的包纸巾共享类，不用让它们对全世界都可见。未导出的包中公有类的公有成员和受保护的成员都提高了两个隐式访问级别，这是正常的公有和受保护级别在模块内部的对等体（intramodular analogues）。

与四个主访问级别不同的是，这两个基于模块的级别主要提供咨询。如果把模块的 JAR 文件放在应用程序的类路径下，而不是放在模块路径下，模块中的包就会恢复其非模块的行为：无论包是否通过模块导出，这些包中公有类的所有公有的和受保护的成员将都有正常的可访问性。严格执行新引入的访问级别的一个示例是 JDK 本身：Java 类库中未导出的包在其模块之外确实是不可访问的

### 要在公有类而非公有域中使用访问方法

有时候，可能需要编写一些退化类，它们没有什么作用，只是用来集中实例域。如果**类可以在它所在的包之外进行访问，就提供访问方法**，以保留将来改变该类的内部表示法的灵活性。如果公有类暴露了它的数据域，要想在将来改变其内部表示法是不可能的，因为公有类的客户端代码已经遍布各处了。

**如果类是包级私有的，或者是私有的嵌套类，直接暴露它的数据域并没有本质的错误**假设这些数据域确实描述了该类所提供的抽象。无论是在类定义中，还是在使用该类的客户端代码中，这种方法比访问方法的做法更不容易产生视觉混乱。虽然客户端代码与该类的内部表示法紧密相连，但是这些代码被限定在包含该类的包中。如有必要，也可以不改变包之外的任何代码，而只改变内部数据表示法。在私有嵌套类的情况下，改变的作用范围被进一步限制在外围类

公有类永远都不应该暴露可变的域。虽然还是有问题，但是让公有类暴露不可变的域，其危害相对来说比较小。但有时候会需要用包级私有的或者嵌套类来暴露域，无论这个类是可变的还是不可变的

### 使可变性最小化

不可变类即其实例不能被修改的类。每个实例中包含的所有信息都必须在创建该实例的时候就提供，并在对象的整个生命周期内固定不变。Java 平台类库中包含许多不可变的类：String、基本类型包装类、BigInteger、BigDecimal。为了使类成为不可变，遵循：

* 不要提供任何会修改对象状态的方法（即设值方法）

* 保证类不会被扩展

  一般声明为 final；让类的所有构造器都变成私有的或者包级私有的。并添加公有的静态工厂来代替公有的构造器

* 声明所有的域都是 final 的

  如果一个指向新创建实例的引用在缺乏同步机制的情况下，从一个线程被传递到另一个线程，就必须确保正确的行为

* 声明所有域都为私有的

  可防止客户端获得访问被域引用的可变对象的权限，并防止客户端直接修改这些对象

* 确保对于任何可变组件的互斥访问

  如果类具有指向可变对象的域，则必须确保该类客户端无法获得指向这些对象的引用。

 大多数重要的不可变类都使用了这种模式。它被称为**函数的（functional）**方法，因为这些方法返回了一个函数的结果，这些函数对操作数运算但并不修改它。与之相对应的更常见的是**过程的（procedural）**或**命令式的（imperative）**方法，使用这些方法时，将一个过程作用在它们的操作数上，会导致它的状态发生改变。这些方法名称都是介词如（plus），而不是动词（add）。这是为了强调该方法不会改变对象的值。BigInteger 类和 BigDecimal 类由于没有遵循这一命名习惯，导致了很多用法错误

**不可变性优点**

* 不可变对象比较简单

  不可变对象可以只有一种状态，即被创建时的状态。如果能确保所有的构造器都建立了这个类的约束关系，就可以确保这些约束关系在整个生命周期内永远不再发生变化。可变的对象可以有任意复杂的状态空间。

* 不可变对象本质是线程安全的，它们不要求同步，可以被自由地共享

  当多个线程并发访问这样的对象时，它们不会遭到破坏。这是获得线程安全的最容易的方法。不可变对象可以被自由共享结果是，永远不需要进行保护性拷贝，不需要也不应该为不可变类提供 clone 或拷贝构造器。

* 不仅可以共享不可变对象，也可以共享它们的内部信息

* 不可变对象为其他对象提供了大量的构件

  无论是可变还是不可变的对象。如果知道一个复杂对象内部的组件对象不会改变，要维护它的不可变性约束是比较容易的。

* 不可变对象提供了失败的原子性

  它们的状态永远不变，因此不存在临时不一致的可能性

**不可变性缺点**

* 对于每个不同的值都需要一个单独的对象。创建这些对象的代价可能很高，特别是大型对象。

  如果执行一个多步骤操作，并且每个步骤都会产生一个新的对象，除了最后的结果之外，其他的对象最终都会被丢弃，此时性能问题就会暴露出来。处理这种情况：先猜测经常会用到哪些多步骤的操作，然后将它们作为基本类型提供。如果多步骤操作已经作为基本类型提供，不可变的类就无须在每个步骤单独创建一个对象；提供一个公有的可变配套类。在 Java 平台中，StringBuilder 即是 String 类的可变配套类

  如果选择让不可变类实现 `Serializable` 接口，并且它包含一个或者多个指向可变对象的域，就必须提供一个显式的 `readObject` 或者`readResolve` 方法，或者使用 `ObjectOutputStream.writeUnshared` 和 `ObjectInputStream.readUnshared` 方法，即便默认的序列化形式是可以接受的，也是如此。

  坚决不要为每个 get 方法编写一个相应的 set 方法。除非有很好的理由要让类成为可变的类，否则它就应该是不可变的。

  对于某些类而言，其不可变性是不切实际的。如果类不能被做成不可变的，仍然应该尽可能地限制它的可变性。降低对象可以存在的状态数，可以更容易地分析该对象的行为，同时降低出错的可能性。因此，除非有