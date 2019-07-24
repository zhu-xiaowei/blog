---
title: kotlin混合开发下
date: 2019-07-20 09:43:40
tags: 技术
---

![](https://img.carlwe.com/kotlin_and_java_logo.png)

<!--more-->

> 回顾：上篇博客介绍了kotlin的一些基本语法和新特性。我们知道了kotlin会更加的实用、简洁和安全。

### Kotlin混合开发原理

上面我们了解到了kotlin如此多的好处，同时也是谷歌官方确认的Android开发的第一语言，那么我们如何在我们现有的java项目中引入kotlin呢，这里就要提到了kotlin的另一个特性：互操作性。

> 互操作性：可以放心的使用现有的库、可以自由的在Java和Kotlin源码文件之间切换、混合开发时可以在不同语言的代码中单步调试、重构java代码时kotlin也会正确执行。

有了这个特性使用起来就方便多啦，这就代表着我们能够直接在现有的项目中直接使用kotlin：在kotlin的代码中可以调用java代码，在java代码中也可以轻松调用kotlin代码。那这背后的原理是什么呢？我们下面来分析。

#### App构建过程介绍

![](https://img.carlwe.com/kotlin_build_process.png)

我们先从App的构建过程讲起，和 Java 一样， Kotlin 也是编译型语言。所以你必须先编译，然后生成.class字节码才能正确执行。不同的是，Kotlin有自己的编译器，会识别对应的kotlin语法，同时拥有kotlin自己的运行时库，提供一些java所不具备的功能。另外用 Kotiin 编译器编译的代码依赖 Kotlin 运行时库。它包括了 Kotlin 自己的标准库类的定义，以及 Kotiin对标准 JavaAPI的扩展。 Gradle 还会帮我们把 Kotlin 运行时库作为依赖加入到你的应用程序中。运行时库需要和你的应用程序一起分发 。所以加入kotlin之后当你的应用打包后，通过dexcount工具你会发现增加了相当一部分kotlin的代码（大概4000个方法），不过包体积的增大相对于kotlin给我们带来的方便可以忽略。

![](https://img.carlwe.com/kotlin_added_methods.png-m)

#### 编译器工作过程

上面介绍到kotlin和java最主要的不同在与编译器，那我们就来看看编译器的工作过程。

![](https://img.carlwe.com/kotlin_compile_progress.png)

java和kotlin的编译过程类似。在大学中上过编译原理的同学对这个肯定不会陌生。简单来说，上面整个过程其实就是翻译我们写代码的过程，下面介绍如下三个概念：

> 词法分析：识别代码中的关键字、元算符，同识别一个句子中的单词类似。
>
> 语法分析：将上面识别到的单词序列组合成各类语法短语，类似学生用多个词语组句一样。
>
> 语义分析：判断组合成的句子是否符合编码规范（变量定义类型是否正确，运算符是否匹配）。类似检查我们说话时是否有病句。

上面流程中语义分析完成后就会进入到目标代码生成阶段。字节码生成器会负责该项工作，生成最后的JVM字节码。那么kotlin和java的最大区别其实就是在于字节码生成器，kotlin会按照自己的语法规则，生成对应的字节码，下面让我们来详细看下目标代码生成的过程。

#### 目标代码生成

![](https://img.carlwe.com/kotlin_target_code.png)

通过最简单的变量生成Set、Get方法为例，我们首先定义了一个Int类型的变量a初始值为1，我们找到kotlin编译器的源码发现，kotlin在目标代码生成阶段多了判断是否需要生成set()、get()方法的逻辑，通过AndroidStuido的Decompiled功能我们能够直接看到编译之后的代码。可以看到自动帮我们生成了setA()和getA()两个方法。

可见Kotlin编译器在编译前端（即词法分析、语法分析、语义分析、中间代码生成）并没有做让人感到很厉害的事情，和Java是基本一致的，所以混合开发就变得水到渠成了。与Java相比，所与众不同的细节是在编译后端（目标代码生成）环节。Kotlin编译器在目标代码生成环节做了很多**类似于Java封装**的事情，比如自动生成Getter/Setter代码的生成、Companion转变成静态类、修改类属性为final不可继承等工作。可以说，大部分Kotlin的特性都在这个环节处理产生。那么总结来说：Kotlin将我们本来在代码层做的一些封装工作转移到了编译后端的阶段，这就是为什么kotlin使用起来如此的简单的原因了。

上面我们了解啦kotlin混合开发的原理。在介绍混合开发之前，我们首先需要知道：

> 同一个文件中的代码要么是kotlin代码要么是java代码。混合开发指的是在不同文件中调用彼此的代码。

混合开发大部分时候并不需要关心太多：我们可以像往常一样直接在java代码中调用kotlin定义的方法和属性。也可以直接在kotlin代码中调用java代码定义的方法和属性，你并不会感觉到会有多大的差异，但是由于语言特性，在某些功能的实现上我们需要做一些特殊的处理。那么下面我们就分别从相互调用的两个方向来进行介绍**（阅读需要有java基础，可选择性跳过）**

### Kotlin中调用Java代码

![](https://img.carlwe.com/use_java_in_kotlin.png)

**1、Getter和Setter**

> 遵循 Java 约定的 getter 和 setter 的方法（名称以 `get` 开头的无参数方法和以 `set` 开头的单参数方法）在 Kotlin 中表示为属性。 `Boolean` 访问器方法（其中 getter 的名称以 `is` 开头而 setter 的名称以 `set` 开头）会表示为与 getter 方法具有相同名称的属性。 例如:

```kotlin
//kotlin
val myHouse = House("5th Avenue,NY.", 200000.0, true)//java类House
myHouse.price = 300000.0   //调用setPrice()方法
myHouse.isNewHouse = false //调用isNewHouse()方法
println("price:${myHouse.price}\nisNewHouse:${myHouse.isNewHouse}")//调用getPrice()方法
```

**2、Java中使用了Kotlin的关键字**

> 一些 Kotlin 关键字在 Java 中是有效标识符：*in*、 *object*、 *is* 等等。 如果一个 Java 库使用了 Kotlin 关键字作为方法，你仍然可以通过反引号（`）字符转义它来调用该方法:

```kotlin
//kotlin
fun main(args: Array<String>) {
    val arr = arrayListOf("kotlin", "java", "and")
    //将 Kotlin 中是关键字的 Java 标识符进行转义
    println(StringUtil.`in`(arr))
}
```

**3、空安全与平台类型**

> Java 中的任何引用都可能是 *null*，这使得 Kotlin 对来自 Java 的对象要求严格空安全是不现实的。 Java 声明的类型在 Kotlin 中会被特别对待并称为*平台类型*。

```java
//java
public interface DataParser<T> {
    void parseData(String input, List<T> output, List<String> errors);
}
```

> kotlin类实现java的DataParser接口时，每个参数的类型是否可空、集合是否可变，可以根据实际情况来定义。

```kotlin
//kotlin
class PersonParse:DataParser<Person>{
    override fun parseData(input:String, 
                           output:MutableList<Person>, 
                           //根据实际情况也可以定义为MutableList<String>
                           errors:MutableList<String?>)
}
```

**4、Java可变参数**

> Java 类有时声明一个具有可变数量参数（varargs）的方法来使用索引

```java
//java
public class JavaArrayExample {
	//接受可变参数
    public List<Integer> removeZero(int... intArr) {
        List<Integer> resultArr = new ArrayList<>();
        for (int value : intArr) {
            if (value != 0) {
                resultArr.add(value);
            }
        }
        return resultArr;
    }
}
```

> 在这种情况下，你需要使用展开运算符 `*` 来传递 `IntArray`

```kotlin
//kotlin
val javaObj = JavaArrayExample()
val array = intArrayOf(0,1,2,3)
println(javaObj.removeZero(*array))
```

**5、Java数组**

> Java 平台上，数组会使用原生数据类型以避免装箱/拆箱操作的开销。 由于 Kotlin 隐藏了这些实现细节，因此需要一个变通方法来与 Java 代码进行交互。 对于每种原生类型的数组都有一个特化的类（`IntArray`、 `DoubleArray`、 `CharArray` 等等）来处理这种情况。 它们与 `Array` 类无关，并且会编译成 Java 原生类型数组以获得最佳性能，Java代码如下：

```java
//java
public class JavaArrayMethod {
    public void removeIndices(int[] indices){
    }
}
```

> kotlin调用java代码

```kotlin
val javaObj = JavaArrayMethod()
val array = intArrayOf(0, 1, 2, 3)
javaObj.removeIndices(array)  // 将 int[] 传给方法
array[1] = array[1] * 2 // 不会实际生成对 get() 和 set() 的调用
for (x in array) {// 不会创建迭代器
    println(x)
}
```

**6、Kotlin中的 Java 泛型**

> Java 的通配符转换成类型投影，Java的原始类型转换成星投影，Java代码：

```java
//java
public class JavaPattern<Animal> {
    public void Test1(JavaPattern<? extends Animal> list) {}
    public void Test2(JavaPattern<? super Animal> list) {}
    
    public static void printColl(ArrayList<?> al) {
        Iterator<?> it = al.iterator();
        while (it.hasNext()) {
            System.out.println(it.next().toString());
        }
    }
}
```

> 转换后的kotlin代码

```kotlin
class KotlinPattern1<Animal> {
    fun Test1(list: KotlinPattern1<out Animal>) {}
    fun Test2(list: KotlinPattern1<in Animal>) {}
    
    fun printColl(al: ArrayList<*>) {
        val it = al.iterator()
        while (it.hasNext()) {
            println(it.next().toString())
        }
    }
}
```

**7、在 Kotlin 中使用 JNI**

> 要声明一个在本地（C 或 C++）代码中实现的函数，你需要使用 `external` 修饰符来标记它

```kotlin
external fun add(x: Int,y: Int): Double
```

### Java中调用Kotlin代码

![](https://img.carlwe.com/use_kotlin_in_java.png)

**1、属性**

> Kotlin 属性会编译成以下 Java 元素：getter 方法，名称通过加前缀 `get` 算出；setter 方法，名称通过加前缀 `set` 算出（只适用于 `var` 属性，kotlin代码：

```kotlin
var firstName: String
```

> 在编译时会生成如下Java代码：

```java
private String firstName;
public String getFirstName() {
    return firstName;
}
public void setFirstName(String firstName) {
    this.firstName = firstName;
}
```

**2、包级函数**

> Kotlin文件`File`中声明的所有的函数和属性，包括扩展函数， 都编译成一个名为`FileKt` 的 Java 类的静态方法，可以使用 `@JvmName` 注解修改生成的 Java 类的类名：

```kotlin
@file:JvmName("Utils")
package com.ltz.kotlintest.usekotlininjava.example2
fun printLowerCase(str: String) {
    println(str.toLowerCase())
}
```

> java中调用

```java
Utils.printLowerCase("Hello XiaoHui");
```

> 如果多个文件中生成了相同的 Java 类名（包名相同并且类名相同或者有相同的 `@JvmName` 注解）通常是错误的。然而，编译器能够生成一个单一的 Java 外观类，它具有指定的名称且包含来自所有文件中具有该名称的所有声明。 要启用生成这样的外观，请在所有相关文件中使用 @JvmMultifileClass 注解

```java
//Fun.kt
@file:JvmName("Utils")
@file:JvmMultifileClass
package com.ltz.kotlintest.usekotlininjava.example2

fun printLowerCase(str: String) {
    println(str.toLowerCase())
}
```

```kotlin
//Fun1.kt
@file:JvmName("Utils")//@JvmName 注解修改生成的 Java 类的类名
@file:JvmMultifileClass
package com.ltz.kotlintest.usekotlininjava.example2

fun printUpperCase(str: String) {
    println(str.toUpperCase())
}
```

```java
//Java
Utils.printUpperCase("hello xiaohui");
Utils.printLowerCase("Hello XiaoHui");
```

**3、静态字段和方法**

- 静态字段

> 在命名对象或伴生对象中声明的 Kotlin 属性会在该命名对象或包含伴生对象的类中具有静态幕后字段。
>
> 通常这些字段是私有的，但可以通过以下方式之一暴露出来：
>
> - `@JvmField` 注解；
> - `lateinit` 修饰符；
> - `const` 修饰符。

```kotlin
//Kotlin
class Key(val value: Int) {
    //伴生对象
    companion object {
        @JvmField //使用 @JvmField 标注这样的属性使其成为与属性本身具有相同可见性的静态字段
        val COMPARATOR: Comparator<Key> = compareBy<Key> { it.value }
    }
}
//命名对象
object Singleton {
     lateinit var key: Key
      const val SingletonConst = 1
}
```

```java
//Java
Key.COMPARATOR.compare(new Key(1), new Key(2));
Singleton.key = new Key(1);
int c = Singleton.SingletonConst;
```

- 静态方法

> Kotlin 可以为命名对象或伴生对象中定义的函数生成静态方法，如果你将这些函数标注为 `@JvmStatic` 的话，编译器既会在相应对象的类中生成静态方法，也会在对象自身中生成实例方法。 例如

```kotlin
//kotlin
class C {
    companion object {//伴生对象
        @JvmStatic fun foo() {}
        fun bar() {}
    }
}
//命名对象
object Obj {
    @JvmStatic fun foo() {}
    fun bar() {}
}
```

```java
//Java 
C.foo(); // 没问题
C.bar(); // 错误：不是一个静态方法
C.Companion.foo(); // 保留实例方法
C.Companion.bar(); // 唯一的工作方式

Obj.foo(); // 没问题
Obj.bar(); // 错误
Obj.INSTANCE.bar(); // 没问题，通过单例实例调用
Obj.INSTANCE.foo(); // 也没问题
```

**4、生成重载**

> 如果你写一个有默认参数值的 Kotlin 函数，在 Java 中只会有一个所有参数都存在的完整参数签名的方法可见，如果希望向 Java 调用者暴露多个重载，可以使用 `@JvmOverloads` 注解。

```kotlin
//kotlin
class Foo @JvmOverloads constructor(x: Int, y: Double = 0.0) {
    @JvmOverloads fun f(a: String, b: Int = 0, c: String = "abc") { …… }
}
```

> 上面的例子最终会生成如下代码：

```java
// 构造函数：
Foo(int x, double y)
Foo(int x)

// 方法
void f(String a, int b, String c) { }
void f(String a, int b) { }
void f(String a) { }
```

**5、用 @JvmName 解决签名冲突**

> 有时我们想让一个 Kotlin 中的命名函数在字节码中有另外一个 JVM 名称，最突出的例子是由于*类型擦除*引发的，下面两个函数在kotlin中能同时定义，因为它们的 JVM 签名是一样的：

```kotlin
fun List<String>.filterValid(): List<String>
fun List<Int>.filterValid(): List<Int>
```

> 我们可以用`@JvmName` 去标注其中的一个（或两个），并指定不同的名称作为参数

```kotlin
fun List<String>.filterValid(): List<String>

@JvmName("filterValidInt")
fun List<Int>.filterValid(): List<Int>
```

> Java中调用

```java
List<String> stringArr = Arrays.asList("abc", "efa", "bde");
List<Integer> intArr = Arrays.asList(1, 2, 3, 4, 5);
KotlinUseJvmNameKt.filterValid(stringArr);
KotlinUseJvmNameKt.filterValidInt(intArr);
```

**6、受检异常**

> Kotlin 没有受检异常。 所以通常 Kotlin 函数的 Java 签名不会声明抛出异常。 于是如果我们有一个这样的 Kotlin 函数

```kotlin
// example.kt
package demo

fun foo() {
    throw IOException()
}
```

> 然后我们想要在 Java 中调用它并捕捉这个异常

```java
// Java
try {
  demo.Example.foo();
}
catch (IOException e) { // 错误：foo() 未在 throws 列表中声明 IOException
  // ……
}
```

> 因为 `foo()` 没有声明 `IOException`，我们从 Java 编译器得到了一个报错消息。 为了解决这个问题，要在 Kotlin 中使用 `@Throws` 注解

```kotlin
@Throws(IOException::class)
fun foo() {
    throw IOException()
}
```

上面这些算是实际项目中混合开发的核心了。我们能有个提前的了解就可以。当我们遇到类似的问题的时候对这个问题有印象知道有办法解决就行，至于怎么解决再去网上查询也不迟。

### Kotin引入的影响

对于Android开发来说，kotlin的引入有如下方面的改变：

* 1.基本不需要再写findViewById. 可通过静态布局引入直接使用布局Id，比ButterKnife更简洁。原理在[这里](https://blog.csdn.net/hust_twj/article/details/80290362)
* 2.会增加很多可空性判断，需要我们来关注和处理。
* 3.如何使用kotlin的语法特性让原有代码更简洁，逻辑更清晰。

### 开发中的问题汇总

**1.资源文件命名**

由于可以直接在Fragment或者Activity中使用资源文件的id作为view的对象，但是以往，资源文件的id通常使用下划线命名例如：`android:id="@+id/buy_input_edit_text"` ，java类中通常使用驼峰法命名。我们在使用时通过findViewById转换成buyInputEditText。但是现在如果直接在kotlin中使用buy_input_edit_text作为对象，会违背命名规范。所以建议使用kotlin后直接在资源文件中使用驼峰法命名id。就像这样：`android:id="@+id/buyInputEditText"` 。

**2.有些文件仍然需要使用findViewById**

在不是Activity、Fragment或自定义view的一些类中使用布局id仍然需要手动的findViewById. 例如adapter、工具类等。但这毕竟是少数，可以接受。

**3.变量定义时的非空设置**

当我们转换一个现有的java文件到kotlin时，Android Stuido会自动帮我们做一些转换，经常碰到的情况是，当前页面的数据在网络请求后才能获取被赋值。所以在转换时默认会被定义成可空类型，在后续使用时每次调用都会进行空判断，或者非空断言：

```kotlin
val info : mInfo? = null
titleTv.text = mInfo?.title
contentTv.text = mInfo?.content
if(mInfo?.isValid!!) ...
```

这样看上去整个类中就会增加密密麻麻的"?"，代码编译后也会生成很多不必要的空检查。所以如果我们能够确认这个属性只有在初始化后才会使用，那么可以添加lateInit修饰符。将代码简化：

```kotlin
lateinit val info : mInfo
titleTv.text = mInfo.title
if(mInfo.isValid) ...
```

如果在使用过程中有些地方不确定被`lateinit`修饰对象是否被初始化，并且需要调用其属性，我们可以给该属性添加判断初始化的方法：

```kotlin
lateinit val info : mInfo
fun isInitialzed() = ::info.isInitialized
```

另外需要注意的是，如果该变量在init{}语句块中有初始化，则不需要添加`lateinit`修饰符。

**4.空判断的一些技巧**

* ？和 ?：的巧妙用法

在java中有三元表达式，可以方便的进行简单的判断并返回值。但是kotlin则需要使用if...else…表达式：

```java
int a = (info != null && info.value != null) ? info.value : 0 //java
```

```kotlin
val a = if(info != null && info.value != null) info.value else 0 //kotlin
```

其实像上面在一连串空判断后不为空时取值本身，否则取默认值的例子很多，我们可以通过kotlin `?` 和 `?:`两个操作符让其变得更简单：

```kotlin
val a = info?.value?:0
```

* if判断条件可能为null的处理

在kotlin中if中的判断结果要么是true要么是false,不能为null，所以很多时候自动转换会出现如下判断：

```kotlin
if(mDialog != null && mDialog!!.isShowing){
    mDialog!!.dismiss()
}
```

这个时候我们进行简化：

```kotlin
if(mDialog?.isShowing == true){
    mDialog!!.dismiss()
}
```

直接进行想要结果的判断就可以，这个是够不管前面的isShowing是false还是null.判断结果都是false.和上面的判断是等效的，同样判断其他类型的值相等也可以。

但是需要注意的是，如果这个时候添加了else.在else中则是两种情况。我们需要做对应的处理。

* `?.let{}` 简化同一个对象的多次非空判断

像上面的例子就可以通此操作符继续优化：

```kotlin
mDialog?.let{
    if(it.isShowing == true){
        it.dismiss()
    }
}
```

可以看到let{}块中使用时可通过it进行访问，同时省去了所有的空检查

* `!!` 非空断言谨慎使用

同时上面的例子我们可以看到，我们已经判断了非空，所有后面 `mDialog!!.dismiss() ` 能够直接使用非空断言。在开发中我们尽量少使用非空断言，除非是在明确了不可能为null的情况。尤其是在通过java一键转kotlin的时候。我们需要对转换之后的逻辑进行确认。

**5.as转换的问题**

当我们在一键转换代码时，会遇到as转换自动为非空类型的问题，这会给代码带来很大的隐患。

```java
Info info = (Info)data.getExtra(INFO); //java代码
```

自动转换成kotlin代码后：

```kotlin
val info = data.getExtra(INFO) as Info 
```

这里需要我们手动的添加可空：`as Info?` ，避免找不到对象造成的类型转换出错。

### 总结

本篇文章从大家从kotlin与java混合开发的原理进行展开。介绍了混合开发中相互调用需要注意的问题，最后结合实际开发中的一些发现，告诉大家需要注意的问题。本文仅列出了当前发现的一些问题。如有后面有新的发现还会持续更新。

如果还没有看过看过上篇对kotlin的简介，可以前往这里查看 👉[《kotlin混合开发上》](<https://carlwe.com/2019/04/30/Kotlin%E6%B7%B7%E5%90%88%E5%BC%80%E5%8F%91%E4%B8%8A/>)