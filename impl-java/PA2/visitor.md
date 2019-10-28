# 访问者模式

在本阶段的分析算法开始前，我们先学习（或复习）一种 Java 程序员常用的设计模式——访问者模式 (visitor pattern)。

## 动机：表达式求值问题

为啥要有访问者模式呢？我们从一个很简单的表达式求值的例子说起：

```java
abstract class Expr {}
class Add extends Expr { Expr lhs; Expr rhs; }
class Sub extends Expr { Expr lhs; Expr rhs; }
class Number extends Expr { int value; }
```

我们先定义了表达式的 AST，包括叶子结点 `Number` 表示一个整数，中间结点 `Add` 和 `Sub` 分别表示加法和减法。
然后我们希望写一个方法，给定表达式，求其值。一种很自然的写法：

```java
int eval(Expr expr) {
    if (expr instanceof Add) { // 分支1
        var e = (Add) expr;
        var l = eval(e.lhs);
        var r = eval(e.rhs);
        return l + r;
    } else if (expr instanceof Sub) { // 分支2
        var e = (Sub) expr;
        var l = eval(e.lhs);
        var r = eval(e.rhs);
        return l - r;
    } else { // 分支3：expr instanceof Number
        var e = (Number) expr;
        return e.value;
    }
}
```

这样确实能达到目的，但是反复的 `instanceof` 和强制类型转换让开发者觉得——这样的代码太丑陋了！
如果程序员粗心把强制转换写错了，鬼知道在运行时的哪个阶段会莫名其妙的出来 `ClassCastException`。
为了解决这个“问题”，Java 开发者设计出了这样一种“巧妙”的方法：

既然我们不愿意强制类型转换，那么不妨把每个 if 分支提取成一个方法，然后让方法的参数就是我们要的那个类型：

```java
interface ExprVisitor<T> {
    T visitAdd(Add e);
    T visitSub(Sub e);
    T visitNumber(Number e);
}
```

接下来，如何让比如说 `visitAdd` 恰好能够应用到一个类型为 `Add` 的表达式上面呢？我们只要让每个表达式都支持一个负责转发的方法即可：

```java
abstract class Expr {
    abstract <T> T accept(ExprVisitor<T> v);
}

class Add extends Expr {
    Expr lhs;
    Expr rhs;

    @Override
    <T> T accept(ExprVisitor<T> v) { return v.visitAdd(this); }
}

class Sub extends Expr {
    Expr lhs;
    Expr rhs;

    @Override
    <T> T accept(ExprVisitor<T> v) { return v.visitSub(this); }
}

class Number extends Expr {
    int value;

    @Override
    <T> T accept(ExprVisitor<T> v) { return v.visitNumber(this); }
}
```

最后将 `eval` 方法实现为一个访问者的实例：

```java
class EvalVisitor implements ExprVisitor<Integer> {

    @Override
    int visitAdd(Add e) {
        var l = e.lhs.accept(this);
        var r = e.rhs.accept(this);
        return l + r;
    }

    @Override
    int visitSub(Sub e) {
        var l = e.lhs.accept(this);
        var r = e.rhs.accept(this);
        return l - r;
    }

    @Override
    int visitNumber(Number e) {
        return e.value;
    }
}

int eval(Expr expr) {
    var v = new EvalVisitor();
    return expr.accept(v);
}
```

这样，对于任意的 `Expr`，我们只要调用它的 `accept` 方法，那么按照 OOP 的动态分派 (dynamic dispatch) 机制，
根据这个表达式对象的具体类型，三个分支之一将会被调用。具体来说，执行 `e.accept(v)` 时：

- 如果 `e instanceof Add`，那么相当于调用的是 `Add` 类定义的 `accept` 方法，该方法会调用 `v.visitAdd`（分支1）
- 如果 `e instanceof Sub`，那么相当于调用的是 `Sub` 类定义的 `accept` 方法，该方法会调用 `v.visitSub`（分支2）
- 如果 `e instanceof Number`，那么相当于调用的是 `Number` 类定义的 `accept` 方法，该方法会调用 `v.visitNumber`（分支3）

进而用了一种更加“优雅”的方法实现了之前那个丑陋的三段 if-else 函数的功能。

## 番外：模式匹配

与其他很多标榜自己为“函数式”的语言一样，Scala 支持模式匹配 (pattern matching)。
如果我们想实现一个对加减法表达式求值的程序，那么只用模式匹配就够了：

```scala
sealed abstract class Expr
case class Add(lhs: Expr, rhs: Expr) extends Expr
case class Sub(lhs: Expr, rhs: Expr) extends Expr
case class Number(value: Int) extends Expr

def eval(expr: Expr): Int = expr match {
  case Add(l, r) => eval(l) + eval(r)
  case Sub(l, r) => eval(l) - eval(r)
  case Number(v) => v
}
```

简单易懂，易于调试。如果你漏掉了一些 case，如忘记了 `Number`：

```scala
def eval(expr: Expr): Int = expr match {
  case Add(l, r) => eval(l) + eval(r)
  case Sub(l, r) => eval(l) - eval(r)
}
```

编译器还会报警告：

```text
warning: match may not be exhaustive.
It would fail on the following input: Number(_)
       def eval(expr: Expr): Int = expr match {
                                   ^
```

你如果没看见，那么运行时求值到 `Number` 时也会抛出异常 `MatchError`，这样你很快就能意识到哪里出错了。

总结一下，为了实现相同的功能，Scala 需要定义1个抽象类、3个具体的类和1个方法，而 Java 需要定义1个抽象类、1个接口、4个具体的类和实现6个方法。
回顾这个设计模式，你会发现，我们其实就是在手动模拟一种典型的模式匹配。如果一个语言将模式匹配作为其基础语言特性，那么我们就不用费劲来手动模拟了。

## 评注：设计模式

设计模式真的是在好好“设计”吗？

无论是学术界还是工业界，已经有越来越多的人对设计模式持反对的态度。他们认为，设计模式是有害的 (Design patterns should be considered harmful)。
如果你读过所谓“精通 Java 设计模式”之流的书，并仔细思考其中的“设计理念”，你会发现，其实很多设计模式是语言设计的缺陷导致的。
反观其他一些编程语言，相当一部分设计模式在它们之中并不存在。为什么？因为根本不需要，只要正常写就行了。
而访问者模式，不就是手动模拟模式匹配和类型转换吗？
