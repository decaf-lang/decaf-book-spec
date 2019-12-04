# Summary

* [语言规范](spec.md)
* Java 框架分阶段指导
  + [本章导引](impl-java/index.md)
  + PA1-A：语法分析器的自动构造
    - [阶段任务](impl-java/PA1-A/index.md)
    - [词法分析](impl-java/PA1-A/lexer.md)
    - [抽象语法树](impl-java/PA1-A/tree.md)
    - [文法分析](impl-java/PA1-A/parser.md)
  + PA1-B：基于 LL(1) 的语法分析器半自动构造
    - [阶段任务](impl-java/PA1-B/index.md)
    - [LL(1) 文法分析](impl-java/PA1-B/parser.md)
    - [错误恢复](impl-java/PA1-B/recovery.md)
  + PA2：语义分析
    - [阶段任务](impl-java/PA2/index.md)
    - [类型、符号和作用域](impl-java/PA2/datatype.md)
    - [访问者模式](impl-java/PA2/visitor.md)
    - [符号表构建](impl-java/PA2/table.md)
    - [类型检查](impl-java/PA2/typecheck.md)
  + PA3：中间代码生成
    - [阶段任务](impl-java/PA3/index.md)
    - [TAC 程序](impl-java/PA3/TAC.md)
    - [面向对象机制](impl-java/PA3/oop.md)
    - [控制流的翻译](impl-java/PA3/control.md)
* Scala 框架分阶段指导
  + [本章导引](impl-scala/index.md)
  + PA1：语法分析器的自动构造
    - [阶段任务](impl-scala/PA1/index.md)
    - [词法分析](impl-scala/PA1/lexer.md)
    - [抽象语法树](impl-scala/PA1/tree.md)
    - [文法分析](impl-scala/PA1/parser.md)
    - [访问者模式](impl-scala/PA1/visitor.md)
  + PA2：语义分析
    - [阶段任务](impl-scala/PA2/index.md)
    - [类型、符号和作用域](impl-scala/PA2/datatype.md)
    - [符号表构建](impl-scala/PA2/table.md)
    - [类型检查](impl-scala/PA2/typecheck.md)
  + PA3：中间代码生成
    - [阶段任务](impl-scala/PA3/index.md)
    - [TAC 程序](impl-java/PA3/TAC.md)
    - [面向对象机制](impl-java/PA3/oop.md)
    - [控制流的翻译](impl-scala/PA3/control.md)
* Rust 框架分阶段指导
  + [本章导引](impl-rust/index.md)
  + PA1-A：语法分析器的自动构造
    - [实验内容](impl-rust/PA1-A/实验内容.md)
    - [lalr1使用指导](impl-rust/PA1-A/lalr1使用指导.md)
      - [编写lexer](impl-rust/PA1-A/编写lexer.md)
      - [impl块的可选属性](impl-rust/PA1-A/impl块的可选属性.md)
      - [产生式和语法动作](impl-rust/PA1-A/产生式和语法动作.md)
      - [解决冲突](impl-rust/PA1-A/解决冲突.md)
      - [一个完整的例子](impl-rust/PA1-A/一个完整的例子.md)
    - [抽象语法树](impl-rust/PA1-A/抽象语法树.md)
    - [框架中部分实现的解释](impl-rust/PA1-A/框架中部分实现的解释.md)
    - [文件结构](impl-rust/PA1-A/文件结构.md)
  + PA1-B：基于 LL(1) 的语法分析器半自动构造
    - [实验内容](impl-rust/PA1-B/实验内容.md)
    - [lalr1使用指导](impl-rust/PA1-B/lalr1使用指导.md)
    - [错误恢复](impl-rust/PA1-B/错误恢复.md)
    - [文件结构](impl-rust/PA1-B/文件结构.md)
  + PA2：语义分析
    - [实验内容](impl-rust/PA2/实验内容.md)
    - [语义分析](impl-rust/PA2/语义分析.md)
    - [符号表](impl-rust/PA2/符号表.md)
    - [visitor模式](impl-rust/PA2/visitor模式.md)
  + PA3：中间代码生成
    - [实验内容](impl-rust/PA3/实验内容.md)
    - [中间代码](impl-rust/PA3/中间代码.md)
    - [中间代码中的类型信息](impl-rust/PA3/中间代码中的类型信息.md)
    - [运行时存储布局](impl-rust/PA3/运行时存储布局.md)
    - [面向对象机制](impl-rust/PA3/面向对象机制.md)
    - [tacvm简述](impl-rust/PA3/tacvm简述.md)
  + PA4：中间代码优化
    - [实验内容](impl-rust/PA4/实验内容.md)
    - [基本块](impl-rust/PA4/基本块.md)
    - [数据流分析概述](impl-rust/PA4/数据流分析概述.md)
    - [数据流优化概述](impl-rust/PA4/数据流优化概述.md)
    - [公共表达式提取](impl-rust/PA4/公共表达式提取.md)
    - [复写传播](impl-rust/PA4/复写传播.md)
    - [常量传播](impl-rust/PA4/常量传播.md)
    - [死代码消除](impl-rust/PA4/死代码消除.md)
  + PA5：寄存器分配
    - [实验内容](impl-rust/PA5/实验内容.md)
    - [图着色基本原理](impl-rust/PA5/图着色基本原理.md)
    - [着色算法](impl-rust/PA5/着色算法.md)
    - [预着色节点](impl-rust/PA5/预着色节点.md)
    - [干涉图节点合并](impl-rust/PA5/干涉图节点合并.md)
    - [调用约定](impl-rust/PA5/调用约定.md)
