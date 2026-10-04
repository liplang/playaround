# 一、第一组：验证 LIP 最基本的“数据依赖”

### 1. Hello Data

最简单的：

```lip
name = "World"
message = "Hello " + name
print(message)
```

验证：

* assignment
* dependency
* expression
* 基本 Go 类型
* 自动执行顺序

核心问题：

> `message` 到底是不是一个 node？

---

### 2. 多级依赖

```lip
a = 10
b = a * 2
c = b + 3
d = c * c
print(d)
```

验证：

```text
a → b → c → d
```

看看是否真的可以把“顺序”从代码的控制流程变成依赖关系。

---

### 3. 两条独立计算

```lip
a = foo()
b = bar()

c = a + b
```

理想：

```text
    foo ──→ a ──┐
                ├→ c
    bar ──→ b ──┘
```

验证：

> **LIP 是否天然知道 `foo()` 和 `bar()` 可以并行？**

---

### 4. 条件依赖

```lip
x = get_value()

if x > 10 {
    y = foo(x)
} else {
    y = bar(x)
}
```

这里非常重要。

到底形成：

```text
x → condition → foo/bar → y
```

还是普通 Go 式控制流？

这是第一次真正逼问：

> **if 在 LIP 中到底是什么？**

---

### 5. 选择而不是执行

```lip
x = get_value()

y = if x > 10 then 100 else 0
```

和 #4 对比。

这两个应该明确区分：

```text
conditional computation
conditional value
```

这会帮助确定 `if` 的语义。

---

# 二、第二组：真正测试 Flow

### 6. Fan-out

```lip
page = fetch(url)

title = extract_title(page)
text  = extract_text(page)
links = extract_links(page)
```

这是一个非常核心的例子。

```text
              ┌→ title
page ─────────┼→ text
              └→ links
```

验证：

> 一个值能不能自然地产生多个并行消费者？

---

### 7. Fan-in

```lip
a = fetch_a()
b = fetch_b()
c = fetch_c()

result = combine(a, b, c)
```

验证 Join。

---

### 8. Map / 批量并行

```lip
urls = get_urls()

pages = [fetch(url) for url in urls]
```

这里会第一次逼问：

> `[f(x) for x in xs]` 到底是什么？

它可能意味着：

* 普通 Go loop
* Flow map
* 自动并行 map
* future 集合

这个例子非常关键。

---

### 9. Map + Reduce

```lip
numbers = get_numbers()

squares = [x * x for x in numbers]

total = sum(squares)
```

对应：

```text
map → reduce
```

这是 Dataflow 非常经典的结构。

---

### 10. 动态数量的 Flow

```lip
urls = discover_urls()

pages = [fetch(url) for url in urls]
```

这里 `urls` 运行时才知道。

这会逼问：

> Dependency Graph 是不是必须在编译期完整确定？

答案很可能是：

**不需要。**

这对 LIP 的架构非常重要。

---

# 三、第三组：Pending / Logical Time

### 11. 网络请求

```lip
page = fetch(url)
text = extract(page)
```

`fetch` 没完成：

```text
page = Pending
text = Pending
```

完成后：

```text
page = Ready
→ text 自动执行
```

这是 LIP 的核心场景之一。

---

### 12. 两个异步请求

```lip
user = fetch_user(id)
posts = fetch_posts(id)

profile = make_profile(user, posts)
```

理想：

```text
fetch_user ──┐
             ├→ profile
fetch_posts ─┘
```

这里可以验证：

* Pending
* Join
* 并行
* tick

---

### 13. 请求超时

```lip
data = fetch(url, timeout=5)
```

逼问：

> timeout 是 Go 函数参数，还是 Flow/runtime 概念？

我倾向前者。

---

### 14. Retry

```lip
data = retry(fetch(url), 3)
```

或者：

```lip
data = fetch(url) retry 3
```

这会逼问：

> retry 是普通函数，还是 Feedback？

这是非常有意思的语言设计实验。

---

### 15. 外部状态变化

例如：

```lip
temperature = sensor.read()
```

然后：

```lip
if temperature > 80 {
    alarm()
}
```

问题：

> sensor.read() 是一次性 computation，还是一个持续存在的 Flow？

这里就开始触及 **state / event / stream**。

---

# 四、第四组：Feedback / Loop

### 16. 普通循环

```lip
sum = 0

for x in numbers {
    sum = sum + x
}
```

问题：

> 这个东西在 LIP 里是否仍然应该写成普通 Go？

我其实倾向：

**是。**

因为这是一个非常好的测试：

> LIP 不应该为了“纯 Dataflow”而把普通算法写得奇怪。

---

### 17. Feedback

真正的数据流反馈：

```text
state → computation → new_state → computation
```

例如：

```lip
x = initial()

while condition(x) {
    x = step(x)
}
```

这里才真正测试：

> Feedback 是不是 LIP 的一等语义？

---

### 18. Agent retry loop

```text
request
   ↓
plan
   ↓
tool
   ↓
result
   ↓
evaluate
   │
   ├── good → answer
   │
   └── bad → plan again
```

这应该是 LIP 非常有代表性的场景。

---

# 五、第五组：真实 Web 后端

### 19. HTTP handler

```text
request
 ↓
parse
 ↓
validate
 ↓
load user
 ↓
load data
 ↓
construct response
```

这个例子非常重要，因为它能回答：

> LIP 到底是不是适合普通 Web 后端？

---

### 20. 并行数据库查询

```text
request
    │
    ├→ query_user
    ├→ query_orders
    └→ query_recommendations
             │
             ↓
          response
```

这是一个 LIP 应该非常舒服的场景。

---

### 21. Cache

```lip
user = cache.get(id)

if user == missing {
    user = db.get(id)
    cache.put(id, user)
}
```

这里会逼问：

> effect / mutation 到底如何处理？

这可能直接影响 LIP runtime 设计。

---

### 22. Pipeline

例如文件处理：

```text
file
 ↓
parse
 ↓
validate
 ↓
transform
 ↓
save
```

这是最简单的 Pipeline。

但特别重要的是拿它和真正的 Dataflow 比较。

---

# 六、第六组：科学计算

### 23. Monte Carlo

```lip
samples = [simulate(i) for i in range(N)]
result = average(samples)
```

这是测试：

> LIP 自动并行是否真的有意义？

---

### 24. 科学计算 DAG

```text
A ─┐
   ├→ B ─┐
C ─┘     │
         ├→ D
E ───────┘
```

比如：

```lip
a = load_data()
b = preprocess_a(a)
c = preprocess_b(a)

d = model_1(b)
e = model_2(c)

result = combine(d, e)
```

这是非常典型的 DAG。

---

### 25. 增量计算

```lip
result = expensive(a, b, c)
```

然后：

```text
a 不变
b 不变
c 改变
```

理想：

```text
只重新计算 result
```

而不是全部重跑。

这会验证：

> **Dependency Graph 是否真的比普通程序结构产生额外价值。**

---

# 七、第七组：AI Agent

### 26. 最简单 Agent

```text
request
 ↓
LLM
 ↓
answer
```

---

### 27. Tool Agent

```text
request
 ↓
plan
 ↓
tool selection
 ↓
tool call
 ↓
result
 ↓
answer
```

---

### 28. 并行 Tool Calling

```text
             ┌→ weather
request → plan├→ search
             └→ database
                    │
                    ↓
                  answer
```

这应该是 LIP 的杀手级测试之一。

---

### 29. Agent with verification

```text
request
 ↓
answer
 ↓
verify
 │
 ├── good → output
 │
 └── bad → revise
              │
              └────→ verify
```

这里同时测试：

* Flow
* Feedback
* Pending
* LLM
* state
* retry

---

### 30. 完整小型 Agent Workflow

最后做一个稍微真实一点的：

```text
用户问题
   ↓
理解问题
   ↓
┌───────────────┐
│ search web    │
│ query database│
│ call tools    │
└───────────────┘
   ↓
merge
   ↓
reason
   ↓
verify
   │
   ├── fail → retry
   │
   └── pass
         ↓
       answer
```

这个例子如果能够用**非常短、非常自然的 LIP**表达出来，我认为就很有说服力了。
