# 语法树

语法树是贯穿 Decaf 编译器前端的重要数据结构。
`TreeNode.java` 定义了所有语法树结点的基类：

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

## 语法树就是一棵树

在上述基类结点中，我们要求子类必须要有

- `kind`：结点类型
- `pos`：在源码中的位置
- `displayName`：结点名称

并实现接口

- `treeArity`：返回这个结点有几个子树
- `treeElementAt`：返回下标为 `index`（从0开始）的那个子树
- `accept`：如何接受一个 `Visitor` 的访问（将会在 PA2 用到，本阶段你只需要照猫画虎地给个实现）

利用 `displayName`, `treeArity` 和 `treeElementAt`，我们相当于知道了这棵树叫什么名字，以及其子树有哪些。
根据 `treeArity` 和 `treeElementAt`，我们很容易实现一个子树迭代器，即 `Iterator<Object> iterator()` 方法的实现。
进一步，在格式化打印一棵树的时候我们能大幅简化代码量。因为我们的打印策略（见 `PrettyTree.java`）其实很简单：

1. 打印名称 (`displayName`)
2. 增加缩进
3. 遍历所有子树 (`iterator()`)，依次递归打印它们
4. 减少缩进

而无需为每一种语法树结点实现格式化打印的方法。
借助此，我们还顺便重写了 `toString()` 方法，让打印结果看起来更加友好，而不是 Java 默认的那个谜一般的地址。

## 语法树的继承

在 `Tree.java` 中，每种语法树结点都对应着一种类型的语法元素，而且它们差不多能与文法中的符号一一对应。
结点间的继承关系也与文法中规定的类似。
在创建新的语法结点时，请继承合适的超类。
