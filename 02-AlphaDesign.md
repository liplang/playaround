# Lip Alpha Design & Specification v0.1

> ### **Describe dependencies. Let the runtime decide execution.**
> ### **描述依赖，让 Runtime 决定执行。**

这两句话，现在已经足以作为 LIP Alpha 的设计宪法。

> **Status: Alpha / Experimental**
>
> 本规范服务于 LIP 第一版编译器与 Runtime 的实现。
>
> 它不是最终语言规范，而是一份用于验证核心模型、约束 Alpha 实现范围、指导编译器架构设计的阶段性 Specification。

---

# 0. Design Manifesto

LIP 的核心不是重新发明一种表达式语言，也不是再创造一种 Pipeline DSL。

LIP 试图重新定义的是：

> **程序如何表达计算之间的关系，以及 Runtime 如何根据这些关系决定执行。**

LIP 的核心原则可以浓缩为：

> **程序员描述依赖，Runtime 决定执行。**

以及：

> **LIP 不把“并行”“异步”“响应式”“Agent”“工作流”分别设计成语言特性；它只定义计算之间的依赖、门控与状态变化，而把由此产生的执行时序交给 Runtime。**

因此，LIP 的程序：

* 表面上应该像普通现代语言；
* 可以使用函数式、命令式乃至 OOP 风格；
* 可以调用 Go 函数和 Go 库；
* 但在程序的更深层，是一张由数据依赖、执行门控和状态变化构成的计算结构；
* Runtime 根据这张结构决定何时执行、哪些可以并行、哪些必须等待。

---

# 1. LIP 是什么

LIP 是一种：

> **以数据依赖为程序骨架、以逻辑时序驱动执行的数据流语言。**

其核心计算模型可以表示为：

```text
                   LIP Program
                        │
                        ▼
                Dependency Structure
                        │
          ┌─────────────┼─────────────┐
          │             │             │
      Data Dependency   Gate        Effect
          │             │             │
          └─────────────┼─────────────┘
                        ▼
                 Execution State
                        │
       ┌────────┬───────┼───────┬────────┐
       ▼        ▼       ▼       ▼        ▼
    Pending    Ready  Running  Error  Cancelled
                        │
                        ▼
                    Scheduler
                        │
                        ▼
                      Tick
                        │
                        ▼
                 Changed State
                        │
                        └────→ Next Tick
```

这里最重要的是：

**Graph 是程序的语义结构，而不是要求程序员直接操作的对象。**

程序员通常不需要写：

```text
node A
node B
connect A -> B
parallel A B
wait B
```

而只需要写：

```lip
a = foo()
b = bar(a)
```

编译器从普通代码中推导出：

```text
a → b
```

---

# 2. Design Principles

LIP Alpha 遵循以下原则。

## 2.1 Dependency is semantics, not syntax

依赖关系是语义，不是语法。

```lip
a = foo()
b = bar(a)
```

自然意味着：

```text
foo → bar
```

不需要：

```lip
pipe(foo, bar)
```

也不需要：

```lip
b = a |> bar
```

---

## 2.2 Pipe is semantics, not syntax

LIP 的计算天然可以理解为：

```text
input → computation → output
```

因此任何计算都可以在语义上视为 Pipe。

但这并不意味着 LIP 必须要求用户显式写 Pipe。

例如：

```lip
x = a + 1
```

内部可以理解为：

```text
a → (+1) → x
```

但程序员只需要写自然的表达式。

---

## 2.3 Source Order is not Execution Order

源代码出现的先后顺序，不是主要的执行顺序。

```lip
a = foo()
b = bar()
c = combine(a, b)
```

语义上：

```text
foo ──┐
      ├──→ combine
bar ──┘
```

因此：

```text
foo
bar
```

如果不存在其他约束，可以并行执行。

### Alpha 限制

为了简化第一版 Resolver：

> **Alpha 暂不允许普通变量的 forward reference。**

例如：

```lip
print(x)
x = 10
```

暂不合法。

这是**语法/编译器限制**，不是核心语义原则。

---

# 3. Dependency Graph

LIP 程序可以被编译器转换为 Dependency Structure。

Dependency Structure 至少包含：

### 3.1 Data Dependency

```lip
a = foo()
b = a + 1
```

产生：

```text
a → b
```

---

### 3.2 Fan-out

```lip
page = fetch(url)

title = extract_title(page)
text  = extract_text(page)
links = extract_links(page)
```

形成：

```text
             ┌→ title
page ────────┼→ text
             └→ links
```

多个下游计算可以独立执行。

---

### 3.3 Fan-in / Join

```lip
a = fetch_a()
b = fetch_b()
c = fetch_c()

result = combine(a, b, c)
```

形成：

```text
a ──┐
b ──┼──→ combine
c ──┘
```

`combine` 必须等待所有必要依赖进入可用状态。

---

# 4. Graph Node 不等于 Syntax Node

这是 Alpha 一个非常重要的架构原则。

以下三者必须区分：

```text
Syntax Node
Graph Node
Execution Instance
```

它们不是同一个东西。

例如：

```lip
pages = [fetch(url) for url in urls]
```

并不意味着：

```text
10000 urls
    ↓
10000 permanent Graph Nodes
```

而应该是：

```text
              Map
               │
       Runtime Dynamic Expansion
          / / / / / / / /
       execution instances
```

因此：

> **Graph 粒度由调度价值决定，而不是由语法节点或集合元素决定。**

---

# 5. Static Structure + Dynamic Expansion

LIP Graph 不要求完全静态。

例如：

```lip
urls = discover_urls()
pages = [fetch(url) for url in urls]
```

编译器可以知道：

```text
pages depends on urls
```

但 `urls` 最终包含多少元素，只有 Runtime 才知道。

因此 Alpha 使用：

> **Static Structure + Runtime Dynamic Expansion**

的模型。

---

# 6. Conditional Computation：`if`

`if` 表示条件计算或值选择。

例如：

```lip
y = if x > 10 then 100 else 0
```

其语义是：

```text
x
 ↓
condition
 ├── true  → 100
 └── false → 0
             ↓
             y
```

---

## 6.1 Statement-level `if`

```lip
if x > 10 {
    y = foo(x)
} else {
    y = bar(x)
}
```

可以理解为：

```text
              condition
              /       \
           foo         bar
              \       /
                join
                  ↓
                  y
```

但这种 Graph 表达是**语义模型**，不是要求编译器必须真的构造这些细粒度 Node。

---

# 7. Execution Gate：`when`

30 个例程中，这是最重要的新结论之一。

纯 Data Dependency 无法表达所有业务上的先后约束。

例如：

```lip
valid = validate(input)

user = load_user(input.user_id)
data = load_data(input)
```

从 Data Dependency 看：

```text
validate ──→ valid

load_user ──┐
            ├── independent
load_data ──┘
```

但是业务上可能要求：

> 只有验证通过以后，才允许读取用户数据。

因此：

```lip
valid = validate(input)

when valid {
    user = load_user(input.user_id)
    data = load_data(input)
}
```

这里：

> `when` 表示 **Execution Gate / Logical Prerequisite**。

---

# 8. `if` 与 `when` 的区别

这是 Alpha 必须明确规定的语义区别。

### `if`

> Conditional computation / value selection

回答：

> **应该产生哪个结果？**

例如：

```lip
x = if valid then a else b
```

---

### `when`

> Execution gate

回答：

> **这个计算什么时候才允许发生？**

例如：

```lip
when valid {
    send_email(user)
}
```

---

因此：

```text
if   → selection
when → permission to execute
```

两者不能简单视为同一个东西。

---

# 9. Pending

LIP Runtime 中，一个计算可能尚未具备执行条件。

例如：

```lip
page = fetch(url)
text = extract(page)
```

当 `fetch` 尚未完成：

```text
page → Pending
text → Pending
```

当 `page` Ready：

```text
page → Ready
text → Ready / Running
```

因此：

> **Pending 不是 Error，也不是 null。**

Pending 表示：

> 该计算在逻辑上存在，但其依赖尚未满足。

---

# 10. 不需要显式 `await`

LIP 的普通依赖等待不需要：

```lip
await page
```

因为：

```lip
text = extract(page)
```

已经完整表达：

```text
text depends on page
```

因此：

> **Dependency itself expresses waiting.**

这也是 LIP 与传统 async/await 模型的重要区别。

---

# 11. Runtime Execution State

Alpha Runtime 至少需要区分以下状态：

```text
Pending
Ready
Running
Completed
Error
Cancelled
```

其中：

* **Pending**：依赖尚未满足
* **Ready**：可以执行
* **Running**：正在执行
* **Completed**：执行完成
* **Error**：执行失败
* **Cancelled**：执行被取消

这些主要属于 Runtime semantics，而不是用户每天操作的普通 Value 类型。

---

# 12. Logical Tick

LIP 的 Tick 是：

> **一次逻辑状态变化所触发的依赖传播过程。**

它不是：

* wall-clock 时间
* OS thread
* event loop iteration
* goroutine

例如：

```text
External State Change
        ↓
      Tick N
        ↓
Dependency Propagation
        ↓
Stable State
        ↓
External State Change
        ↓
      Tick N+1
```

同一个 Node：

> **默认在一个 Tick 中最多执行一次。**

这有利于：

* 确定性
* trace
* caching
* incremental computation
* replay
* scheduling

---

# 13. State

普通值：

```lip
x = foo()
```

表示计算结果。

State：

```lip
count = state(0)
```

表示：

> 一个跨 Logical Tick 保持身份的状态单元。

例如：

```lip
count = state(0)

double = count * 2
```

当 State 从：

```text
0 → 1
```

发生变化：

```text
Tick 1:
count = 0
double = 0

Tick 2:
count = 1
double = 2
```

---

## Alpha Decision

Alpha **承认 State 是核心语义概念**。

但 State mutation 的完整语法：

```text
count = count + 1
```

究竟如何定义，暂时不完全规定。

### Open Question

* State 是否允许直接赋值？
* 是否需要 `set count = ...`？
* State update 是否自动产生新 Tick？
* State 与 Feedback 的正式关系是什么？

---

# 14. External State 与 Event

外部世界可以改变 LIP 所观察的状态。

例如：

```lip
temperature = sensor.read()

when temperature > 80 {
    alarm()
}
```

Host / Runtime 可以在外部状态发生变化时触发：

```text
External Change
      ↓
New Tick
```

### Alpha 原则

完整 Event System 暂不进入 Core。

Alpha 优先采用：

```text
Host → Runtime → Tick
```

而不是先定义：

```lip
on event { ... }
```

### Open Question

未来是否需要：

```lip
on request
on message
on timer
```

之类的语言级 Event primitive。

---

# 15. Incremental Computation

LIP 不应每个 Tick 都无条件重新执行整个 Graph。

例如：

```lip
a = fetch_data()
b = expensive_compute(a)
c = format(b, style)
```

如果只有：

```text
style changed
```

那么理想情况下：

```text
a unchanged
b unchanged
c recompute
```

因此：

> **LIP Runtime 应支持基于依赖变化的增量传播。**

---

## Alpha 简化

Alpha 必须定义：

> 输入变化会触发依赖重新传播。

但暂时不规定复杂的 Value Equality 体系。

### Open Question

Change Detection 到底基于：

* identity
* equality
* structural equality
* user-defined equality

暂不决定。

Memoization / cache 也暂时视为 Runtime optimization。

---

# 16. Map

例如：

```lip
urls = get_urls()
pages = [fetch(url) for url in urls]
```

语义上：

```text
Map(fetch, urls)
```

每个元素之间如果没有依赖：

> Runtime 可以并行执行。

但 LIP **不承诺具体并行方式**。

例如 Runtime 可以使用：

* goroutine
* worker pool
* sequential execution
* bounded concurrency

都不改变语言语义。

---

# 17. Map + Reduce

```lip
numbers = get_numbers()

squares = [x * x for x in numbers]

total = sum(squares)
```

这里：

```text
numbers
   ↓
 Map
   ↓
squares
   ↓
Reduce
   ↓
total
```

Map/Reduce 是语言语义上的计算结构。

但不要求：

```text
每一个 x = 一个 Runtime Node
```

---

# 18. Parallelism

LIP 不提供：

```lip
parallel {
    ...
}
```

作为一般情况下的必要语法。

因为：

```lip
a = foo()
b = bar()
c = combine(a, b)
```

已经表达：

```text
foo ──┐
      ├──→ combine
bar ──┘
```

如果 `foo` 与 `bar` 没有其他约束：

> Runtime SHOULD schedule them concurrently.

因此：

> **并行是 Dependency Graph 的执行结果，而不是程序员必须声明的语法。**

---

# 19. Data Dependency ≠ Complete Dependency

这是 Alpha 非常重要的限制。

例如：

```lip
send_email(user)
write_log("email sent")
```

两个操作可能没有数据依赖，但：

```text
send_email
      ↓
write_log
```

可能存在业务上的逻辑顺序。

因此 LIP 的最终调度模型不是：

```text
Data Dependency → Scheduler
```

而是：

```text
Data Dependency
       +
Execution Gate
       +
Effect / Ordering Constraint
       ↓
Execution State
       ↓
Scheduler
```

---

# 20. Effect

Alpha 承认：

> Effect 会影响合法的执行顺序。

但：

**Alpha 不引入完整 Effect Type System。**

例如可以存在轻量 metadata：

```text
PURE
READ_ONLY
EXTERNAL_WRITE
```

或者类似：

```go
// effect: external_write
```

具体形式暂定。

Runtime 可以据此采取更保守的调度策略。

---

## Alpha 原则

优先：

```text
Explicit Data Dependency
```

其次：

```text
Effect / Ordering Metadata
```

而不是：

```text
所有 side effect 全部串行
```

后者会严重削弱 LIP 的价值。

### Open Question

完整 Effect System 是否进入未来版本。

---

# 21. Error

计算失败是 Runtime 的合法状态：

```text
Pending
Ready
Running
Error
```

例如：

```lip
a = fetch()
b = a + 1
c = b * 2
```

如果：

```text
a → Error
```

则依赖它的：

```text
b
c
```

不能正常执行。

Error 因此参与 Dependency Propagation。

---

# 22. Error Recovery

Alpha 不为：

```text
retry
fallback
recover
ignore
```

分别设计语言 primitive。

优先使用普通函数：

```lip
data = fetch_with_fallback(url)
```

或者：

```lip
result = recover(fetch(url))
```

具体错误处理由 Go function / library 决定。

原则是：

> **LIP 决定什么时候可以执行；Go 决定一次执行失败之后具体怎么处理。**

---

# 23. Cancellation

Cancellation 属于 Runtime Concern。

如果：

```text
a → b → c
```

而 `a` 被取消：

```text
a → Cancelled
```

则依赖它的：

```text
b
c
```

通常也失去执行条件。

因此：

> Cancellation SHOULD propagate through dependent computations.

但：

```text
detach
background
independent task
```

等语义暂不进入 Alpha。

### Open Question

是否允许一个 computation 从当前 Flow 中 detach。

---

# 24. Feedback

LIP 的 Dependency Structure **概念上允许 Feedback**。

例如 Agent workflow：

```text
plan
 ↓
execute
 ↓
evaluate
 ↓
bad?
 ↓
replan
 ↓
execute
```

但 Alpha 不把所有循环都解释成 Graph-level Feedback。

---

## Local Loop

普通算法：

```lip
sum = 0

for x in numbers {
    sum = sum + x
}
```

应该正常存在。

它可以直接编译为 Go：

```go
for ...
```

不需要把每一次循环都变成 Runtime Tick。

因此：

> **Local algorithm semantics 与 Flow-level reactive semantics 必须分离。**

---

# 25. Agent

Agent 不是 LIP 的特殊对象。

例如：

```lip
request = user_input()

plan = llm_plan(request)
result = execute(plan)
evaluation = llm_evaluate(result)

if evaluation.success {
    answer = result
} else {
    plan = llm_replan(request, result, evaluation)
    result = execute(plan)
}
```

它只是：

```text
Dependency
+
Gate
+
State
+
Feedback
+
External Effects
```

因此：

> **LIP ≠ Agent DSL。**

Agent Workflow 是 LIP 可以自然表达的一种程序。

---

# 26. Workflow

同理，Workflow 不应该成为另一套语言。

例如：

```text
request
   ↓
parse
   ↓
validate
   ↓
load user ────┐
              ├──→ response
load data ────┘
```

本身就是 Dependency Structure。

因此：

> Workflow 是 LIP 程序的一种组织方式，而不是额外的语言实体。

---

# 27. Reactive Programming

LIP 可以表现出 Reactive Programming 的性质：

```text
State Change
     ↓
Dependency Propagation
     ↓
Affected Computations
     ↓
New Stable State
```

但 LIP Alpha 不定义完整 Observable / Stream / Operator 系统。

这与 RxGo 等模型不同。

LIP 更关注：

```text
State / Data
    ↓
Dependency Graph
    ↓
Logical Tick
    ↓
Scheduler
    ↓
Stable State
```

而不是：

```text
Observable
    ↓
Operators
    ↓
Event Stream
    ↓
Scheduler
```

因此 Reactive 是 LIP 的**语义性质**，不是必须暴露给程序员的 API 风格。

---

# 28. Runtime Architecture

Alpha Runtime 可以保持非常小。

核心组件建议包括：

```text
runtime/
    node.go
    value.go
    state.go
    tick.go
    scheduler.go
    trace.go
    error.go
```

职责：

### Node

表示可调度的计算单元。

### Value

保存计算结果及其状态。

### State

保存跨 Tick 的状态身份。

### Tick

表示一次逻辑传播轮次。

### Scheduler

决定：

```text
Ready
→ Running
→ Completed / Error
```

以及是否并行执行。

### Trace

记录：

```text
node
tick
start
finish
state
dependency
error
```

---

# 29. Compiler Architecture

第一版编译器建议：

```text
Source
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
Name / Dependency Resolution
  │
  ▼
Dependency Graph / Tiny IR
  │
  ▼
Go Code Generation
  │
  ▼
gofmt
  │
  ▼
go build
```

---

# 30. LIP 与 Go

LIP 不重新实现：

* 类型系统
* GC
* struct
* interface
* function
* package
* standard library
* network
* database
* OS interface
* 基础并发机制

这些由 Go 提供。

LIP 主要增加：

```text
Dependency
Flow
Gate
State
Tick
Scheduling
Pending
Trace
```

因此：

```text
LIP
 ↓
Go
 ↓
Go Compiler
 ↓
Executable
```

---

# 31. Go Interop

Alpha 优先采用：

> **编译期生成 Go，并直接使用 Go package/function。**

例如：

```lip
import net/http

response = http.get(url)
```

最终可以生成相应 Go 代码。

不建议 Alpha 依赖 Go `plugin` 机制。

原因是：

> LIP 的自然边界就是 source-level / compile-time interop，而不是动态加载 Go binary。

---

# 32. Python / 外部语言

Python 暂不进入 Alpha Core。

未来可以通过：

```text
JSON
RPC
Unix Socket
HTTP
Subprocess
```

等方式连接。

例如：

```text
LIP
 ↓
Go Runtime
 ↓
Python Worker
```

这样不会污染第一版语言核心。

---

# 33. Performance Philosophy

Alpha 的第一目标不是“和 Go 一样快”。

第一目标是：

> **证明 LIP 的模型能够运行、能够表达真实程序，并且确实给程序员带来明显收益。**

因此：

```text
semantic correctness
        >
usability
        >
runtime overhead
        >
micro-optimization
```

但对于普通纯计算：

> Compiler SHOULD generate direct Go code whenever practical.

Runtime overhead 应主要集中在真正需要 Runtime 管理的部分：

```text
dependency tracking
scheduling
state propagation
dynamic expansion
trace
```

而不应该让：

```lip
x = a + b
```

也产生不必要的巨大 Runtime 开销。

---

# 34. Trace / Inspect / Debug

由于 LIP 的程序骨架是 Dependency Structure，Runtime 天然可以记录：

```text
Tick 12

fetch_user
    Ready → Running → Completed

fetch_orders
    Ready → Running → Completed

make_response
    Pending → Ready → Running
```

未来可以支持：

```text
inspect graph
watch node
trace tick
explain why pending
explain why not executed
```

这些属于：

> **Runtime / Tooling capability，而不是语言核心语法。**

Alpha Runtime SHOULD 保留足够 trace information，使这些工具未来可实现。

---

# 35. Determinism / Replay

LIP 的 Logical Tick 模型天然适合：

```text
Trace
Replay
Debugging
```

例如：

```text
Tick 1
Tick 2
Tick 3
```

可以记录：

```text
input state
node execution
output
error
state transition
```

从而允许未来实现 deterministic replay。

但：

### Open Question

对于：

* random
* current time
* network
* LLM
* external side effect

如何实现严格 replay，Alpha 暂不规定。

---

# 36. Local Imperative Code

LIP **不排斥命令式编程**。

例如：

```lip
result = complex_algorithm(data)
```

内部完全可以：

```go
for ...
if ...
mutable variables ...
struct methods ...
```

甚至使用 OOP。

因此：

> LIP 的 Dataflow 语义主要存在于 **computation boundaries / scheduling boundaries**，而不是强迫介入每一条局部指令。

这是非常重要的逃生口。

---

# 37. Graph Boundary

Alpha 不要求：

```text
every expression = graph node
```

例如：

```lip
result = complicated_algorithm(data)
```

可以只是：

```text
        ┌──────────────────────┐
data →  │ complicated_algorithm │ → result
        └──────────────────────┘
```

内部如何计算，由 Go 普通语义解决。

因此：

> **Node 是调度单元，而不是语法单元。**

---

# 38. Alpha Core Syntax

经过 30 个例程，目前真正需要进入 Alpha Core 的东西非常少。

建议暂定：

```text
variable binding
expression
function call
if
conditional expression
when
for
while
collection comprehension
state
function definition
import
```

除此之外：

> 新增语言 primitive 必须证明无法合理地由现有语义 + Go library 实现。

---

# 39. Alpha 明确不做

第一版明确不做：

```text
完整 Effect Type System
完整 Stream System
完整 Event DSL
Agent DSL
Workflow DSL
async/await primitive
复杂 reactive operator system
自定义 VM
自定义 GC
完整 OOP system
复杂 scheduler language
Python 原生语言嵌入
IDE
```

这些并不意味着 LIP 永远不会支持它们。

只是：

> **如果某个能力可以通过普通 LIP + Go library + Runtime 实现，就没有理由现在把它做成语言 primitive。**

---

# 40. Open Questions

Alpha 必须保留一个明确的 Open Questions 区，而不是把未决定内容偷偷塞进语义。

目前至少包括：

### OQ-01 — State Mutation

```lip
count = count + 1
```

的正式语义是什么？

---

### OQ-02 — State 与 Feedback

State update 是否自动形成 Feedback Edge？

---

### OQ-03 — Change Detection

如何判断：

```text
old value == new value
```

---

### OQ-04 — Error Model

是否最终引入类似：

```text
Result<T, E>
```

的语言级模型？

---

### OQ-05 — Effect System

是否需要正式：

```text
PURE
READ_ONLY
IDEMPOTENT_WRITE
EXTERNAL_WRITE
```

Effect System？

---

### OQ-06 — Event Syntax

是否需要：

```lip
on event { ... }
```

等语言级 Event primitive？

---

### OQ-07 — Feedback Syntax

是否需要显式：

```lip
feedback
```

或者其他 reactive cycle syntax？

---

### OQ-08 — Cancellation / Detach

是否允许：

```text
background
detach
independent task
```

---

### OQ-09 — Transaction

数据库 transaction 是否应该被 LIP Runtime 感知？

---

### OQ-10 — Concurrency Limits

是否提供：

```lip
map(..., concurrency=10)
```

之类的语言级约束，还是完全由 Runtime / library 决定？

---

### OQ-11 — Deterministic Replay

外部世界不可确定时，Replay 的正式语义如何定义？

---

### OQ-12 — Forward Reference

Alpha 暂不允许。

未来是否开放？

---

# 41. Alpha Conformance Examples

第一版编译器至少应该能够通过以下类别的测试：

```text
01  Hello Data
02  Dependency Chain
03  Independent Computation
04  Conditional Execution
05  Conditional Value
06  Fan-out
07  Fan-in / Join
08  Map
09  Map + Reduce
10  Dynamic Flow
11  Pending / Async Fetch
12  Parallel IO
13  Timeout
14  Retry
15  External State
16  Local Loop
17  Feedback
18  Agent Retry Loop
19  HTTP Handler
20  Parallel Database Query

21  Incremental Computation
22  State Update
23  Event / Trigger
24  Error Propagation
25  Error Recovery
26  Cancellation
27  Dynamic Fan-out / Fan-in
28  Side Effect / Ordering
29  Agent Workflow
30  Complete Web / AI Workflow
```

这些不是“示例程序”这么简单。

它们实际上构成：

> **LIP Alpha 的语义压力测试集。**

任何 Alpha 设计修改，都应该重新检查这 30 个例程。

---

# 42. 最终设计原则

经过 30 个例程之后，LIP Alpha 最值得固定下来的原则，我认为是下面这 12 条：

### 1. Dependency is semantics, not syntax.

依赖关系由普通代码自然产生，不需要额外 DSL。

### 2. Pipe is semantics, not syntax.

LIP 可以把计算理解为 Pipe，但不强迫用户写 Pipe。

### 3. Dataflow is the program skeleton.

数据依赖构成程序的骨架。

### 4. Source order is not execution order.

代码排列不是主要调度依据。

### 5. `if` selects; `when` gates.

`if` 解决条件计算；`when` 解决执行门控。

### 6. Pending is a runtime state.

普通依赖等待不需要 `await`。

### 7. Graph structure is not execution instances.

Graph Node、Syntax Node、Map Element 必须分离。

### 8. State change drives logical time.

External State Change 可以产生新的 Logical Tick。

### 9. Data dependency is not the whole dependency.

Gate 和 Effect / Ordering Constraint 同样可能影响调度。

### 10. Local algorithms remain ordinary code.

函数式、命令式、OOP 都可以自然存在。

### 11. Runtime decides execution.

并行、等待、调度、增量传播等不应该成为用户负担。

### 12. Keep the language small.

如果某个能力能够通过：

```text
existing LIP semantics
+
Go function
+
library
+
runtime
```

自然解决，就不要轻易增加新的语言 primitive。

---

# 43. LIP 的最终核心模型

如果把整份 Spec 再压缩到只剩一张图，我认为现在应该是：

```text
                         LIP
                          │
                          ▼
                ┌─────────────────┐
                │ Dependency      │
                │ Structure       │
                └────────┬────────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
            Data        Gate       Effect
             │           │          │
             └───────────┼──────────┘
                         ▼
                ┌─────────────────┐
                │ Execution State │
                └────────┬────────┘
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
          Pending       Ready       Error
                         │
                         ▼
                    Scheduler
                         │
                 ┌───────┴───────┐
                 ▼               ▼
             Sequential       Parallel
                 │               │
                 └───────┬───────┘
                         ▼
                       Tick
                         │
                         ▼
                    New State
                         │
                         └──────→ Next Tick
```

而程序员看到的仍然只是：

```lip
input = receive()

valid = validate(input)

when valid {
    user = load_user(input.user_id)
    data = load_data(input)

    result = process(user, data)
}

response = make_response(result)
```

**这就是最接近 Lip 本质的地方：**

程序员没有写：

```text
parallel
async
await
join
spawn
schedule
event
workflow
agent
retry
```

但这些能力在正确的语义下，**都可能自然地从 Dependency + Gate + State 中涌现出来。**
