# LIP Design Memo

## 设计思想与来路

> **这不是语言规范（Specification），也不是实现文档。**
>
> 本文记录 LIP 最初的设计想法、问题意识、关键概念以及设计取舍。它回答的不是“LIP 必须怎样实现”，而是：
>
> **我们为什么想设计这样一门语言，以及当时希望它最终成为什么样子。**
>
> 因此，本文中的语法、术语和具体机制都可能在后续实现中改变；真正需要保留下来的，是背后的设计动机。

---

# 一、LIP 是什么？

LIP 是一门**以数据流、管道计算和时序逻辑为核心的编程语言**。

它希望把几种通常分开的编程思想统一起来：

```text
Dataflow
Functional Programming
Reactive Programming
Temporal / Synchronous Programming
```

但 LIP 并不希望因此成为一门“只能写 Pipeline 的 DSL”。

它在表面上应该仍然是一门**自然、简洁、类似现代通用语言的编程语言**：

```lip
a = foo(x)
b = bar(x)
c = baz(a, b)
```

而在底层，它把程序理解为一张：

> **随着数据和状态变化而不断推进的依赖图。**

因此，LIP 可以用一句话概括：

> **LIP 是一种以数据依赖为程序骨架、以逻辑时序驱动执行的数据流语言。**

---

# 二、它看起来像普通代码，骨子里却是一张 Flow

LIP 最重要的特点，是**程序的表面形式和内部计算模型之间存在两层视角**。

程序员写：

```lip
a = foo(x)
b = bar(x)
c = baz(a, b)
```

看起来只是普通的变量和函数。

但 LIP 内部看到的是：

```text
             ┌──→ foo ──→ a ──┐
x ───────────┤                 ├──→ baz ──→ c
             └──→ bar ──→ b ──┘
```

这张图就是一个 **Flow**。

因此：

* `a` 依赖 `foo(x)`
* `b` 依赖 `bar(x)`
* `c` 同时依赖 `a` 和 `b`
* `foo` 和 `bar` 没有相互依赖，因此可以并行
* `baz` 自动等待 `a`、`b` 都准备好

程序员不需要显式描述这些调度关系。

这也是 LIP 最核心的思想：

> **程序员描述数据依赖，运行时根据依赖关系组织计算。**

---

# 三、什么是“管道计算”？

LIP 所说的 Pipe，并不只是传统意义上的：

```text
A → B → C → D
```

而是更一般的：

> **数据从一个计算节点流向另一个计算节点。**

因此一个计算：

```text
x → f → y
```

就是最基本的 Pipe。

多个计算可以分叉：

```text
        ┌→ f → a
x ──────┤
        └→ g → b
```

也可以汇合：

```text
a ─┐
   ├→ h → c
b ─┘
```

也可以形成反馈：

```text
      ┌────────┐
      ↓        │
x → f → y → g ─┘
```

因此，LIP 最终希望把程序中常见的：

* 顺序计算
* 分支
* 合并
* 条件
* 循环
* 反馈
* 聚合
* 并行

都理解为某种数据流关系。

但需要特别强调：

> **Pipe 是 LIP 的语义，而不是要求程序员不断书写 `|>`。**

例如：

```lip
y = (x + 1) * 2
```

完全应该保持自然。

编译器可以在内部理解为：

```text
x → (+1) → (*2) → y
```

所以 LIP 不是“Pipeline 语法”，而是：

> **以 Pipeline / Dataflow 为底层计算模型的普通语言。**

---

# 四、什么是“时序”？

LIP 不仅关心：

> **数据依赖谁？**

还关心：

> **数据什么时候发生变化？**

因此 LIP 的 Flow 不是一张静态的图，而是一张可以随着时间不断推进的图。

一个输入发生变化，可以产生一个 **Logical Tick**：

```text
Tick 1

输入变化
   ↓
依赖传播
   ↓
节点重新计算
   ↓
新的数据继续传播
   ↓
直到这一轮达到稳定状态
```

下一次变化：

```text
Tick 2
```

再次推进。

这里的“时间”首先不是现实世界的秒、毫秒，也不等同于传统的：

```text
async
await
event loop
callback
```

它更接近：

> **程序状态发生了一次可观察变化，并由此引发一轮依赖传播。**

因此 LIP 希望形成一种：

> **语义上同步，执行上可以并行的时序模型。**

程序员看到的是清晰、确定的逻辑时序；运行时则可以利用 Flow 中天然存在的并行性。

---

# 五、什么是“待定状态”？

LIP 还希望把一个普通程序中经常被迫隐藏起来的事实显式化：

> **一个数据并不一定已经准备好了。**

例如：

```lip
x = fetch(url)
y = process(x)
```

在 `fetch` 尚未完成时：

```text
x = Pending
y = Pending
```

而不是：

```text
x = null
```

也不是：

```text
y = error
```

当 `x` 最终获得结果：

```text
x = Ready(data)
```

Flow 自动继续：

```text
x
↓
process
↓
y
```

因此：

> **Pending 不是错误，而是“依赖尚未满足”的正常状态。**

这使 LIP 天然适合表达：

* 外部输入
* 网络请求
* 长时间计算
* AI Agent
* 等待工具调用
* 增量计算
* 持续运行的程序

---

# 六、Flow 是什么？

在 LIP 中，可以把 **Flow** 理解成：

> **一组计算节点以及它们之间的数据依赖关系。**

例如：

```text
             ┌→ A ─┐
Input ───────┤      ├→ C
             └→ B ─┘
```

就是一个 Flow。

Flow 并不只是传统意义上的“函数调用序列”。

它可能：

```text
分叉
汇合
等待
并行
反馈
重新计算
暂停
继续
```

因此一个 Flow 本质上更接近：

> **一个可以随数据变化而不断推进的计算图。**

不过在 Alpha 阶段，我们并不急于把 Flow 本身做成一种普通的数据类型。

也就是说：

> **Flow 是程序的组织结构，而不是首先要成为一种新的复杂运行时对象。**

---

# 七、LIP 的几个关键概念可以放在一起看

如果把上面的东西压缩起来，LIP 的整体模型大致是：

```text
                  Data
                    │
                    ▼
              ┌──────────┐
              │   Flow   │
              │          │
              │ A → B    │
              │ ↓   ↓    │
              │ C → D    │
              └────┬─────┘
                   │
             Logical Tick
                   │
                   ▼
              Scheduler
                   │
          ┌────────┴────────┐
          ▼                 ▼
       Ready              Pending
          │                 │
          └──────→ Flow ────┘
```

换句话说：

### Data

是程序处理的东西。

### Pipe / Dependency

描述数据如何从一个计算进入另一个计算。

### Flow

把这些计算组织成一张依赖图。

### Pending

表示依赖尚未满足，计算暂时不能继续。

### Logical Tick

表示一次状态变化所引起的计算推进。

### Runtime

负责根据 Flow 自动安排执行。

---

# 八、一个完整例子

例如：

```lip
input request

plan = llm(request)
tool = choose_tool(plan)
result = call(tool)
answer = llm(result)

when answer {
    output(answer)
}
```

表面上，这就是非常普通的程序。

内部则是：

```text
request
   │
   ▼
  llm
   │
   ▼
 plan
   │
   ▼
choose_tool
   │
   ▼
 tool
   │
   ▼
 call
   │
   ▼
result
   │
   ▼
 llm
   │
   ▼
answer
   │
   ▼
output
```

如果某一步需要等待：

```text
call(tool)
```

那么 Flow 可以停在：

```text
Pending
```

而不是把整个程序写成一堆：

```text
async
await
callback
```

如果未来存在两条没有依赖关系的路径：

```text
request
 ├──→ search
 │
 └──→ retrieve
```

运行时就可以自动并行。

这也是为什么 LIP 特别适合 AI Agent：

> **Agent 中大量存在“等待输入 → 产生结果 → 触发下一步 → 再等待”的过程，而这正是 LIP 的基本计算模型。**

---

# 九、LIP 并不排斥普通编程

虽然底层是 Flow，但 LIP 不希望所有代码都变成：

```text
node
node
node
pipe
pipe
pipe
```

相反：

```lip
fn calculate(x) {
    total = 0

    for i in 0..100 {
        total += i * x
    }

    total
}
```

这种普通的命令式代码完全可以存在。

因此 LIP 更准确的定位是：

> **Dataflow-oriented general-purpose language**

而不是：

> **Pipeline-only language**

外层用 Flow 组织程序。

Flow 内部仍然可以使用普通的：

* 函数式计算
* 命令式代码
* OOP
* 数据结构
* 算法

---

# 十、LIP 与 Go 的基本关系

为了让整个项目足够小，LIP 不打算重新实现一套完整的语言基础设施。

最初的设想是：

```text
             LIP
              │
              ▼
        Dependency / Flow
              │
              ▼
          Go source
              │
              ▼
         Go compiler
              │
              ▼
        Native program
```

LIP 尽可能直接采用 Go 的：

* 数据类型
* struct
* interface
* function
* package
* import
* 标准库
* 类型系统
* runtime

因此：

> **Go 是 LIP 的计算底座；LIP 主要重新定义程序的组织方式。**

这也是为什么第一版不需要做一个庞大的编译器。

---

# 十一、RxGo 的位置

RxGo 对 LIP 很有启发意义。

LIP 借鉴 Reactive Programming、Stream、Operator、Scheduler 等思想，但并不直接建立在 RxGo 之上。

两者关注点并不完全相同：

```text
RxGo
Event / Stream
      ↓
Observable
      ↓
Operators
      ↓
Scheduler
```

而 LIP 更希望：

```text
Data / State
      ↓
Dependency Graph
      ↓
Logical Tick
      ↓
Scheduler
      ↓
Stable State
```

因此：

> **RxGo 是参考对象，不是 LIP 的地基。**

---

# 十二、最终希望形成的感觉

理想中的 LIP 应该让程序员产生这样的感觉：

```text
“我只是把数据之间的关系写出来了，
至于什么时候算、哪些可以并行、
哪里需要等待、哪些东西需要重新计算，
让运行时自己处理。”
```

代码仍然是：

```text
简洁的
自然的
容易阅读的
容易学习的
```

但其背后是：

```text
数据流
+
依赖图
+
逻辑时序
+
自动调度
+
可暂停状态
+
自动并行
```

因此可以把 LIP 最终的愿景压缩成一句话：

> **表面像普通现代语言，骨子里是一张随逻辑时序推进的数据依赖图。**

---

# 十三、为什么要保持“小”

LIP 当前最重要的实验，不是证明：

> “我们能够设计一门完整的新语言。”

而是验证：

> **“数据依赖 + 管道计算 + 逻辑时序”是否真的能成为一种更好的程序组织方式。**

因此第一阶段应该尽可能复用 Go，而不是重新发明：

```text
类型系统
OOP
GC
VM
async runtime
完整 Stream 系统
Symbolic Computation
IDE
```

目标是：

> **约 2～3 万行 Go，在几周内形成一个真正可以写东西的 Alpha。**

如果这个 Core 被证明是舒服的，再让语言自然成长。

---

# 十四、当前最值得记住的设计原则

最后，可以把整个 Memo 浓缩成下面这些原则：

> **1. 数据依赖是程序的骨架。**

> **2. Pipe 是语义，不是强制性的语法。**

> **3. Flow 是计算的组织结构。**

> **4. Pending 是正常的运行状态，而不是错误。**

> **5. Logical Tick 描述状态变化和计算传播。**

> **6. 语义上同步，执行上可以并行。**

> **7. 程序员描述依赖，Runtime 决定调度。**

> **8. Flow 负责组织，普通代码负责具体计算。**

> **9. Go 提供语言和生态的底座，LIP 只增加新的计算组织模型。**

> **10. RxGo 是参考，而不是基础设施。**

> **11. 不追求一开始解决所有问题。**

> **12. 先用一个极小的 Core 证明思想，再让语言成长。**

---

## 一个最简的心智模型

如果一年以后只剩下一张图还能想起来，我希望是这张：

```text
                         LIP
                          │
             ┌────────────┼────────────┐
             │            │            │
           Data          Flow         Time
             │            │            │
             │      dependency graph   │
             │            │        logical tick
             │            │            │
             └────────────┼────────────┘
                          │
                     Pending / Ready
                          │
                          ▼
                      Runtime
                          │
                   automatic scheduling
                          │
                  ┌───────┴───────┐
                  ▼               ▼
               sequential      parallel
                          │
                          ▼
                         Go
```

**这就是 LIP 最初想做的东西。**

后面的所有具体语法、类型设计、编译器结构，理论上都应该服务于这个模型；如果未来某个具体设计和这个模型发生冲突，就回到这里重新问：**我们究竟是在解决问题，还是只是在给语言增加功能？**
