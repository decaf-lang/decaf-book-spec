# 翻译过程

把带类型标注的 AST 翻译成 JVM 字节码的过程是：

- 对于每个类，分别为其生成同名的 Class 文件，其中记录了该类的元信息和所有成员；
- 对于类中每个变量，需要在 Class 文件中记录其类型，设置访问权限为 `ACC_PROTECTED`；
- 对于类中每个方法，需要在 Class 文件中记录其类型，设置访问权限为 `ACC_PUBLIC`，静态方法还要设置 `ACC_STATIC`，然后把函数体翻译为 JVM 指令序列。

## ASM 库

`org.objectweb.asm` 库提供了一些对于 JVM 字节码的包装。
使用该库，你可以在无需知道每条 JVM 指令字节码编码等细节的情况下，完成对 JVM 字节码的生成并写入到文件中。
该库提供了 `ClassWriter` 类用于管理一个 Class 文件，以及 `MethodVisitor` 类完成对于一个函数体的生成。
我们可以使用 `MethodVisitor` 类提供的一组 `visit*` 方法完成 JVM 指令的生成。
在 PA3 阶段使用的 `decaf.lowlevel.tac.ProgramWriter` 是仿照 asm 库实现的，因此你在使用时一定会有似曾相识的感觉。

## 类的翻译

多亏了 JVM 原生对于面向对象机制的支持，一个 Decaf 类能非常直白地对应到一个 Class 文件。这个功能实现在 `JVMGen` 类中的 `emitClass` 方法：

```scala
def emitClass(clazz: ClassDef): JVMClass = {
  implicit val cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES + ClassWriter.COMPUTE_MAXS)

  // set up meta data
  cw.visit(...)

  // First add the default constructor:
  val mv = cw.visitMethod(Opcodes.ACC_PUBLIC, CONSTRUCTOR_NAME, CONSTRUCTOR_DESC, null, null)
  mv.visitCode()
  mv.visitVarInsn(Opcodes.ALOAD, 0)
  mv.visitMethodInsn(Opcodes.INVOKESPECIAL, superClass, CONSTRUCTOR_NAME, CONSTRUCTOR_DESC, false) // call super
  mv.visitInsn(Opcodes.RETURN)
  mv.visitMaxs(-1, -1) // pass in random numbers, as COMPUTE_MAXS flag enables the computation
  mv.visitEnd()

  // Then generate every user-defined member:
  clazz.fields.foreach {
    case field: VarDef => cw.visitField(...)
    case method: MethodDef => emitMethod(method)
  }
  cw.visitEnd()

  JVMClass(clazz.name, cw.toByteArray)
}
```

它会最终返回一个 `JVMClass`，调用其 `writeFile` 方法能生成最终所需要的 `.class` 文件。

## 方法的翻译

针对于 Decaf 中的成员方法和静态方法，我们需要为其生成相应的 JVM 函数。这个功能由 `JVMGen` 类中的 `emitMethod` 实现：

```scala
def emitMethod(method: MethodDef)(implicit cw: ClassWriter): Unit = {
  implicit val mv = cw.visitMethod(...)

  // Allocate indexes (in JVM local variable array) for every argument.
  // For member methods, index 0 is reserved for `this`.
  implicit val ctx = new Context(method.isStatic)
  method.params.foreach { p => ctx.declare(p.symbol) }

  // Visit method body and emit bytecode.
  mv.visitCode()
  implicit val loopExits: List[Label] = Nil
  emitStmt(method.body)
  appendReturnIfNecessary(method)
  mv.visitMaxs(-1, -1)
  mv.visitEnd()
}
```

注意我们需要调用 `cw` 的成员方法 `visitMethod`，传入方法的元数据，才能得到一个真正的 `MethodVisitor mv`。
接下来，先保留临时变量数组的前面几个元素作为函数参数。注意对于成员方法，0 对应于 this。
然后，依次遍历所有方法体的语句，生成 JVM 指令。
若该方法没有返回值，且最后一条指令又不是返回指令的话，需要插入一条返回指令作为结束，否则 JVM 在运行时会报错（通过 `appendReturnIfNecessary` 辅助函数实现）。
最后，调用 `visitMaxs` 自动计算栈帧相关信息，包括临时数组的大小。
调用 `visitEnd` 退出。

## 语句与表达式的翻译

语句的翻译实现在

```scala
def emitStmt(stmt: Stmt)(implicit mv: MethodVisitor, loopExits: List[Label], ctx: Context): Unit
```

表达式的翻译实现在

```scala
def emitExpr(expr: Expr)(implicit mv: MethodVisitor, ctx: Context): Unit
```

实现方法与 PA3 类似，这里不再赘述。
需要留意的是，在 JVM 这种栈式虚拟机下，指令生成与 TAC 相比其实更加简单。
但是，这里的一个难点在于，你的脑海中始终要有一个操作数栈的概念，因为这些指令的输入输出都跟栈紧密相关。
这有助于理清楚每条指令的具体含义。
