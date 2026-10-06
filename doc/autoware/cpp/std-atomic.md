# ch01.std::atomic::store  std::atomic::load 的所有std::memory_order_参数及其配对规则

<< std::memory_order 全部枚举 + store/load 可用范围 + 配对规则  >>

C++11 一共6种内存序：

```cpp
std::memory_order_relaxed
std::memory_order_consume // C++17 标记弃用，几乎不用
std::memory_order_acquire
std::memory_order_release
std::memory_order_acq_rel
std::memory_order_seq_cst
```

## 一、哪些序能给 store / load 使用

| 内存序   | 可用于 `atomic::store` | 可用于 `atomic::load` | 简要含义 |
| ------- | ---| ---| --- |
| relaxed | ✅ | ✅ | 仅保证原子性，**无同步、无可见性约束**，允许CPU/编译器自由重排 |
| consume | ❌ | ✅ | 依赖链同步；C++17弃用，新项目直接避开 |
| acquire | ❌ | ✅ | **读侧屏障**：load之后的代码不能被挪到load前面 |
| release | ✅ | ❌ | **写侧屏障**：store之前的代码不能被挪到store后面 |
| acq_rel | ✅ | ✅ | 读+写双屏障；**只用于read-modify-write(RMW)操作**（fetch_add, compare_exchange等） |
| seq_cst | ✅ | ✅ | 全局全序，最强；默认值，开销最大 |

> 
> 硬性语法规则：
> 
> 
> 1. `store` **不能**用 acquire / consume / acq_rel（标准禁止）
> 2. `load` **不能**用 release（标准禁止）
> 3. acq_rel 设计给**读改写**，虽然语法上某些平台允许给store/load，但**语义错误，严禁这么写**

## 二、配对规则（核心！）

> 
> 配对的意义：**当load读到store写入的值时，同步生效**；没读到，则同步不生效。

### 1. release ↔ acquire（最常用）

- 写端：`a.store(val, memory_order_release)`
- 读端：`a.load(memory_order_acquire)`
✅ 成功配对：
写线程中，**release store之前所有内存写**，对读到这个值的acquire读线程全部可见。> 
> 经典生产者消费者，就是这个组合。

### 2. release ↔ consume（基本废弃）

- store: release
- load: consume
只保证**依赖该原子变量的那些变量可见**，不保证全部前置写。C++17起弃用，不要在新项目写consume。

### 3. seq_cst ↔ seq_cst

`store(..., seq_cst)` + `load(seq_cst)`
不仅有release/acquire那样的可见性，**所有线程看到所有seq_cst操作有同一个全局顺序**。
代价更高。原子默认就是seq_cst。

### 4. relaxed + relaxed

**没有任何同步配对！**
`store(relaxed)` + `load(relaxed)`
只保证原子读写本身不会撕裂，**不保证其他普通变量的可见性**。
只能用于单纯计数器，不能用来做线程间同步。

### 5. acq_rel（RMW操作专用，如fetch_add）

`fetch_add(1, memory_order_acq_rel)`
等价于：**读的时候acquire，写的时候release**。

- 前面别的线程release store，本线程acq_rel RMW读到它 → 获得release的可见性
- 本线程acq_rel写入，别的线程acquire load读到 → 看到本线程RMW之前所有写

## 三、配对速查表

| Store 内存序 | 可以配对的 Load 内存序 |
| --- | --- |
| release | acquire、consume、seq_cst |
| relaxed | 无有效同步配对（仅原子性） |
| seq_cst | acquire、consume、seq_cst |

> 
> 反向：
> Load(acquire) 可以匹配：release / seq_cst 的store。

## 四、常见错误清单

1. ❌ store(..., acquire)：编译报错
2. ❌ load(..., release)：编译报错
3. ❌ store(release) + load(relaxed)：**不能同步！** 就算读到值，也不保证前面普通变量可见
4. ❌ store(relaxed) + load(acquire)：**不能同步！** acquire拿不到任何发布的数据
5. ❌ 把acq_rel用到单纯store或单纯load：语义错误，不推荐

## 五、极简示例汇总

### release + acquire（推荐）

```cpp
std::atomic<bool> flag{false};
int data;

// 生产者
data = 100;
flag.store(true, std::memory_order_release);

// 消费者
while (!flag.load(std::memory_order_acquire));
// 读到true，data一定是100
```

### relaxed（仅计数，无同步）

```cpp
std::atomic<int> cnt{0};
cnt.store(1, std::memory_order_relaxed);
int v = cnt.load(std::memory_order_relaxed);
// 只保证读写不撕裂，不能用来同步别的变量
```

### seq_cst（默认，全局序）

```cpp
flag.store(true, std::memory_order_seq_cst);
while (!flag.load(std::memory_order_seq_cst));
```

## 补充：RMW操作（fetch_add / cmpxchg）

RMW可以用：relaxed, acquire, release, acq_rel, seq_cst

- acq_rel：读端acquire，写端release
- cmpxchg有两个参数：`expected, desired, success_order, fail_order`> 
> 失败分支**不能使用release/acq_rel**，只能relaxed/acquire/seq_cst。

如果你需要，我可以给一个表格，区分**happens-before / synchronizes-with**这两个术语，这是理解配对背后底层模型的关键。

=============================

# ch02.内存序  只跟原子变量有关吗?

`store(relaxed)` + `load(relaxed)` 只保证原子读写本身不会撕裂，**不保证其他普通变量的可见性**。 请问这些内存序 不是只跟原子变量有关吗?跟其他普通变量有何关系?

## 核心一句话

**内存序约束的不是原子变量本身，约束的是「跨线程，普通内存读写和原子操作之间的重排 & 可见性」**。
原子变量只是**同步的“标记/桥梁”**，不是说内存序只管这个原子。

## 1. CPU/编译器会乱序（两种重排）

重排分两类，都是为了提速：

1. **编译器重排**：编译阶段调整代码指令顺序（只要单线程结果不变）
2. **CPU 乱序执行 + 缓存一致性**：CPU硬件层面，指令执行、写回缓存的顺序可以乱。

> 
> 单线程下，重排完全无害；**多线程才会翻车**。

举你熟悉的例子：

```cpp
int data = 0;
std::atomic<bool> flag{false};

// 线程A
data = 42;                  // 普通变量写
flag.store(true, relaxed);  // 原子store relaxed

// 线程B
while(!flag.load(relaxed)){} // 原子load relaxed
std::cout << data;           // 普通变量读
```

你直觉：`data=42` 写在flag前面，B读到flag=true，`data`肯定是42。
**但 relaxed + relaxed 没有同步关系！**

可能发生两种坏情况：

1. 编译器把 `flag.store` 挪到 `data=42` **前面**（指令重排）
2. CPU先把flag的写入刷到缓存，`data=42`还留在CPU本地缓存，没同步到B的CPU缓存。

线程B看到`flag=true`，但读到`data=0`。
👉 **原子flag本身读写没有撕裂（原子性保证），但是普通变量data的可见性没保证。**

## 2. 各个内存序到底做了什么（大白话）

内存序本质是**给编译器+CPU加限制：哪些跨指令重排不允许做**。

- `release`（store）：**禁止把store前面的读写，重排到store之后**。相当于：在store之前的所有写，要“发布出去”。
- `acquire`（load）：**禁止把load后面的读写，重排到load之前**。相当于：读到这个原子值后，才能去拿前面发布的数据。
- `relaxed`：**不加任何这类重排限制！只保证原子变量本身读写不撕裂**。
编译器、CPU可以自由挪动普通变量读写，跨这个原子操作。

> 
> 原子变量的**原子性**和**内存序（同步/可见性）是两件独立东西**
> 
> 
> - 原子性：保证读/写这个原子变量的时候，不会读到半截值（不会撕裂）。所有memory_order都自带这个。
> - 内存序：控制**原子操作 和 别的普通变量读写**之间的重排、缓存可见性。

## 3. synchronizes-with（同步配对）

当 `store(release)` 写入的值，被另一边 `load(acquire)` 读到：

> 
> 这个store **synchronizes-with** 这个load。
> 一旦建立这个同步关系：
> 写线程在release store**之前**所有内存写入（不管是普通变量还是别的原子），对读线程acquire load**之后**的代码全部可见。

而 `relaxed + relaxed`：**不存在 synchronizes-with 关系**。
哪怕load读到了store写进去的值，也不会建立跨线程的可见性传递。
原子本身没问题，但普通变量的状态不会跟着同步。

## 4. 一个常见误区纠正

> 
> “memory_order是原子变量的属性，所以只管原子变量”
> ❌ 错。
> `memory_order` 是**施加在本次原子操作上的内存屏障语义**，它的作用域是**所有内存访问**，只是屏障由原子操作触发。
> 原子操作是“触发屏障的载体”，屏障影响全部内存（普通变量+其他原子）。

## 5. 什么时候可以放心用relaxed？

只有一种场景：**只用这个原子变量本身，不拿它去同步任何其他变量**。
比如单纯计数器：

```
std::atomic<int> cnt{0};
// 多个线程只读写cnt，没有其他变量依赖cnt的值
cnt.fetch_add(1, std::memory_order_relaxed);
```

这里没有别的普通变量要同步，relaxed完全没问题。

---

如果你想，我可以画一个happens-before的关系图，对比：
release/acquire 版本 vs relaxed/relaxed版本，看两者happens-before链条差别。

=======================

# ch03.memory_order内存屏障语义 的作用域
 memory_order 是施加在本次原子操作上的内存屏障语义，它的作用域是"所有内存访问 ",请问这个所有内存访问的作用域是什么,是运行到这个时刻为止的所有内存操作吗,包括赋值等?

## 一句话先回答

**是的，包含普通变量赋值（读写），也包含其它原子变量读写；但不是“全局所有内存操作永久冻结”，而是约束【原子操作】和【其它内存访问】之间能不能互相重排。**

> 
> 重点：不是“执行到这里，把全部内存立刻刷到全局”（很多人误以为内存屏障是刷缓存）。
> C++内存模型是**抽象模型（happens-before）**，不是CPU硬件缓存刷新指令。硬件屏障只是CPU上的实现手段。

## 1. 作用范围：哪些内存访问会被约束？

所有**在同一个线程内**，发生在这个原子操作**前面/后面**的内存访问：

- ✅ 普通变量读写：`int a=1; a=10;` 这类赋值、读变量
- ✅ 其它`std::atomic<T>`的load/store/RMW
- ✅ 数组、结构体成员读写

**只约束【本线程】的指令重排；不直接约束另一个线程的指令。**

举 `store(..., release)`：

```
// 同一个线程内
x = 1;        //普通写
y.store(2);   //别的原子
flag.store(true, memory_order_release); // release原子store
```

`release` 的规则：

> 
> **本线程中，所有在这个store之前的读写（x、y），不能被编译器/CPU重排到这个store之后。**
> 但是：store**之后**的代码，可以随便往前挪，不受release限制。

> 
> ⚠️ 不是说“执行到store这一行，立刻把x,y刷到全局内存”。
> 只是**禁止重排**。只有当另一个线程用`acquire load`读到这个store写入的值，才触发跨线程可见性传递。

## 2. 不是“运行到此刻为止全部内存”一次性全局提交

有一个极易踩坑的误区：

> 
> ❌ 错误理解：执行release store的时候，把当前线程所有已经写过的内存全部同步到别的CPU。

✅ 标准模型的真实含义：

1. 只限制**本线程内指令重排顺序**；
2. 只有当另一个线程`acquire load`**成功读到本次release写入的值**，才建立`synchronizes-with`；
3. 一旦建立同步：
 写线程中，**happens-before这个release store的所有写操作**，对读线程中`acquire load`之后的代码可见。

> 
> 注意：happens-before是**代码序上的先行关系**，不是物理时间先后。

## 3. 对比 acquire 帮你对照理解

`load(..., acquire)`

```cpp
// 线程B
while(!flag.load(memory_order_acquire)){} //acquire load
auto v1 = a;
auto v2 = b;
```

acquire约束：

> 
> **本线程中，load后面的内存访问(a,b的读取)，不能重排到load的前面。**
> 前面的代码可以随便往后挪。

当读到release写入的值：
写线程release之前所有写，对本线程acquire之后的内存访问可见。

## 4. relaxed为什么没有这个效果？

`relaxed` **没有任何重排限制**。
同一个线程里，普通变量读写可以自由挪到原子store前后，编译器怎么优化都行。
仅仅保证**这个原子变量本身读写不会撕裂**。没有`synchronizes-with`。

## 5. 边界：哪些东西不在这个内存序约束里？

1. **其它线程内部的指令重排**：内存序不会阻止另一个线程自己内部的重排；只能建立跨线程同步链。
2. 不影响**不参与happens-before链条**的内存访问。
3. 函数调用、系统调用这些，不是内存访问，不受内存序约束（当然函数内部的内存访问依然遵守）。

## 极简总结

1. memory_order施加在原子操作上，**约束的是【同一个线程】内，普通变量/其它原子变量的读写 和 当前原子操作之间的重排**；包含赋值。
2. release：本线程，**原子store之前的内存访问不能后移**。
3. acquire：本线程，**原子load之后的内存访问不能前移**。
4. 它不是“瞬间刷新全局内存”；跨线程可见性，**只有release+acquire配对读到值时才生效**。

如果你想，我可以画一个happens-before链条对比：release/acquire 和 relaxed两个版本，直观看到普通变量什么时候能保证可见。

===========================

# ch04. 是否可以用原子操作及其内存序 完全代替各种mutex?

## 一句话结论

**不能完全替代 mutex，只能在特定场景替代；绝大多数复杂同步场景，mutex 依然更合适。**

先分清两者本质差异：

- `std::atomic` + 内存序：**底层内存同步原语**，操作粒度是单个原子变量，只做「原子读写 + 内存可见性/重排约束」，**没有临界区所有权，不自动保护一组多个变量**。
- `std::mutex`：**临界区锁**，一旦 lock()，可以保护**任意多条代码、任意多个变量**，保证：互斥（同一时间只有一个线程进入临界区）+ 内存同步。

## 一、什么时候可以用 atomic 代替 mutex？

条件：**只需要修改/读取单个变量，不需要同时保护一堆变量，逻辑简单**。
典型例子：计数器、状态标记（bool flag）、版本号。

```
// 多线程计数，用 atomic relaxed 替代锁
std::atomic<int> cnt{0};
cnt.fetch_add(1, std::memory_order_relaxed);
```

```
// 生产者消费者单flag，release/acquire替代锁
std::atomic<bool> ready{false};
```

这类场景，atomic 性能通常优于 mutex（mutex会进入内核，有上下文切换开销）。

## 二、什么时候绝对不能用 atomic 替代 mutex？（重点）

只要满足下面任意一条，**别硬上 atomic**：

### 1. 需要同时保护**多个独立变量**

```
int a, b;
// 要保证 a 和 b 一起修改，作为一个整体原子更新
```

atomic 只能保证**单个原子变量**原子操作。
你没法用两个独立 `std::atomic<int>` 做到「a、b同时修改，别的线程要么看到旧的a+b，要么看到新的a+b」。

> 
> 两个独立原子操作，中间线程可以插队，会读到半更新状态。
> 这种场景：要么 mutex，要么打包成一个结构体用 `std::atomic<struct>`（仅当结构体大小支持硬件原子指令，限制很大）。

### 2. 临界区内有复杂逻辑、分支、函数调用

```
// 临界区：
if (a > 10) {
    b += calc(a);
    log(a,b);
}
```

mutex 可以把整块代码包起来，保证整块逻辑原子执行。
atomic 只能保护单条原子指令，**无法把一长段业务代码变成原子操作**。

### 3. 需要条件等待（等待某个条件成立）

mutex 搭配 `std::condition_variable` 非常方便：线程等待，条件满足唤醒。
单纯原子变量只能写自旋循环（busy-wait），CPU空转，浪费资源。

> 
> 当然可以用 `std::atomic::wait`（C++20）做等待，但它也只是等待原子值变化，依然不能保护多变量临界区。

### 4. 需要递归锁、独占/共享读写锁（rwlock）、所有权

mutex 有所有权概念：谁lock，谁必须unlock。
atomic 没有所有权，没有“持有锁”这个概念。

## 三、容易混淆的关键点：原子性 ≠ 临界区

`memory_order` 只管**内存可见性与指令重排**，**不提供互斥（mutual exclusion）**。

举个反面例子：

```
std::atomic<int> x{0};
// 线程1
x.store(x.load(std::memory_order_acquire)+1, std::memory_order_release);
// 线程2
x.store(x.load(std::memory_order_acquire)+1, std::memory_order_release);
```

❌ **有竞态！会丢更新！**
load 和 store 是**两次独立原子操作**，中间线程可以插队。
想要“读改写原子”，必须用RMW：`fetch_add` / `compare_exchange`。

> 
> 这就是：**单条RMW原子操作才具备原子的读改写；分开load+store就算配上acquire/release，也不是原子复合操作。**

## 四、对比总结表

| 特性 | std::atomic + memory_order | std::mutex |
| --- | --- | --- |
| 保护对象 | 单个原子变量 | 任意代码、任意数量变量 |
| 互斥临界区 | 仅RMW单条指令具备 | 任意代码块 |
| 性能 | 轻量，用户态，无内核切换（大部分平台） | 较重，竞争强时陷入内核 |
| 等待 | C++20有atomic::wait，只能等原子值 | cv + mutex，支持复杂条件等待 |
| 适用逻辑 | 简单标记、计数器、版本号 | 复杂业务、多变量、长临界区 |

## 五、工程上的经验准则

1. **优先 mutex**：逻辑复杂、多个变量、临界区代码长 → mutex，写起来简单，不容易踩内存序、ABA、竞态bug。atomic 极其容易写出隐藏bug。
2. **用atomic**：仅仅单个变量读写/RMW，追求低延迟，且你完全吃透内存模型、ABA问题。
3. **不要强行用atomic替换mutex**：很多人踩坑，为了“性能”把mutex改成atomic，最后引入极难复现的内存模型bug，调试成本极高。

## 补充小例子：ABA问题

就算你用CAS（compare_exchange，原子RMW），也会遇到ABA，这是atomic独有的坑，mutex天然不存在这个问题。

如果你想，我可以写一段代码演示：**分开load+store配上release/acquire，依然会数据丢失**。


====================================
# ch05.memory_order_release 用于数据保护

```cpp
int data1 = 0;
int data2= 0;
std::atomic<bool> flag{false};
// 线程A
data1 = 41; // 普通变量写
data2 = 42;                  // 普通变量写
flag.store(true, memory_order_release);  // 原子store release
// 线程B
while(!flag.load(memory_order_acquire)){} // 原子load acquire
std::cout << "data1="<<data1<<",data2="<<data2;           // 普通变量读
```

## 结论先说
✅ **这段代码是安全的**。
线程B一旦`load(acquire)`读到`true`，**一定能看到 `data1=41`、`data2=42`**，不会读到0。

## 原理拆解
1. 线程A：
```cpp
data1 = 41;
data2 = 42;
flag.store(true, std::memory_order_release);
```
`release` 的约束：**本线程内，store之前所有内存访问不能重排到store之后**。
编译器/CPU 绝对不能把 `data1=41` / `data2=42` 挪到 `flag.store` 的后面。

2. 线程B：
```cpp
while(!flag.load(std::memory_order_acquire)){}
cout << data1 << data2;
```
`acquire` 的约束：**load之后的内存访问（读data1、data2）不能重排到load前面**。

3. 同步关系：
当B的`acquire load`读到A `release store`写入的`true`：
> A的`store` synchronizes-with B的`load`
于是，**A中所有 happens-before 这个release store的写操作，对B中acquire load之后的代码全部可见**。
`data1`、`data2`的写入就在这个集合里面。

## 但是！有一个巨大限制
> 这个同步**只保护「读到flag=true」这一次**。
> 如果线程A后续再次修改`data1/data2`，**没有再次通过flag做release发布，B是看不到新修改的！**

举个错误扩展版本（会出问题）：
```cpp
// 线程A
data1 = 41;
data2 = 42;
flag.store(true, std::memory_order_release);

// 一段时间之后，A直接修改data1/data2，不再碰flag！
data1 = 100;
data2 = 200;  // ❌ 没有release同步！

// 线程B此时flag已经是true，不会再进while循环，直接读data1/data2
// B不一定能看到100,200，可能读到旧值
```
只有**再次执行一次release store**，才能把新的data1/data2发布出去。

## 再对比：如果改成 relaxed + relaxed
```cpp
flag.store(true, std::memory_order_relaxed);
while(!flag.load(std::memory_order_relaxed)){}
```
此时就算B读到flag=true，**不保证data1/data2是41/42**，有可能读到0。
因为没有`synchronizes-with`关系，release/acquire的跨线程可见性传递不存在。

## 工程上的注意点
这个模式就是经典**单标志生产者消费者**，适合**一次性发布一组数据**。
⚠️ 它**不是锁**：
- 不能多线程并发修改data1/data2；
- 只能生产者一次性写完data1/data2，再发布flag；
- 一旦flag置为true，生产者不能再随意修改data1/data2，否则会数据竞争UB。

> 一旦flag=true之后，A再写data1/data2，B同时读data1/data2：这就是**数据竞争，未定义行为UB**！
> 这是很多人踩坑的地方。

如果你想，我可以写一个坏例子，展示flag置true之后生产者继续修改data，触发UB。

============================================

# ch06.坏例子：flag=true 之后生产者继续修改 data，产生**数据竞争（Data Race，UB未定义行为）**

```cpp
#include <atomic>
#include <thread>
#include <iostream>

int data1 = 0;
int data2 = 0;
std::atomic<bool> flag{false};

void producer()
{
    // 第一轮：正常发布
    data1 = 41;
    data2 = 42;
    flag.store(true, std::memory_order_release);

    // ========== 错误！flag已经true，继续修改data1/data2 ==========
    // 此时消费者线程可能正在并行读 data1/data2
    data1 = 100;
    data2 = 200;
}

void consumer()
{
    while (!flag.load(std::memory_order_acquire))
    {
        // 等待
    }
    // 读到flag=true，直接读取data1、data2
    std::cout << "data1=" << data1 << ", data2=" << data2 << "\n";
}

int main()
{
    std::thread t_prod(producer);
    std::thread t_cons(consumer);
    t_prod.join();
    t_cons.join();
    return 0;
}
```

## 问题拆解

1. `flag.store(release)` 只**同步本次发布之前**的写操作（41,42）。
2. `data1=100; data2=200;` 在 release store **之后**，没有任何同步保护。
3. 两个线程：
   - 生产者：**写 data1/data2**
   - 消费者：**同时读 data1/data2**> 
   > 同一个普通变量，一个线程写、另一个线程无同步地读 → **数据竞争，C++标准直接判定为UB（未定义行为）**

UB意味着什么：

- 可能输出 `41,42`
- 可能输出 `100,200`
- 可能读到撕裂值：`data1=100，data2=42`（半更新状态）
- 编译器优化乱序、程序崩溃，一切都有可能，**行为不可预测**。

## 怎么修复？

### 方案1：用版本号（双buffer思想，推荐）

不要原地修改旧数据，准备新数据，用**新的原子flag/版本号发布**。

```cpp
#include <atomic>
#include <thread>
#include <iostream>

int data1 = 0;
int data2 = 0;
std::atomic<int> version{0};

void producer()
{
    // 第1版
    data1 = 41;
    data2 = 42;
    version.store(1, std::memory_order_release);

    // 第2版：写新值，再更新版本号发布
    data1 = 100;
    data2 = 200;
    version.store(2, std::memory_order_release);
}

void consumer()
{
    int v;
    do {
        v = version.load(std::memory_order_acquire);
    } while (v == 0);
    std::cout << "version=" << v << " data1=" << data1 << ", data2=" << data2 << "\n";
}
```

> 
> 规则：**写完一组data，再更新version做release发布；消费者读到新版本，才去读取data**。
> 但是⚠️ 这个版本依然有坑：如果生产者连续快速覆盖多轮，消费者有可能跳过某些版本。

### 方案2：std::mutex（最简单稳妥）

如果需要频繁读写、多轮更新，直接mutex保护data1/data2，避免手动内存模型踩坑。

## 关键总结

`release/acquire` 只能保证**发布那一刻快照**。
一旦flag置true：
✅ 可以：只读data，不再修改data
❌ 禁止：生产者继续原地写data，消费者并行读data（UB）

如果你想，我可以再写一个双buffer无锁版本，解决多次更新的场景，避免原地修改的UB。