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

 

**原始方法参数 / 存根方法**

```csharp
[AsyncStateMachine(typeof(PrintAndWaitStateMachine))]
[DebuggerStepThrough]
private static unsafe Task PrintAndWait(TimeSpan delay)
{
    var machine = new PrintAndWaitStateMachine  //初始化状态机（包括方法参数）
    {
        delay = delay,
        builder = AsyncTaskMethodBuilder.Create(),
        state = -1
    };
    machine.builder.Start(ref machine); //运行状态机，直到它需要等待
    return machine.builder.Task; //返回表示该 async 操作的 Task
}
```



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
    this.builder.SetStateMachine(stateMachine);  //将 builder 与装箱后的状态机连接起来
}
```



这个清单看起来已经相当复杂了，但我要提醒你：**绝大部分工作实际上是在 MoveNext 方法中完成的**，而我目前完全省略了它的实现。

清单 6.2 的目的，是先**搭好舞台、提供结构**，这样当你真正看到 MoveNext 的实现时，它才会显得合理。下面我们将按顺序分析清单中的各个部分，首先从**存根方法（stub method）**开始。

## 存根方法：准备阶段与迈出第一步

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

**状态机字段的分类**

这些字段大致可以分为五类：

- 当前状态（例如：未开始、在某个 await 处暂停等）
- 用于与 async 基础设施通信并提供返回 Task 的 builder
- Awaiter
- 参数和局部变量
- 临时栈变量

**状态字段**

`state` 是一个整数，可能的取值如下：

- `-1` —— 尚未开始，或正在执行（这两种情况无需区分）
- `-2` —— 已完成（成功或失败）
- 其他值 —— 在某个特定的 await 表达式处暂停

**Builder 类型**

builder 的类型取决于 async 方法的返回类型：

- C# 7 之前：
  - `AsyncVoidMethodBuilder`
  - `AsyncTaskMethodBuilder`
  - `AsyncTaskMethodBuilder<T>`
- C# 7 之后支持自定义 Task 类型：
  - 使用 `AsyncTaskMethodBuilderAttribute` 指定 builder 类型

------

**Awaiter 字段的复用**

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

**局部变量是否需要字段**

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

**临时栈变量**

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
编译器会复用同类型的临时变量字段，只生成最少数量。

## **`MoveNext()` 方法（高层视角）**

我现在还不会展示清单 6.1 中 `MoveNext()` 方法反编译后的代码，因为它很长且令人畏惧。在你了解了流程的样子后，它会更容易掌握，所以我在这里抽象地描述一下。

每次调用 `MoveNext()`，状态机就前进一步。每次它到达一个 `await` 表达式时，如果等待的值已经完成，它会继续执行；否则会暂停。在以下任何一种情况发生时，`MoveNext()` 会返回：

- 状态机需要暂停以等待一个未完成的值。
- 执行到达方法的末尾或一个 `return` 语句。
- 抛出一个异常且未在异步方法中被捕获。

注意，在最后一种情况下，`MoveNext()` 方法并不会以抛出异常而结束。相反，与异步调用关联的任务会变为故障状态。（如果这让你感到惊讶，请参阅 5.6.5 节以回顾异步方法关于异常的行为。）

图 6.2 展示了一个关注 `MoveNext()` 方法的异步方法的通用流程图。图中没有包含异常处理，因为流程图没有表示 `try/catch` 块的方式。当你最终看到代码时，会看到它是如何管理的。同样，我也没有展示 `SetStateMachine` 在哪里被调用，因为流程图本身已经足够复杂了。



![image-20260119224033013](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260119224033090.png)

关于 `MoveNext()` 方法的最后一点：它的返回类型是 `void`，而不是任务类型。只有桩方法需要返回任务，这个任务是在构建器的 `Start()` 方法调用 `MoveNext()` 执行第一步之后从状态机的构建器中获取的。所有其他对 `MoveNext()` 的调用都是从暂停状态恢复状态机的基础设施的一部分，这些调用不需要关联的任务。你将在 6.2 节中看到所有这一切在代码中的样子（现在不用等很久了），但首先，简单提一下 `SetStateMachine`。

> 状态机模式是连接人类线性思维与计算机非线性执行的经典桥梁。编译器将我们编写的顺序异步代码拆解成基于状态的 `MoveNext()` 调用，本质上是将“时间”维度显式编码为“状态”维度，从而在单线程（或可控线程切换）的约束下模拟出并发执行的假象。这启示我们：面对复杂控制流问题时，将其转化为状态机模型往往是降维打击的有效手段。

 

## **SetStateMachine 方法与状态机的装箱之舞**

我已经展示过 `SetStateMachine` 的实现。它很简单：
```csharp
void IAsyncStateMachine.SetStateMachine(IAsyncStateMachine stateMachine)
{
    this.builder.SetStateMachine(stateMachine);
}
```
在发布版本中，实现总是这样。（在调试版本中，状态机是一个类，其实现是空的。）这个方法的目的在高层面上很容易解释，但其细节却很繁琐。当状态机执行第一步时，它作为桩方法的一个局部变量存在于栈上。如果它暂停了，它必须将自己装箱（放到堆上），以便在恢复时所有信息仍然保留。

装箱之后，会使用装箱后的值作为参数，在这个装箱后的值上调用 `SetStateMachine`。换句话说，在基础设施的核心深处，有类似于这样的代码：
```csharp
void BoxAndRemember<TStateMachine>(ref TStateMachine stateMachine)
    where TStateMachine : IStateMachine
{
    IStateMachine boxed = stateMachine; // 装箱发生在这里
    boxed.SetStateMachine(boxed);
}
```
实际情况并非完全如此简单，但这传达了正在发生的事情的本质。`SetStateMachine` 的实现然后确保 `AsyncTaskMethodBuilder` 持有对它所关联的状态机的单一装箱版本的一个引用。这个方法必须在装箱后的值上调用；它只能在装箱之后调用，因为那时你才拥有对装箱后值的引用，并且如果你在装箱之后在未装箱的值上调用它，那不会影响装箱后的值。（记住，`AsyncTaskMethodBuilder` 本身也是一个值类型。）这个复杂的“舞蹈”确保了当向等待器传递一个延续委托时，该延续将在同一个装箱实例上调用 `MoveNext()`。

这样做的结果是，状态机如果不需要装箱就完全不会装箱，如果需要则只装箱一次。装箱之后，所有事情都发生在装箱版本上。这一切复杂的代码都是为了效率。

我发现这段小小的“舞蹈”是整个异步机制中最有趣也最奇特的部分之一。它听起来似乎完全没有意义，但由于装箱的工作方式，它是必要的，而装箱对于在状态机暂停时保存信息也是必要的。

不完全理解这段代码是完全没问题的。如果你发现自己需要在底层调试异步代码，可以再回来看这一节。对于所有其他意图和目的，这段代码更像是一个新奇的东西，而非必须掌握的内容。

以上就是状态机的组成部分。本章剩余的大部分内容将专门讨论 `MoveNext()` 方法以及它在各种情况下的操作。我们将从简单的情况开始，并逐步深入。

---

 

> 状态机的装箱机制揭示了一个深刻的工程权衡：如何在不牺牲性能的前提下，让栈上的轻量状态机在需要暂停时获得堆上的“永生”能力。编译器通过精妙的“装箱之舞”，在首次暂停时进行一次精确的堆分配，并确保后续所有操作都指向这个唯一的堆对象。这就像是为一个临时的舞台演员（栈状态机）在关键时刻制造了一个永久性的全息投影（堆状态机），所有后续演出都通过这个投影进行，从而保证了状态的持续性和一致性。这种设计体现了系统编程中“零成本抽象”的追求：常见情况（无需暂停）下零开销，特殊情况（需要暂停）下仅支付必要成本。
>

 

> `SetStateMachine` 方法是状态机从栈存活到堆的关键环节。其核心流程是：当异步方法首次因等待未完成操作而暂停时，状态机（值类型）会被装箱到堆上，然后立即在这个堆实例上调用 `SetStateMachine` 方法，将自身引用传递给内部的 `AsyncTaskMethodBuilder`。这套“自己设置自己”的看似怪异的流程，本质是为了让构建器能够持有一个稳定的堆引用，从而确保后续的延续回调能正确恢复到同一个状态机实例。这是编译器为了平衡效率（避免不必要的堆分配）和正确性（暂停时必须持久化状态）而设计的精巧机制。



 

# 简单的 `MoveNext()` 实现

我们将从你在清单 6.1 中看到的简单异步方法开始。它简单并不是因为它短（尽管这有帮助），而是因为它不包含任何循环、`try` 语句或 `using` 语句。它的控制流简单，因此产生一个相对简单的状态机。让我们开始吧。

## 一个完整的具体例子

我将先向你展示完整的方法。不要指望一下子完全理解它，但请花几分钟浏览一下。手头有了这个具体例子，更容易理解更通用的结构，因为你随时可以回头看看该结构的每个部分在这个例子中是如何呈现的。冒着让你觉得乏味的风险，再次展示清单 6.1，作为编译器输入内容的提醒：

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
下面的清单是反编译代码的一个版本，为了可读性进行了略微重写。（是的，这是“易读”版本。）

**清单 6.3 清单 6.1 的反编译 `MoveNext()` 方法**
```csharp
void IAsyncStateMachine.MoveNext()
{
    int num = this.state;
    try
    {
        TaskAwaiter awaiter1;
        switch (num)
        {
            default:
                goto MethodStart;
            case 0:
                goto FirstAwaitContinuation;
            case 1:
                goto SecondAwaitContinuation;
        }
MethodStart:
        Console.WriteLine("Before first delay");
        awaiter1 = Task.Delay(this.delay).GetAwaiter();
        if (awaiter1.IsCompleted)
        {
            goto GetFirstAwaitResult;
        }
        this.state = num = 0;
        this.awaiter = awaiter1;
        this.builder.AwaitUnsafeOnCompleted(ref awaiter1, ref this);
        return;
FirstAwaitContinuation:
        awaiter1 = this.awaiter;
        this.awaiter = default(TaskAwaiter);
        this.state = num = -1;
GetFirstAwaitResult:
        awaiter1.GetResult();
        Console.WriteLine("Between delays");
        TaskAwaiter awaiter2 = Task.Delay(this.delay).GetAwaiter();
        if (awaiter2.IsCompleted)
        {
            goto GetSecondAwaitResult;
        }
        this.state = num = 1;
        this.awaiter = awaiter2;
        this.builder.AwaitUnsafeOnCompleted(ref awaiter2, ref this);
        return;
SecondAwaitContinuation:
        awaiter2 = this.awaiter;
        this.awaiter = default(TaskAwaiter);
        this.state = num = -1;
GetSecondAwaitResult:
        awaiter2.GetResult();
        Console.WriteLine("After second delay");
    }
    catch (Exception exception)
    {
        this.state = -2;
        this.builder.SetException(exception);
        return;
    }
    this.state = -2;
    this.builder.SetResult();
}
```
代码量很大，你可能注意到它有很多 `goto` 语句和代码标签，这在手写的 C# 代码中几乎看不到。目前，我预计它有些难以理解，但我希望先展示一个具体的例子，这样你可以在需要时随时参考。我将进一步将其分解为通用结构，然后是 `await` 表达式的具体细节。到本节结束时，清单 6.3 对你来说可能看起来仍然极其丑陋，但你会更好地理解它在做什么以及为什么这样做。

## **`MoveNext()` 方法的通用结构**

我们进入了异步这棵洋葱的下一层。`MoveNext()` 方法是异步状态机的核心，其复杂性提醒我们让异步代码正确运行是多么困难。状态机越复杂，你越应该庆幸是 C# 编译器而不是你必须编写这些代码。

> **注意**：是时候为了简洁引入更多术语了。在每个 `await` 表达式处，被等待的值可能已经完成，也可能仍未完成。如果在等待它时它已经完成，状态机会继续执行。我称此为**快速路径**。如果它尚未完成，状态机会调度一个延续并暂停。我称此为**慢速路径**。

提醒一下，`MoveNext()` 方法在异步方法首次被调用时被调用一次，然后在每次需要从 `await` 表达式的暂停状态恢复时再被调用一次。（如果每个 `await` 表达式都走快速路径，`MoveNext()` 将只被调用一次。）该方法负责以下事项：
- 从正确的位置开始执行（无论是原始异步代码的开始还是中间某个位置）
- 在需要暂停时保存状态，包括局部变量和代码内的位置
- 在需要暂停时调度一个延续
- 从等待器获取返回值
- 通过构建器传播异常（而不是让 `MoveNext()` 自身抛出异常而失败）
- 通过构建器传播任何返回值或方法完成信号

考虑到这些，下面的清单展示了 `MoveNext()` 方法通用结构的伪代码。你将在后续章节看到由于额外的控制流，这会变得如何更复杂，但这是一种自然的扩展。

**清单 6.4 `MoveNext()` 方法的伪代码**
```csharp
void IAsyncStateMachine.MoveNext()
{
    try
    {
        switch (this.state)
        {
            default: goto MethodStart;
            case 0: goto Label0A;
            case 1: goto Label1A;
            case 2: goto Label2A;
            // ... 根据 await 表达式的数量有多少 case
        }
MethodStart:
        // 第一个 await 表达式之前的代码
        // 设置第一个等待器
        // ... 快速路径和慢速路径在此处重新汇合
Label0A:
        // 从延续中恢复的代码
Label0B:
        // 其余代码，带有更多标签、等待器等...
    }
    catch (Exception e)
    {
        this.state = -2;
        builder.SetException(e); // 通过构建器传播所有异常
        return;
    }
    this.state = -2;
    builder.SetResult(); // 通过构建器传播方法完成信号
}
```
那个大的 `try/catch` 块覆盖了来自原始异步方法的所有代码。如果其中任何东西抛出异常，无论异常是如何抛出的（通过等待一个故障操作、调用一个会抛出异常的同步方法，或者直接抛出异常），该异常都会被捕获，然后通过构建器传播。只有特殊的异常（例如 `ThreadAbortException` 和 `StackOverflowException`）才会导致 `MoveNext()` 以异常结束。

在 `try/catch` 块内部，`MoveNext()` 方法的开始总是一个 `switch` 语句，用于根据状态跳转到方法内的正确代码段。如果状态是非负数，这意味着你正在从 `await` 表达式之后恢复。否则，假定你是第一次执行 `MoveNext()`。

**其他状态怎么办？**
在 6.1 节中，我将可能的状态列为未开始、执行中、暂停和完成（其中暂停是每个 `await` 表达式的单独状态）。为什么状态机不区分处理未开始、执行中和完成状态？

答案是，`MoveNext()` 永远不应该在执行中或完成状态下被调用。你可以通过编写一个损坏的等待器实现或使用反射来强行调用它，但在正常操作下，`MoveNext()` 仅在启动或恢复状态机时被调用。甚至没有为未开始和执行中设置不同的状态值；两者都使用 -1。完成状态有一个状态值 -2，但状态机从不检查这个值。

需要注意的一个棘手之处是状态机中的 `return` 语句与原始异步代码中的 `return` 语句之间的区别。在状态机内部，当状态机在为等待器调度延续后暂停时，会使用 `return`。原始代码中的任何 `return` 语句最终都会落到状态机 `try/catch` 块外的底部，在那里方法的完成信号通过构建器传播。

如果你比较清单 6.3 和 6.4，希望你能看到我们的具体例子是如何适应这个通用模式的。至此，我已经解释了由你开始的那个简单异步方法生成的代码的几乎所有内容。唯一缺失的部分是围绕 `await` 表达式具体发生了什么。



## **深入 `await` 表达式**

让我们再次思考一下，在异步方法中每次遇到 `await` 表达式时（假设你已经对操作数求值得到了可等待对象）必须发生的事情：

1.  通过调用 `GetAwaiter()` 从可等待对象中获取等待器，并将其存储在栈上。
2.  检查等待器是否已经完成。如果是，可以直接跳到获取结果（第 9 步）。这就是**快速路径**。
3.  看起来你走的是**慢速路径**。好吧。通过 `state` 字段记住你到达的位置。
4.  将等待器保存在一个字段中。
5.  与等待器一起调度一个延续，确保当延续执行时，你能回到正确的状态（必要时进行装箱之舞）。
6.  从 `MoveNext()` 方法返回：如果是第一次暂停，则返回给原始调用者；否则返回给调度该延续的代码。
7.  当延续触发时，将状态设置回运行中（值为 -1）。
8.  将等待器从字段中复制回栈上，并清空字段，以便可能帮助垃圾回收器。现在你已准备好重新加入快速路径。
9.  从等待器获取结果（此时无论你走哪条路径，等待器都在栈上）。即使没有结果值，你也必须调用 `GetResult()`，以便在必要时让等待器传播错误。
10. 继续愉快地执行，如果有结果值，则使用它来执行原始代码的其余部分。

记住这个列表，让我们回顾一下清单 6.3 中对应于第一个 `await` 表达式的一部分代码。

**清单 6.5 清单 6.3 中对应于单个 await 的代码段**
```csharp
awaiter1 = Task.Delay(this.delay).GetAwaiter();
if (awaiter1.IsCompleted)
{
    goto GetFirstAwaitResult;
}
this.state = num = 0;
this.awaiter = awaiter1;
this.builder.AwaitUnsafeOnCompleted(ref awaiter1, ref this);
return;
FirstAwaitContinuation:
awaiter1 = this.awaiter;
this.awaiter = default(TaskAwaiter);
this.state = num = -1;
GetFirstAwaitResult:
awaiter1.GetResult();
```
不出所料，这段代码精确地遵循了上述步骤。两个标签代表了根据路径需要跳转到的两个位置：
- 在**快速路径**中，你跳过慢速路径的代码。
- 在**慢速路径**中，当延续被调用时，你跳回代码中间。（记住，这就是方法开头的 `switch` 语句的作用。）

对 `builder.AwaitUnsafeOnCompleted(ref awaiter1, ref this)` 的调用是执行装箱之舞（必要时回调 `SetStateMachine`；每个状态机只发生一次）并调度延续的部分。在某些情况下，你会看到调用 `AwaitOnCompleted` 而不是 `AwaitUnsafeOnCompleted`。它们仅在执行上下文的处理方式上有所不同。你将在 6.5 节更详细地了解这一点。

可能看起来有点不明确的一个方面是局部变量 `num` 的使用。它总是在与 `state` 字段相同的时间被赋值，但总是读取它而不是字段。（它的初始值是从字段中复制的，但这是唯一一次读取字段。）我相信这纯粹是为了优化。无论何时读取 `num`，你都可以将其视为 `this.state`。

查看清单 6.5，原本只是以下代码：
```csharp
await Task.Delay(delay);
```
却对应了 16 行代码。好消息是，除非你进行这类练习，否则几乎永远不需要看到所有这些代码。也有一个小的坏消息，即代码膨胀意味着即使是很小的异步方法——即使是使用 `ValueTask<TResult>` 的方法——也无法被 JIT 编译器合理地内联。但在大多数情况下，与 async/await 带来的好处相比，这是一个微不足道的代价。

这是控制流简单的简单情况。有了这个背景，你可以探索几个更复杂的情况。





# **控制流如何影响 `MoveNext()`**

到目前为止，我们看到的例子只是一系列方法调用，只有 `await` 操作符引入了复杂性。当你想要使用所有常用的控制流语句来编写真实代码时，情况会变得困难一些。在本节中，我将展示控制流的两个要素：循环和 `try/finally` 语句。这并不是要面面俱到，但应该足以让你窥见编译器必须执行的“控制流体操”，以便你在需要时理解其他情况。

## **`await` 表达式之间的控制流很简单**

在我们进入复杂的部分之前，我先举一个例子，说明引入控制流并不会比同步代码增加生成代码的复杂性。在下面的清单中，在我们的示例方法中引入了一个循环，因此你将打印三次 `Between delays` 而不是一次。

**清单 6.6 在 `await` 表达式之间引入循环**
```csharp
static async Task PrintAndWaitWithSimpleLoop(TimeSpan delay)
{
    Console.WriteLine("Before first delay");
    await Task.Delay(delay);
    for (int i = 0; i < 3; i++)
    {
        Console.WriteLine("Between delays");
    }
    await Task.Delay(delay);
    Console.WriteLine("After second delay");
}
```
反编译后是什么样子？非常像清单 6.2！唯一的区别是，原来的：
```csharp
GetFirstAwaitResult:
awaiter1.GetResult();
Console.WriteLine("Between delays");
TaskAwaiter awaiter2 = Task.Delay(this.delay).GetAwaiter();
```
变成了：
```csharp
GetFirstAwaitResult:
awaiter1.GetResult();
for (int i = 0; i < 3; i++)
{
    Console.WriteLine("Between delays");
}
TaskAwaiter awaiter2 = Task.Delay(this.delay).GetAwaiter();
```
状态机中的变化与原始代码中的变化完全相同。没有额外的字段，也没有关于如何继续执行的复杂性；它只是一个循环。

我提到这一点的原因是为了帮助你思考为什么在我们的下一个例子中需要额外的复杂性。在清单 6.6 中，你永远不需要从外部跳入循环，也永远不需要暂停执行并跳出循环，从而暂停状态机。这些情况是由循环内的 `await` 表达式引入的。现在让我们来看这种情况。

## **在循环内等待**

到目前为止，我们的例子包含两个 `await` 表达式。为了在引入其他复杂性时保持代码的可管理性，我将把它减少到一个。下面的清单展示了在本小节中将要反编译的异步方法。

**清单 6.7 在循环中等待**
```csharp
static async Task AwaitInLoop(TimeSpan delay)
{
    Console.WriteLine("Before loop");
    for (int i = 0; i < 3; i++)
    {
        Console.WriteLine("Before await in loop");
        await Task.Delay(delay);
        Console.WriteLine("After await in loop");
    }
    Console.WriteLine("After loop delay");
}
```
`Console.WriteLine` 调用主要作为反编译代码中的路标，这样可以更容易地映射到原始清单。

编译器为此生成了什么？我不打算展示完整的代码，因为大部分与你之前看到的类似。（不过，所有代码都在可下载的源代码中。）桩方法和状态机几乎与前面的例子完全相同，只是状态机中多了一个对应于循环计数器 `i` 的字段。有趣的部分在 `MoveNext()` 中。

你可以用 C# 忠实地表示代码，但不能使用循环结构。问题是，在状态机从 `Task.Delay` 处的暂停返回后，你想要跳入原始循环的中间。在 C# 中，你不能用 `goto` 语句做到这一点；如果 `goto` 语句不在该标签的作用域内，C# 语言禁止使用指定该标签的 `goto` 语句。

这没关系；你可以用很多 `goto` 语句来实现 `for` 循环，而根本不需要引入任何额外的作用域。这样，你就可以毫无问题地跳入循环中间。下面的清单展示了 `MoveNext()` 方法体反编译代码的主要部分。我只包含了 `try` 块内的部分，因为这是我们在这里关注的。（其余部分是简单的样板代码。）

**清单 6.8 不使用任何循环结构反编译的循环**
```csharp
switch (num)
{
    default:
        goto MethodStart;
    case 0:
        goto AwaitContinuation;
}
MethodStart:
Console.WriteLine("Before loop");
this.i = 0; // For 循环初始化部分
goto ForLoopCondition;
ForLoopBody: // For 循环体
Console.WriteLine("Before await in loop");
TaskAwaiter awaiter = Task.Delay(this.delay).GetAwaiter();
if (awaiter.IsCompleted)
{
    goto GetAwaitResult;
}
this.state = num = 0;
this.awaiter = awaiter;
this.builder.AwaitUnsafeOnCompleted(ref awaiter, ref this);
return;
AwaitContinuation: // 状态机恢复时的跳转目标
awaiter = this.awaiter;
this.awaiter = default(TaskAwaiter);
this.state = num = -1;
GetAwaitResult:
awaiter.GetResult();
Console.WriteLine("After await in loop");
this.i++; // For 循环迭代部分
ForLoopCondition: // 检查循环条件，如果成立则跳回循环体
if (this.i < 3)
{
    goto ForLoopBody;
}
Console.WriteLine("After loop delay");
```
我本可以完全跳过这个例子，但它提出了一些有趣的点。首先，C# 编译器不会将异步方法转换成不使用 async/await 的等效 C# 代码。它只需要生成适当的 IL。在某些地方，C# 的规则比 IL 更严格。（有效的标识符集合是另一个例子。）

其次，尽管反编译器在查看异步代码时可能有用，但有时它们会产生无效的 C#。当我第一次反编译清单 6.7 的输出时，输出包含一个 `while` 循环，其中有一个标签，以及循环外的一个 `goto` 语句试图跳入循环。有时，通过告诉反编译器不要那么努力地生成地道的 C#，你可以得到有效（但更难读）的 C# 代码，这时你会看到大量的 `goto` 语句。

第三，如果你还没有被说服，你应该不想手写这类代码。如果你必须为这类任务编写 C# 4 代码，你无疑会以非常不同的方式来做，但它仍然会比你在 C# 5 中可以使用的异步方法丑陋得多。

你已经看到了在循环内等待可能会给人类带来一些压力，但这并不会让编译器出汗。在我们最后的控制流例子中，你将给它一些更困难的工作：一个 `try/finally` 块。





### **译文**

**6.3.3 在 `try/finally` 块中等待**

提醒一下，在 `try` 块中使用 `await` 一直是有效的，但在 C# 5 中，在 `catch` 或 `finally` 块中使用它是无效的。这个限制在 C# 6 中被取消，不过我不打算展示任何利用这一点的代码。

> **注意**：这里的可能性实在太多了，无法一一列举。本章的目的是让你深入了解 C# 编译器如何处理 async/await，而不是提供详尽的转换列表。

在本节中，我将只展示一个在仅包含 `finally` 块的 `try` 块中等待的例子。这可能是最常见的 `try` 块类型，因为它等同于 `using` 语句的结构。下面的清单展示了将要反编译的异步方法。同样，所有的控制台输出只是为了更容易理解状态机。

**清单 6.9 在 `try` 块中等待**
```csharp
static async Task AwaitInTryFinally(TimeSpan delay)
{
    Console.WriteLine("Before try block");
    await Task.Delay(delay);
    try
    {
        Console.WriteLine("Before await");
        await Task.Delay(delay);
        Console.WriteLine("After await");
    }
    finally
    {
        Console.WriteLine("In finally block");
    }
    Console.WriteLine("After finally block");
}
```

你可能会想象反编译的代码看起来像这样：
```csharp
switch (num)
{
    default:
        goto MethodStart;
    case 0:
        goto AwaitContinuation;
}
MethodStart:
...
try
{
    ...
AwaitContinuation:
    ...
GetAwaitResult:
    ...
}
finally
{
    ...
}
...
```
这里，每个省略号 `...` 代表更多的代码。但是，这种方法有一个问题：即使在 IL 中，也不允许从 `try` 块外部跳转到内部。这有点像上一节中看到的循环问题，但这次不是 C# 规则，而是 IL 规则。

为了实现这一点，C# 编译器使用了一种我喜欢称之为“蹦床”的技术。（这不是官方术语，尽管该术语在其他地方用于类似目的。）它跳转到紧挨着 `try` 块之前的位置，然后 `try` 块内的第一段代码会跳转到块内的正确位置。

除了蹦床技术，`finally` 块也需要小心处理。在三种情况下，你会执行生成代码中的 `finally` 块：
- 你到达了 `try` 块的末尾。
- `try` 块抛出了异常。
- 由于 `await` 表达式，你需要在 `try` 块内暂停。

（如果异步方法包含 `return` 语句，那将是另一种情况。）如果 `finally` 块正在执行是因为你正在暂停状态机并返回给调用者，那么原始异步方法的 `finally` 块中的代码就不应该执行。毕竟，在逻辑上你是在 `try` 块内暂停，并将在延迟完成后在那里恢复。幸运的是，这很容易检测：局部变量 `num`（其值总是与 `state` 字段相同）在状态机仍在执行或已完成时为负数，在你暂停时为非负数。

所有这些加在一起，就产生了下面的清单，它同样是 `MoveNext()` 外层 `try` 块内的代码。尽管代码量仍然很大，但大部分与你之前看到的类似。我将与 `try/finally` 相关的特定部分用**粗体**高亮显示。

**清单 6.10 反编译的 `try/finally` 内的等待**
```csharp
switch (num)
{
    default:
        goto MethodStart;
    case 0:
        goto AwaitContinuationTrampoline; // 跳转到蹦床前，以便它能将执行弹到正确位置
}
MethodStart:
Console.WriteLine("Before try");
AwaitContinuationTrampoline: // `try` 块内的蹦床
try
{
    switch (num)
    {
        default:
            goto TryBlockStart;
        case 0:
            goto AwaitContinuation; // 真正的延续目标
    }
TryBlockStart:
    Console.WriteLine("Before await");
    TaskAwaiter awaiter = Task.Delay(this.delay).GetAwaiter();
    if (awaiter.IsCompleted)
    {
        goto GetAwaitResult;
    }
    this.state = num = 0;
    this.awaiter = awaiter;
    this.builder.AwaitUnsafeOnCompleted(ref awaiter, ref this);
    return;
AwaitContinuation:
    awaiter = this.awaiter;
    this.awaiter = default(TaskAwaiter);
    this.state = num = -1;
GetAwaitResult:
    awaiter.GetResult();
    Console.WriteLine("After await");
}
finally
{
    if (num < 0) // 如果你正在暂停，则有效地忽略 `finally` 块
    {
        Console.WriteLine("In finally block");
    }
}
Console.WriteLine("After finally block");
```

我保证这是本章最后一个反编译示例。我之所以要达到这个复杂度级别，是为了在你将来需要时帮助你理解生成的代码。这并不是说你查看这些代码时不需要保持清醒的头脑，特别是要记住编译器可以执行许多转换以使代码比我展示的更简单。正如我之前所说，在我总是使用 `switch` 语句来表示“跳转到 X”代码的地方，编译器有时可以使用更简单的分支代码。在阅读源代码时，多种情况下的一致性很重要，但这对于编译器来说无关紧要。

到目前为止，我忽略的一个方面是为什么等待器必须实现 `INotifyCompletion`，但也可以实现 `ICriticalNotifyCompletion`，以及这对生成的代码有什么影响。现在让我们仔细看看。

---

 

> 编译器处理 `try/finally` 内的 `await` 时所采用的“蹦床”技术，本质上是一种在底层指令集限制（禁止跨 `try` 块边界跳转）与高级语言语义（允许在 `try` 块内任意位置暂停）之间寻找通路的智慧。它通过增加一个间接层（先跳到 `try` 块入口，再通过内部 `switch` 跳转到实际位置）来规避限制，这启示我们：当在某个抽象层面遇到不可逾越的障碍时，回到更基础的层面增加一个间接层，往往是实现语义无损转换的有效策略。同时，通过在 `finally` 块中检查状态来区分“正常退出”与“异步暂停”，也体现了异步执行模型对传统控制流语义的精细调整。
>

 

> 本节深入探讨了在 `try/finally` 块中使用 `await` 时编译器生成的复杂代码结构。核心挑战在于 IL 禁止从 `try` 块外部直接跳转到内部，因此编译器采用了“蹦床”模式：状态机先跳转到 `try` 块之前的标签，然后通过 `try` 块内部的另一个 `switch` 跳转到正确的恢复点。此外，`finally` 块中的代码需要根据状态（`num < 0` 表示正在执行或完成，`num >= 0` 表示因等待而暂停）有条件地执行，以确保异步暂停时不会错误执行清理逻辑。这些机制共同确保了异步代码的异常处理和资源清理语义与同步代码保持一致。

 

# **执行上下文与流动**

在 5.2.2 节中，我描述了用于控制代码执行线程的同步上下文。这只是 .NET 中众多上下文之一，尽管它可能是最著名的。上下文提供了一种透明维护环境信息的途径。例如，`SecurityContext` 跟踪当前安全主体和代码访问安全性。你无需显式地传递所有这些信息；它们会跟随你的代码，在几乎所有情况下都能正确工作。有一个单独的类用于管理所有其他上下文：`ExecutionContext`。

> **深入且令人畏惧的内容**
> 我几乎不打算包含这一节。这几乎是我关于异步知识的极限。如果你需要了解其内部细节，你将会希望了解比这里所包含的更多内容。
> 我之所以涵盖这部分内容，只是因为否则就无法解释为什么构建器中同时有 `AwaitOnCompleted` 和 `AwaitUnsafeOnCompleted`，或者为什么等待器通常实现 `ICriticalNotifyCompletion`。

提醒一下，`Task` 和 `Task<T>` 管理着任何被等待的任务的同步上下文。如果你在 UI 线程上并等待一个任务，你的异步方法的延续也将在 UI 线程上执行。你可以通过使用 `Task.ConfigureAwait` 来选择退出这一点。你需要这个来明确地说：“我知道我的方法的其余部分不需要在同一个同步上下文中执行。” 执行上下文则不同；你几乎总是希望在你的异步方法继续时使用相同的执行上下文，即使是在不同的线程上。

执行上下文的这种保留被称为**流动**。执行上下文被认为可以跨越 `await` 表达式流动，这意味着你所有的代码都在相同的执行上下文中运行。是什么确保了这一点呢？嗯，`AsyncTaskMethodBuilder` 总是这样做，而 `TaskAwaiter` 有时这样做。这正是事情变得棘手的地方。

`INotifyCompletion.OnCompleted` 方法只是一个普通方法；任何人都可以调用它。相比之下，`ICriticalNotifyCompletion.UnsafeOnCompleted` 被标记为 `[SecurityCritical]`。它只能被受信任的代码（例如框架的 `AsyncTaskMethodBuilder` 类）调用。

如果你曾经编写自己的等待器类，并且关心在部分受信任的环境中正确且安全地运行代码，你应该确保你的 `INotifyCompletion.OnCompleted` 代码流动执行上下文（通过 `ExecutionContext.Capture` 和 `ExecutionContext.Run`）。你也可以实现 `ICriticalNotifyCompletion`，并在这种情况下不流动执行上下文，相信异步基础设施已经这样做了。实际上，这是对等待器仅被异步基础设施使用的常见情况的一种优化。在可以安全地只做一次的情况下，没有必要捕获和恢复两次执行上下文。

在编译异步方法时，编译器将在每个 `await` 表达式处创建对 `builder.AwaitOnCompleted` 或 `builder.AwaitUnsafeOnCompleted` 的调用，具体取决于等待器是否实现 `ICriticalNotifyCompletion`。这些构建器方法是泛型的，并有限制条件，以确保传递给它们的等待器实现了适当的接口。

如果你曾经实现自己的自定义任务类型（再次强调，除了教育目的外，这极不可能），你应该遵循与 `AsyncTaskMethodBuilder` 相同的模式：在 `AwaitOnCompleted` 和 `AwaitUnsafeOnCompleted` 中都捕获执行上下文，这样当被要求调用 `ICriticalNotifyCompletion.UnsafeOnCompleted` 时是安全的。说到自定义任务，既然你已经看到了编译器如何使用 `AsyncTaskMethodBuilder`，让我们回顾一下自定义任务构建器的要求。

---

 

> 执行上下文的流动机制揭示了异步编程中一个常被忽视的深层维度：除了线程和同步上下文之外，代码执行还承载着一系列“环境”信息（如安全身份、文化设置等）。`async/await` 通过 `ExecutionContext.Capture/Run` 自动传递这些信息，确保异步操作不会意外丢失安全边界或区域设置，这体现了 .NET 框架对“透明上下文传播”这一复杂问题的系统性解决思路。然而，这种自动流动也可能成为性能瓶颈（例如在热点路径上频繁捕获上下文），因此框架提供了 `ICriticalNotifyCompletion` 这种“受信任的优化通道”，允许基础设施在确保安全的前提下避免重复捕获。
>
>  
>
> 本节解释了执行上下文（`ExecutionContext`）在异步操作中的流动机制。为了保持安全上下文、文化设置等环境信息在异步延续中不丢失，编译器根据等待器是否实现 `ICriticalNotifyCompletion` 接口，决定调用构建器的 `AwaitOnCompleted`（要求等待器自行处理上下文流动）或 `AwaitUnsafeOnCompleted`（由构建器负责上下文流动，等待器可免除此责任）。这种设计平衡了安全性与性能：普通等待器需确保上下文正确传播，而受信任的等待器则可依赖基础设施完成此工作。



# **自定义任务类型回顾**
清单 6.11 再次展示了清单 5.10 中的构建器部分，你最初是在那里首次看到自定义任务类型。在你查看了这么多反编译的状态机之后，这组方法现在可能感觉熟悉多了。你可以将本节作为对 `AsyncTaskMethodBuilder` 方法如何被调用的回顾，因为编译器对所有构建器的处理方式都是相同的。

**清单 6.11 一个示例自定义任务构建器**
```csharp
public class CustomTaskBuilder<T>
{
    public static CustomTaskBuilder<T> Create();
    public void Start<TStateMachine>(ref TStateMachine stateMachine)
        where TStateMachine : IAsyncStateMachine;
    public CustomTask<T> Task { get; }
    public void AwaitOnCompleted<TAwaiter, TStateMachine>
        (ref TAwaiter awaiter, ref TStateMachine stateMachine)
        where TAwaiter : INotifyCompletion
        where TStateMachine : IAsyncStateMachine;
    public void AwaitUnsafeOnCompleted<TAwaiter, TStateMachine>
        (ref TAwaiter awaiter, ref TStateMachine stateMachine)
        where TAwaiter : INotifyCompletion
        where TStateMachine : IAsyncStateMachine;
    public void SetStateMachine(IAsyncStateMachine stateMachine);
    public void SetException(Exception exception);
    public void SetResult(T result);
}
```
我按照方法被调用的正常时间顺序进行了分组。桩方法调用 `Create` 来创建一个构建器实例，作为新创建的状态机的一部分。然后它调用 `Start` 让状态机执行第一步，并返回 `Task` 属性的结果。

在状态机内部，每个 `await` 表达式会根据上一节的讨论，生成对 `AwaitOnCompleted` 或 `AwaitUnsafeOnCompleted` 的调用。假设是类任务的设计，第一次这样的调用最终会调用 `IAsyncStateMachine.SetStateMachine`，后者又会调用构建器的 `SetStateMachine`，以便以一致的方式解决任何装箱问题。详情请回顾 6.1.4 节。

最后，状态机通过调用构建器上的 `SetException` 或 `SetResult` 来指示异步操作已完成。这个最终状态应该传播给最初由桩方法返回的自定义任务。

本章是本书迄今为止最深入的探讨。书中没有其他地方如此详细地查看 C# 编译器生成的代码。对许多开发者来说，本章的所有内容可能都是多余的；你并不真的需要这些来编写正确的 C# 异步代码。但对于充满好奇心的开发者，我希望它有所启发。你可能永远不需要反编译生成的代码，但对底层发生的事情有所了解是有益的。如果你确实需要详细查看正在发生的事情，我希望本章能帮助你理解你所看到的内容。

我用了两章来介绍 C# 5 这一个主要特性。在下一个简短的章节中，我将介绍剩余的两个特性。在了解了异步的细节之后，它们会带来一些轻松感。

**总结**
- 异步方法通过使用构建器作为异步基础设施，被转换为桩方法和状态机。
- 状态机跟踪构建器、方法参数、局部变量、等待器以及在延续中恢复的位置。
- 编译器创建代码，以便在方法恢复时能够回到方法的中间位置。
- `INotifyCompletion` 和 `ICriticalNotifyCompletion` 接口有助于控制执行上下文的流动。
- 自定义任务构建器的方法由 C# 编译器调用。 





