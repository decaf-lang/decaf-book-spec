# 错误恢复

在课程讲义 Lecture04 中，我们介绍了应急恢复和短语层恢复的方法。这里，我们提出一种介于二者之间的错误恢复方法：

与应急恢复的方法类似，当分析非终结符$$A$$时，若当前输入符号$$a \notin Begin(A)$$，则先报错，
然后跳过输入符号串中的一些符号，直至遇到$$Begin(A) \cup End(A)$$中的符号：

- 若遇到的是$$Begin(A)$$中的符号，可恢复分析$$A$$；
- 若遇到的是$$End(A)$$中的符号，则$$A$$分析失败，返回 `null`，继续分析$$A$$后面的符号。

这个处理方法与应急恢复方法的不同之处在于：

- 我们用集合$$Begin(A)=\{s \mid M[A,s] 非空\}$$（其中，$$M$$为预测分析表）来代替$$First(A)$$。由于$$First(A) \subseteq Begin(A)$$，我们能少跳过一些符号。
- 我们用集合$$End(A)=Follow(A) \cup F$$ 来代替$$Follow(A)$$。其中，$$F$$集合（即 `parseSymbol` 函数的第二个参数 `follow`）包含了$$A$$各父节点的$$Follow$$集合。因此，我们既能少跳过一些符号，同时由于结束符 EOF 必然属于文法开始符号的$$Follow$$集合，本算法还无需额外考虑因读到 EOF 而陷入死循环的问题。这个处理方法借鉴了短语层恢复中$$EndSym$$的设计。
- 另外，当匹配终结符失败时，只报错，但不消耗此匹配失败的终结符，而是将它保留在剩余输入串中。这部分的处理已经在 `matchToken` 函数中实现了。

一般来说，错误恢复是一个非常难的问题。上述方法显然也会有在一些 Decaf 程序上产生误报。

## 避免重复报错

由于实现的原因，本框架的 `parseSymbol` 函数有可能会多次连续调用 `issue` 方法报告同样的语法错误 (`DecafError`)。
为了避免输出重复的错误信息，在 `LLParser` 类中，我们重载了 `issue` 方法，在添加错误之前先检查它是不是刚添加过了：

```java
@Override
public void issue(DecafError error) {
    if (!errors.isEmpty()) {
        var last = errors.get(errors.size() - 1);
        if (error.toString().equals(last.toString())) { // ignore
            return;
        }
    }

    super.issue(error);
}
```
