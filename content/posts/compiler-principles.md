+++
date = '2026-06-20T01:31:43+08:00'
draft = false
title = '编译原理'
tags = ['compile','Go']
+++

# 编译原理：从源代码到可执行程序

高级语言中的变量、函数、结构体、接口、模块等概念更接近抽象语义；机器实际执行的是指令、寄存器操作、内存访问和跳转。编译器的核心任务是将高级语言程序转换为机器可执行的目标形式，并在这个过程中完成语法约束、类型约束、控制流约束、内存布局和目标平台约束的处理。

典型编译流程可以抽象为：

```text
源代码
  -> 词法分析：字符流 -> Token 流
  -> 语法分析：Token 流 -> AST
  -> 语义分析：AST -> 带类型、作用域、绑定关系的 AST
  -> 中间代码生成：AST -> IR
  -> 代码优化：IR -> 优化后的 IR
  -> 目标代码生成：IR -> 汇编 / 机器指令
  -> 汇编与链接：目标文件 -> 可执行文件
```

这条流水线的关键不是“把文本翻译成另一段文本”，而是不断把程序转换为更适合下一阶段处理的结构化表示。Token 适合语法分析，AST 适合表达源代码结构，IR 适合数据流和控制流优化，机器指令适合在具体硬件上执行。

## 编译过程概览

以一段简单的 Go 代码为例：

```go
package main

func add(a, b int) int {
	return a + b
}
```

在编译器内部，它会经历类似下面的结构转换：

```text
源码文本
  "func add(a, b int) int { return a + b }"

Token 流
  FUNC IDENT(add) LPAREN IDENT(a) COMMA IDENT(b) IDENT(int) RPAREN ...

AST
  FuncDecl
    Name: add
    Params: a int, b int
    Results: int
    Body:
      ReturnStmt
        BinaryExpr(+)
          Ident(a)
          Ident(b)

语义信息
  add: func(int, int) int
  a: int
  b: int
  a + b: int

IR
  v1 = Arg <int> {a}
  v2 = Arg <int> {b}
  v3 = Add <int> v1 v2
  Ret v3

目标代码
  从调用约定指定的位置读取参数
  执行整数加法
  将结果放到返回值寄存器
  返回调用方
```

编译器通常分为前端、中端和后端：

- 前端负责源语言相关处理，包括词法、语法、语义、类型系统和作用域。
- 中端负责语言无关的优化，核心对象通常是 IR、CFG、SSA 等中间表示。
- 后端负责目标平台相关处理，包括指令选择、寄存器分配、栈帧布局、调用约定和机器码生成。

这种分层使源语言和目标平台之间不需要强耦合。一个语言前端可以生成统一 IR，再由多个后端分别生成 x86-64、ARM64、RISC-V 等平台代码。

## 词法分析

词法分析的输入是字符流，输出是 Token 流。Token 是带类别的最小语法单元，通常包含类型、字面量、源码位置等信息。

例如：

```go
x := 1 + 2
```

词法分析结果可以表示为：

```text
IDENT    literal="x"  pos=1:1
DEFINE   literal=":=" pos=1:3
INT      literal="1"  pos=1:6
ADD      literal="+"  pos=1:8
INT      literal="2"  pos=1:10
EOF
```

Token 通常可以抽象为类似结构：

```go
type Token struct {
	Kind    TokenKind
	Literal string
	Line    int
	Column  int
}
```

词法分析器不需要理解表达式优先级，也不需要知道 `x` 是否已经声明。它只负责把字符序列切成稳定的词法单元。

词法规则通常由正则表达式或有限状态机描述：

```text
identifier = letter { letter | digit | "_" }
integer    = digit { digit }
operator   = "+" | "-" | "*" | "/" | ":=" | "==" | "!="
space      = " " | "\t" | "\n"
```

扫描字符串字面量时，状态机会比普通标识符复杂：

```text
初始状态
  -> 遇到 " 进入字符串状态
  -> 遇到 \ 进入转义状态
  -> 遇到匹配的 " 结束字符串
  -> 遇到换行或 EOF 报错
```

例如：

```go
s := "a\nb"
```

词法分析器需要把源码里的两个字符 `\` 和 `n` 识别为转义序列，而不是普通字符。非法字符串：

```go
s := "abc
```

会在词法阶段报错，因为字符串字面量没有闭合。

## 语法分析

语法分析的输入是 Token 流，输出是 AST。AST 只保留程序结构中对语义有意义的部分，括号、分号、逗号等纯语法元素通常只用于辅助构建树。

例如：

```go
return a + b * c
```

根据乘法优先级高于加法，AST 可以表示为：

```text
ReturnStmt
  BinaryExpr(+)
    Ident(a)
    BinaryExpr(*)
      Ident(b)
      Ident(c)
```

如果写成：

```go
return (a + b) * c
```

AST 会变成：

```text
ReturnStmt
  BinaryExpr(*)
    BinaryExpr(+)
      Ident(a)
      Ident(b)
    Ident(c)
```

这说明括号本身未必保留在 AST 中，但括号改变了树的形状。

表达式语法可以用简化的 EBNF 表示：

```text
expr    = term { ("+" | "-") term }
term    = factor { ("*" | "/") factor }
factor  = ident | integer | "(" expr ")"
```

这组规则可以解析 `a + b * c`，因为 `expr` 处理加减，`term` 处理乘除，乘除天然位于更深层结构中。

许多编译器会使用递归下降或 Pratt Parser 处理表达式。以 Pratt Parser 为例，它会给操作符设置绑定强度：

```text
* / : 20
+ - : 10
==  : 5
```

解析 `a + b * c` 时，`*` 的绑定强度高于 `+`，因此先形成 `b * c`，再与 `a` 组成加法表达式。

语法阶段只能检查“结构是否合法”，不能检查“含义是否合法”。例如：

```go
var x int = "hello"
```

这段代码语法结构完整，因此语法分析可以成功；类型不匹配属于语义分析阶段的问题。

## 语义分析

语义分析负责给 AST 附加语言规则层面的约束信息，包括类型、作用域、名称绑定、常量求值、控制流合法性等。

以变量声明为例：

```go
var x int = "hello"
```

语法树可以正常构造：

```text
VarDecl
  Name: x
  Type: int
  Value:
    StringLiteral("hello")
```

语义分析会检查：

```text
declared type: int
value type:    string
assignment:    string -> int 不可赋值
```

因此该错误属于类型检查错误，而不是语法错误。

语义分析通常依赖符号表。符号表记录名字与声明之间的绑定关系：

```go
func area(width, height int) int {
	return width * height
}
```

可以得到类似信息：

```text
scope: package
  area -> func(int, int) int

scope: function area
  width  -> int, parameter
  height -> int, parameter
```

表达式 `width * height` 的类型推导过程：

```text
lookup(width)  -> int
lookup(height) -> int
operator(*)    -> int * int -> int
return type    -> int
function result type -> int
match
```

作用域检查也在这个阶段完成。例如：

```go
func f() int {
	if true {
		x := 1
	}
	return x
}
```

`x` 只存在于 `if` 代码块内部，`return x` 处无法绑定到合法声明，因此语义分析会报未定义标识符错误。

控制流检查同样属于语义分析的一部分：

```go
func abs(x int) int {
	if x >= 0 {
		return x
	}
}
```

该函数声明返回 `int`，但存在一条控制流路径没有返回值。语义分析或后续控制流分析会识别这类问题。

## 中间代码生成

AST 适合表达源码结构，但不适合直接做复杂优化。现代编译器通常会把 AST 降低为 IR。IR 更接近执行逻辑，并且弱化了源语言表面语法。

例如：

```go
x = a + b * c
```

三地址码形式可以表示为：

```text
t1 = b * c
t2 = a + t1
x  = t2
```

每条指令最多包含一个核心操作，数据依赖关系更清晰。

控制流通常会被拆成基本块。基本块是只有一个入口和一个出口的连续指令序列：

```go
func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}
```

对应控制流图可以抽象为：

```text
B0:
  v1 = Arg a
  v2 = Arg b
  v3 = Gt v1 v2
  If v3 -> B1, B2

B1:
  Ret v1

B2:
  Ret v2
```

控制流图，通常称为 CFG，是很多优化的基础。死代码删除、循环优化、支配关系分析、变量活跃性分析都依赖 CFG。

现代编译器还经常使用 SSA 形式。SSA 的特点是每个变量只赋值一次。遇到控制流合并时，需要使用 `phi` 节点表示变量来源：

```go
func f(cond bool) int {
	x := 1
	if cond {
		x = 2
	}
	return x
}
```

SSA 形式可以抽象为：

```text
B0:
  x1 = 1
  If cond -> B1, B2

B1:
  x2 = 2
  Jump B2

B2:
  x3 = Phi(x1 from B0, x2 from B1)
  Ret x3
```

`Phi` 节点表示：如果控制流从 B0 直接到达 B2，`x3` 取 `x1`；如果从 B1 到达 B2，`x3` 取 `x2`。SSA 让变量定义和使用关系非常明确，便于做常量传播、死代码删除、公共子表达式消除等优化。

## 代码优化

代码优化的对象通常是 IR。优化必须保持程序在语言规范下的可观察行为不变。这个约束非常关键，因为许多代数上看似等价的表达式，在固定宽度整数、浮点数、并发内存模型、异常机制下未必等价。

常见优化包括：

- 常量折叠：编译期直接计算常量表达式。
- 常量传播：把已知常量传播到使用点。
- 死代码删除：删除结果不会被使用的计算。
- 公共子表达式消除：相同表达式只计算一次。
- 循环不变量外提：把循环中不变的表达式移到循环外。
- 函数内联：把小函数体展开到调用点。
- 逃逸分析：判断对象是否必须分配到堆上。
- 强度削弱：用更低成本的操作替代高成本操作。

常量折叠示例：

```go
func calc() int {
	x := 1 + 2
	return x * 10
}
```

可能先变成：

```text
x = 3
return x * 10
```

再经常量传播和折叠变成：

```text
return 30
```

死代码删除示例：

```go
func f(a int) int {
	x := a * 10
	y := a + 1
	return y
}
```

如果 `x` 没有被使用，且 `a * 10` 没有副作用，则可以删除：

```text
y = a + 1
return y
```

公共子表达式消除示例：

```go
func f(a, b int) int {
	x := (a + b) * 10
	y := (a + b) * 20
	return x + y
}
```

优化前：

```text
t1 = a + b
x  = t1 * 10
t2 = a + b
y  = t2 * 20
ret x + y
```

优化后：

```text
t1 = a + b
x  = t1 * 10
y  = t1 * 20
ret x + y
```

循环不变量外提示例：

```go
func sum(a []int, k int) int {
	total := 0
	for i := 0; i < len(a); i++ {
		total += a[i] * (k + 1)
	}
	return total
}
```

如果 `k` 在循环中不变，`k + 1` 可以被外提：

```text
t = k + 1
for i := 0; i < len(a); i++ {
	total += a[i] * t
}
```

### 溢出语义与表达式改写

表达式改写必须受语言语义约束。以中点计算为例：

```go
mid := (left + right) / 2
```

从数学角度看，可以尝试改写为：

```go
mid := left + (right-left)/2
```

但在固定宽度整数中，`left + right` 可能溢出。编译器不能只根据代数等价进行改写，还需要证明改写前后在语言规范下行为一致。

在 IR 中，该表达式可能表示为：

```text
t1 = Add left, right
t2 = Div t1, 2
```

优化器要改写它，通常需要经过：

```text
模式匹配：Div(Add(x, y), 2)
语义检查：整数是否有符号，除法如何取整，溢出是否定义
范围分析：x + y 是否可能溢出，y - x 是否可能溢出
重写规则：生成新的 IR
```

如果语言规定有符号整数溢出是未定义行为，优化器可以在“不发生溢出”的前提下做更多代数优化。C/C++ 的有符号整数溢出属于这类语义。

如果语言规定整数溢出按固定宽度回绕，编译器必须保留回绕行为。Go 的整数溢出具有确定语义，因此编译器不能随意把 `(left + right) / 2` 改成防溢出版本，否则溢出场景下的结果会改变。

更底层的 IR 还可能携带“不会溢出”的标记。以 LLVM IR 为例：

```text
%s = add nsw i32 %a, %b
```

`nsw` 表示 no signed wrap，也就是该有符号加法不会溢出。只有在这类约束成立时，优化器才能安全使用依赖“不溢出”的规则。

无溢出平均值还可以用位运算实现：

```text
avg = (a & b) + ((a ^ b) >> 1)
```

这个公式常用于无符号整数平均值。对于有符号整数、负数、不同语言的右移规则和除法取整方向，还需要额外处理。因此编译器优化不会只匹配字符串形态，而是基于 IR、类型、范围、溢出语义和目标语言规范共同判断。

### 逃逸分析

Go 代码中，逃逸分析会决定对象可以放在栈上还是必须放到堆上：

```go
func newValue() *int {
	x := 10
	return &x
}
```

`x` 的地址被返回到函数外部，当前函数返回后仍可能被访问，因此 `x` 必须逃逸到堆上。

另一个例子：

```go
func value() int {
	x := 10
	return x
}
```

`x` 只在函数内部使用，通常可以保留在栈上，甚至直接放入寄存器。逃逸分析结果会影响 GC 压力、分配成本和对象生命周期。

## 目标代码生成

目标代码生成把优化后的 IR 转换为目标平台上的指令。这个阶段会处理指令选择、寄存器分配、栈帧布局、调用约定、指令调度等问题。

例如 IR：

```text
v1 = Arg a
v2 = Arg b
v3 = Add v1, v2
Ret v3
```

在某个寄存器调用约定下，可能被转换为类似指令：

```asm
MOVQ a_reg, AX
ADDQ b_reg, AX
RET
```

具体寄存器、指令名称、参数传递位置都依赖目标平台和 ABI。

指令选择会把 IR 操作映射为机器指令。例如：

```text
x = y * 8
```

在某些平台上可以选择乘法指令：

```asm
IMUL y, 8
```

也可以选择移位：

```asm
SHL y, 3
```

后者通常成本更低，但是否可用取决于类型、溢出语义、目标平台指令集和优化等级。

寄存器分配决定哪些值放在寄存器，哪些值溢出到栈上。假设某段 IR 同时需要很多临时变量：

```text
t1 = a + b
t2 = c + d
t3 = e + f
t4 = t1 * t2
t5 = t4 + t3
```

如果可用寄存器不足，某些临时值需要 spill 到栈上：

```text
store t2 -> [sp+offset]
...
load [sp+offset] -> register
```

寄存器分配质量会直接影响运行性能。

栈帧布局负责安排局部变量、临时 spill 槽、保存的寄存器、返回地址等内容：

```text
高地址
  调用者栈帧
  返回地址
  保存的寄存器
  局部变量
  spill 槽
低地址
```

Go 可以通过下面的命令查看某个文件的汇编输出：

```bash
go tool compile -S main.go
```

汇编输出可以结合 IR、调用约定和栈帧布局理解，而不是只看单条指令。

## 汇编与链接

目标代码生成之后，通常还需要汇编和链接。编译器前面生成的可能是汇编文本或目标文件，最终可执行文件由链接器组织完成。

链接器主要处理：

- 符号解析：函数名、全局变量名最终指向哪个定义。
- 重定位：代码和数据在最终地址空间中的位置修正。
- 库链接：静态库或动态库的引用处理。
- 段布局：`.text`、`.data`、`.bss`、只读数据等段的组织。
- 入口点：程序从哪个地址开始执行。

例如两个文件：

```go
// a.go
package main

func main() {
	println(add(1, 2))
}
```

```go
// b.go
package main

func add(a, b int) int {
	return a + b
}
```

编译 `a.go` 时，`add` 可以先作为外部符号存在。链接阶段会把 `add` 的调用绑定到 `b.go` 生成的函数地址。

对于 Go 这类带运行时的语言，链接结果还包含调度器、GC、栈增长、panic/recover、反射元数据等运行时支持。因此最终二进制不仅包含业务函数，也包含语言运行时所需代码和数据。

## 解释执行、JIT 与 AOT

编译原理同样适用于解释器、虚拟机和 JIT。

AOT，Ahead Of Time，表示运行前完成编译。Go、C、Rust 常见编译链路属于这一类：

```text
源码 -> 编译器 -> 机器码 -> 运行
```

解释执行会在运行时直接执行 AST、字节码或其他中间形式：

```text
源码 -> AST / 字节码 -> 解释器逐条执行
```

JIT，Just In Time，表示运行过程中根据热点代码动态编译：

```text
源码 -> 字节码
运行时收集 profile
热点函数 -> JIT 编译为机器码
继续执行
```

很多现代运行时不是单一模式，而是混合模式。例如 JavaScript 引擎可能先解释执行，发现热点函数后进行基线编译，再对稳定热点路径做更激进优化；如果运行时假设失效，再进行反优化，回退到较低层级执行。

JIT 优化会使用运行时信息。例如某个对象属性访问在大量执行中总是命中同一种对象形状，JIT 可以生成针对该形状的快速路径：

```text
if object.shape == cachedShape:
  read field at fixed offset
else:
  fallback to generic lookup
```

这种优化依赖运行时 profile，AOT 编译器通常无法获得同样的信息。

## 编译原理与工程实践

编译原理中的“解析 -> 结构化表示 -> 语义检查 -> 优化 -> 执行计划”模式，在很多工程系统中都会出现。下面几个例子比抽象描述更接近实际代码。

### 语言机制实现

闭包不是简单的函数指针。它需要捕获外层变量，并把函数代码和捕获环境一起保存：

```go
func addBase(base int) func(int) int {
	return func(x int) int {
		return base + x
	}
}
```

编译器可能把闭包降低为类似结构：

```text
Closure {
  code: addBase.func1
  env: {
    base: int
  }
}
```

调用闭包时，运行时不仅调用函数代码，还要把 `env` 作为隐藏参数传入。`base` 如果逃出当前函数栈帧，还会触发逃逸分析并影响分配位置。

`defer` 也不是普通函数调用：

```go
func f() {
	defer cleanup()
	work()
}
```

它会被编译为“注册延迟调用 + 函数退出时执行”的控制流结构。涉及 panic/recover 时，还要和运行时的异常展开机制配合。

### 性能分析中的编译器行为

性能问题不只来自算法复杂度，也可能来自编译器决策。

例如接口调用：

```go
type Writer interface {
	Write([]byte) error
}

func save(w Writer, data []byte) error {
	return w.Write(data)
}
```

`w.Write` 可能是动态派发，编译器在无法确定具体类型时，很难直接内联目标函数。若某个调用点能被证明只有一个具体实现，编译器才可能做去虚化和内联。

再例如逃逸：

```go
func build() *Item {
	item := Item{}
	return &item
}
```

这里 `item` 逃逸到堆上，会产生分配和 GC 成本。性能分析时，`go build -gcflags="-m"` 这类命令可以观察逃逸和内联信息。最终性能不是源码字面形态直接决定的，而是由语义分析、逃逸分析、内联、SSA 优化和后端生成共同决定。

### DSL 与规则引擎

规则引擎可以看作小型编译器。例如一条风控规则：

```text
age >= 18 && country == "CN" && score > 700
```

处理过程可以拆成：

```text
Token:
  IDENT(age) GE INT(18) AND IDENT(country) EQ STRING("CN") AND IDENT(score) GT INT(700)

AST:
  And
    And
      Compare(age, >=, 18)
      Compare(country, ==, "CN")
    Compare(score, >, 700)

语义检查:
  age: int
  country: string
  score: int
  比较操作类型合法

执行计划:
  load age
  compare >= 18
  short-circuit if false
  load country
  compare == "CN"
  short-circuit if false
  load score
  compare > 700
```

如果规则很多，还可以做优化：

```text
把低成本、高过滤率条件放到前面
把常量表达式提前折叠
把重复字段读取合并
把规则编译成字节码或函数
```

这与编译器中的短路求值、常量折叠、公共子表达式消除非常接近。

### SQL 优化器

数据库查询优化器也具有编译器特征。SQL 输入不是直接执行，而是先被转换为逻辑计划，再转换为物理计划：

```sql
SELECT u.name
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.amount > 100;
```

可能形成逻辑计划：

```text
Project(u.name)
  Filter(o.amount > 100)
    Join(u.id = o.user_id)
      Scan(users)
      Scan(orders)
```

优化器可以进行谓词下推：

```text
Project(u.name)
  Join(u.id = o.user_id)
    Scan(users)
    Filter(o.amount > 100)
      Scan(orders)
```

物理计划还需要选择：

```text
Hash Join
Nested Loop Join
Index Scan
Table Scan
```

这和编译器后端的指令选择类似：逻辑语义相同，但物理执行方式不同，成本模型决定最终选择。

### 模板引擎与表达式求值

模板引擎同样包含编译过程：

```text
Hello, {{ user.name }}!
{{ if user.vip }}VIP{{ end }}
```

可以被解析为：

```text
Text("Hello, ")
Expr(GetField(user, "name"))
Text("!")
If(GetField(user, "vip"))
  Text("VIP")
End
```

运行时可以解释 AST，也可以把模板编译成字节码或目标语言函数。对高频模板而言，预编译通常能减少重复解析成本。

## 总结

编译器不是一次性把源码文本替换成机器码，而是经过多层结构转换：

```text
字符流 -> Token -> AST -> 语义信息 -> IR -> 优化后的 IR -> 目标代码 -> 可执行文件
```

词法分析解决字符如何组成 Token，语法分析解决 Token 如何组成结构，语义分析解决结构是否满足语言规则，IR 让控制流和数据流分析变得可操作，优化阶段在保持语义的前提下改写程序表示，目标代码生成把抽象操作映射到具体机器指令。

编译原理关注的核心问题是：如何表示程序、如何证明转换合法、如何在约束条件下选择成本更低的执行形式。这个问题不仅存在于编译器，也存在于规则引擎、SQL 优化器、模板引擎、表达式求值器和多类执行计划生成系统中。

## 附录：英文缩写

| 缩写 | 全称 | 说明 |
| --- | --- | --- |
| IR | Intermediate Representation | 中间表示。位于源码和目标代码之间的程序表示，常用于优化和后端代码生成。 |
| AST | Abstract Syntax Tree | 抽象语法树。语法分析后的树形结构，保留程序的核心语法结构。 |
| CFG | Control Flow Graph | 控制流图。由基本块和跳转边组成，用于表示程序执行路径。 |
| SSA | Static Single Assignment | 静态单赋值形式。每个变量只赋值一次，便于数据流分析和优化。 |
| Phi | Phi Function | SSA 中用于合并不同控制流来源变量值的特殊节点。 |
| EBNF | Extended Backus-Naur Form | 扩展巴科斯范式。用于描述编程语言或 DSL 的语法规则。 |
| EOF | End Of File | 文件结束标记。词法分析器通常会在 Token 流末尾追加 EOF。 |
| LLVM | Low Level Virtual Machine | 一套编译器基础设施，包含 IR、优化器和多平台后端。 |
| nsw | No Signed Wrap | LLVM IR 中的标记，表示有符号整数运算不会发生回绕溢出。 |
| ABI | Application Binary Interface | 应用二进制接口。规定调用约定、寄存器使用、栈布局、二进制兼容等规则。 |
| AOT | Ahead Of Time | 提前编译。程序运行前生成目标代码。 |
| JIT | Just In Time | 即时编译。程序运行过程中根据热点代码动态生成机器码。 |
| DSL | Domain Specific Language | 领域特定语言。面向特定业务或技术领域的小语言。 |
| SQL | Structured Query Language | 结构化查询语言。数据库查询优化器通常会把 SQL 转换为逻辑计划和物理计划。 |
| GC | Garbage Collection | 垃圾回收。运行时自动管理堆对象生命周期的机制。 |
