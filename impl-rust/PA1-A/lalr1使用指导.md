# lalr1使用指导

[lalr1](https://github.com/MashPlant/lalr1)是一个基于lalr(1)或者ll(1)文法的parser generator，其中lalr(1)的部分在本次pa1a中使用，ll(1)的部分在pa1b中使用。lalr1并没有经过系统的测试和长期的使用，可能还有一些不稳定的因素，如果产生了影响使用的bug，请大家积极汇报。

lalr(1)分析表的生成算法是基于龙书给出的一个较为高效的算法，感兴趣的同学可以直接阅读代码或者参考龙书相关章节，这里不详细介绍了。生成decaf的分析表大约耗时0.1s，相比于rustc的编译时间可以忽略不计了。

lalr1有两套代码生成的方式，比较传统的一种是读取配置文件并输出rust代码；另一种是利用rust的过程宏，在rust代码中嵌入产生式的信息，在编译时生成rust代码作为rustc的输入。我们的实验使用后者。

过程宏大致的编写方式是在你希望让它成为一个parser的struct的impl块上添加`#[lalr1(Start)]`标记，其中`Start`是一个非终结符，是parser希望规约出来的最终结果。

另外一个必要的属性是`#[lex(TomlOfLexer)]`，其中`TomlOfLexer`以toml字符串的形式配置了词法解析器。之所以选择借助嵌入在过程宏中的toml而不是直接使用过程宏，是因为这样更能清晰地描述lexer。正则表达式到有限状态机的转换由[re2dfa](https://github.com/MashPlant/re2dfa)来提供，与lalr1一样，它也没有怎么经过测试，如果大家遇到使用上的bug请及时汇报。

此外，还可以为impl块添加一些可选的属性，后面会详细描述。

对于impl块中的每个函数，都使用rust的正常函数的语法来编写语义动作，同时用函数级别的属性来描述产生式。

生成出来的rust代码中不会保留这个impl块，也就是说这里面编写的东西都不会直接被编译。输出的结果包含两个enum，即`TokenKind`和`StackItem`，两个struct，即`Token`和`Lexer`，并为你希望成为parser的struct提供一个`parse`函数。根据我的使用经验，现在的ide或者编辑器不太可能把这些符号识别出来并且给予正常的语法提示，如果希望了解具体的api的话，可以使用之前提到的`#[expand]`，也可以使用`rust doc`，它会在展开过程宏后再分析api。

生成出来的`Lexer`是可以独立于`Parser`使用的，下面的代码可以依次输出从一个字符串中解析出的全部token：

```rust
let mut lexer = Lexer::new("your string here".as_bytes());
loop {
  let token = lexer.next();
  println!("{:?}", token);
  if token.ty == TokenKind::_Eof { break; }
}
```