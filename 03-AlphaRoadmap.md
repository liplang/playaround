# LIP Alpha Compiler Roadmap

## 总体架构

建议第一版固定成：

```text
.lip
 │
 ▼
Lexer
 │
 ▼
Parser
 │
 ▼
AST
 │
 ▼
Name / Scope Resolution
 │
 ▼
Dependency Analysis
 │
 ▼
LIP IR / Dependency Graph
 │
 ▼
Scheduling / Runtime Metadata
 │
 ▼
Go Code Generator
 │
 ▼
gofmt
 │
 ▼
go build
 │
 ▼
Executable
```

不要一开始做 LLVM、自己的 VM、自己的 GC，也不要试图直接把 LIP 编译成机器码。

**Go 就是 Alpha 阶段的后端。**

---

# Phase 0：先冻结 Alpha 语言边界

### 目标

先把 Spec 0.1 变成一个**可实现的语言子集**。

暂时只保证：

```text
变量绑定
表达式
函数调用
if
conditional expression
when
for
while
comprehension
function
import
state
```

同时明确：

**Alpha 不做：**

```text
async/await
Stream
Event DSL
Agent DSL
Workflow DSL
Effect Type System
复杂 Feedback syntax
复杂 Error DSL
自定义 VM
```

这一阶段最重要的不是写代码，而是建立：

```text
Spec
 ↓
Grammar
 ↓
Semantic Rules
 ↓
Examples
```

三者必须能够互相对应。

### 特别注意

从现在开始，最好建立一个：

```text
tests/conformance/
```

把目前 01～30 个例程全部保存下来。

以后每增加一个语义，都必须问：

> **这个语义能不能被一个可执行测试验证？**

---

# Phase 1：Lexer + Parser

这一阶段先完全不要考虑 Runtime。

目标：

```lip
a = foo()
b = a + 1

if b > 10 {
    print(b)
}
```

能够稳定解析成 AST。

例如：

```text
Program
 ├── Binding(a)
 │    └── Call(foo)
 ├── Binding(b)
 │    └── Binary(+)
 │         ├── Identifier(a)
 │         └── Literal(1)
 └── If
      └── ...
```

### 建议

Parser 不要急着把“Dependency”塞进去。

**AST 应该忠实表达 source language。**

Dependency 是下一层分析出来的东西。

这是一个非常重要的架构边界：

> **AST 表达“程序写了什么”；Dependency Graph 表达“计算依赖什么”。**

---

# Phase 2：Name Resolution / Scope

解决：

```text
a = 10
b = a + 1
```

中的：

> `a` 到底指向哪个 binding？

包括：

* local scope
* function scope
* block scope
* shadowing
* imports
* undefined variable
* duplicate binding

Alpha 可以故意保持简单。

### 我建议

**暂时禁止 forward reference：**

```lip
b = a + 1
a = 10
```

报错即可。

这样会大幅降低第一版 compiler 的复杂度。

以后再开放。

---

# Phase 3：Dependency Graph —— 真正的核心

这是整个编译器最重要的一层。

例如：

```lip
a = foo()
b = bar()
c = combine(a, b)
```

得到：

```text
       ┌──→ a ──┐
input?           ├──→ c
       └──→ b ──┘
```

更准确地说：

```text
Node A
Node B
Node C

A → C
B → C
```

### 这里有一个非常重要的设计原则

不要：

> AST Node = Graph Node

必须保持：

> **Graph Node ≠ Syntax Node ≠ Execution Instance**

例如：

```lip
squares = [x * x for x in numbers]
```

不能因为有 1,000,000 个元素，就产生 1,000,000 个静态 Graph Node。

Graph 应描述：

```text
Map(numbers, x*x)
```

Runtime 再产生 execution instances。

---

# Phase 4：先做“假的 Scheduler”

这一阶段非常关键。

**不要一开始就做复杂并发。**

先让 Runtime 能够：

```text
Pending
Ready
Running
Completed
Error
Cancelled
```

并且按照 Dependency Graph：

```text
A
↓
B
↓
C
```

自动执行。

第一版甚至可以：

> **完全单线程、顺序执行。**

只要：

```text
Dependency 正确
Pending 正确
Gate 正确
Error propagation 正确
```

就成功了一大半。

---

# Phase 5：Go Code Generation

现在开始：

```text
LIP → Go
```

第一版甚至可以采用非常朴素的：

```text
strings.Builder
+
template
+
gofmt
```

没必要立即上 Go AST。

例如：

```lip
a = foo()
b = bar()
c = combine(a, b)
```

先生成：

```go
a := foo()
b := bar()
c := combine(a, b)
```

对于完全静态、没有 Runtime 语义介入的代码，**尽可能生成普通 Go。**

这是一个很重要的优化原则：

> **能直接生成 Go，就不要进入 Runtime。**

---

# Phase 6：引入 Dependency Runtime

到了这里才真正体现 LIP 的特色。

例如：

```lip
a = fetch_a()
b = fetch_b()
c = combine(a, b)
```

不能再简单生成：

```go
a := fetch_a()
b := fetch_b()
c := combine(a, b)
```

因为这样 Go 会按照 source order 执行。

需要生成类似：

```text
Runtime
 ├── Node A
 ├── Node B
 └── Node C
       ↑
      A,B
```

Runtime 决定：

```text
A ─┐
   ├─→ C
B ─┘
```

于是：

```text
A/B 可以并行
C 等待 A/B
```

### 但这里千万不要急着做复杂 scheduler。

Alpha 可以：

```text
Phase 6a
单线程 dependency scheduler

Phase 6b
goroutine scheduler

Phase 6c
worker pool / concurrency limits
```

这样风险低很多。

---

# Phase 7：`when`

这是 Alpha 非常值得优先实现的特色。

```lip
valid = validate(input)

when valid {
    user = load_user(id)
    data = load_data(id)
}
```

语义：

```text
validate
   ↓
 valid
   ↓
 Gate
 ┌─┴────┐
 ↓      ↓
user   data
```

这里必须严格区分：

```text
if
```

和：

```text
when
```

### `if`

决定：

> **计算哪个值。**

### `when`

决定：

> **计算是否获得执行资格。**

这是 Compiler / IR 层面必须明确区分的。

---

# Phase 8：Pending / Error

然后加入真正的 Runtime execution state：

```text
Pending
Ready
Running
Completed
Error
Cancelled
```

例如：

```lip
page = fetch(url)
text = extract(page)
```

Runtime：

```text
page  = Pending
text  = Pending

fetch complete

page  = Completed
text  = Ready

extract starts
```

这里非常重要：

> **Pending 不是 LIP 用户层面的普通 Value。**

不要让 compiler 把它暴露成：

```text
Option<Value>
```

或者：

```text
Value | Pending
```

之类的语言类型。

它应该是 Runtime execution state。

---

# Phase 9：Map / Dynamic Expansion

再实现：

```lip
urls = discover_urls()
pages = [fetch(url) for url in urls]
```

Compiler 编译的是：

```text
Map(fetch, urls)
```

而不是：

```text
fetch(url1)
fetch(url2)
fetch(url3)
...
```

Runtime 才知道：

```text
urls.length == N
```

然后动态展开。

这一阶段会真正验证之前非常重要的原则：

> **Static Structure + Dynamic Expansion**

---

# Phase 10：State + Tick

最后再把：

```lip
count = state(0)

double = count * 2
```

接入 Runtime。

概念上：

```text
Tick 1

count = 0
 ↓
double = 0


Tick 2

count = 1
 ↓
double = 2
```

然后实现：

> **只重新计算受到变化影响的节点。**

例如：

```text
a
↓
b
↓
c
```

如果 a 没变：

```text
a unchanged
b unchanged
c unchanged
```

而：

```text
style → c
```

变化：

```text
a unchanged
b unchanged
c recompute
```

这才开始真正体现 LIP 的：

> **Dependency-driven incremental computation**

---

# Phase 11：并行化

这时候再做：

```text
A ──┐
    ├── C
B ──┘
```

自动：

```go
go A()
go B()
wait
C()
```

但不要把 Go `goroutine` 暴露给 LIP 用户。

这是 Runtime implementation detail。

### 最重要的一条

不要保证：

> “没有 dependency 就一定并行。”

应该保证：

> **没有 dependency 的 computation 具有并行执行的可能性。**

因为以后还会加入：

```text
Effect
Resource limit
Concurrency limit
Ordering
```

---

# Phase 12：Effect / Ordering

最后处理：

```lip
send_email(user)
write_log("email sent")
```

这两个没有 Data Dependency。

但业务上可能要求：

```text
send_email
     ↓
write_log
```

这里先不要设计完整 Effect Type System。

Alpha 可以采用轻量 metadata：

```text
pure
read_only
idempotent_write
external_write
```

甚至先只是 compiler/runtime 内部 metadata。

然后 scheduler：

```text
Data Dependency
+
Gate
+
Ordering
+
Effect Constraint
        ↓
Scheduler
```

这会让架构以后可以扩展，而不需要推翻。

---

# 最后：Trace / Inspect

我反而建议**很早就做，甚至 Phase 4 就开始留接口。**

因为 LIP 最大的问题不是“能不能执行”，而是：

> **程序员怎么知道 Runtime 到底在干什么？**

例如：

```text
Tick 17

fetch_user     Completed
fetch_posts    Running
combine        Pending
```

或者：

```text
Node: combine
State: Pending

Waiting for:
  ✓ user
  ✗ posts
```

这会成为 LIP 非常重要的开发体验。

未来甚至可以：

```text
lip run main.lip --trace
lip inspect
lip watch
```

---

# 我建议的实际研发顺序

如果你真的准备让一个人/一个 AI 按这个路线实现，我会压缩成下面 **8 个 Milestone**：

| Milestone | 内容                          | 目标              |
| --------- | --------------------------- | --------------- |
| **M0**    | Spec + Grammar              | 冻结 Alpha 子集     |
| **M1**    | Lexer + Parser + AST        | 能解析语言           |
| **M2**    | Resolver + Dependency Graph | 真正理解 LIP        |
| **M3**    | Basic Go Codegen            | LIP → Go 跑起来    |
| **M4**    | Runtime + Pending + Gate    | LIP 核心语义出现      |
| **M5**    | Map + Dynamic Expansion     | 动态依赖出现          |
| **M6**    | State + Tick + Incremental  | LIP 的第二核心       |
| **M7**    | Parallel Scheduler + Effect | 真正发挥 Runtime 优势 |

然后：

```text
M7
 ↓
01–30 全部 Conformance Tests
 ↓
Alpha 0.1
```

---

# 最需要警惕的 6 个坑

这个比具体技术路线还重要。

### ① 不要把 LIP 编译器写成“Go 代码翻译器”

这是最大的坑。

错误方向：

```text
LIP syntax
   ↓
Go syntax replacement
```

正确方向：

```text
LIP
 ↓
Semantic Analysis
 ↓
Dependency Graph / IR
 ↓
Go
```

**Dependency Graph 才是 Compiler 的灵魂。**

---

### ② 不要让 AST 决定 Runtime Node

一定牢记：

> **Syntax Node ≠ Graph Node ≠ Execution Instance**

否则 Map、循环、函数调用、普通表达式很快都会把 Runtime 搞爆。

---

### ③ 不要过早追求并行

第一版：

```text
Dependency
→ sequential scheduler
```

完全没问题。

如果单线程语义都没验证清楚，直接上 goroutine，debug 会非常痛苦。

---

### ④ 不要过早设计 Reactive 系统

这是我们前面最大的经验之一。

不要因为：

```text
state
tick
feedback
```

就开始设计：

```text
Observable
Stream
Event
Operator
Subscription
```

**State 是核心；Reactive 是 Runtime 行为。**

---

### ⑤ 不要把所有东西都做成 Runtime Node

例如：

```lip
for x in numbers {
    sum = sum + x
}
```

完全可以编译成普通 Go：

```go
for _, x := range numbers {
    sum += x
}
```

否则 LIP 会变成一个巨大的 interpreter。

---

### ⑥ Runtime API 一定要保持小

Alpha Runtime 我甚至建议先控制在几个核心概念：

```text
Node
Value
State
Tick
Scheduler
ExecutionState
Trace
```

而不是几十个抽象。

---

# 最后给你一个我认为非常重要的研发原则

整个 Alpha Compiler 可以始终围绕这一条检查：

> **如果一个问题可以由 Go 解决，就不要让 LIP 解决。**
>
> **如果一个问题属于“计算之间的关系与执行时序”，才应该让 LIP 解决。**

所以：

```text
Go
├── type
├── struct
├── interface
├── function
├── package
├── IO
├── HTTP
├── database
├── algorithm
├── loop
└── ordinary computation

LIP
├── dependency
├── gate
├── state
├── tick
├── scheduling
├── pending
├── incremental propagation
└── execution lifecycle
```

这样 LIP 才能保持我们现在最珍贵的那个特征：

> **表面上，它仍然是一门简单、正常、好写的编程语言；复杂性没有消失，而是被移动到了 Runtime。**

而这也正是整个项目最值得验证的假设：

> **如果“依赖驱动”真的成立，那么程序员可以用接近普通编程语言的代码表达复杂的并发、异步、增量和状态驱动程序，而无需亲自管理这些执行机制。**

这应该就是 Alpha 编译器真正要证明的东西。
