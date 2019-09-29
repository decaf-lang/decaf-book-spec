# 阶段任务

第一阶段的任务是完成编译器的最前端——语法分析器的构造。

## 背景介绍

主流的语法分析器构造方法大致可以分为两类：

### 自动生成器 (Parser generator)

这类工具往往需要要求用户根据该工具规定好的格式输入一个文法的描述，如类似于 EBNF 的文法，并自动生成该文法对应的分析器。
这类工具以 lex/yacc 等一批经典工具为代表。所生成的分析器往往是源码，便于用户集成到自己的编译器中使用。
此外，一些工具（如 Antlr）还可以自动生成相应的库函数，方便用户直接在编译器中调用。
一般来说，生成源码的好处在于集成方便，对第三方库的依赖少（甚至无依赖），但是缺点在于，用户需要了解所生成代码中有哪些可以调用的函数和可以操作的变量，
耦合性较大，也往往需要再编写一些额外的代码来跟生成的代码进行交互。为了减轻这种耦合性，我们可以选择能生成库函数的工具，来保持开发上的便捷。
在 Java 版的实现中，我们采用了前一种更加传统的方式。在 Scala 版的实现中，我们采用了后一种方式。

### 组合子库 (Parser combinators library)

所谓“组合子”，一般指函数式编程中随处可见的高阶函数。说得再理论点，它就是 lambda 演算系统中不含有自由变元 (free variable) 的项 (term)，
简称“闭项” (closed term)。使用组合子库进行语法分析器的构造，是函数式语言中常见的做法。
Haskell 著名的 [Parsec](http://hackage.haskell.org/package/parsec) 库开创了这一先河。
与采用自动生成器的方法不同，使用组合子库在构造语法分析器，就和你调其他库没有任何区别——无需提前生成代码，分析器在运行时直接运行，并解析输入串。
因此，你甚至可以在运行时新增操作符、改变文法，实现更加“动态”的语法分析。
但是，使用组合子库的一个大问题在于，如果开发者不熟悉库函数的行为，那么很有可能构造出来的分析器和期望的不同，且这种不同只有在运行时才会暴露出来。
相比之下，自动生成器对输入文法有所要求（如要求其是 LALR(1)、LL(1) 等），若不满足则会报出错误或警告。
Decaf 项目暂时未采用这种方式，感兴趣的同学可从 Haskell 的 [Parsec](http://hackage.haskell.org/package/parsec)，
或者 Scala 的 [scala-parser-combinators](https://github.com/scala/scala-parser-combinators) 开始学习。

除此之外，很多工业界的编译器在完成自举 (bootstrap) 之后，往往会手工构造语法分析器。
这样做有两个目的：第一，验证该编程语言是否自己能够写出它自己的语法分析器。
第二，人工构造有助于更好地优化语法分析过程，更自由地根据上下午来调整分析的流程，并在分析出错时给出更加合理的报错信息。
完全人工构造工作量大，因此本框架未采用。但是，我们在下一阶段设计了一种工作量可控的“半手工”构造方案。
该方案基于 LL(1) 分析，同学们只需要编写高层次的 LL(1) 分析算法和错误恢复算法，而更加底层的分析表由工具自动生成。

本阶段的任务是，通过 JFlex 和 Jacc 这两个工具完成语法分析器的自动构造。

## 相关文件

与本阶段相关的文件有：

```text
src
└── main
    ├── jflex
    │   └── Decaf.jflex         词法描述
    ├── jacc
    │   └── Decaf.jacc          文法描述
    └── java
        └── decaf
            ├── frontend
            │   ├── parsing
            │   │   ├── AbstractLexer.java      抽象词法分析器，提供词法分析的辅助方法
            │   │   ├── AbstractParser.java     抽象文法分析器，提供文法分析的辅助方法
            │   │   ├── JaccParser.java         本阶段文法分析入口
            │   │   ├── SemValue.java           文法分析器使用的语义值
            │   │   └── Tokens.java             单词列表
            │   ├── tree
            │   │   ├── TreeNode.java           语法树基类定义
            │   │   ├── Tree.java               语法树定义
            │   │   ├── Visitor.java            语法树访问者
            │   │   └── Pos.java                语法符号在源码中的位置
            └── printing
                ├── PrettyPrinter.java          格式化缩进打印器
                └── PrettyTree.java             语法树缩进打印
```