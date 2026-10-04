# LIP

> **LIP — Dependency-Oriented Programming Language**
> 一种以依赖关系描述计算、由 Runtime 决定执行时序的编程语言。

## 为什么需要 LIP？

过去几十年的主流编程语言，都在不同程度上要求程序员描述**程序如何执行**。

```text
程序 = 指令序列          → 命令式
程序 = 函数组合          → 函数式
程序 = 会传播变化的值/事件 → Reactive
程序 = 节点 + 数据通道    → Dataflow
```

这些范式各有价值，但它们都有一个共同的历史假设：

> **控制执行顺序，是程序员的职责。**

于是，程序员不得不越来越多地管理：

* 顺序与并行
* 等待与同步
* 异步与调度
* 事件与响应
* 状态与更新
* 缓存与重新计算

而现代计算机实际上越来越擅长做另一件事：

> **根据计算之间的依赖关系，决定什么应该在什么时候运行。**

换句话说：

> **机器本来就会以依赖图的方式执行。**
> **只是我们的语言仍然要求程序员把这个图“伪装成一串指令”。**

---

## LIP 的核心思想

LIP 做一个简单的转变：

> **程序 = 计算之间的依赖关系。**

程序员描述：

> **什么依赖什么。**

Runtime 决定：

> **何时执行、是否并行、何时重新计算，以及如何响应状态变化。**

例如：

```lip
a = fetch_a()
b = fetch_b()
c = combine(a, b)
```

这里真正重要的不是：

```text
先执行 a
再执行 b
最后执行 c
```

而是：

```text
a ──┐
    ├──→ c
b ──┘
```

因此 `a` 和 `b` 没有依赖关系，Runtime 可以让它们并行执行；`c` 则自动等待二者完成。

**程序员不需要写 goroutine、wait、join、async、await。**

---

## LIP 与 Reactive 的区别

LIP 不是 Reactive Programming 的又一种实现。

> **Reactive 把“响应”本身做成程序员需要操作的东西。**

于是程序员需要面对：

```text
Observable
Subscription
Operator
Scheduler
Event
Stream
Backpressure
Cancellation
...
```

而 LIP 希望：

> **程序员只描述关系，响应机制藏在 Runtime 后面。**

例如：

```lip
temperature = sensor.read()

when temperature > 80 {
    alarm()
}
```

程序员只描述：

> `alarm` 的执行依赖于 `temperature > 80`。

至于传感器什么时候变化、什么时候产生新的计算、是否取消旧计算、如何调度，都交给 Runtime。

因此：

> **LIP 不是“响应式编程语言”，而是一种“依赖驱动的编程语言”。**

Reactive 只是 LIP 在状态发生变化时可能表现出来的一种运行性质，而不是 LIP 的核心模型。

---

## 控制流只是结果，而不是起点

传统编程语言通常把：

> **控制流**

当作程序的第一等结构。

LIP 则反过来：

> **控制流只是依赖关系确定以后，Runtime 产生的一种执行结果。**

同一组依赖关系，可以产生：

```text
顺序执行
并行执行
异步执行
增量重新计算
状态变化后的重新传播
```

而程序本身不必因此改变。

这意味着，LIP 不需要分别设计：

```text
parallel
async
await
reactive
workflow
agent
```

这些机制。

它只需要描述：

> **依赖、门控与状态变化。**

然后把执行时序交给 Runtime。

---

## 一个更大的例子

```lip
request = receive_request()
input = parse(request)
valid = validate(input)

when valid {
    user = load_user(input.user_id)
    documents = load_documents(input.user_id)

    chunks = split(documents)
    embeddings = [embed(chunk) for chunk in chunks]

    answer = llm_answer(input.question, embeddings)
    response = make_response(answer)
}
```

程序员没有描述：

* 哪些任务并行
* 什么时候等待
* 如何 join
* 如何展开 Map
* 哪些结果发生变化后需要重新计算

这些都是依赖关系自然产生的。

因此：

> **LIP 的目标不是让程序员更好地控制执行，而是让程序员不必控制那些本来就应该由 Runtime 控制的执行。**

---

## The Core Idea

```text
Traditional Programming

Program
   ↓
Control Flow
   ↓
Execution


LIP

Program
   ↓
Dependencies
   ↓
Runtime
   ↓
Scheduling
   ↓
Execution
```

**Describe the dependencies. Let the Runtime decide the execution.**

> **LIP：描述关系，而不是描述执行。**
