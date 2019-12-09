# JVM 字节码简介

这一节将会介绍有关 JVM 字节码的基础知识。
更多有关 JVM 运行机制和指令集的介绍，请移步[官方文档](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)。
请注意：本次实验生成的字节码以 Java 8 版本 (major version 52) 为准。

## Class 文件的组织

JVM 字节码以类为单位进行组织，每个类对应于一个 class 文件。例如考虑这个简单的 Decaf 类：

```java
class Main {
    static void main() {
        Print("hello world");
    }
}
```

为其生成的 `Main.class` 文件的内容（使用 `javap -v Main` 指令反汇编）是：

```text
public class Main
  minor version: 0
  major version: 52
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #2                          // Main
  super_class: #4                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 2, attributes: 0
Constant pool:
   #1 = Utf8               Main
   #2 = Class              #1             // Main
   #3 = Utf8               java/lang/Object
   #4 = Class              #3             // java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = NameAndType        #5:#6          // "<init>":()V
   #8 = Methodref          #4.#7          // java/lang/Object."<init>":()V
   #9 = Utf8               main
  #10 = Utf8               ([Ljava/lang/String;)V
  #11 = Utf8               java/lang/System
  #12 = Class              #11            // java/lang/System
  #13 = Utf8               out
  #14 = Utf8               Ljava/io/PrintStream;
  #15 = NameAndType        #13:#14        // out:Ljava/io/PrintStream;
  #16 = Fieldref           #12.#15        // java/lang/System.out:Ljava/io/PrintStream;
  #17 = Utf8               hello world
  #18 = String             #17            // hello world
  #19 = Utf8               java/io/PrintStream
  #20 = Class              #19            // java/io/PrintStream
  #21 = Utf8               print
  #22 = Utf8               (Ljava/lang/String;)V
  #23 = NameAndType        #21:#22        // print:(Ljava/lang/String;)V
  #24 = Methodref          #20.#23        // java/io/PrintStream.print:(Ljava/lang/String;)V
  #25 = Utf8               Code
{
  public Main();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #8                  // Method java/lang/Object."<init>":()V
         4: return

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #16                 // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #18                 // String hello world
         5: invokevirtual #24                 // Method java/io/PrintStream.print:(Ljava/lang/String;)V
         8: return
}
```

可以看出，一个 class 文件由以下三部分构成：

- 元信息：包括类名、版本号 (`minor version`, `major version`)、访问权限 (`ACC_PUBLIC`)、父类 (`super_class`)、接口信息 (`interfaces`) 等。
- 常量池：包括该文件中所有会用到的常量，如 `#17` 表示 `hello world` 常量字符串；以及 JVM 对各类成员及其类型的内部引用，如 `#24` 是对 `java.io.PrintStream.print` 方法的引用。
- 类成员：包括该类的所有成员（继承自父类且没有覆盖的除外），每个成员都会记录其**访问权限**（以 `ACC_` 开头的那些）和**类型**（JVM 中叫做 descriptor）。
对于方法，其代码段 (code) 会记录下 JVM 指令序列，以及元信息，包括局部变量个数 (`locals`)、参数列表大小 (`args_size`) 等。

## 数据的存储

除了把常量存放在常量池以外，JVM 还支持以下两种方式存储数据：

### 操作数栈

JVM 是一种栈式虚拟机。在这种计算模型中，几乎所有指令都依赖于从栈顶弹出一个或多个数据作为操作数，指令运行完成后，结果再压入栈顶。
例如，为了实现整数加法运算，我们先将两个操作数压栈，再执行 `IADD` 指令，结果会存入栈顶。
使用栈的一个好处在于，指令能省下很多地址（操作数），因为我们约定了一个指令的输入输出都在栈的哪些位置，无需再在指令中给出。
在上面这个例子中，`IADD` 就是一条零地址的指令。而在 TAC 中，`ADD` 是一条三地址指令。
由于指令更加简单，这也大大简化代码生成的工作。
与 TAC 相比，在本阶段，我们无需用类似于 `Temp` 的变量缓存下指令的输出结果。

### 临时变量

虽然操作数栈很好用，但它毕竟是个栈，而栈并不能很好地支持按照偏移量来访问其中的元素。
为了弥补这个缺陷，JVM 在每个方法的调用帧内存储一个临时变量数组，并提供相应指令完成按下标进行读写。
临时变量的作用有：(1) 函数调用时传参；(2) 记录程序局部变量的值。
简而言之，临时变量数组相当于 TAC 中的临时变量 (Temp) 构成的数组。

## 类型

与 TAC 不同，JVM 区分不同位数的整数、数组以及对象引用，并为它们分别设计了相匹配的指令。
Decaf 中的各类型与 JVM 类型的对应关系如下表：

| Decaf | JVM |
| :---: | :-: |
| `int`  | `int` |
| `bool` | `int`/`byte` |
| `string` | `Ljava/lang/String;` |
| 数组 | 数组类型 |
| 对象 | 引用类型 |

需要特别强调的是：虽然 JVM 支持布尔类型，但是没有针对布尔类型提供相应的指令。
因此，在默认情况下我们总是用一个 JVM 的 32位整数表示该布尔值——`0` 表示 `false`，`1` 表示 `true`。
一个特别的情况是，布尔类型的数组 (Decaf `bool[]`) 直接用 JVM 的布尔数组类型表示（其底层实现是每个元素占一个字节）。

在 JVM 中，类型都用一个字符串来表示，被称为**descriptor**。其中，域（field，即成员变量）的 descriptor 的语法表示为：

```text
FieldDesc ::= FieldType
FieldType ::= BaseType | ObjectType | ArrayType
BaseType  ::= 'B' | 'C' | 'D' | 'F' | 'I' | 'J' | 'S' | 'Z'
ObjectType ::= 'L' ClassName ';'
ArrayType ::= '[' FieldType
```

其中，我们会涉及到的基本类型 (`BaseType`) 有：`'B'` 表示字节，`'I'` 表示整数，`'Z'` 表示布尔。
对象类型 (`ObjectType`) 以 `'L'` 开头，后面跟上类名，并以一个分号结束。
如 `Ljava/lang/String;` 就表示 Java 的字符串类型 `java.lang.String`，注意在这里 `.` 被替换成了 `/`。
数组类型 (`ArrayType`) 以 `'['` 开头，后面跟上数组元素的类型。如 `[Ljava/lang/String;` 表示字符串数组类型。

方法的 descriptor 的语法表示为：

```text
MethodDesc ::= '(' FieldType* ')' ReturnDesc
ReturnDesc ::= FieldType | 'V'
```

一个方法的 descriptor 由两部分组成：(1) 各参数的类型，它们直接拼接起来，并用一对圆括号括起来；
(2) 返回值的类型，注意无返回值时用 `'V'` (i.e. void) 表示。
例如，主函数的 descriptor 为 `([Ljava/lang/String;)V`，即 Java 里面的 `void(String[])`。
`(IZLjava/lang/Thread;)Ljava/lang/Object;` 表示 Java 里面的 `Object(int, boolean, Thread)`。

## 指令集概述

以下简要介绍我们在 Decaf 中用到到一些 JVM 指令。
更加详细的请查阅 [文档](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html)。
为方便，本节列出的所有指令都链接了它们的文档，请点击指令名称查看。

### Load/Store

这里的 load/store 跟 TAC 是完全不同的！JVM 的 load/store 是指在**临时变量**与**操作数栈**之间进行数据交换。

- 把一个临时变量的值压入操作数栈：[`ILOAD`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.iload) / [`ALOAD`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.aload)
- 把操作数栈栈顶元素写入临时变量：[`ISTORE`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.istore) / [`ASTORE`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.astore)
- 把常量压入操作数栈：[`LDC`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.ldc)

上述指令都需要输入一个参数——目标临时变量的编号，即它在那个临时变量数组里的下标。
LOAD 指令执行完成后，栈顶元素为读取的数据。STORE 指令要求在执行前，栈顶元素为待写入的数据。
作为 JVM 的惯例，以 `I` 开头的指令操作整数，以 `A` 开头的指令操作引用。

> 注意：JVM 要求在读取一个临时变量前该变量已经有值（即被写入过）。因此，我们在程序中声明一个变量时，就立即将其写为初始值。
> 没有指定初始值的，我们可以赋默认值 0。

### Arithmetic & Logical

以下是整数相关的算术、逻辑运算指令：

- 加法：[`IADD`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.iadd)
- 减法：[`ISUB`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.isub)
- 乘法：[`IMUL`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.imul)
- 除法：[`IDIV`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.idiv)
- 取模：[`IREM`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.irem)
- 取负：[`INEG`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.ineg)
- 逻辑或：[`IOR`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.ior)
- 逻辑与：[`IAND`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.iand)

指令执行前，要求右操作数位于次栈顶，且左操作数位于栈顶。
指令执行完毕，栈顶元素为计算结果。

### Control Transfer

JVM 支持类似于 TAC 和汇编中的条件/无条件跳转指令，以实现高级语言中的如分支、循环等包含控制流转让的语句。

- 无条件跳转：[`GOTO`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.goto)
- 条件跳转：[`IF_ICMP<cond>`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.if_icmp_cond) / [`IF_ACMP<cond2>`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.if_acmp_cond) / [`IFNULL`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.if_null)

参数为要跳转到的目标标签。

其中，`IF_ICMP` 系列适用于比较两个整数，条件 `<cond>` 取 `EQ`, `NE`, `LE`, `LT`, `GE`, `GT` 之一（含义显然）。
指令执行前，要求待比较的两个数分别位于次栈顶和栈顶。当二者满足 `<cond>` 条件时发生跳转，否则继续往后顺序执行。
而 `IF_ACMP` 系列适用于比较两个引用，条件 `<cond2>` 只能取 `EQ` 和 `NE`，即引用无法比较大小。

### Array

JVM 内置数组类型。
采用 [`NEWARRAY`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.newarray) / [`ANEWARRAY`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.anewarray) 指令创建新数组。
其中，`NEWARRAY` 用于创建基本类型的数组，其参数为数组元素的类型；而 `ANEWARRAY` 用于创建引用的数组，无参数。
指令执行前，要求栈顶元素为数组长度。执行完毕后，栈顶元素为数组的引用。

用 [`ARRAYLENGTH`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.arraylength) 指令来获取数组长度。
指令执行前，要求栈顶元素为数组引用。执行完毕后，栈顶元素为长度。

类似于 `ILOAD` / `ISTORE`，数组也支持 LOAD / STORE 操作（第1个字母标识类型，第2个字母 `A` 表示 array）：

- 把数组元素压入操作数栈：[`BALOAD`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.baload) / [`IALOAD`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.iaload) / [`AALOAD`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.aaload)
- 把操作数栈顶元素写入数组：[`BASTORE`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.bastore) / [`IASTORE`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.iastore) / [`AASTORE`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.aastore)

### Object

JVM 不支持直接操作内存的指令。我们在创建对象（实例）的时候需要使用指令 [`NEW`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.new)，其参数为该对象的类型。新建对象的引用将被压入操作数栈。

对于类中的成员域（成员变量），我们采用 [`GETFIELD`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.getfield) 和 [`PUTFIELD`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.putfield) 指令分别进行读和写。
参数为待读写的域名称，操作数栈的栈顶元素为读取结果或待写入值。

此外，[`INSTANCEOF`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.instanceof) 指令支持 Decaf 和 Java 的 instance-of 检查。
该指令的参数为一个类型 `A`，且指令执行前待检查对象的引用需要位于栈顶。
如果该对象是 `A` 的子类型，那么检查结果为真，`1` 被压入栈顶；否则 `0` 被压入栈顶。
类似地，使用 [`CHECKCAST`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.checkcast) 指令可以完成向上转型。
若转型失败，运行时会抛出 `ClassCastException`。

### Method Invocation

目前的框架中使用到了以下三种 JVM 支持的函数调用方式：

- [`INVOKESTATIC`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.invokestatic)：调用类里面声明的静态方法
- [`INVOKESPECIAL`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.invokespecial)：调用构造方法
- [`INVOKEVIRTUAL`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.invokevirtual)：调用成员方法

这些指令的参数为待调用函数的名称，且参数已经依次按压入操作数栈。

此外，JVM 还支持 [`INVOKEINTERFACE`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.invokeinterface) 和 [`INVOKEDYNAMIC`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.invokedynamic)。

函数返回采用 [`RETURN`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.return) 指令（无返回值时）；
或者 [`IRETURN`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.ireturn) 等指令（有返回值时），这时要求操作数栈栈顶元素为待返回的值。

> 注意：JVM 要求任何函数必须要在最后一条指令结束时返回。
> 但是，我们知道一个无返回值的函数体内可能没有返回语句。一个简单的解决方法是，若最后一条指令不是返回指令时，则强行插入一条 `RETURN` 指令。

### 指令使用案例

[这里](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-3.html) 给出了诸多 Java 代码翻译成 JVM 指令的案例。

## 调用惯例

JVM 在调用一个需要 `k` 个参数的函数时，先从栈顶弹出 `k` 个元素，依次作为第 `k, k - 1, ... 1` 个参数。
调用发生后，程序执行的控制权由调用者 (caller) 转移到被调用者 (callee)。
在被调用者的临时变量数组中，前 `k` 个元素被依次初始化为第 `1, 2, ..., k` 个参数。
此后，程序从调用者代码的第一条开始执行。
执行到返回指令时，控制权由被调用者回到调用者，且此时操作数栈栈顶记录了返回值（如果函数返回值不是 `void`）。

特别地，对于成员方法，我们需要在本来的参数列表前面加入一个 self 参数传入 this 对象。
因此，0 号临时变量（若存在）的值总是 this 对象的引用。
