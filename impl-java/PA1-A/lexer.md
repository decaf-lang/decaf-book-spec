# 词法分析

在词法分析阶段，输入的文本会被解析为一个一个独立的单词 (token)，它们构成的单词流 (token stream) 将作为文法分析器的输入。
本阶段我们采用 [JFlex](http://jflex.de) 来自动构建词法分析器。为此，我们需要将词法规范用 JFlex 可识别的格式表达出来。

## 单词类型

在 `src/main/jflex/Decaf.jflex` 中，我们将所有 Decaf 程序中可能出现的单词都定义出来了。分为以下几类：

- 关键字：如 `class`
- 操作符和分隔符：默认单字符的单词直接用其 ASCII 编码，因此该文件中只需定义那些含有两个或更多字符的操作符，如 `<=`
- 标识符：即程序中的变量名、类名等
- 空白字符：需要被忽略
- 常量：整数、布尔、字符串和 null
- 其他字符：报错

大部分词法的规则形如

```text
pattern '{' action '}'
```

其中 `pattern` 是用正则表达式描述的待匹配单词的模式。关于 JFlex 所支持的正则表达式及其语法，请参见[用户手册](https://jflex.de/manual.pdf)。
而 `action` 是语义动作，即当我们匹配上了这个模式以后，我们要做的事情。在本框架中，语义动作就是将生成的单词记录下来，将来给文法分析器使用。
原则上，你设计的词法规范应该是“正交”的，即不同模式间没有重叠。
但在实践中，如果某个串对应着多个模式与之匹配，那么 JFlex 会按照 `.jflex` 文件中的顺序，选择最先出现的那个模式进行匹配。

## 空白处理

Decaf 不是一门需要游标卡尺的语言，所有空格、制表符、换行符等空白字符以及代码注释都不影响文法的分析。因此，我们不妨在词法分析阶段直接把它们忽略掉：

```jflex
{WHITESPACE}        { /* Just ignore */ }
{NEWLINE}           { /* Just ignore */ }
{S_COMMENT}         { /* Just ignore */ }
```

即让它的语义动作为空，那么当词法分析器解析出这些模式来以后，就啥也不做。

## 字符串处理

在 Decaf 中，一个合法的字符串被一对双引号包围，不能换行，且中间出现的字符如果是转义字符，必须是下列之一：

```text
\n  \t  \r  \"  \\
```

除此之外可以是任意字符。与普通的代码不同，双引号内部的那些空白字符是不可忽略的——你需要原样把它们解析出来。
为了区分普通代码与双引号内部这两个截然不同的环境，我们使用了 JFlex 的模式切换特性，在读取开头的那个双引号时进入字符串模式：

```jflex
<YYINITIAL>\"       { startPos = getPos();
                      yybegin(S);
                      buffer = new StringBuilder();
                      buffer.append('"'); }
```

这里我们先把字符串的起始引号所在的位置缓存到 `startPos` 中，用 `yybegin` 开启这个特别的字符串模式 `S`。
然后把第这个引号给添加到 `buffer` 中。

在读取结束的那个双引号时退出字符串模式（恢复正常的代码模式 `YYINITIAL`）：

```jflex
<S>\"               { buffer.append('"');
                      yybegin(YYINITIAL);
                      return stringConst(buffer.toString(), startPos); }
```

并调用我们在 `AbstractLexer` 中提供的辅助方法 `stringConst` 构建该字符串常量对应的单词。
这个 `stringConst` 虽然只返回了一个 `int` 作为单词，那么解析出来的字符串到哪里去了呢？
实际上，我们会为每个单词在创建一个文法符号 `SemValue`，解析出来的字符串会被记录到该 `SemValue`：

```java
protected int stringConst(String value, Pos pos) {
    var token = parser.tokenOf(Tokens.STRING_LIT);
    parser.semValue = new SemValue(Tokens.STRING_LIT, pos);
    parser.semValue.strVal = value;
    return token;
}
```

在字符串模式下，我们依次分析字符串出现的字符，有以下这些情形：

- 如果遇到换行符 (模式为 `<S>{NEWLINE}`)，那么报错 `NewlineInStrError`。
- 如果遇到行尾 (模式为 `<S><<EOF>>`)，由于此时我们仍在字符串模式中，说明这个字符串还没有闭合程序就结束了，那么报错 `UntermStrError`。
- 如果遇到合法的转义符 (模式为 `<S>{ESC}`)，或者其他任意非转义符 (模式为 `<S>.`，由于它出现在 `<S>\"` 的后面，所以也不能是引号)，
那么就正常把它加到 `buffer` 中。
- 如果遇到非法的转义符 (模式为 `<S>{BAD_ESC}`)，如 `\a`，那么报错 `BadEscCharError`。

在词法分析阶段，只有出现未识别的字符时才有必要紧急退出词法分析：

```jflex
.                   { issueError(new UnrecogCharError(getPos(), yycharat(0)));      }
```

因为这个时候我们很难“恢复”——到底这个字符应该直接忽略掉，
还是因为用户想输入其他符号但是输错了一个字符？我们不得而知。除此之外，遇到其他错误时不应该中断词法分析。
因此，即使我们在上述情形中报了错，也不能终止分析，而是继续解析剩下的字符。

## 与文法分析器的联系

虽然理论上我们可以将词法分析和文法分析看成两个独立的阶段，且事实上我们也可以先做完词法分析，得到单词流之后，再来做文法分析，进而生成语法树。
但实际上，像 lex/yacc/Antlr 等工具在运行的时候，是按需 (by need) 解析单词的。
具体来说，当文法分析器需要查看下一个单词的时候，就向词法分析器“发送请求”（即：调词法分析器提供的接口），获取下一个单词。
这样，如果输入串是符合文法的，那么最终词法分析器会读到 EOF，且文法分析器会完成一次对文法开始符号的推导。

## 多文法分析器适配

在 PA1-A 与 PA1-B 阶段，我们使用的文法分析工具不同，但是使用的词法分析工具都是 JFlex。
由于不同文法分析工具都会为 Decaf 的单词生成自己的一套整数值编码，为了支持不同的文法分析器，我们需要在 JFlex 中返回那个文法分析器对应的编码。
为此，`decaf.frontend.parsing` 中实际出现了三套单词编码：

- `Tokens.java` 是一套与文法分析工具无关的标准编码
- Jacc 工具会自动生成它的编码
- ll1pg 工具会自动生成它的编码

接下来，我们在 `AbstractParser` 类中要求所有具体 `Parser` 都实现以下编码转换函数：

```java
public abstract int tokenOf(int code);
```

它输入一个 `Tokens.java` 中定义的标准单词编码 `code`，返回这个具体 `Parser` 对应的编码。
这样，我们只要在 `JaccParser.java` 和 `LLParser.java` 中分别实现这个接口，就能将标准编码分别对应到 Jacc 和 ll1pg 生成的编码上来。
而在 JFlex 的语义动作代码中，我们总是传入标准单词的编码，由 `AbstractLexer` 中的辅助函数来调用该转换函数。
