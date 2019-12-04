# Decaf中的TAC简介

中间代码的目的是为了更好地对要翻译的内容进行加工处理，例如对代码进行优化等。中间代码的种类有多种，例如逆波兰表达式、三元式、TAC（三地址码，即四元式）、DAG（有向无环图）、类LISP表达式等等。不同的中间代码有不同的适用场合，例如有些适合进行代码逻辑分析，有些适合用来进行源语言层次的优化，有些适合用来做机器层次的优化等等，因此同一个编译器内可能会使用多种中间代码（源代码->中间代码1->中间代码2->…->目标代码）。在Decaf项目中我们只使用TAC作为我们的编译器中间代码，每条TAC最多可以有三个参数，例如加法指令中含有两个操作数地址和一个结果地址（所谓地址这里是指临时变量）。我们采用的是“源代码->AST->TAC->MIPS代码”的翻译过程。

我们在实验框架中的lowlevel/tac/TacInstr类下里面已经定义好了需要用到的所有的TAC种类（事实上你几乎不会直接使用它们，而是通过lowlevel/tac/FuncVisitor类的方法来创建TAC实例）。

下面这张表是Decaf中使用的TAC简介，更加详细的内容请参考框架代码及注释。
<table>
    <tr>
        <th>名字</th>
        <th>操作符</th>
        <th>显示形式示例</th>
        <th>含义</th>
    </tr>
    <tr>
        <th  colspan="4">赋值操作 </th>
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
        <th>x = VTABLE\<C\></th>
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
        <th  colspan="4">运算操作  </th>
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
        <th>c = (a > b) </th>
        <th>若a大于b则c为1，否则为0 </th>
    </tr>
    <tr>
        <th>Binary</th>
        <th>GEQ</th>
        <th>c = (a >= b) </th>
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

TAC表示中使用的数据对象如下

<table>
    <tr>
        <th>名字  </th>
        <th>含义 </th>
    </tr>
    <tr> 
        <th>Temp</th>
        <th>临时变量</th>
    </tr>
    <tr> 
        <th>Label </th>
        <th>标号</th>
    </tr>
    <tr> 
        <th>TacFunc</th>
        <th>函数块</th>
    </tr>
    <tr> 
        <th>VTable</th>
        <th>类的虚函数表</th>
    </tr>
</table>

其中Temp与实际机器中的寄存器相对应。在Decaf框架中，我们用临时变量来表示函数的形式参数（parameter/formal argument）以及函数的局部变量（但是不表示类的成员变量）。在PA3的AST扫描中，我们将会为所有函数的LocalVarDef时为其关联Temp对象。在后面的AST扫描过程中可以通过VarSymbol的temp变量来获得所关联的Temp对象。此外，在翻译的过程中还可以通过FuncVisitor的freshTemp()函数获取一个新的表示32位整数的临时变量。与实际寄存器不同的是，一个Decaf程序中可以使用的临时变量的个数是无限的。

Label表示标号，即代码序列中的特定位置（也称为“行号”）。在Decaf框架中有两种标号，一种是函数的入口标号，另一种是一般的跳转目标标号。正如我们在前面介绍，TAC是一种比较接近汇编语言的中间表示，因此诸如分支语句、循环语句等等将会转换成在一系列行号之中进行跳转的操作（即有些语言中的GOTO语句）。在PA3中，我们在扫描的时候为各Function对象创建TacFunc对象，其中就有函数的入口行号信息，在后面的AST扫描过程中可以用TacFunc的entry来获得函数的函数块然后得到入口行号对象。

Temp和Label都是用于函数体内的数据对象，在Decaf框架的TAC表示中，我们用TacFunc对象来表示源程序中的一个函数定义。与符号表中的Function对象不同，TacFunc对象并不包括函数的返回值、参数表等等信息，而仅仅包括了函数的入口标号以及函数体的语句序列。

VTable所表示的是一个类的虚函数表，即一个存放着各虚函数入口标号的数组。关于虚函数表的细节请参看后面的相应章节。

最后，一个VTable列表加上一个TacFunc列表，就组成了一个完整的TAC程序。

从上面对TAC中间表示的描述可以看出，该中间表示是一种比AST更低级、但比汇编代码高级的表示方式（具有“函数”、“虚函数表”等概念）。

在我们给出的Decaf代码框架中，已经为大家把创建各种TAC的方法封装成FuncVisitor类，大家需要利用该类建立各种数据对象、并且对函数体的各种语句和表达式进行翻译。需要注意的是，在开始翻译函数体之前需要调用FuncVisitor的构造函数来开始函数体的翻译过程，在翻译完函数体以后需要调用Translater的visitEnd ()函数来结束函数体的翻译过程（否则将不能形成正确的TacFunc对象）。
