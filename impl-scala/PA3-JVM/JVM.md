# JVM 字节码简介

这一节将会介绍有关 JVM 字节码的基础知识。
更多有关 JVM 运行机制和指令集的介绍，请移步[官方文档](https://docs.oracle.com/javase/specs/jvms/se8/html/)。
请注意：本次实验生成的字节码以 Java 8 版本 (major version 52) 为准。

## class 文件的组织

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

- 元信息：包括类名、版本号 (minor version, major version)、访问权限 (ACC_PUBLIC)、父类 (super_class)、接口信息 (interfaces) 等
- 常量池：包括该文件中所有会用到的常量，如 `#17` 表示 `hello world` 常量字符串；以及 JVM 对各类成员及其类型的内部引用，如 `#24` 是对 `java.io.PrintStream.print` 方法的引用
- 类成员：包括该类的所有成员（继承自父类且没有覆盖的除外），每个成员都会记录其访问权限和类型（在 JVM 中叫做 descriptor）。
对于方法，其代码 (Code) 段会记录下 JVM 指令序列，以及元信息，包括局部变量个数 (locals)、参数列表大小 (args_size)。

## JVM 的类型

与 TAC 不同，JVM 区分不同位数的整数、数组以及对象引用，并为它们分别设计了相匹配的指令。
Decaf 中的各类型与 JVM 类型的对应关系如下表：

| Decaf | JVM |
| :---: | :-: |
| `int`  | `int` |
| `bool` | `int`/`byte` |
| `string` | `Ljava/lang/String;` |
| 对象 | 引用类型 |

TODO
