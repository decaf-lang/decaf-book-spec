# 控制流的翻译

TAC 指令不支持 Decaf 语言中的条件、循环等高层次的控制流语句，而仅支持底层的跳转语句。
为此，我们需要将这些高层次的控制流语句翻译为语义等价的跳转语句。
这里，我们仅给出一个 while 循环的例子（`TacEmitter.java`）：

```java
@Override
default void visitWhile(Tree.While loop, FuncVisitor mv) {
    var exit = mv.freshLabel();
    Function<FuncVisitor, Temp> test = v -> {
        loop.cond.accept(this, v);
        return loop.cond.val;
    };
    Consumer<FuncVisitor> body = v -> {
        loopExits.push(exit);
        loop.body.accept(this, v);
        loopExits.pop();
    };
    emitWhile(test, body, exit, mv);
}
```

这里我们先调用 `freshLabel` 得到一个新的标签，它将在 TAC 指令中用来标识循环的出口。
而 `test` 是一个函数（lambda 表达式），它在定义的时候还未执行，在调用它的时候能够生成计算循环条件的那一段 TAC 指令。
之所以这样，是因为这一段 TAC 代码在辅助函数 `emitWhile` 执行的时候，才真正地生成并插入到 `FuncVisitor` 中。
同理，`body` 也是一个函数，在调用它的时候能够生成执行一次循环体的 TAC 指令。
最终，辅助函数 `emitWhile` 将实际上完成与循环语义等价的 TAC 指令生成。
其功能是，将如下形式的循环

```java
while (cond) {
    block
}
```

转换为语义等价的 TAC 程序，形如

```text
entry:
    cond = do test
    if (cond == 0) branch exit
    do block
    branch entry
exit:
```

即：进入 `entry` 标签后，先计算循环条件是否满足，把结果存在 `cond` 临时变量中。
若条件为真，则进入循环体；否则，退出循环。这一步采用条件跳转指令实现。
由于 `exit` 标签标记了整个循环的出口，那么在条件为假时，就跳转到 `exit` 标签；否则往下执行一次循环体。
循环体执行完毕后，我们还需要再次从循环头开始执行下一次，故直接跳转到 `entry` 标签即可。
