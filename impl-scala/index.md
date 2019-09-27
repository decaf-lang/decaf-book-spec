# Scala 框架分阶段实验指导

本书的这一部分介绍 Scala 框架各阶段实验的主要任务以及框架中一些重要的实现方法。

## 分阶段设计

作为系统软件的重要一员，编译器是连接用户与机器的桥梁。编译器的核心功能是：
将某种对人类（相对）友好的高级语言语义保持地（但很多编译器并不能完全做到）转换为机器可执行的低级语言。
随着计算机学科的飞速发展，编译器的功能已远不止于此，代码优化 (optimization)、静态分析 (static analysis/lint)，乃至属性验证 (verification)
和程序综合 (synthesis)，已逐渐成为现代工业级编译器的标配。

编译器是一个复杂的软件，复杂到即便用上很多现代软件工程的技术手段，也远不是任何普通团队和个人能轻易掌控的。
但是，编译器也不神秘，它不就是个把高级语言翻译成低级语言的程序吗？这里最大的挑战是：高级语言往往很高级，低级语言往往很低级。
如何打破这跨越时代的变迁？分层！

分层这个概念大家并不陌生：计算机的软硬件分层、计算机网络的 OSI 七层模型与 TCP/IP 五层模型、商业网站的开后端分离、甚至深度学习框架也开始搞开后端分离。
既然高级语言和低级语言差异很大，不如设计一系列中间语言，逐个击破，多次翻译，那么就在有限的步数内解决问题。
Decaf 的编译器也是如此，在 Decaf 源码和 Mips 汇编之间，我们设计了抽象语法树、带类型标注的抽象语法树、TAC 这三种中间语言。

- 在语法分析阶段，Decaf 源码被转换为抽象语法树（PA1)
- 在语义分析阶段，抽象语法树被转换为带类型标注的抽象语法树（若程序无类型错误）（PA2）
- 在中间代码生成阶段，带类型标注的抽象语法树被转换为 TAC（PA3）
- 在汇编码生成阶段，TAC 被最终转换为 Mips 汇编（PA5）

此外，在生成汇编码前，我们还可以对 TAC 进行一系列优化，以生成执行效率更高的 TAC（PA4）。

在 Scala 框架中，上述每个阶段都被定义为一个 `Phase`：

```scala
abstract class Phase[In, Out](...) extends Task[In, Out] with ErrorIssuer
```

每个阶段都是一个可能失败的函数 `In => Option[Out]`，其中 `In` 为该阶段作为输入的某种语言表示，而 `Out` 为该阶段作为输出的某种语言表示。
例如，文法分析阶段就是从某个输入流读取 Decaf 源码，输出语法树：

```scala
class Parser extends Phase[InputStream, Tree](...)
```

而一次完整的编译过程，无非就是把这些阶段像流水线一样给拼接 (pipe) 起来。由于每个阶段都是一个有可能失败的函数，这种拼接就对应于函数的某种组合：

```scala
trait Task[-T, +U] extends Function[T, Option[U]] {
  def |>[R](g: Task[U, R]): Task[T, R] = x => this (x) flatMap g
}
```

具体来说，`f |> g` 就是，先做 `f`，如果成功，把输出的结果作为输入传给 `g`，返回 `g` 的结果（成功或者失败）。
如果 `f` 已经失败了（执行 `f` 的过程中产生错误），那么直接终止整个流程，返回失败。
用 Haskell 的话讲，这 (`|>`) 不就是个 Kleisli composition 吗？

借助这个 `|>` 组合子，我们能方便地定义出各阶段编译器要执行的任务：

```scala
val parse = new Parser
val typeCheck: Task[InputStream, TypedTree.Tree] = parse |> new Namer |> new Typer
val tac: Task[InputStream, TacProg] = typeCheck |> new TacGen
val jvm: Task[InputStream, List[JVMClass]] = typeCheck |> new JVMGen
val optimize: Task[InputStream, TacProg] = tac |> new Optimizer
val mips: Task[InputStream, String] = optimize |> new Asm
```

## 代码结构

本框架的源码部分的包组织如下：

```text
decaf
    .driver         主控（命令行选项解析、任务调度）
        .error      错误定义及处理
    .frontend       前端
        .tree       语法树定义
        .annot      语法树标注（类型、符号、作用域）定义
        .parsing    语法分析
        .typecheck  语义分析（符号表构建、类型检查）
        .tac        中间代码 TAC 生成
    .backend        后端
        .dataflow   数据流分析
        .opt        TAC 优化
        .reg        寄存器分配
        .asm        汇编码生成
    .jvm            JVM 字节码生成
    .printing       格式化打印
    .util           辅助类
    .lowlevel       底层工具链（Java 编写，未来会分离出来）
        .instr      （伪）指令定义
        .label      标签定义
        .tac        TAC 工具链
        .log        简单 Logger（供调试使用）
```

此外，`src/main/antlr4` 包含了 Antlr 需要使用的词法和文法描述文件。

## 项目构建

目前本框架基于 Scala 的官方工具 `sbt` 构建。虽然它有时会很慢，但对于这个项目来说还算可以接受。
使用 `sbt` 的一个主要优点在于，你可以直接 `sbt console` 打开 Scala REPL，方便地对项目中编写好的类和方法进行调试和测试。
工程引入了两个插件：

- Antlr 插件能在编译时进行代码生成
- assembly 插件便于创建独立的 `.jar` 包（命令 `sbt assembly`）

此外，你可以使用 `sbt doc` 生成 ScalaDoc 风格的 API 文档，以便查阅和搜索。

## 与 Java 混合编程

本框架会涉及到与 Java 混合编程的部分有：

- 文法分析器：调用 Antlr 生成的 Java 方法
- TAC 生成：调用 `decaf.lowlevel.tac` 中提供的 `ProgramWriter` 辅助完成 TAC 的生成和模拟执行，未来将支持生成 TAC 字节码
- JVM 字节码生成：调用 `org.objectweb.asm` 提供的方法辅助完成 JVM 字节码生成
- 目标平台定义：`decaf.lowlevel` 中提供了 Mips 等目标平台的基本定义，如（会用到的）指令集和寄存器等

由于 Scala 在设计时就考虑了与 JVM 平台其他语言（以 Java 为代表）的兼容性，使用 Scala 和 Java 混合编程并不困难。
在混合编程时，如何在 Java 与 Scala 的数据结构之间转换是一个高频需求。
为此，Scala 在 `scala.jdk.CollectionConverters` 包中提供了一组辅助函数，但是你仍需通过显式调用 `.asJava`/`.asScala` 的方式完成转换。
得益于 Scala 的隐式转换特性，我们在 `decaf.util.Conversions` 中封装了一组隐式转换方法。
你只要在代码中导入它们，就能实现 `java.util.List` 与 `scala.List`，以及 `java.util.Optional` 与 `scala.Option` 之间的无缝转换。
