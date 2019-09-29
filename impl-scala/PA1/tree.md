# 抽象语法树

抽象语法树 (Abstract Syntax Tree, AST) 是贯穿 Decaf 编译器整个前端的重要数据结构。
“抽象语法”是相对于“具体语法”而言的。在具体语法树 (Concrete Syntax Tree) 上，所有语法符号都会被记录。
而在 AST 上，那些我们在编译器后面的阶段不再需要关心的符号（如逗号等）则被忽略。
简而言之，AST 是一种在忽略冗余语法细节的前提下，对源代码**结构**的忠实表达。

在 Scala 框架中，我们分别定义了不带类型标注的 AST (`SyntaxTree`) 和带类型标注的 AST (`TypedTree`)。
它们有诸多类似的 AST 结点。考虑二元运算表达式 `Binary`。无论是否有类型，`Binary` 都至少包含了三个元素：二元操作符及两个操作数。
但是，在 `SyntaxTree` 中，这两个操作数是不带类型的，因此它们的类型应该是 `SyntaxTree.Expr`。
而在 `TypedTree` 中，这两个操作数是带类型的，因此它们的类型应该是 `TypedTree.Expr`。此外，我们还要能够得到 `TypedTree.Expr` 的类型。
为此，我们期望能有一种方法，在不重复写 `Binary` 两次的情况下，也能定义出这两种实际上不同的语法结点。

## AST 模板

在 `TreeTmpl.scala` 中，我们给出了一个一般化的 AST 模板。
该模板包含了 `SyntaxTree` 和 `TypedTree` 共有的语法结点。同时，我们允许每个语法结点有一个额外的域来存储如类型这种“标注” (annotation)。
标注定义在 `decaf.frontend.annot` 包中。例如：

```scala
trait TreeTmpl {
  type ExprAnnot <: Annot

  trait Expr extends Node with Annotated[ExprAnnot]

  case class Binary(op: BinaryOp, lhs: Expr, rhs: Expr)
                   (implicit val annot: ExprAnnot) extends Expr
}
```

我们将该语法树结点的三个元素定义在第一个参数组里。其中，`Expr` 是指这个 trait 里面的那个 `Expr` 类型。
而标注则定义在第二个参数组里，并作为隐式参数 (implicit parameter)。
这个标注的类型是抽象的，我们先约束其上界为 `Annot`（所有标注类型都得是它的子类型），然后等实例化的时候再指定即可。
这样做的好处是当这个标注为空时，我们可以定义一个隐式的空标注，那么在构造该结点时就可以直接忽略第二个参数组了。

## 模板实例化

在我们实例化上述模板时，例如实例化成 `SyntaxTree`：

```scala
implicit object NoAnnot extends Annot

object SyntaxTree extends TreeTmpl {
  type ExprAnnot = NoAnnot.type
}
```

由于 `SyntaxTree` 无需标注，我们先定义一个 `NoAnnot` 当作空标注。
并在 `SyntaxTree` 中重写 `ExprAnnot` 类型成员为 `NoAnnot` 这个单例对象的类型 `NoAnnot.type`。
这样，Scala 编译器在进行 trait mixin 时，上述代码会被展开为（示意）：

```scala
trait TreeTmpl // just a type

implicit object NoAnnot extends Annot

object SyntaxTree extends TreeTmpl {
  type ExprAnnot = NoAnnot.type

  trait Expr extends Node with Annotated[NoAnnot.type]

  case class Binary(op: BinaryOp, lhs: SyntaxTree.Expr, rhs: SyntaxTree.Expr)
                   (implicit val annot: NoAnnot.type) extends SyntaxTree.Expr
}
```

这样我们就得到所期望的 `SyntaxTree.Binary` 了。由于 `NoAnnot` 是一个隐式值，且类型为 `NoAnnot.type`。那么在构造语法树结点时，

```scala
val expr1: Expr = ???
val expr2: Expr = ???
val node = Binary(TreeNode.ADD, expr1, expr2)
```

会被展开为

```scala
val expr1: Expr = ???
val expr2: Expr = ???
val node = Binary(TreeNode.ADD, expr1, expr2)(NoAnnot)
```

注意隐式参数 `NoAnnot` 被编译器自动查找到并填充。

完全类似地，`TypedTree` 的定义也很简单：

```scala
object TypedTree extends TreeTmpl {
  type ExprAnnot = Type
}
```

只需知道表达式的标注是 `Type`。但是，目前我们只能通过 `.annot` 来读取其标注的类型。
这似乎不是特别友好——光从 `annot` 的名字看不出来它到时是何种标注。而 `typ` 似乎是一个对于类型标注来说更合理的名字。
为了实现这个特性，我们使用 Scala 的隐式类 (implicit class)：

```scala
object TypeImplicit {
  implicit class TypeAnnotatedHasType(self: Annotated[Type]) {
    def typ: Type = self.annot
  }
}
```

即：让任何标注了 `Type` 的对象 `self` 都具有一个 `typ` 方法，它的返回 `self.annot`，而此时 `self.annot: Type`。
于是，我们就可以直接这样取得一个语法树结点的类型：

```scala
val node: TypedTree.Expr = ???
node.typ // : Type, == node.annot
```

## AST 就是一棵树

从名称来看，AST 就是一棵树。在本框架的实现中，我们也贯彻了这一想法，即让每一个 AST 结点都具有树状的结构。
具体来说，对于任何一个 AST 结点，我们需要知道它有几棵子树，以及这些子树分别是什么。
特别地，如果一个 AST 结点不包含任何子树，那么它就是叶子结点。

Scala 的 case class 天生支持我们期望的功能——只需将所有 AST 结点定义成 case class，并且把子树依次作为该类（主）构造函数的第一个参数组。
回顾我们对于 `Binary` 的定义：

```scala
case class Binary(op: BinaryOp, lhs: SyntaxTree.Expr, rhs: SyntaxTree.Expr)(...)
```

其构造函数的第一个参数组包含了 `op`, `lhs` 和 `rhs`。它们构成了 `Binary` 的三棵子树：二元操作符、左操作数和右操作数。

在本框架中，这样设计的最大好处在于——大幅简化格式化打印 AST 的代码量！打印策略的实现（见 `PrettyTree.scala`）非常简单：

1. 打印名称 (`productPrefix`)
2. 增加缩进
3. 遍历所有子树 (`productIterator`)，依次递归打印它们
4. 减少缩进

注意到 `productPrefix` 和 `productIterator` 都已经是 case class 的成员。
Scala 编译器在实现 case class 时，会让它们继承 `scala.Product` 并自动实现上述两个接口。

## 评注：Scala Implicit

Implicit 是 Scala 的一大特色，隐式转换、隐式参数极大地提高了编码效率，也使得 Scala 成为一门极易于构建内置式领域特定语言
(embedded domain-specific language) 的通用程序语言。
但是，由于 implicit 变幻莫测，对新手不太友好，滥用 implicit 会导致项目变得难以维护，甚至行为不可控。这导致很多用户并不喜欢这个特性。
这有点像 Haskell 的 monad，新手觉得晦涩难懂，专家用得出神入化。
Scala 的下一代版本将会对 implicit 这一特性进行大改版，以符合更多工业界人的口味。
在本框架中，我们仅在必需的场合使用了部分常见 implicit 特性，以降低编码冗余。
