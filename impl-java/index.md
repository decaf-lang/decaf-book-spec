# Java 版 Decaf 编译器分阶段实验指导

本书的这一部分介绍 Java 框架各阶段实验的主要任务以及框架中一些重要的实现方法。

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

在 Java 框架中，上述每个阶段都被定义为一个 `Phase`：

```java
public abstract class Phase<In, Out> implements Task<In, Out>, ErrorIssuer
```

每个阶段都是一个可能失败的函数 `Function<In, Optional<Out>>`，其中 `In` 为该阶段作为输入的某种语言表示，而 `Out` 为该阶段作为输出的某种语言表示。
例如，文法分析阶段就是从某个输入流读取 Decaf 源码，输出语法树：

```java
public Task<InputStream, Tree.TopLevel> parse() {
  return new JaccParser(config);
}
```

而一次完整的编译过程，无非就是把这些阶段像流水线一样给拼接 (pipe) 起来。由于每个阶段都是一个有可能失败的函数，这种拼接就对应于函数的某种组合：

```java
public interface Task<T, R> extends Function<T, Optional<R>> {
    default <V> Task<T, V> then(Task<R, V> next) {
        return t -> this.apply(t).flatMap(next);
    }
}
```

具体来说，`f.then(g)` 就是，先做 `f`，如果成功，把输出的结果作为输入传给 `g`，返回 `g` 的结果（成功或者失败）。
如果 `f` 已经失败了（执行 `f` 的过程中产生错误），那么直接终止整个流程，返回失败。
用 Haskell 的话讲，这 (`then`) 不就是个 Kleisli composition 吗？

借助这个 `then` 函数，我们能方便地定义出各阶段编译器要执行的任务：

```java
public class TaskFactory {

    public Task<InputStream, Tree.TopLevel> parse() { // PA1-A
        return new JaccParser(...);
    }

    public Task<InputStream, Tree.TopLevel> parseLL() { // PA1-B
        return new LLParser(...);
    }

    public Task<InputStream, Tree.TopLevel> typeCheck() { // PA2
        return parse().then(new Namer(...)).then(new Typer(...));
    }

    public Task<InputStream, TacProg> tacGen() { // PA3
        return typeCheck().then(new TacGen(...));
    }

    public Task<InputStream, TacProg> optimize() { // PA4
        return tacGen().then(new Optimizer(...));
    }

    public Task<InputStream, String> mips() { // PA5
        var emitter = new MipsAsmEmitter();
        return tacGen().then(new Asm(...));
    }
}
```

## 代码结构

本框架源码部分的包组织如下：

```text
decaf
    .driver         主控（命令行选项解析、任务调度）
        .error      错误定义及处理
    .frontend       前端
        .tree       语法树定义
        .parsing    语法分析
        .type       类型定义
        .symbol     符号定义
        .scope      作用域定义
        .typecheck  语义分析（符号表构建、类型检查）
        .tacgen     中间代码 TAC 生成
    .backend        后端
        .dataflow   数据流分析
        .opt        TAC 优化
        .reg        寄存器分配
        .asm        汇编码生成
            .mips   Mips 平台相关
    .printing       格式化打印
    .lowlevel       底层工具链（未来会分离出来）
        .instr      （伪）指令定义
        .label      标签定义
        .tac        TAC 工具链
        .log        简单 Logger（供调试使用）
```

此外：

- `src/main/jflex` 包含了 JFlex 需要使用的词法描述文件（PA1 共用）
- `src/main/jacc` 包含了 Jacc 需要使用的文法描述文件（PA1-A）
- `src/main/ll1pg` 包含了 ll1pg 需要使用的文法描述文件（PA1-B）

## 项目构建

目前本框架基于 [gradle](http://gradle.org) 构建。该工具以 Groovy（默认）或 Kotlin（可配置）作为脚本语言。
由于 Groovy/Kotlin 已经是通用编程语言，因此你可以在脚本中编写任意复杂的业务逻辑。
项目中提供的 `gradle.build` 已经完成了基本配置。你可以自行添加新的库依赖
（如果不是 maven central 仓库请设置 `repositories` 并保证该库可公开下载），或者按需求增加自定义的 task。

工程引入了两个插件：

- JFlex 插件能在编译时进行词法分析器代码生成
- Jacc 插件能在编译时进行文法分析器代码生成

请使用 `gradle jar` 生成独立的 `.jar` 包。运行时请加上 `--enable-preview` 选项以开启 JDK 12 的新特性。

此外，你还可以使用 `gradle javadoc` 生成 JavaDoc 风格的 API 文档，以便查阅和搜索。
