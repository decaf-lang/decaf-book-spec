# LL(1) 文法分析

## 基本原理

在Lecture04中，我们介绍了递归下降的LL(1)分析方法，并给出了一系列非终结符对应的分析函数。例如为了分析文法

```text
<function> -> FUNC ID ( <parameter_list> ) <statement>
```

我们定义分析函数 `ParseFunction`：

```java
void ParseFunction() {
    MatchToken(FUNC);
    MatchToken(ID);
    MatchToken(LPAREN);
    ParseParameterList();
    MatchToken(RPAREN);
    ParseStatement();
}
```

从 `ParseFunction` 的实现可以总结出：为了完成对非终结符 `<function>` 的分析，只需依次对 `<function>` 产生式右部的各符号进行分析。

- 遇到终结符（如`FUNC`）时，调用`MatchToken`函数来匹配；
- 遇到非终结符（如`<parameter_list>`）时，递归调用它对应的分析函数（如`ParseParameterList`）进行分析。

此外，为了将分析出的非终结符`<function>`所对应的分析结果（这里考虑其AST结点的值）记录下来，
我们可以像 PA1-A 那样，用一个`SemValue`类的对象记录每个语法符号对应的结果，然后根据这些结果完成某个用户定义的语义动作。
据此，上述分析函数修改为：

```java
SemValue ParseFunction() {
    SemValue[] params = new SemValue[6 + 1];
    params[1] = MatchToken(FUNC);
    params[2] = MatchToken(ID);
    params[3] = MatchToken(LPAREN);
    params[4] = ParseParameterList();
    params[5] = MatchToken(RPAREN);
    params[6] = ParseStatement();

    params[0] = new SemValue();
    // do user-defined actions
    return params[0];
}
```

这里，我们用长度比产生式右部符号数多1的SemValue数组，来缓存对产生式右部各符号进行分析得到的结果（`params[1]`到`params[6]`）。
然后，我们执行用户定义的语义动作，即用户访问并修改`params`数组中的数据。
最终，返回`params[0]`作为非终结符`<function>`的分析结果。

以上讨论了非终结符对应于单一产生式的情形。
如果某个非终结符对应于多个产生式（即存在多种分析方法），那么我们需要向后查看一个单词来决定使用哪一个产生式（哪一种分析方法），
然后再利用上述方法分析该产生式右部的符号。
在Java语言中，我们用一个 switch-case 语句即可实现根据下一个单词选择相应产生式的逻辑。具体例子请见Lecture04，这里不再赘述。

## 文法分析入口

形如 `ParseFunction` 的分析函数是类似的，为了避免重复，我们在 `LLParser` 类中把它们统一为通用的 `parseSymbol` 函数，
并把非终结符（如 `<function>`）作为第一个参数传入：

```java
SemValue parseSymbol(int symbol, Set<Integer> follow)
```

其中 `symbol` 为待分析的非终结符。若分析成功，则返回值存储了该 `symbol` 所对应AST结点的值；若分析失败，则返回 `null`。
我们提供的代码框架中已经实现了不带错误恢复功能的 LL(1) 分析算法，实现思路与

```java
SemValue ParseFunction()
```

类似。请注意，框架中给出的实现仅在输入程序语法正确的情况下有效；针对语法错误的程序输入，抛异常是正常现象。
实验开始前，请仔细阅读这部分的代码，有必要时插入一些打印语句输出调试信息，以便理解该算法的思路。
暂时，你无需关心第二个参数 `follow`。

## 文法简化

针对 Decaf 语言规范中的文法

```text
simple      ::= var ('=' expr)? | lValue '=' expr | expr | ε
```

由于将其改写为等价的 LL(1) 文法十分复杂。作为*妥协*，本阶段我们将上述文法扩展为

```text
simple      ::= var ('=' expr)? | expr ('=' expr)? | ε
```

即先在文法层面允许 `lValue` 为任意 `expr`，然后在语义动作中检查等号左边的表达式是否是 `lValue`：

```java
// SimpleStmt : Expr Initializer
if ($1.expr instanceof LValue) {
    var lv = (LValue) $1.expr;
    $$ = svStmt(new Assign(lv, $2.expr, $2.pos));
} /* ... */
```

但在之后的各阶段，我们依然以语言规范（PA1-A 的实现符合规范）为准。

## LL(1) 文法改写

在 `parseSymbol` 函数中，我们调用了如 `query`、`act` 等 ll1pg 工具生成的 `LLTable` 类中的方法。
ll1pg 工具会读取 `Decaf.spec` 文件所描述的 LL(1) 文法，并生成预测分析表（通过 `query` 接口获得），
以及语义动作表（通过 `act` 接口获得）。
文法描述文件的格式与 Jacc 类似，详情请查阅 [项目 Wiki](https://github.com/paulzfm/ll1pg/wiki/1.-Specification-File)。

> 警告：课程使用的 ll1pg 版本与 `master` 分支上的有所不同。
> 此 Wiki 描述的 `<headers>` 文法与 `Decaf.spec` 不一致。除 `%tokens` 外，实验时请勿修改其他头部声明。

在 `build.gradle` 脚本中已经设置了每次编译 Java 代码前运行 ll1pg 工具，生成 `LLTable.java`，
默认路径为 `build/generated-src/ll1pg/LLTable.java`。
如果 `Decaf.spec` 文件所描述的文法不是 LL(1)，那么会给出警告。
例如，框架给出的文法并非严格 LL(1)，ll1pg 工具运行时给出如下警告：

```text
Warning: conflict productions at line 260:
ElseClause -> ELSE Stmt
ElseClause -> <empty>
```

请阅读 [项目 Wiki](https://github.com/paulzfm/ll1pg/wiki/2.-Resolving-Conflicts)，了解此工具对于是如何解决冲突的。

> 注意：修改 `Decaf.spec` 后，你需要仔细阅读给出的警告。
> 如果工具默认的冲突解决方式不是你想要的，那么请将相关部分的文法重写为严格的 LL(1)，以避免潜在的错误。
