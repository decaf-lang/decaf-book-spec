# 阶段任务

这一阶段的任务是与 PA1-A 相同，也是进行语法分析器的构造。
不同之处在于，这一阶段要求你借助 LL(1) 分析方法进行递归下降语法分析，并支持一定程度的**错误恢复**——即语法分析器在遇到语法错误时不终止分析过程，
而是继续分析，直至读到 EOF。

## 词法分析

本阶段的无需考虑手工构造词法分析器，请沿用 PA1-A 阶段 JFlex 工具生成的词法分析器，即保留上一阶段的 `Decaf.jflex`。

## LL(1) 分析表生成

本阶段实验给出了一种工作量可控的语法分析“半手工”构造方案——先利用
[ll1pg](https://github.com/paulzfm/ll1pg/tree/course) 工具从文法描述文件自动生成 LL(1) 分析表，
再完善 `LLParser.java` 中的 LL(1) 分析算法（会调用相关接口来访问工具生成的 LL(1) 分析表），以支持错误恢复。
框架中已经给出的分析算法只能处理语法正确的程序。对于存在语法错误的程序，此算法有可能抛出运行时错误。
同学们需要为其添加错误恢复功能，使其也能正确处理存在语法错误的程序，不抛出任何运行时错误。

## 相关文件

与本阶段相关的文件有：

```text
src
└── main
    ├── jflex
    │   └── Decaf.jflex         词法描述（同 PA1-A）
    ├── ll1pg
    │   └── Decaf.spec          LL(1) 文法描述
    └── java
        └── decaf
            ├── frontend
            │   ├── parsing
            │   │   ├── AbstractLexer.java      抽象词法分析器，提供词法分析的辅助方法
            │   │   ├── AbstractParser.java     抽象文法分析器，提供文法分析的辅助方法
            │   │   ├── LLParser.java           本阶段文法分析入口
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

除 `LLParser.java` 外，其他文件均与 PA1-A 一致。
