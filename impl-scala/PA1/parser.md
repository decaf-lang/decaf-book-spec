# 文法分析

本框架的文法分析器仍然由 Antlr 生成，它基于 LL(*) 分析，对其原理感兴趣的同学可以参考
[PLDI'11](https://www.antlr.org/papers/LL-star-PLDI11.pdf)。

## Antlr 中的优先级与结合性

`DecafLexer.g4` 文件包含了所有 Decaf 的文法。Antlr 的文法描述形式如同 EBNF，且支持 `?`/`+`/`*` 等常见的简写记号。
关于这些记号的准确含义，请阅读[文档](https://github.com/antlr/antlr4/blob/master/doc/parser-rules.md#subrules)。

Antlr 默认中缀运算符是**左结合**的。
你可以通过特别的标注来修改结合性，如 `<assoc=right>`，详见[文档](https://github.com/antlr/antlr4/blob/master/doc/left-recursion.md#formal-rules)。

Antlr 似乎没有提供特别的语法来声明运算符的优先级。但是，它会默认按照产生式出现的顺序，**由上而下**尝试。
例如，为了表达“乘法优先级高于加法”，你可以在定义产生式时先定义乘法的，后定义加法的。

## 左递归

LL 文法的一个“通病”在于它不支持左递归 (left-recursion)，大多数基于 PEG 和回溯的组合子库也有这个问题——一遇到左递归就自动进入死循环。
但是，像算术表达式这种在程序语言中频繁出现的元素，如果都让用户自己来消除左递归就显得很不友好。
在最新版的 Antlr 中，我们可以直接写出[左递归](https://github.com/antlr/antlr4/blob/master/doc/left-recursion.md)的文法，如：

```g4
expr: expr '*' expr
    | expr '+' expr
    | expr '(' expr ')'
    | id
    ;
```

注意这里隐含了优先级顺序：乘法高于加法，加法高于函数调用，函数调用高于一个普通的标识符。它会被自动展开为如下的 LL(1) 文法：

```g4
expr[int pr] : id
               ( {4 >= $pr}? '*' expr[5]
               | {3 >= $pr}? '+' expr[4]
               | {2 >= $pr}? '(' expr[0] ')'
               )*
             ;
```

注意：这里其实是定义了一组产生式——`expr[0], expr[1], ..., expr[5]`。
但是，Antlr 目前只能消除很简单的左递归，如果你的文法中存在间接左递归，甚至嵌套的左递归，那么 Antlr 极有可能无法自己解决，并报错。

## 终结符的写法

在词法分析阶段，我们已经为每个终结符取了一个名字，例如用 `DOT` 表示字符 `.`，用 `CLASS` 表示字符串 `class`。
为了简化文法的描述，Antlr 允许我们直接用字符或字符串常量直接指代原来的单词，如用 `'.'` 指代单词 `DOT`，用 `'class'` 指代单词 `CLASS`。
但是，使用未定义的常量是非法的，如在 Decaf 语言中未定义的单词 `'interface'`。

## AST 构造

### 用 Listener 还是 Visitor？

Antlr 支持生成 `Listener` 和 `Visitor`。
前者会自动生成一堆可以 hook 文法分析动作的监听器 (listener)，让用户可以在进入和退出某条特定产生式时执行特定的动作。
在 Java 框架中选用的 Jacc 工具，大体就属于这一类。
而后者会自动根据文法描述，为每个非终结符生成一个 CST 结点（继承自 `ParserRuleContext`），为每个终结符生成一个 `TerminalNode` 类的结点，
同时生成 Java 社区喜闻乐见的访问者（visitor）接口。
提示：如果你不知道“访问者模式”，请先阅读完下一节，再回来阅读后面的内容。

采用后者来实现 AST 构造更加简单，因为 Antlr 实际上已经生成了 CST，我们的任务就是遍历 CST 生成 AST。
其实 Decaf 编译器前端的每个阶段不都在重复着相同的故事吗？遍历一棵树，逐结点地翻译到该阶段所期望的输出形式（往往也是一棵树）——
PA1 是 AST，PA2 是带类型标注的 AST，PA3 是 TAC。
这种设计理念为许多现代编译器所推崇，如在 [Dotty](http://dotty.epfl.ch/) 编译器中，几乎所有阶段都统一采用树变换 (tree transformation)
来实现——每个阶段的任务就是把一种语法树变换为另一种语法树。

### `ParserRuleContext`

一个 `ParserRuleContext` 包含了产生式中各符号所对应的语法元素（也是一个 `ParserRuleContext` 或者 `TerminalNode`）。例如，文法

```g4
lValue
    : (expr '.')? id            # lValueVar
    | expr '[' expr ']'         # lValueIndex
    ;
```

对应于 `LValueVarContext`（第一个产生式）和 `LValueIndexContext`（第二个产生式）两个语法元素。
这里我们用 `#` 给不同的产生式打标签，便于在生成的代码中找到它们分别对应的语法元素。

类 `LValueVarContext` 的代码（Antlr 生成的）如下：

```java
public static class LValueVarContext extends LValueContext {
    public IdContext id() { /* code */ }
    public ExprContext expr() { /* code */ }
    public TerminalNode DOT() { /* code */ }

    @Override
    public <T> T accept(ParseTreeVisitor<? extends T> visitor) { /* code */ }
}
```

不难发现，文法中的 `expr` 对应于这里的 `expr()`，`id` 对应于这里的 `id()`，而终结符 `DOT` 对应于这里的 `DOT()`。
由于在产生式中，`(expr '.')` 是可选的，因此要小心 `expr()` 和 `id()` 有可能是 `null`。而 `id()` 一定是有值的。
每个非终结符对应的 `ParserRuleContext` 还会自动生成相应的 `accept` 方法，用于接受访问者的访问。

类似地，`LValueIndexContext` 的代码如下：

```java
public static class LValueIndexContext extends LValueContext {
    public List<ExprContext> expr() { /* code */ }
    public ExprContext expr(int i) { /* code */ }
    public TerminalNode LBRACK() { /* code */ }
    public TerminalNode RBRACK() { /* code */ }

    @Override
    public <T> T accept(ParseTreeVisitor<? extends T> visitor) { /* code */ }
}
```

由于文法中出现了两个 `expr`，所以 Antlr 生成的 `expr()` 将返回一个列表，分别对应于那两个表达式的 `ExprContext`。
同时还提供了 `expr(int)` 方法便于只取出特定的那个 `ExprContext`。
如果你觉得这样用下标访问太暴力了，我们可以修改文法，分别为两个 `expr` 命名，比如第一个叫 `array`，第二个叫 `index`：

```g4
lValue
    : (expr '.')? id                    # lValueVar
    | array=expr '[' index=expr ']'     # lValueIndex
    ;
```

那么 `LValueIndexContext` 就会多出两个成员：

```java
public static class LValueIndexContext extends LValueContext {
    public ExprContext array;
    public ExprContext index;
    /* others */
}
```

### 多个访问者

本节介绍在 `Parser.scala` 中，如何用多个访问者来完成语法树的翻译。
之所以我们决定采用多个访问者而不是一个，是因为只用一个访问者解决起来不优雅。
我们先看看为什么会这样？

首先，Antlr 自动生成的访问者长成这样：

```java
public class DecafParserBaseVisitor<T> extends AbstractParseTreeVisitor<T>
    implements DecafParserVisitor<T> {

    @Override
    public T visitTopLevel(DecafParser.TopLevelContext ctx) { /* code */ }

    @Override
    public T visitClassDef(DecafParser.ClassDefContext ctx) { /* code */ }

    /* ... */
}
```

每个方法都访问一种类型的 `ParserRuleContext`，并返回某种类型 `T` 的值。
但是在同一个访问者里面，这个 `T` 必须是**同一**（或者它们有公共的上界，homogeneous）类型。
这符合我们的需求吗？考虑到我们的 AST 结点有好多种 (heterogenous) 类型，那么这个 `T` 最小也得取成 `Node`。
这样会带来一个大问题：我们在用类型为 `Node` 的返回值去构造更上层的 AST 结点时，就不得不把 `Node` 再向下转型，比如：

```scala
val e: Node
e.asInstanceOf[Expr] // : Expr
```

这样非常丑陋。一种可行的解决方案是，将不同类型的 AST 结点分开为独立的访问者，例如考虑 `Expr` 和 `Stmt`：

```scala
object ExprVisitor extends DecafParserBaseVisitor[Expr] { /* code */ }
object StmtVisitor extends DecafParserBaseVisitor[Stmt] { /* code */ }
```

然后在访问如 while 语句时，用 `ExprVisitor` 访问循环条件，而用 `StmtVisitor` 访问循环体：

```scala
override def visitWhile(ctx: DecafParser.WhileContext): Stmt = positioned(ctx) {
  val cond = ctx.cond.accept(ExprVisitor)
  val body = ctx.body.accept(StmtVisitor)
  While(cond, body)
}
```

这样就无需向下转型。
