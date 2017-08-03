---
layout: post
title:  浅谈 Scala 中的语言特性实现（一）
date:   2017-08-03 03:39:03 +0700
categories: [scala]
---

## 前言

Scala 是我最近正在研究、学习和使用的一门语言，Scala 的学习过程与其他语言类似，语法、类型、特性对于有过编程基础的人其实很好消化，但对我而言 Scala 又是一门与众不同的语言，不同点主要有二：

一是 Scala 是我正式接触的第一门函数式编程语言，编程思想存在明显的差异，虽然写过几年程序，但作为一个 Scala 新人写出来的代码风格与公司的老手风格居然可以相差甚远（典型的比如 `for (1 <- 1 to 10)` 和 `(1 to 10).map`）。

二是这一门函数式编程语言居然是基于 JVM 构建，并实现了 Java 中原本没有的诸多实用特性（高阶函数、类型推断、模式匹配、隐式转换，等等），对于原来习惯 Java 写 OOP 的程序员来说，还是那股熟悉的感觉，却又能摆脱 Java 自身的限制和束缚，宛如手上多了一把利器，那种感觉太舒服不过了。

___Seven Languages in Seven Weeks___ 一书在访谈 [Martin Odersky](https://en.wikipedia.org/wiki/Martin_Odersky) 时候，Martin 博士有段话我很是喜欢，向前辈致敬：

> 我坚信讲将函数式编程和面向对象两种编程范式统一起来将具有很大的实用价值。不过函数式编程社区对面向对象编程社区不屑一顾的态度和面向对象程序员坚持函数式编程知识一种学术活动的信条都让我十分沮丧。因此，我想表明这两种模式可以统一，而且一些新的更强大的功能可能因此产生。

知其然亦要知其所以然，因为我打算写一系列小文，浅谈一下 Scala 中不同语言特性是如何实现的，当做个人的学习笔记。能力、经验和水平都十分有限，不对之处还望各位指正。

## Scala 中的 Class

先从最简单的 `class` 说起，如下代码，我们构造了一个类，内部包含一个有参方法和一个无参方法，一个主构造器和一个辅助构造器：

``` scala
class TestClass {
  def isEven(i: Int) = i % 2 == 0
  def isOdd(i: Int) = i % 2 == 1
}
```

我们使用 `scalac TestClass.scala` 编译成字节码并用 `jd-cli *.class` 反编译，得到结果：

``` java
public class TestClass {
  public boolean isEven(int i) {
    return i % 2 == 0;
  }

  public boolean isOdd(int i) {
    return i % 2 == 1;
  }
}
```

这很符合我们的预期，Scala 的 `Class` 与 Java 的 `Class` 本质上基本是一样的。

__// TODO: 主构造器和辅助构造器 __

## var 和 val 的实现

继续看例子：

``` scala
class TestClass {
  val valValue = 0
  var varValue = 1
}
```

编译和反编译：

``` java
public class TestClass {
  public int valValue() {
    return valValue;
  }

  private final int valValue = 0;

  public int varValue() {
    return varValue;
  }

  public void varValue_$eq(int x$1) {
    varValue = x$1;
  }

  private int varValue = 1;
}
```

`valValue` 和 `varValue` 其实都是 `private` 字段，外部无法直接访问，`valValue` 提供了 `intValue()` 方法获取其值（也就是 getter）而 `varValue` 不仅提供了 `varValue()`，还提供了 `varValue_$eq(int x$1)` 用于赋值（既有 getter 又有 setter）。

这里需要稍微注意一点，Java 是 [允许方法和字段同名的](https://stackoverflow.com/questions/9960560/java-instance-variable-and-method-having-same-name)。

__ // TODO: 重写字段 __

字段的重写本质上也是 `getter` 和 `setter` 的重写。


## Scala 中的 Object

再看 Scala 中的 `Object`，我们知道 Scala 中的 `Object` 分为单例对象和伴生对象，我们先看前者：

``` scala
object App {
  def isEven(i: Int) = i % 2 == 0
  def isOdd(i: Int) = i % 2 == 1
}
```

同样的方法编译和反编译，我们会发现 scalac 实际上生成了两个 class 文件，一个是 `App.class`，另一个是 `App$.class`：

``` java
// decompile App$.class
public final class App$ {
  public static final  MODULE$;

  static {
    new ();
  }

  public boolean isEven(int i) {
    return i % 2 == 0;
  }

  public boolean isOdd(int i) {
    return i % 2 == 1;
  }

  public void main(String[] args) {}

  private App$() {
    MODULE$ = this;
  }
}

// decompile App.class
public final class App {
  public static void main(String[] paramArrayOfString) {
    App..MODULE$.main(paramArrayOfString);
  }

  public static boolean isOdd(int paramInt) {
    return App..MODULE$.isOdd(paramInt);
  }

  public static boolean isEven(int paramInt) {
    return App..MODULE$.isEven(paramInt);
  }
}
```

`App$` 是一个单例类（构造函数前带 `private`，内部包含当前类的一个实例 `MODULE$`），`App` 提供相应的静态方法给外部调用，而静态方法实际上调用的都是 `App$.MODULE$` 的实例方法。

为什么不直接用 `App$` 而还要额外生成 `App` 呢，这个 [回答](https://stackoverflow.com/a/12785219/2122077) 的解释是 `Object` 既要有静态类的特征（包含类方法）又要有非静态类的特征（包含实例方法）并且允许这两个方法同名，单个类是无法实现的。虽然有点道理，但我觉得这并不是很能说服人，具体的原因还是在后面的学习过程中继续思考吧。

接下来看伴生对象：

``` scala
class App {
  def func(i: Int) = App.isEven(i)
  def func2(i: Int) = func(i)
}

object App {
  def isEven(i: Int) = i % 2 == 0
  def isOdd(i: Int) = i % 2 == 1
}
```

反编译：

``` java
public final class App$ {
  public static final  MODULE$;

  static {
    new ();
  }

  public boolean isEven(int i) {
    return i % 2 == 0;
  }

  public boolean isOdd(int i) {
    return i % 2 == 1;
  }

  private App$() {
    MODULE$ = this;
  }
}

import scala.reflect.ScalaSignature;
public class App {
  public static boolean isOdd(int paramInt) {
    return App..MODULE$.isOdd(paramInt);
  }

  public static boolean isEven(int paramInt) {
    return App..MODULE$.isEven(paramInt);
  }

  public boolean func(int i) {
    return App..MODULE$.isEven(i);
  }

  public boolean func_2(int i) {
    return func(i);
  }
}
```

当外部调用 `object App` 时候中的方法或者成员时，实际上调用的是 `App$.MODULE$` 而非 `App` 的方法和成员，这也是为什么 Scala 中的一个类不能直接访问其伴生对象中定义的方法和字段的原因 —— 两者并不是同一个类，当我们试图去 `import App._` 时候，实际上是 `import WHAT?`

## 高阶函数

## Scala 中的 Case Class

## Scala 中的 Trait

## Scala 中的 包

## 一个应用实例

最近在读 `scala.Enumeration` 的源码实现，一个来自 ___《Scala for the Impatient》___  一书的`Enumeration` 例子：

``` scala
// Source Code Here
```

思考一个问题：`Enumeration` 是怎么在运行时，获取得到枚举值的名字的？回顾 `val` 的实现，我们很容易想通：用反射即可，对应的源码实现如下：

``` java
abstract class Enumeration (initial: Int) extends Serializable {
  // ...
  private def populateNameMap() {
    val fields = getClass.getDeclaredFields
    def isValDef(m: JMethod) = fields exists (fd => fd.getName == m.getName && fd.getType == m.getReturnType)

    // The list of possible Value methods: 0-args which return a conforming type
    val methods = getClass.getMethods filter (m => m.getParameterTypes.isEmpty &&
      classOf[Value].isAssignableFrom(m.getReturnType) &&
      m.getDeclaringClass != classOf[Enumeration] &&
      isValDef(m))
    methods foreach { m =>
      val name = m.getName
      // invoke method to obtain actual `Value` instance
      val value = m.invoke(this).asInstanceOf[Value]
      // verify that outer points to the correct Enumeration: ticket #3616.
      if (value.outerEnum eq thisenum) {
        val id = Int.unbox(classOf[Val] getMethod "id" invoke value)
        nmap += ((id, name))
      }
    }
  }
  // ...
}
```

因为 `val` 会对应生成一个 `method` 和一个 `field`，所以本质上我们只需要获取返回类型为 `Value` 并且存在 `field` 名字与其相同的 `method` 的名字（外加一些其他条件）即可，最后把所有名字和其对应的 `id` 放在一个 `Map` 中，用时只需查一下 `Map`。

---
References:

1. [Why is a Scala companion object compiled into two classes (both Java and .NET compilers) ? - StackOverflow](https://stackoverflow.com/questions/12783994/why-is-a-scala-companion-object-compiled-into-two-classesboth-java-and-net-com)
2. [A Look at How Scala Compiles to Java](http://blog.thegodcode.net/post/239967776/a-look-at-how-scala-compiles-to-java)
3. [Scala Object 实现](http://gaocegege.com/Blog/scala/scalaobj)
4. [Higher-order function - Wikipedia](https://en.wikipedia.org/wiki/Higher-order_function)
5. [Scala 中的语言特性是如何实现的 - 崔鹏飞的 Octopress Blog](http://cuipengfei.me/blog/2013/05/05/how-are-scala-language-features-implemented/)
6. [Java instance variable and method having same name - StackOverflow](https://stackoverflow.com/questions/9960560/java-instance-variable-and-method-having-same-name)
