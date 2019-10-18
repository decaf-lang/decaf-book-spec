# lalr1使用指导

本次pa会用到lalr1的ll(1)文法部分，大多数概念与之前是相同的，这里只介绍不一样的部分。

当用作ll(1)文法的parser generator时，lalr1没有提供利用优先级和结合性来解决冲突的机制，所以运算符的优先级和结合性在这里都没有意义了，大家需要手动地用产生式的组合来描述"优先级和结合性"这种性质。类似于左递归和左公因子之类的问题也需要手动解决。对于冲突的产生式，lalr1会汇报冲突警告，并且使用出现在最前面的产生式。

decaf语言由于允许`if`语句的`else`分支为空，所以不是严格的ll(1)语言。在解析非终结符`MaybeElse`时遇到终结符`Else`时，有两个产生式，即`MaybeElse -> Else Stmt`和`MaybeElse ->`，都可以被选择，lalr1会汇报一个警告并且选择前者。这里实现的逻辑是"最近悬挂if匹配"，常见的编程语言中也都是这个逻辑。

ll(1)的lexer部分与之前基本一样，不过在`#[lex(TomlOfLexer)]`中的`priority`字段写一个空数组即可，反正写了也用不到。

生成的代码包含了lalr(1)版本中的东西，即"两个enum，即`TokenKind`和`StackItem`，两个struct，即`Token`和`Lexer`，并为你希望成为parser的struct提供一个`parse`函数"。此外还有：

1. `u32`常数`NT_NUM`，表示非终结符的数目
2. 包在`lazy_static!`中的`HashSet<u32>`数组`FOLLOW`，用非终结符编号作下标访问，得到它的follow集合
3. 包在`lazy_static!`中的`HashMap<u32, (u32, Vec<u32>)>`数组`TABLE`，用非终结符编号作下标访问，得到它的预测分析表。预测分析表是一个终结符到`(产生式编号, 产生式右端项内容)`的映射，其中`产生式右端项内容`，即`Vec<u32>`，就是每个右端项的编号
4. 为你希望成为parser的struct添加了一个`fn act(&mut self, prod: u32, value_stk: Vec<StackItem<'p>>) -> StackItem<'p>`函数，其中`prod`即`产生式编号`，`value_stk`是每个右端项对应的解析结果，返回值为以`value_stk`为参数执行`prod`对应的语法动作后的结果

相比于lalr(1)版本中的`StackItem`，这里的`StackItem`增加了一个variant：`StackItem::_Fail`，表示解析失败的文法符号。如果`value_stk`中有任何一个`StackItem::_Fail`，则`act`函数的返回值就会是`StackItem::_Fail`。

利用`#[verbose(OutputPath)]`可以查看每个文法符号和产生式的编号，文法符号编号保证所有非终结符的编号都在`0..NT_NUM`范围内，所有终结符都在`NT_NUM..`范围内。其实在lalr(1)版本中的`verbose`也会输出这些信息，不过在pa1a中应该用不到这些信息来辅助调试，pa1b中可能会用到。

`parse`函数会转发调用一个未提供实现的`_parse`函数，`_parse`需要利用上面提供的这些东西来实现一个支持一定程度的错误恢复的parser。
