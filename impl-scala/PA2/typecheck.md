# 类型检查

这一阶段对应于代码中的 `Typer`，具体任务是：分析程序中的每一个表达式和语句，检查其是否类型正确，并构造出 `TypedTree` 上的对应结点。

## 自顶向下类型检查

一种经典的类型检查过程按照语法树的结构自顶向下，并依据一系列归纳定义的定型规则 (typing rules) 来进行检查。
考虑一个很简单的算术表达式语言：

```text
expr ::= expr '+' expr | n | s
```

其中 `n` 指一个整数常量，`s` 指一个字符串常量。我们设计如下两条定型规则：

- T-Int 规则：任何整数常量具有整数类型。
- T-Add 规则：若表达式 `e1` 具有整数类型，且表达式 `e2` 具有整数类型，那么表达式 `e1 + e2` 具有整数类型。

对于表达式 `2 + 4` 进行类型检查的过程如下：

1. 根据 T-Int 规则，`2` 具有整数类型。
2. 根据 T-Int 规则，`4` 具有整数类型。
3. 由以上两条，根据 T-Add 规则，`2 + 4` 具有整数类型。

定型规则未定义的视为类型错误，例如我们无法根据上述两条定型规则来推断 `1 + "str"` 的类型，那么它就无类型 (ill-typed)。
> 上述定型规则是用自然语言描述的，这是“非形式化”且可能不严格的。
> 一个严谨的编程语言，需要用形式化系统来精确刻画其定型规则。
> 对此感兴趣的同学，可从经典教材 Types and Programming Languages 入门。

虽然 Decaf 语言的定型规则比上面的玩具例子要复杂得多，但是在已知定型规则的情况下编码实现类型检查算法并不困难——你只需把规则逐条翻译成代码。
> 这个观点也不是放之四海而皆准，因为在很多类型系统特别花哨（如依赖类型）的语言中，定型规则往往高度不确定 (non-deterministic)，甚至还需要进行（有可能还是不可判定的）逻辑推理。

## 特别机制

此外，框架代码还实现了以下两种特别的机制，以得出更友好和更有用的错误信息。

### 类型猜测

根据定型规则，例如我们在分析一个“大”表达式的类型之前，会先分析出其中各“小”表达式的类型。
对于无法得出正确类型的表达式，我们会尽量先推测出一个合理的类型，即使类型检查不通过，也把推测的类型标注在 AST 结点上，而不总是标注 `ERROR`。
因为当用户修复这个表达式后，那么它的类型就应该是推测出来的类型。例如 `instanceof` 表达式，只要正确，必然是 bool 类型。
这样，即使 `instanceof` 是错的，包含它的更“大”的表达式仍然可以假设它是 bool 来继续分析。这样我们可以得出更多有意义的结果。

### 防止错误扩散

另一方面，如果“小”表达式已经被标注为 `ERROR`，那么“大”表达式显然类型检查也通过不了。
为了避免那个“小”表达式的错误扩散到所有包含它的“大”表达式中，我们会特判这种情形，不重复报错。
但是，若这个“大”表达式的其他部分产生了新的错误，则依然需要报告这些新的错误。

## 语句检查

在 Decaf 中，有关语句的类型检查主要分为两个部分——对语句中出现的表达式进行类型检查，和记录该语句是否返回了值。前者的必要性不言自明。
后者是为了保证所有声明了返回类型（非 void）的方法都一定会返回一个值，且通过 return 语句的类型检查，这个值必然是所声明的返回类型（的一个子类型）。

### 定型规则

各语句对其中出现的表达式的类型有如下要求：

- 赋值语句中，右值类型是左值类型的子类型
- if/while/for 语句中，条件表达式是 bool 类型
- return 语句返回值类型是方法返回类型的子类型
- print 语句表达式的类型只能是 int, bool 或 string

此外，break 语句必须出现在循环体中。

### 返回值规则

除以下给出的规则外，其他情况下的语句均不返回值：

- 一个 if 语句返回值，当且仅当它的 false 分支存在，且两个分支均返回了值
- 一个 return 语句返回值，当且仅当其返回值对应的表达式存在
- 一个语句块返回值，当且仅当它非空，且最后一条语句返回了值

## 表达式的类型检查

表达式只需做类型检查。为简单，我们用记号 `e : t` 表示表达式 `e` 的类型为 `t`，若我们猜测 `e` 的类型也为 `t`，则记为 `e : !t`。
定型规则和猜测机制如下：

- 常量表达式：具有该常量的类型（int, bool, string, null）
- `ReadInt() : int`
- `ReadLine() : string`
- 一元取反表达式：`!e : !bool` iff `e : bool`
- 一元取负表达式：`-e : !int` iff `e : int`
- 二元算术表达式（`op` 属于 `+,-,*,/,%`）：`e1 op e2 : !int` iff `e1 : int` and `e2 : int`
- 二元逻辑表达式（`op` 属于 `&&,||`）：`e1 op e2 : !bool` iff `e1 : bool` and `e2 : bool`
- 二元比较表达式（`op` 属于 `<,<=,>,>=`）：`e1 op e2 : !bool` iff `e1 : int` and `e2 : int`
- 二元判等表达式（`op` 属于 `==,!=`）：`e1 op e2 : !bool` iff `t1 <: t2` or `t2 <: t1` given `e1 : t1` and `e2 : t2`
- `new t[] : t[]` iff `t !== void`
- `new A() : class A` 若 `A` 有定义
- `this : class A` 若当前方法是成员方法且定义在类 `A` 中
- `x : t` 若 `x` 是当前作用域有定义的符号且类型为 `t`
- `obj.x : t` 若 `obj : class A`，`x` 对当前作用域可见且类型为 `t`
- `a[e] : !t` given `a : t[]`, iff `e : int`
- `a.length() : int` iff `a : t[]`
- 若 `e1 : t1, ..., en : tn`，则 `f(e1, ..., en) : t` iff `f : (s1, ..., sn) => s`, `t1 <: s1`, ..., `tn <: sn`, `s <: t`
- `instanceof(e, class A) : !bool` 若 `e` 是一个对象且 `A` 有定义
- `(class A) e : class A` 若 `A` 有定义

> 这里我们引用易于普通人理解的自然语言描述了定型规则，今后我们可能会制定出更加形式化的规范。
