# TAC 程序

## 临时变量与标签

首先介绍两个基本概念。

**临时变量**

它定义在 `decaf.lowlevel.instr.Temp`，用来表示函数的形式参数（parameter/formal argument）以及函数的局部变量（但是不表示类的成员变量）。
在 AST 扫描过程中，我们将会为所有函数的 `LocalVarDef` 创建其关联的 `Temp` 对象，并记录在 `VarSymbol` 的 `temp` 域中。
在翻译过程中通过 `decaf.lowlevel.tac.FuncVisitor` 提供的 `freshTemp()` 方法来获取一个新的临时变量。
每个临时变量在内存中都占4个字节，并采用一个32位有符号整数来表示。

**标签**

它定义在 `decaf.lowlevel.instr.Label`，用来标记一段指令序列的开始。
从底层的角度来看，每个标签本质上就是一个地址，且往往是某一段连续内存的起始地址。
在本框架中，标签有三种作用：

- 作为跳转目标：TAC 指令不支持 Decaf 语言中的条件、循环等控制流语句，而是将它们都翻译成更加底层的跳转语句（类似于 C 语言中的 goto）。
在本阶段中，我们在所有有可能跳转到的指令前均插入一个标签，作为跳转语句的跳转目标。
- 作为函数入口：函数调用伴随着必然的控制流跳转。因此，每个函数的入口都对应着一个函数标签 `decaf.lowlevel.label.FuncLabel`（它继承了 `Label`）。
- 作为虚表标号：由于虚表（见后）对应着一段连续的内存，我们也为每个虚表分配一个虚表标签 `decaf.lowlevel.label.VTableLabel`（它继承了 `Label`），便于对虚表中的各项进行访问（通过 `LoadVTbl` 指令，见后）。

## 数据类型

TAC 程序是**无类型**的，或者说它仅支持一种类型：32位（4字节）整数。
由于一个整数类型和布尔类型由于都能用一个32位整数装得下，故它们直接用**值**存储。
而字符串、数组、对象等更复杂的数据则仅用一个32位地址表示，即它们用**引用**存储。
具体来说，Decaf 语言中的各类型按照如下形式在 TAC 程序中表示：

- 整数类型（值存储）：直接用整数值表示。
- 布尔类型（值存储）：用 32 位整数的 0 表示 `false`，1 表示 `true`。
- 字符串类型（引用存储）：用一段连续内存的首地址表示，从首地址开始，每个字节依次存储该字符串的每个字符的值，并在末尾添加字符 `\0` 标记该字符串的结束。在内存中，我们不记录字符串的长度。
- 数组类型（引用存储）：用一段连续内存的首地址表示，从首地址开始，以4字节为单位，依次存储该数组中的各个元素的值或引用，且在首地址前的4字节中记录该数组的长度（用一个32位整数表示）。
- 对象/实例（引用存储）：用一段连续内存的首地址表示，从首地址开始，前4个字节记录了该类虚表的首地址，接下来以4字节为单位，依次存储该类的所有成员变量的值或引用。

## TAC 程序的构成

一个 TAC 程序由

- 很多个**虚表** (virtual table)，和
- 很多个**函数** (function)

构成，其中每个函数都由指令的序列构成。对应于 `decaf.lowlevel.tac.TacProg` 中的 `vtables` 和 `funcs`。
`decaf.lowlevel.tac.ProgramWriter` 类给出了一组辅助方法便于我们生成 `TacProg`。
在完成所有虚表和函数的创建过程后，调用 `ProgramWriter` 类的 `visitEnd` 方法将返回一个创建好的 `TacProg`。

### 虚表

一般来说，每个 Decaf 类会对应到一个虚表，用来记录该类的元信息和所有成员方法，对应到 `decaf.lowlevel.tac.VTable` 中的以下成员：

- `parent`：父类的虚表（若存在）
- `className`：该类的名称
- `getItems()`：该类成员方法的入口（均用 `decaf.lowlevel.label.FuncLabel` 来表示）

调用 `ProgramWriter` 类的 `visitVTables` 方法会自动创建好所需的 `VTable`。
注意在初始化 `ProgramWriter` 类实例的时候，我们要给定 `List<ClassInfo> classes` 参数描述每个类的元信息 `ClassInfo`。
具体来说，一个 `ClassInfo` 包含以下字段：

- `name`：类名称
- `parent`：父类（若存在）名称
- `memberVariables`：类的成员变量名的集合
- `memberMethods`：类的成员方法名的集合
- `staticMethods`：类的静态方法名的集合
- `isMainClass`：是否是主类

在模拟器运行的时候，每个虚表被加载到一段连续的内存区域中：

- 第1个字节存储父类虚表 (`parent`) 的指针。若无父类，则为0。
- 第2个字节存储一个字符串的起始地址，该字符串就是类的名称 (`className`)。
- 从第3个字节开始，每个字节均是一个函数的入口地址，分别对应着各成员方法的入口，其顺序与 `getItems()` 的一致。

### 指令

与汇编指令类似，每条 TAC 指令由操作码和（最多3个）操作数构成。
操作数可能会有：临时变量 (`decaf.lowlevel.instr.Temp`)、标签 (`decaf.lowlevel.label.Label`) 和常量。
各指令的含义如下表所示：

<table>
    <tr>
        <th>名字</th>
        <th>操作符</th>
        <th>显示形式示例</th>
        <th>含义</th>
    </tr>
    <tr>
        <th colspan="4">赋值操作</th>
    </tr>
    <tr>
        <th>Assign </th>
        <th> </th>
        <th>a = b </th>
        <th>把变量b的值赋给变量a </th>
    </tr>
    <tr>
        <th>LoadVTbl       </th>
        <th></th>
        <th>x = VTABLE&lt;C&gt;</th>
        <th>把类C的虚表加载到x中 </th>
    </tr>
    <tr>
        <th>LoadImm4       </th>
        <th></th>
        <th>x = 34 </th>
        <th>加载整数常量到变量x中 </th>
    </tr>
    <tr>
        <th>LoadStrConst       </th>
        <th></th>
        <th>x = “Hello World” </th>
        <th>加载字符串常量到变量x中</th>
    </tr>
    <tr>
        <th colspan="4">运算操作  </th>
    </tr>
    <tr>
        <th>Unary</th>
        <th>NEG</th>
        <th>c = - a </th>
        <th>把变量a的相反数赋给变量c </th>
    </tr>
    <tr>
        <th>Unary</th>
        <th>LNOT</th>
        <th>c = ! a </th>
        <th>把变量a逻辑非的结果放到c </th>
    </tr>
    <tr>
        <th>Binary  </th>
        <th>ADD</th>
        <th>c = (a + b) </th>
        <th>把a和b的和放到c中 </th>
    </tr>
    <tr>
        <th>Binary</th>
        <th>SUB</th>
        <th>c = (a - b)</th>
        <th>把a和b的差放到c中 </th>
    </tr>
    <tr>
        <th>Binary</th>
        <th>MUL</th>
        <th>c = (a * b) </th>
        <th>把a和b的积放到c中 </th>
    </tr>
    <tr>
        <th>Binary</th>
        <th>DIV</th>
        <th>c = (a / b) </th>
        <th>把a除以b的商放到c中</th>
    </tr>
    <tr>
        <th>Binary</th>
        <th>MOD</th>
        <th>c = (a % b) </th>
        <th>把a除以b的余数放到c中 </th>
    </tr>
    <tr>
        <th>Binary</th>
        <th>EQU</th>
        <th>c = (a == b)</th>
        <th>若a等于b则c为1，否则为0 </th>
    </tr>
    <tr>
        <th>Binary</th>
        <th>NEQ</th>
        <th>c = (a != b) </th>
        <th>若a不等于b则c为1，否则为0 </th>
    </tr>
    <tr>
        <th>Binary</th>
        <th>LES</th>
        <th>c = (a &lt; b) </th>
        <th>若a小于b则c为1，否则为0 </th>
    </tr>
    <tr>
        <th>Binary</th>
        <th>LEQ</th>
        <th>c = (a &lt;= b) </th>
        <th>若a小于等于b则c为1，否则为0 </th>
    </tr>
    <tr>
        <th>Binary</th>
        <th>GTR</th>
        <th>c = (a &gt; b) </th>
        <th>若a大于b则c为1，否则为0 </th>
    </tr>
    <tr>
        <th>Binary</th>
        <th>GEQ</th>
        <th>c = (a &gt;= b) </th>
        <th>若a大于等于b则c为1，否则为0 </th>
    </tr>
    <tr>
        <th>Binary</th>
        <th>LAND</th>
        <th>c = (a && b) </th>
        <th>把a和b逻辑与操作的结果放到c</th>
    </tr>
    <tr>
        <th>Binary</th>
        <th>LOR</th>
        <th>c = (a || b) </th>
        <th>把a和b逻辑或操作的结果放到c </th>
    </tr>
    <tr>
        <th  colspan="4">控制流管理  </th>
    </tr>
    <tr>
        <th>Branch</th>
        <th></th>
        <th>branch _L2 </th>
        <th>无条件跳转到行号_L2所表示的地址</th>
    </tr>
    <tr>
        <th>CondBranch </th>
        <th>BEQZ</th>
        <th>if (c == 0) branch _L1</th>
        <th>如果c为0则跳转到_L1所表示地址 </th>
    </tr>
    <tr>
        <th>CondBranch </th>
        <th>BNEZ</th>
        <th>if (c != 0) branch _L1</th>
        <th>如果c不为0则跳转到_L1所表示地址 </th>
    </tr>
    <tr>
        <th>Return</th>
        <th></th>
        <th>return c</th>
        <th>结束函数并把c的值作为返回值返回 </th>
    </tr>
    <tr>
        <th  colspan="4">函数调用相关操作  </th>
    </tr>
    <tr>
        <th>Parm</th>
        <<th></th>
        <th>parm a  </th>
        <th>变量a作为调用的参数传递</th>
    </tr>
    <tr>
        <th>IndirectCall </th>
        <th></th>
        <th>x = call a</th>
        <th>取出a中函数地址，并调用，结果放x </th>
    </tr>
    <tr>
        <th>DirectCall </th>
        <th></th>
        <th>x = call Label </th>
        <th>根据函数标签，调用函数，结果放x </th>
    </tr>
    <tr>
        <th  colspan="4">内存访问操作  </th>
    </tr>
    <tr>
        <th >Memory</th>
        <th>LOAD</th>
        <th>x = *(y - 4) </th>
        <th>把地址为y-4的单元的内容加载到x </th>
    </tr>
    <tr>
        <th >Memory</th>
        <th>STORE</th>
        <th>*(x + 4) = y </th>
        <th>把y保存到地址为x+4的内存单元中</th>
    </tr>
    <tr>
        <th  colspan="4">其他  </th>
    </tr>
    <tr>
        <th>Mark</th>
        <th></th>
        <th>_L5: </th>
        <th>定义一个行号_L5（全局的） </th>
    </tr>
    <tr>
        <th>Memo</th>
        <th></th>
        <th>memo 'XXX'</th>
        <th>注释：XXX（供模拟器使用） </th>
    </tr>
</table>

上述这些指令的数据结构均定义在 `decaf.lowlevel.tac.TacInstr` 中。
但是，你**无需**通过调用这些类的构造方法来创建指令，而是通过 `decaf.lowlevel.tac.FuncVisitor` 中提供的 `visit*` 系列辅助方法来向一个函数中插入新指令。

### 函数

一个 TAC 函数用来记录与该函数相关的一些元信息和构成函数体的所有指令。框架中对应 `decaf.lowlevel.tac.TacFunc`：

- `entry`：函数入口标签。请注意：**主函数**具有特别标签 `decaf.lowlevel.label.FuncLabel.MAIN_LABEL`。
- `numArgs`：函数参数个数。
- `getInstrSeq()`：函数体，是一个指令序列（类型为 `List<TacInstr>`）。

TAC 不区分所谓的“成员方法”和“静态方法”。
在模拟器运行的时候，一个 TAC 函数对应的所有指令会按 `getInstrSeq()` 的顺序加载到一段连续的指令内存中，且第一条指令的地址就是该函数的入口地址（即 `entry`）。
模拟器会自己维护一个函数调用栈来处理必要的上下文切换，如保存和恢复各函数的临时变量——准确来说是各函数**栈帧**中的临时变量，因为一个函数有可能多次被调用，每次调用的时候那些临时变量都只有它自己能看见。
TAC 的函数**调用惯例** (calling convention) 如下：
若一个函数有 `n` 个参数，那么保留编号从 0 到 `n - 1` 的临时变量，按照参数输入的顺序（通过 `parm` 指令）依次传递，例如：

```text
parm A
parm B
parm C
call XXX
```

意味着在刚进入调用方时：

```text
_T0 = A
_T1 = B
_T2 = C
```

`FuncVisitor` 类提供了一系列辅助方法，以便在 `decaf.frontend.tacgen` 中更加方便地创建一个 TAC 函数。
其使用流程是：

1. 通过在上一层调用 `ProgramWriter` 类的 `visitFunc` 方法可以得到我们需要用的 `FuncVisitor` 类。
由于调用时已经给定函数的参数个数 `n`，`FuncVisitor` 会自动保留编号从 0 到 `n - 1` 的临时变量。
2. 依次遍历该函数的所有 Decaf 语句和表达式，生成相应的 TAC 指令，借助 `visit*` 系列方法插入到该函数当前指令序列的最后（在这个意义下，把 "visit" 理解为 "make" 或者 "append" 也许更合适）。
此过程中，调用 `freshLabel` 和 `freshTemp` 方法分别会创建新的标签（仅用于作为函数内的跳转目标）和新的临时变量（从 `n` 开始编号）。
3. 所有指令插入完毕后，调用 `visitEnd` 方法结束该 TAC 函数的创建。

## 运行时库函数

一般来说，编译器在把源程序转换为目标机器的汇编程序或者机器代码的过程中，除了直接生成机器指令来实现一些功能以外，有时还会调用运行时库函数所提供的功能。
所谓**运行时库** (runtime library)，是指一系列预先实现好的函数的集合（请注意跟 Decaf 语言规范中的“标准库函数”不同），这些函数往往是针对特定的运行平台实现的，帮助编程语言实现一些平台相关的功能，例如C语言中的 `libc` 库（在 Windows 平台上通常是 `MSVCRT.dll`），又例如 Java 语言的类库（例如 `rt.jar`）等。通常，这样的运行库是随着所使用的编译器（或者解释器）的不同而不同的。

在Decaf中，为了实现一些平台相关的功能，我们也提供了一系列的运行时库函数，这些函数涉及到内存动态分配、输入输出等等功能。
这些函数我们都定义在 `lowlevel.tac.Intrinsic` 中，通过 `DirectCall` 的方式来进行调用。
注意库函数调用的传参方式和其他函数调用是一样的，也需要先用 `Parm` 压入参数。

以下是对Decaf运行时库中所提供的8种运行时库函数的具体介绍：

<table>
    <tr>
        <th>名称</th>
        <th>功能</th>
        <th>输入</th>
        <th>输出</th>
    </tr>
    <tr>
        <th>ALLOCATE</th>
        <th>分配内存，如果失败则自动退出程序</th>
        <th>参数1：要分配的内存块大小（单位为字节）</th>
        <th>内存块的首地址</th>
    </tr>
    <tr>
        <th>READ_LINE</th>
        <th>从标准输入读取一行（最多63个字符）</th>
        <th>无</th>
        <th>读到的字符串的首地址</th>
    </tr>
    <tr>
        <th>READ_INT</th>
        <th>从标准输入读取一个整数</th>
        <th>无</th>
        <th>读到的整数值</th>
    </tr>
    <tr>
        <th>STRING_EQUAL</th>
        <th>比较两个字符串是否相等（长度相等且各字符对应相等）</th>
        <th>参数1：字符串1的首地址；参数2：字符串2的首地址</th>
        <th>若相等则返回true，否则返回false</th>
    </tr>
    <tr>
        <th>PRINT_INT</th>
        <th>向标准输出打印一个整数</th>
        <th>参数1：要打印的整数值</th>
        <th>无</th>
    </tr>
    <tr>
        <th>PRINT_STRING</th>
        <th>向标准输出打印一个字符串</th>
        <th>参数1：要打印的字符串的首地址</th>
        <th>无</th>
    </tr>
    <tr>
        <th>PRINT_BOOL</th>
        <th>向标准输出打印一个布尔值</th>
        <th>参数1：要打印的布尔值</th>
        <th>无</th>
    </tr>
    <tr>
        <th>HALT</th>
        <th>立即结束并退出程序</th>
        <th>无</th>
        <th>无</th>
    </tr>
</table>

## TAC 模拟器的运行时布局

一般来说，程序运行时的内存空间从逻辑上分为“代码区”和“数据区”两个主要部分。
顾名思义，代码区用于存放可执行的代码，而数据区用于存放程序运行所需的数据。
TAC 模拟器在实现上也采用了这种代码与数据分开的方式。
代码区存储 TAC 各函数的指令，数据区存储虚表、字符串常量和调用栈。
此外，如数组、对象等动态创建的数据通过运行时库函数 `ALLOCATE` 分配在内存的堆区域。

在实现中，我们不得不采用一些 Java 的数据结构来模拟上述各区域（因为 Java 不像 C 那样能直接操作内存）：

- 代码区：`Vector<decaf.lowlevel.tac.TacInstr>`
- 字符串常量池：`decaf.lowlevel.tac.StringPool`
- 调用栈：`Stack<decaf.lowlevel.tac.Frame>`，注意每个函数对应的临时变量会直接以数组形式存放在调用帧里
- （堆）内存：`decaf.lowlevel.tac.Simulator.Memory`，注意虚表是直接存在这里的
