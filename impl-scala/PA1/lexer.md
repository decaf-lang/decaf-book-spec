# 词法分析

在词法分析阶段，输入的文本会被解析为一个一个独立的单词 (token)，它们构成的单词流 (token stream) 将作为文法分析器的输入。
本阶段我们采用 Antlr 来自动构建词法分析器。为此，我们需要将词法规范用 Antlr 可识别的语法表达出来，如 `src/main/antlr4/DecafLexer.g4`。

## 单词类型

在 `src/main/antlr4/DecafLexer.g4` 中，我们将所有 Decaf 程序中可能出现的单词都定义出来了。分为以下几类：

- 关键字：如 `class`
- 操作符：算术、逻辑等操作符，如 `+`
- 分隔符：括号、逗号、分号
- 标识符：即程序中的变量名、类名等
- 空白字符：需要被忽略
- 简单常量：整数、布尔和 null
- 字符串常量：实际上词法分析阶段只分析出了构成字符串的每个字符
- 其他字符：报错

大部分词法的规则形如

```text
token ':' pattern ';'
```

其中 `token` 是单词的名称，`rule` 是具体的模式。Antlr 采用一种类似于正则表达式的写法来描述模式，同时支持用 `|` 表示选择，用 `~` 表示取反。
原则上，你设计的词法规范应该是“正交”的，即不同模式间没有重叠。但在实践中，如果某个串对应着多个模式与之匹配，那么 Antlr 会按照 `.g4` 文件中的顺序，
选择最先出现的那个模式进行匹配。

## 空白处理

Decaf 不是一门需要游标卡尺的语言，所有空格、制表符、换行符等空白字符以及代码注释都不影响文法的分析。因此，我们不妨在词法分析阶段直接把它们忽略掉：

```g4
WHITESPACE:         [ \t\r\n]+ -> skip;
COMMENT:            '//' ~[\r\n]* -> skip;
```

这里使用 `skip` 这个 Antlr 的内置命令来完成“忽略”操作。当词法分析器解析出一个 `WHITESPACE` 或 `COMMENT` 单词时，Antlr 知道这是个“虚无”的单词。
他还会继续往下读取输入串，直至解析出一个普通的单词。

## 字符串处理

在 Decaf 中，一个合法的字符串被一对双引号包围，不能换行，且中间出现的字符如果是转义字符，必须是下列之一：

```g4
\n  \t  \r  \"  \\
```

除此之外可以是任意字符。与普通的代码不同，双引号内部的那些空白字符是不可忽略的——你需要原样把它们解析出来。
为了区分普通代码与双引号内部这两个截然不同的环境，我们使用了 Antlr 的模式切换特性，在读取开头的那个双引号时进入字符串模式：

```g4
OPEN_STRING:        '"' -> pushMode(IN_STRING);
```

在读取结束的那个双引号时退出字符串模式（恢复正常的代码模式）：

```g4
CLOSE_STRING:       '"' -> popMode;
```

在词法分析阶段，只有出现未识别的字符 (`UNRECOG_Char`) 时才有必要紧急退出词法分析，因为这个时候我们很难“恢复”——到底这个字符应该直接忽略掉，
还是因为用户想输入其他符号但是输错了一个字符？我们不得而知。除此之外，遇到其他错误时不应该中断词法分析。
因此，我们还需要考虑那些在字符串中可能出现的非法符号，包括非法的转义字符（如 `\a`）、换行（`\r` 和 `\n`）和未终止的字符串。
最后一种情况只有在我们处于字符串模式，但是却遇到了 EOF (end-of-file) 的情况：

```g4
UNTERM_STRING:      EOF -> popMode;
```

我们引入这个特别的单词 `UNTERM_STRING` 来刻画这种情形。但是，Antlr 似乎并不能在读到 EOF 的时候产生一个 `UNTERM_STRING`。
为此，我们不得不在 `src/main/scala/decaf/frontend/parsing/Lexer.scala` 重写 `emitEOF` 方法以魔改默认生成的单词。
详情请见代码注释。

更多 Antlr 词法规则的描述请参见[文档](https://github.com/antlr/antlr4/blob/master/doc/lexer-rules.md)。

## 与文法分析器的联系

虽然理论上我们可以将词法分析和文法分析看成两个独立的阶段，且事实上我们也可以先做完词法分析，得到单词流之后，再来做文法分析，进而生成语法树。
但实际上，像 lex/yacc/Antlr 等工具在运行的时候，是按需 (by need) 解析单词的。
具体来说，当文法分析器需要查看下一个单词的时候，就向词法分析器“发送请求”（即：调词法分析器提供的接口），获取下一个单词。
这样，如果输入串是符合文法的，那么最终词法分析器会读到 EOF，且文法分析器会完成一次对文法开始符号的推导。