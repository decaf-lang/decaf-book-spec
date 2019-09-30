# tacvm简述

[tacvm](https://github.com/MashPlant/tacvm)是用来执行tac的解释器。

tacvm从文本中parse出tac语句，然后经过一定转换后执行。目前我们还没有将tacvm中的tac与框架中的tac统一化，也就是说目前仍然无法通过编写tac文本来直接构造可供编译器后续处理的tac。这的确是一个很重要的feature，例如llvm的ir就有内存形式，bit code形式，文本ir形式这三种形式，三者是完全一致，彼此一一对应的，这给予了llvm ir很大的灵活性和通用性。只是我们的tac的设计初期就没有怎么考虑这个问题，现在要统一起来工作量较大，希望在将来能够实现这个feature。

从tac文件中可以基本看出tac文件的大致格式，这里只大致讲一下tac中不能完全体现出来的内容：

- 程序有两种顶层元素，即`VTBL`和`FUNCTION`，括号里的标识符可以使用字母，数字，`.`和`_`
- `VTBL`和`FUNCTION`的内容(大括号里的内容)都是用换行分隔的，对空格缩进不敏感，但是对换行敏感，这其中不允许出现空白的一行
- `VTBL`中可以出现以下内容
  - `"string"`，填入对应的字符串指针
  - `int`，填入对应整数
  - `FUNCTION<identifier>`，填入`identifier`对应的函数指针
  - `VTBL<identifier>`，填入`identifier`对应的虚表指针
- `VTBL`的内容和顺序并不要求符合我们的编译器框架中的内容和顺序，即编译器的虚表中只用到了tacvm中的虚表的有限的功能，未来想实现更多功能时也可以利用这个虚表(例如static变量什么的)
- tac函数的参数传递过程是这样的：
  - 调用者使用`parm`指令把参数存入一个临时的空间中
  - `call`指令时，将临时空间中的内容复制到被调用的函数的前若干个连续的虚拟寄存器中，之后清空临时空间

tacvm会检测tac代码的运行错误，这里直接列出tacvm中表示所有的错误种类的代码：

```rust
#[derive(Debug)]
pub enum Error {
  // program calls _Halt explicitly
  Halt,
  // base % 4 != 0 or off % 4 != 0 or allocate size % 4 != 0
  UnalignedMem,
  // base == 0
  NullPointer,
  // base is in an invalid object
  MemOutOfRange,
  // base is in a valid obj, base + off is not or in another obj
  ObjOutOfRange,
  // access to string id is invalid(typically a string id is not a valid mem address)
  StrOutOfRange,
  // instruction fetch out of range
  IFOutOfRange,
  // call a register which is not a valid function id
  CallOutOfRange,
  // call stack exceeds a given level(specified in RunConfig)
  StackOverflow,
  // instructions exceeds a given number(specified in RunConfig)
  TLE,
  // the number of function arguments > function's stack size
  TooMuchArg,
  // div 0 or mod 0
  Div0,
  // fails in reading or writing things
  IO,
}
```

相信注释写的比较清晰了，这里补充几点：

- 所有函数都必须显式地返回，即使是decaf返回void的函数，否则执行到函数末尾时会触发`IFOutOfRange`
  - 虽然在decaf语法层面上允许返回void的函数不显式写出`return`，但是tac层面的`return`是不能省略的，正常的ir或者汇编代码应该都是这样要求的
- 目前decaf的tac翻译策略中还没有检测空指针的错误，这个错误目前在tac层面可以由tacvm检测，但是在汇编阶段则没有任何保护了
  - 也许未来我们会考虑在到tac的翻译中加入对于对象，字符串和数组的空指针检测。显然，如果没有高效的编译优化，这样的检测会产生很大的运行开销。目前我们的编译优化还很初级，还没有达到这个程度
  - 空指针检测也不一定需要依靠tac代码或者汇编代码来体现，另一种可行的方案是充分利用os提供的虚拟内存和信号机制，[SIGSEGV as control flow - How the JVM optimizes your null checks](https://jcdav.is/2015/10/06/SIGSEGV-as-control-flow/)对此有一定的描述
  - 现阶段，大家直接忽略这个问题就好了，可以假装我们已经合理地处理了空指针

tacvm可以作为一个库链接到程序中(我们的测试程序就是这样用的)，也可以作为一个独立的程序。为了避免跨平台的问题，框架中并没有提供编译好的版本，大家如果想作为一个独立的程序运行tacvm，自己去github上clone一份然后编译即可。

目前tacvm提供以下的运行参数：

```
--inst_count # 指定info_output是否输出执行的指令条数
--stacktrace # 指定info_output中是否在tac发生运行时错误时输出stacktrace
--info_output # tacvm的运行信息输出路径
--inst_limit # tacvm至多执行多少条指令
--stack_limit # tacvm至多允许多少层调用栈
--vm_input # tacvm的输入路径(如ReadLine()等函数用到)
--vm_output # tacvm的输出路径(如Print()等函数用到)
<file> # tacvm的输入tac文件
```

测试程序中`inst_limit`和`stack_limit`设置的都不是很大(分别是100000和1000)，不过也都够用了。