---
weight: 1
title: "异步实现"
---

本章内容包括：

- 异步代码的结构
- 与框架中的 builder 类型进行交互
- 在 async 方法中执行单个步骤
- 理解执行上下文在 `await` 表达式之间的流动
- 与自定义任务类型进行交互

我清楚地记得 2010 年 10 月 28 日那个夜晚。Anders Hejlsberg 正在 PDC 上介绍 `async/await`，就在他的演讲开始前不久，一大批可下载的资料被发布出来，其中包括 C# 规范变更的草案、C# 5 编译器的一个社区技术预览版（CTP），以及 Anders 演讲所用的幻灯片。在某个时刻，我一边在线观看演讲，一边浏览幻灯片，同时在安装 CTP。等 Anders 讲完时，我已经开始编写 async 代码并进行各种尝试了。

在接下来的几周里，我开始拆解这些内容，仔细研究编译器究竟生成了哪些代码，尝试自己写一个与 CTP 自带库类似的、极其简化的实现，并从各个角度对它进行探索。随着新版本的发布，我逐渐弄清楚发生了哪些变化，也越来越熟悉幕后发生的一切。我看到得越多，就越能体会到编译器愿意替我们生成多少样板代码。这就像在显微镜下观察一朵美丽的花：美感依然存在，但你会发现其中蕴含的东西远比第一眼看到的要丰富得多。

当然，并不是每个人都像我这样。如果你只是想依赖我之前描述过的行为，单纯相信编译器会把事情做好，那也完全没问题。或者，你也可以暂时跳过这一章，等以后再回来看；本书后续的内容并不依赖这一章。你几乎不太可能需要把代码调试到本章所讨论的这种层次，但我相信这一章能让你更深入地理解 `async/await` 是如何组合在一起的。在看过生成的代码之后，无论是 *awaitable pattern*，还是自定义任务类型的要求，都会变得更加容易理解。

我不想把这件事说得太玄，但研究这些实现细节确实能在语言和开发者之间建立一种更深层次的联系。

作为一种粗略的近似，我们可以假装 C# 编译器是把使用了 `async/await` 的 C# 代码，转换成不使用 `async/await` 的 C# 代码。当然，编译器实际上是在更低的层面上工作，使用中间表示并最终生成 IL。事实上，在 `async/await` 的某些方面，生成的 IL 根本无法用普通的 C# 表达出来，但这些地方解释起来并不困难。

> **Debug 与 Release 构建的差异，以及未来可能的变化**
>
> 在撰写本章时，我注意到 async 代码在 Debug 构建和 Release 构建之间存在差异：在 Debug 构建中，生成的状态机是类（class），而不是结构体（struct）。（这是为了提供更好的调试体验，尤其是在“编辑并继续”（Edit and Continue）场景中更为灵活。）在我写第三版时并非如此；编译器的实现已经发生了变化，而且未来也可能再次改变。如果你反编译由 C# 8 编译器生成的 async 代码，它的形式可能会与这里展示的内容略有不同。
>
> 虽然这可能让人感到意外，但其实不必过于担心。按定义来说，实现细节本来就可能随时间变化。研究某一具体实现方式所能获得的洞见并不会因此失效。只是需要注意：这类学习不同于“C# 的规则是什么，以及它们只会以明确规定的方式发生变化”。
>
> 在本章中，我展示的是 **Release 构建** 生成的代码。差异主要体现在性能方面，而我认为大多数读者更关心的是 Release 构建下的性能表现，而不是 Debug 构建。

生成的代码有点像洋葱——一层一层的复杂结构。我们将从最外层开始，逐步深入，最终到达最棘手的部分：`await` 表达式，以及 awaiter 与 continuation 之间的“舞蹈”。为了简洁起见，我只展示异步方法，而不涉及 async 匿名函数；两者背后的机制是相同的，重复讲解并不会带来新的收获。

------

# 生成代码的结构

正如我在第 5 章中提到的，实现方式（无论是这种近似描述，还是实际编译器生成的代码）都是以**状态机**的形式存在的。编译器会生成一个私有的嵌套结构体来表示异步方法，同时还必须包含一个与原始方法签名相同的方法。我把这个方法称为**桩方法（stub method）**；它本身内容不多，但它启动了整个流程。

> **注意**
> 在本章中，我经常会提到状态机“暂停”。这对应于 async 方法执行到某个 `await` 表达式，并且被等待的操作尚未完成的情况。正如你在第 5 章中所看到的，此时会调度一个 continuation，在等待的操作完成后继续执行 async 方法的剩余部分，然后 async 方法返回。类似地，我们也会说 async 方法“执行了一步”，指的是两次暂停之间所执行的代码。这些并不是官方术语，但作为简写非常有用。

状态机用于跟踪 async 方法内部当前执行到的位置。从逻辑上看，存在四种状态，按执行顺序排列如下：

- 尚未开始
- 正在执行
- 已暂停
- 已完成（成功或失败）

只有“已暂停”这一组状态取决于 async 方法的结构。方法中的每一个 `await` 表达式都对应一个独立的状态，用于在稍后恢复执行。当状态机正在执行时，并不需要记录当前具体执行的是哪一行代码；那时它只是普通代码，CPU 会像执行同步代码一样跟踪指令指针。只有当状态机需要暂停时，才会记录状态；其根本目的就是允许稍后从中断的位置继续执行。图 6.1 展示了这些状态之间的转换关系。

![image-20260118235911848](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260118235911884.png)

------

为了更具体一些，我们来看一段真实代码。下面的清单展示了一个简单的 async 方法。它并不是最简单的形式，但可以同时演示多个要点。

**清单 6.1 一个简单的入门 async 方法**

```csharp
static async Task PrintAndWait(TimeSpan delay)
{
    Console.WriteLine("Before first delay");
    await Task.Delay(delay);
    Console.WriteLine("Between delays");
    await Task.Delay(delay);
    Console.WriteLine("After second delay");
}
```

在这个阶段，有三个需要注意的点：

- 方法有一个参数，这个参数需要在状态机中使用。
- 方法中包含两个 `await` 表达式。
- 方法返回 `Task`，因此需要返回一个在最后一行执行完成后结束的任务，但没有具体的返回值。

这段代码很适合用作示例，因为它没有循环，也没有 `try/catch/finally` 块需要处理；控制流除了 `await` 之外都很简单。现在让我们看看编译器为这段代码生成了什么。

------

> **不妨自己动手试试**
>
> 我通常会混合使用 `ildasm` 和 Redgate Reflector 来做这类工作，并把反编译器的优化级别设置为 C# 1，以防止反编译器帮我们“还原”出 async 方法本身。市面上还有其他反编译工具，不论你选哪一种，我都建议同时查看 IL。我见过反编译器在处理 `await` 时出现细微 bug，通常体现在执行顺序上。
>
> 如果你不想做这些，也完全没关系；但如果你曾经好奇编译器对某个代码结构到底做了什么，而本章又没有给出答案，那就不妨亲自去看看。只要记住 Debug 和 Release 构建之间的差异即可。另外，不要被编译器生成的名字吓到，它们确实会让代码更难读。

借助这些工具，你可以把清单 6.1 反编译成类似清单 6.2 的代码。C# 编译器生成的许多名称在 C# 中并不是合法标识符；为了让代码可运行，我把它们改写成了合法形式。在其他地方，我也重命名了一些标识符以提高可读性。再往后，我还对状态机中 case 和 label 的顺序做了一些调整；这些调整在逻辑上与生成代码完全等价，但更容易阅读。在某些地方，即便只有两个分支，我也使用了 `switch` 语句，而编译器可能实际使用的是 `if/else`。在这些地方，`switch` 表示的是一种更通用的形式，适用于有多个跳转点的情况；而在简单场景下，编译器可以生成更精简的代码。

**清单 6.2 清单 6.1 的生成代码（不包括 MoveNext 的实现）**

（以下注释保持原意，略）

> 原始方法参数
> 桩方法
> 初始化状态机（包括方法参数）
> 运行状态机直到需要等待
> 返回表示该异步操作的 Task
> 私有状态机结构体
> 状态机的状态（从哪里继续）
> Builder
> 用于在恢复时获取结果的 Awaiter
> 连接 builder 与装箱后的状态机

这段代码看起来已经相当复杂了，但我必须提醒你：真正的大部分工作都发生在 `MoveNext` 方法中，而我目前完全省略了它的实现。清单 6.2 的目的只是搭建舞台、提供整体结构，好让你在看到 `MoveNext` 的实现时能够理解它的意义。接下来，我们将逐个分析这些部分，从桩方法开始。



### 清单 6.2

**清单 6.1 生成的代码（不包括 MoveNext 方法）**

**原始方法参数 / 存根方法**

```csharp
[AsyncStateMachine(typeof(PrintAndWaitStateMachine))]
[DebuggerStepThrough]
private static unsafe Task PrintAndWait(TimeSpan delay)
{
    var machine = new PrintAndWaitStateMachine
    {
        delay = delay,
        builder = AsyncTaskMethodBuilder.Create(),
        state = -1
    };
    machine.builder.Start(ref machine);
    return machine.builder.Task;
}
```

> 初始化状态机（包括方法参数）
> 运行状态机，直到它需要等待
> 返回表示该 async 操作的 Task

------

**用于状态机的私有结构体**

```csharp
[CompilerGenerated]
private struct PrintAndWaitStateMachine : IAsyncStateMachine
{
    public int state;                         // 状态机的状态（从哪里继续）
    public AsyncTaskMethodBuilder builder;    // 构建器
    private TaskAwaiter awaiter;              // awaiter，用于在恢复时获取结果
    public TimeSpan delay;                    // 参数

    void IAsyncStateMachine.MoveNext()
    {
        // 状态机的主要工作逻辑在这里
    }
}
[DebuggerHidden]
void IAsyncStateMachine.SetStateMachine(IAsyncStateMachine stateMachine)
{
    this.builder.SetStateMachine(stateMachine);
}
```

> 将 builder 与装箱后的状态机连接起来

------

这个清单看起来已经相当复杂了，但我要提醒你：**绝大部分工作实际上是在 MoveNext 方法中完成的**，而我目前完全省略了它的实现。

清单 6.2 的目的，是先**搭好舞台、提供结构**，这样当你真正看到 MoveNext 的实现时，它才会显得合理。下面我们将按顺序分析清单中的各个部分，首先从**存根方法（stub method）**开始。

------

## 6.1.1

### 存根方法：准备阶段与迈出第一步

清单 6.2 中的存根方法除了 `AsyncTaskMethodBuilder` 之外都很简单。
`AsyncTaskMethodBuilder` 是一个**值类型**，属于通用的 async 基础设施的一部分。在本章的后续内容中，你会看到状态机是如何与这个 builder 交互的。

```csharp
[AsyncStateMachine(typeof(PrintAndWaitStateMachine))]
[DebuggerStepThrough]
private static unsafe Task PrintAndWait(TimeSpan delay)
{
    var machine = new PrintAndWaitStateMachine
    {
        delay = delay,
        builder = AsyncTaskMethodBuilder.Create(),
        state = -1
    };
    machine.builder.Start(ref machine);
    return machine.builder.Task;
}
```

应用到该方法上的特性（attributes）本质上是**为工具服务的**。
它们不会影响正常的执行流程，你也不需要了解它们的细节就能理解生成的异步代码。

**状态机总是在存根方法中被创建，并携带三类信息：**

- 方法的所有参数（这里仅有 `delay`），每个参数对应状态机中的一个字段
- builder，其类型取决于 async 方法的返回类型
- 初始状态，始终为 `-1`

> **注意**
> `AsyncTaskMethodBuilder` 这个名字可能会让你联想到反射，但它并不是在 IL 层面创建方法之类的东西。
> builder 提供的是生成代码用来传播成功或失败、处理 await 等功能的支持。如果你觉得把它当作一个“helper（辅助器）”更好理解，也完全可以这样想。

创建完状态机之后，存根方法会让状态机的 builder 启动它，并**通过引用**传入状态机本身。

在接下来的几页中，你会看到大量的“按引用传递”，其根本原因是**效率与一致性**。

- 状态机是一个**可变的值类型**
- `AsyncTaskMethodBuilder` 也是一个**可变的值类型**

通过 `ref` 传递 `machine` 给 `Start` 方法，可以避免拷贝状态机，从而：

- 提高效率
- 确保 `Start` 内部对状态的修改在返回后仍然可见

尤其重要的是：**builder 的状态很可能在 Start 过程中发生变化**。
这也是为什么你必须在 `Start` 调用和随后访问 `Task` 时，都使用 `machine.builder`。

假设你尝试这样重构代码：

```csharp
var builder = machine.builder;
builder.Start(ref machine);
return builder.Task;
```

这是一个**无效的重构尝试**。

这样写会导致 `builder.Start()` 内部对 builder 的修改无法反映到 `machine.builder` 中（反之亦然），因为这里操作的是 **builder 的一个拷贝**。

这也解释了为什么 `machine.builder` 必须是一个**字段**而不是属性。
你不希望对状态机中 builder 的副本进行操作，而是要直接操作状态机真正持有的那个值。

这正是你**不应该自己手写这类代码**的原因之一，也是为什么**可变值类型 + 公共字段**几乎总是个坏主意。（你会在第 11 章看到，它们在极其谨慎的设计下也能派上用场。）

------

启动状态机并不会创建新线程。
它只是执行状态机的 `MoveNext()` 方法，直到：

- 状态机需要因为 `await` 而暂停
- 或者整个异步方法执行完成

换句话说：**它只执行了一步**。

无论哪种情况，`MoveNext()` 都会返回，随后 `machine.builder.Start()` 返回，这时你就可以把代表整个异步方法的 `Task` 返回给调用方。

builder 负责：

- 创建这个 Task
- 并在异步方法执行过程中正确地推进 Task 的状态

这就是存根方法。
接下来，我们来看状态机本身。

## 状态机的结构

我依然省略了状态机中绝大部分代码（`MoveNext()` 方法的实现），这里只是再次提醒你它的整体结构：

```csharp
[CompilerGenerated]
private struct PrintAndWaitStateMachine : IAsyncStateMachine
{
    public int state;
    public AsyncTaskMethodBuilder builder;
    private TaskAwaiter awaiter;
    public TimeSpan delay;
}
void IAsyncStateMachine.MoveNext()
{
    // 实现被省略
}
[DebuggerHidden]
void IAsyncStateMachine.SetStateMachine(IAsyncStateMachine stateMachine)
{
    this.builder.SetStateMachine(stateMachine);
}
```

再次强调，特性并不重要。**真正重要的是以下几点：**

- 它实现了 `IAsyncStateMachine` 接口，这是 async 基础设施使用的接口（只有这两个方法）
- 字段：用于在状态机每一步之间保存所需的信息
- `MoveNext()`：
  - 状态机启动时调用一次
  - 每次从暂停状态恢复时调用一次
- `SetStateMachine()`：
  - 在 release 构建中始终具有相同的实现

你其实已经见过一次 `IAsyncStateMachine` 的用法，只不过有点“隐藏”。
`AsyncTaskMethodBuilder.Start()` 是一个泛型方法，它约束类型参数必须实现 `IAsyncStateMachine`。

`Start()` 在做完一些准备工作之后，会调用 `MoveNext()`，让状态机迈出 async 方法的第一步。

------

### 状态机字段的分类

这些字段大致可以分为五类：

- 当前状态（例如：未开始、在某个 await 处暂停等）
- 用于与 async 基础设施通信并提供返回 Task 的 builder
- Awaiter
- 参数和局部变量
- 临时栈变量

#### 状态字段

`state` 是一个整数，可能的取值如下：

- `-1` —— 尚未开始，或正在执行（这两种情况无需区分）
- `-2` —— 已完成（成功或失败）
- 其他值 —— 在某个特定的 await 表达式处暂停

#### Builder 类型

builder 的类型取决于 async 方法的返回类型：

- C# 7 之前：
  - `AsyncVoidMethodBuilder`
  - `AsyncTaskMethodBuilder`
  - `AsyncTaskMethodBuilder<T>`
- C# 7 之后支持自定义 Task 类型：
  - 使用 `AsyncTaskMethodBuilderAttribute` 指定 builder 类型

------

### Awaiter 字段的复用

其他字段更复杂，因为它们取决于 async 方法的具体实现，编译器会尽量**减少字段数量**。

关键原则是：

> **只有在状态机恢复后仍然需要使用的值，才需要字段。**

Awaiter 是字段复用的第一个例子。
一个状态机在任意时刻只能 await 一个值，因此：

- 每种 awaiter 类型只需要一个字段

例如：

- `await Task<int>` 两次
- `await Task<string>` 一次
- `await Task` 三次

最终只会生成三个字段：

- `TaskAwaiter<int>`
- `TaskAwaiter<string>`
- `TaskAwaiter`

> **注意**
> 这里讨论的是由 `await` 表达式生成的 awaiter。
> 如果你手动调用 `GetAwaiter()` 并赋值给局部变量，那它就只是普通局部变量。

------

### 局部变量是否需要字段

如果一个局部变量**只在两个 await 之间使用**，而不是跨 await 使用，那么它可以留在 `MoveNext()` 的栈上，而不需要变成字段。

例如：

```csharp
public async Task LocalVariableDemoAsync()
{
    int x = DateTime.UtcNow.Second;
    int y = DateTime.UtcNow.Second;
    Console.WriteLine(y);
    await Task.Delay();
    Console.WriteLine(x);
}
```

- `x` 在 await 之后仍被使用 → **需要字段**
- `y` 只在 await 之前使用 → **不需要字段**

> **注意**
> 编译器通常能很好地控制字段数量，但并不完美。
> 例如：如果两个变量类型相同、都跨 await 使用，但它们从不同时处于作用域中，理论上可以复用字段——但目前编译器还不会这样做。

------

### 临时栈变量

当 `await` 是更大表达式的一部分时，需要保存中间结果，这就需要**临时栈变量**。

例如：

```csharp
public async Task TemporaryStackDemoAsync()
{
    Task<int> task = Task.FromResult(10);
    DateTime now = DateTime.UtcNow;
    int result = now.Second + now.Hours * await task;
}
```

即使你知道 `Task.FromResult` 返回的是已完成任务，**编译器并不知道**，它必须生成能够暂停和恢复的状态机。

为了遵守 C# 的求值顺序规则：

- `now.Second`
- `now.Hours`

都必须在 `await task` 之前求值，并在恢复后继续使用，因此需要字段。

你可以把它理解为编译器先重写成这样：

```csharp
int tmp1 = now.Second;
int tmp2 = now.Hours;
int result = tmp1 + tmp2 * await task;
```

然后再把这些局部变量转成字段。

与普通局部变量不同的是：
**编译器会复用同类型的临时变量字段，只生成最少数量。