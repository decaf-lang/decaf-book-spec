# 文法分析

本框架的文法分析器由 [Jacc](http://web.cecs.pdx.edu/~mpj/jacc/) 生成。
Jacc 是一个类似于 [yacc](http://dinosaur.compilertools.net/yacc/index.html) 的工具，同样基于 LALR(1) 分析。
只不过，它能生成 Java 源码。

## Jacc 中的优先级与结合性

`Decaf.jacc` 文件包含了所有 Decaf 的文法。
Jacc 文法描述形式如同 BNF，遗憾的是，它**不支持** EBNF 中诸如 `?`/`+`/`*` 等符号。
因此，遇到这样的文法时，我们不得不手动将其转换为普通的 BNF。
转换方式不难想到，你可以参考我们已经定义出来的文法，并与语言标准中给出 EBNF 进行比对。

在 `Decaf.jacc` 文件的导言区，我们用如下格式的一张“表”定义出了 Decaf 语言操作符的优先级和结合性：

```jacc
%left OR
%left AND
%nonassoc EQUAL NOT_EQUAL
%nonassoc LESS_EQUAL GREATER_EQUAL '<' '>'
%left  '+' '-'
%left  '*' '/' '%'
%nonassoc UMINUS '!'
%nonassoc '[' '.'
%nonassoc ')' EMPTY
%nonassoc ELSE
```

不难猜测出，结合性由 `%left` 等命令指定，而优先级按照声明的顺序**由低到高**。
更多细节请查阅[用户手册](http://web.cecs.pdx.edu/~mpj/jacc/jacc.pdf)。

## 左递归

Jacc 采用 LR 分析，因此左递归并不会引起冲突。你可以在书写文法时自由使用左递归。

## 语义动作

与 JFlex 类似，在 Jacc 中定义文法时，我们可以在每个产生式（右部）后面跟上一个用大括号括起来的 Java 代码片段。
这个代码片段被称为“语义动作”，即当用这个产生式进行规约以后，我们要做的事情。
在本框架中，语义动作就是将该产生式左部非终结符对应的那个语法树结点构造出来，并存储到*语义值*中。
如果语法分析成功，那么最终我们会用到开始符号对应的那个产生式进行规约，规约时的语义动作就是把语法树的根结点构造出来：

```jacc
TopLevel :  ClassList
            {
                tree = new TopLevel($1.classList, $1.pos);
            }
            ;
```

这里的 `tree` 是语法分析阶段我们期望的输出。

每个语法符号（终结符和非终结符）都会对应着一个语义值，其定义参见 `SemValue.java`。
需要特别指出的是，在 LR 分析过程中，我们常用一个语义值**栈**来维护当前状态下各个语法符号对应的语义值。
这就无形中要求所有语义值都得是**同一**（或者它们有公共的上界，homogeneous）类型。
但是我们的语法树分为好多种 (heterogenous) 类型，比如语句、表达式等等。
一种可行的方法是栈里面全部存所有语法树结点的基类 `TreeNode`，这样无论你是啥结点都可以表示。
但问题在于，当你需要读取一个特定的类型，如表达式的时候，就得不得把这个 `TreeNode` 对象向下转型为 `Expr`。这个方法十分丑陋。
作为惯例，Jacc 工具要求用户设计出来一个 `SemValue` 类，把要用到的那些具体的类型全部作为这个类的成员：

```java
class SemValue {
    final Kind kind;

    // For parser
    Tree.ClassDef clazz;
    List<Tree.ClassDef> classList;

    Tree.Field field;
    List<Tree.Field> fieldList;

    List<Tree.LocalVarDef> varList;

    Tree.TypeLit type;

    Tree.Stmt stmt;
    List<Tree.Stmt> stmtList;
    Tree.Block block;

    Tree.Expr expr;
    List<Tree.Expr> exprList;
    Tree.LValue lValue;

    Tree.Id id;

    /* ... */
}
```

这样就不用强转了，只需直接访问指定的成员即可，如 `.stmt`, `.expr` 等。用 C++ 的话说，`SemValue` 就是个 *union*。

以下列 if 语句为例：

```jacc
Stmt :  IF '(' Expr ')' Stmt ElseClause
        {
            $$ = svStmt(new If($3.expr, $5.stmt, Optional.ofNullable($6.stmt), $1.pos));
        }
```

产生式左部 `Stmt` 所对应的语义值用 `$$` 表示，而产生式右部的符号按照从左往右的顺序依次用 `$1`, `$2`, ... 表示。
在语法树中，一个 if 语句用类 `If` 表示，它的构造函数签名如下：

```java
public If(Expr cond, Stmt trueBranch, Optional<Stmt> falseBranch, Pos pos)
```

我们通过 `$3.expr` 访问到括号中的表达式，`$5.stmt` 访问到第一个语句，`$6.stmt` 访问到第二个语句（如果没有，则是 null）。
最后，我们需要设置该结点的位置——`IF` 那个单词对应的位置。

此外，`AbstractParser` 类提供了一组用于构造特定类型语义值的函数，如这里的 `svStmt` 函数输入一个语法树上的 `Stmt` 结点，
返回一个 `kind` 是语句的 `SemValue`。请在开始实验前阅读并了解所提供的接口，便于在开发时使用。

由于 IDE 一般不会支持像 `.jacc` 这种“小众”格式的源码，为了方便你写出符合 Java 语法的语义动作代码，
你可以先在 `SemValue.java` 提供的 `UserAction` 函数里面实现，然后再粘贴到 `.jacc` 文件中。
