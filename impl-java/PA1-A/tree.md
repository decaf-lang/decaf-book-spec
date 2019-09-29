# 抽象语法树

抽象语法树 (Abstract Syntax Tree, AST) 是贯穿 Decaf 编译器整个前端的重要数据结构。
“抽象语法”是相对于“具体语法”而言的。在具体语法树 (Concrete Syntax Tree) 上，所有语法符号都会被记录。
而在 AST 上，那些我们在编译器后面的阶段不再需要关心的符号（如逗号等）则被忽略。
简而言之，AST 是一种在忽略冗余语法细节的前提下，对源代码**结构**的忠实表达。

## AST 就是一棵树

从名称来看，AST 就是一棵树。在本框架的实现中，我们也贯彻了这一想法，即让每一个 AST 结点都具有树状的结构。
具体来说，对于任何一个 AST 结点，我们需要知道它有几棵子树，以及这些子树分别是什么。
特别地，如果一个 AST 结点不包含任何子树，那么它就是叶子结点。

为达到上述目的，我们在 `TreeNode.java` 定义了所有 AST 结点的基类：

```java
public abstract class TreeNode implements Iterable<Object> {
    public final Tree.Kind kind;
    public final Pos pos;

    public final String displayName;
    public abstract int treeArity();
    public abstract Object treeElementAt(int index);

    public abstract <C> void accept(Visitor<C> v, C ctx);

    @Override
    public Iterator<Object> iterator() { /* code */ }

    @Override
    public String toString() { /* code */ }
}
```

我们要求子类必须要有

- `kind`：结点类型
- `pos`：在源码中的位置
- `displayName`：结点名称

并实现接口

- `treeArity`：返回这个结点有几个子树
- `treeElementAt`：返回下标为 `index`（从0开始）的那个子树
- `accept`：如何接受一个 `Visitor` 的访问（将会在 PA2 用到，本阶段你只需要照猫画虎地给个实现）

不难发现，这里的 `treeArity` 和 `treeElementAt` 包含了树的信息。根据它们，我们容易实现一个子树迭代器 `iterator`。
在本框架中，这样设计的最大好处在于——大幅简化格式化打印 AST 的代码量！打印策略的实现（见 `PrettyTree.java`）非常简单：

1. 打印名称 (`displayName`)
2. 增加缩进
3. 遍历所有子树 (`iterator()`)，依次递归打印它们
4. 减少缩进

注意到 `displayName` 和 `iterator()` 已经是基类的成员，我们无需对不同类型的 AST 结点再分别实现一套打印方法。
借助此，我们还顺便重写了 `toString()` 方法，让打印结果看起来更加友好，而不是 Java 默认的那个谜一般的地址。

## AST 各结点的定义

在 Java 这种 OO 语言中，人们常用不同的类来定义 AST 上不同类型的结点，并用继承把结点间的关系刻画出来。
一般而言，产生式右部对应的 AST 结点是其左部的子类。
本框架中，各 AST 结点定义在 `Tree.java` 中。

除了标识符 (`Id`) 和修饰符 (`Modifiers`) 外，其他 AST 结点都继承自 `TreeNode`。
特别地，还可能存储一些在语义分析和中间代码生成阶段与该结点关联的信息（“标注”），如对于表达式 `Expr`：

```java
public abstract static class Expr extends TreeNode {
    // For type check
    public Type type;
    // For tac gen
    public Temp val;
    // ...
}
```

在进行语义分析时，`type` 用于记录该表达式的类型。在进行中间代码生成时，`val` 用于记录该表达式在 TAC 中分配得到的临时变量（伪寄存器）。
在语法分析阶段，你无需关注它们。
