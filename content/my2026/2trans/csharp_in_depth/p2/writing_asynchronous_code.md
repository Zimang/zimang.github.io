---
weight: 1
title: "编写异步代码"
---


**本章涵盖**

- 异步代码编写的含义
- 用 `async` 修饰符声明异步方法
- 用 `await` 操作符进行异步等待
- C# 5 以来 `async/await` 的语言演进
- 异步代码的使用准则

多年来，异步编程一直是开发者的痛点。众所周知，它能避免在等待耗时操作时阻塞线程，但正确实现它却令人头疼。

即使在 .NET 框架（从宏观看仍相对年轻）中，我们也经历了三种试图简化此过程的模型：
- .NET 1.x 的 **BeginFoo/EndFoo** 模式，使用 `IAsyncResult` 和 `AsyncCallback` 传递结果。
- .NET 2.0 的**基于事件的异步模式**，如 `BackgroundWorker` 和 `WebClient` 的实现。
- .NET 4.0 引入并在 .NET 4.5 扩展的**任务并行库**。

尽管 TPL 设计卓越，但用它编写健壮、可读的异步代码仍很困难。虽然其对并行的支持很好，但异步编程的某些方面更适合在语言层面而非单纯在库中解决。

C# 5 的核心特性通常称为 **async/await**，它构建于 TPL 之上。它允许你编写形似同步、却在适当时机使用异步的代码。曾经的回调嵌套、事件订阅和零散的错误处理已不复存在；取而代之的是，异步代码能清晰表达其意图，并以开发者熟悉的结构为基础。C# 5 引入的语言构造允许你 **`await`** 一个异步操作。这种“等待”看起来很像普通的阻塞调用——在操作完成前后续代码不会执行，但其神奇之处在于能在不阻塞当前线程的情况下做到这一点。如果这听起来完全矛盾，别担心，本章会让你豁然开朗。

为简化起见，我在介绍原始 C# 5 功能的同时，也包含了 C# 6 和 C# 7 的新特性，并标明了变化点。

.NET Framework 4.5 版全面拥抱异步，大量操作都提供了遵循**基于任务的异步模式**的异步版本，确保了跨 API 的一致性体验。同样，作为通用 Windows 应用基础的 **Windows 运行时平台**，强制要求所有长时间运行（或可能长时间运行）的操作必须是异步的。许多现代 API 也重度依赖异步，如 Roslyn 和 HttpClient。简言之，大多数 C# 开发者都至少会在部分工作中用到异步。

> **注意**：Windows 运行时平台常称为 **WinRT**，请勿与 ARM 版 Windows 8.x 的 **Windows RT** 混淆。

澄清一下，C# 并未变得全能，能猜测你何处需要并发或异步。编译器是聪明的，但它不会试图消除异步执行固有的复杂性。你仍需仔细思考，但 async/await 之美在于，它移除了以往所需的所有繁琐、令人困惑的样板代码。没有了那些让代码异步化的干扰，你可以专注于真正的难点。

**警告**：这是一个相当高级的主题。它具有双重特性：极其重要（实际上，即使是初级开发者也需要对其有合理理解），但一开始理解起来又相当棘手。

本章从“普通开发者”视角关注异步性，让你无需理解过多细节即可使用 async/await。第6章将深入探讨实现的复杂性。理解幕后原理会让你成为更好的开发者，但你完全可以在深入学习前，利用本章所学高效使用 async/await。即使在本章内，我们也会以迭代的方式审视这个特性，步步深入。

# **引入异步函数**

C# 5 引入了**异步函数**的概念。它总是一个用 `async` 修饰符声明的方法或匿名函数，并可以使用 `await` 操作符进行等待表达式。

> **提示**：匿名函数指 lambda 表达式或匿名方法。

从语言角度看，**await 表达式**是精彩之处：如果表达式等待的操作尚未完成，异步函数会立即返回，并在值可用时（在合适的线程上）从停止处继续执行。这种“未完成则不执行下一条语句”的自然流程得以保持，且无需阻塞。

## **初次邂逅异步**

让我们从一个实际演示异步的简单例子开始。我们常常抱怨网络延迟导致应用卡顿，但这恰好容易展示异步为何重要——尤其是在使用如 Windows Forms 这样的 GUI 框架时。我们的第一个示例是一个微型 Windows Forms 应用，它获取本书主页的 HTML 文本，并在标签中显示其长度。

**代码清单 5.1 异步显示网页长度**
```csharp
public class AsyncIntro : Form
{
    private static readonly HttpClient client = new HttpClient();
    private readonly Label label;
    private readonly Button button;

    public AsyncIntro()
    {
        label = new Label { Location = new Point(10, 20), Text = "Length" };
        button = new Button { Location = new Point(10, 50), Text = "Click" };
        button.Click += DisplayWebSiteLength; // 关联事件处理器
        AutoSize = true;
        Controls.Add(label);
        Controls.Add(button);
    }

    async void DisplayWebSiteLength(object sender, EventArgs e)
    {
        label.Text = "Fetching...";
        string text = await client.GetStringAsync("http://csharpindepth.com"); // 开始异步获取页面
        label.Text = text.Length.ToString(); // 更新 UI
    }

    static void Main()
    {
        Application.Run(new AsyncIntro()); // 入口点，运行窗体
    }
}
```
代码第一部分以直接的方式创建 UI 并为按钮关联事件处理器。重点在于 `DisplayWebSiteLength` 方法。点击按钮时，异步获取主页文本，然后更新标签显示 HTML 字符长度。

> **注意**：即使 `Task` 实现了 `IDisposable`，我也没有处置 `GetStringAsync` 返回的任务。通常你无需处置任务。Stephen Toub 在其博客中有详细解释。

如果移除 `async` 和 `await` 上下文关键字，将 `HttpClient` 改为 `WebClient`，将 `GetStringAsync` 改为 `DownloadString`，代码仍能编译运行，但 UI 在获取页面内容时会冻结。而运行异步版本（尤其在慢速网络下），你会看到 UI 保持响应；在获取网页时，你仍可以移动窗口。

> **注意**：`HttpClient` 可视为新版改进的 `WebClient`，是 .NET 4.5 及以后首选的 HTTP API，且只包含异步操作。

大多数开发者都熟悉 Windows Forms 开发中线程的两条黄金法则：
- 不要在 UI 线程上执行任何耗时操作。
- 不要跨 UI 线程访问任何 UI 控件。
如今你或视 Windows Forms 为遗留技术，但多数 GUI 框架遵循相同规则，而遵守它们比说出来要难。作为练习，你可以尝试几种不使用 async/await 实现类似清单 5.1 功能的方式。对于这个极其简单的例子，使用基于事件的 `WebClient.DownloadStringAsync` 方法还不算太糟，但一旦涉及更复杂的流控制（错误处理、等待多个页面完成等），遗留代码会迅速变得难以维护，而 C# 5 代码则能以自然的方式进行修改。

此时，`DisplayWebSiteLength` 方法感觉有些神奇：你知道它完成了所需功能，但不知其所以然。让我们稍作剖析，细节留待后叙。

## **解析第一个例子**

首先稍微展开这个方法。在清单 5.1 中，我直接在 `HttpClient.GetStringAsync` 的返回值上使用 `await`，但你可以将调用与等待部分分离：

```csharp
async void DisplayWebSiteLength(object sender, EventArgs e)
{
    label.Text = "Fetching...";
    Task<string> task = client.GetStringAsync("http://csharpindepth.com");
    string text = await task;
    label.Text = text.Length.ToString();
}
```
请注意，`task` 的类型是 `Task<string>`，但 `await task` 表达式的类型仅仅是 `string`。在这个意义上，`await` 操作符执行了一种**解包**操作——至少当等待的值是 `Task<TResult>` 时是这样。（稍后你会看到，也可以等待其他类型，但 `Task<TResult>` 是个很好的起点。）这是 `await` 与异步不直接相关但让生活更轻松的一个方面。

`await` 的主要目的是在等待耗时操作完成时避免阻塞。你可能想知道这在线程层面具体是如何工作的。你在方法的开始和结尾都设置了 `label.Text`，因此有理由假设这两条语句都是在 UI 线程上执行的，但显然在等待网页下载时你并没有阻塞 UI 线程。

诀窍在于，方法在遇到 `await` 表达式时会立即返回。在此之前，它像任何其他事件处理程序一样，在 UI 线程上同步执行。如果你在调试器中于第一行设置断点，你会看到堆栈跟踪显示按钮正在引发其 `Click` 事件。当执行到 `await` 时，代码会检查结果是否已可用；如果不可用（几乎总是如此），它会调度一个**延续操作**，在 web 操作完成时执行。在此例中，该延续执行方法的剩余部分，有效地跳转到 `await` 表达式之后。该延续在 UI 线程中执行，这正是你能操作 UI 所需要的。

> **定义**：**延续**本质上是一个回调，在异步操作（或任何 `Task`）完成时执行。在异步方法中，延续维持着方法的状态。就像闭包维护其变量环境一样，延续会记住它到达的位置，以便在执行时能从那里继续。`Task` 类有一个专门用于附加延续的方法：`Task.ContinueWith`。

如果你在 `await` 表达式后的代码中设置断点并再次运行代码，假设 `await` 表达式需要调度延续，你会看到堆栈跟踪中不再有 `Button.OnClick` 方法。该方法早已执行完毕。调用堆栈现在本质上是朴素的 Windows Forms 事件循环，上面叠加了几层异步基础设施。调用堆栈类似于你为了在后台线程中适当更新 UI 而调用 `Control.Invoke` 时所看到的情况，但这一切都已为你自动完成。

起初，注意到调用堆栈在你脚下发生戏剧性变化可能会令人不安，但这对于实现高效异步是绝对必要的。

编译器通过创建一个复杂的状态机来实现这一切。这是第6章将要探讨的实现细节，但现在你将专注于 async/await 提供的功能。首先，你需要一个更具体的描述，来说明你要实现什么以及语言的规范。





### 
# **思考异步性**

如果你让一位开发者描述异步执行，他很可能会开始谈论多线程。尽管这是异步典型应用的重要组成部分，但它并非异步执行的必要条件。要充分理解 C# 5 异步功能的原理，最好先抛开对线程的任何想法，回归基础。

## **异步执行基础**

异步性直击 C# 开发者所熟悉的执行模型的核心。请看下面简单的代码：

```csharp
Console.WriteLine("First");
Console.WriteLine("Second");
```
你期望第一个调用完成后，第二个调用才开始。执行流程按顺序从一条语句流向下一条。但异步执行模型并非如此运作。相反，它完全关乎 **延续**。当你开始执行某项操作时，你告诉该操作当它完成后你希望发生什么。你可能听说过或使用过术语“回调”，它表达了相同的理念，但其含义比我们这里所指的更广泛。在异步上下文中，我使用“延续”这个术语，特指那些能保存程序状态的回调，而不是为其他目的（如 GUI 事件处理程序）的任意回调。

在 .NET 中，延续很自然地用委托表示，它们通常是接收异步操作结果的动作。这就是为什么在 C# 5 之前，要使用 `WebClient` 中的异步方法，你需要关联各种事件来指定在成功、失败等情况下应执行什么代码。问题在于，为一系列复杂的步骤创建所有这些委托会变得非常复杂，即使有 lambda 表达式的好处也是如此。当你试图确保错误处理正确时，情况甚至更糟。（顺利的话，我可以对亲手编写的异步代码的成功路径相当有信心。但对于失败时的正确反应，我通常不那么确定。）

本质上，C# 中的 `await` 所做的一切就是让编译器为你构建一个延续。然而，对于一个可以如此简单表述的概念，它对代码可读性和开发者心境的提升效果却是非凡的。

我之前对异步性的描述是一种理想化的描述。基于任务的异步模式的现实情况略有不同。延续不是传递给异步操作，而是异步操作启动并返回一个**令牌/信物**，供你稍后用于提供延续。它代表了正在进行的操作，该操作可能在返回到调用代码之前就已经完成，也可能仍在进行中。然后，每当你想表达“在此操作完成之前，我无法继续推进”这个意思时，就会用到这个令牌。通常，这个令牌以 `Task` 或 `Task<TResult>` 的形式存在，但并非必须如此。

> **注意**：此处描述的令牌与**取消令牌**不同，尽管两者都强调同一个事实：你无需了解幕后发生的事情；你只需要知道这个令牌允许你做什么。

在 C# 5 异步方法中，执行流程通常遵循以下步骤：
1.  执行一些工作。
2.  启动一个异步操作并记住它返回的令牌。
3.  可能再执行一些工作。（通常，在异步操作完成之前，你无法取得任何进一步进展，此时这一步是空的。）
4.  等待异步操作完成（通过令牌）。
5.  执行更多工作。
6.  完成。

如果你不关心“等待”部分的确切含义，你可以在 C# 4 中完成所有这些。如果你愿意阻塞直到异步操作完成，令牌通常会为你提供某种方式来实现。对于 `Task`，你可以直接调用 `Wait()`。但在那时，你占用了一个宝贵的资源（一个线程）却没有做任何有用的工作。这有点像打电话订披萨外卖，然后站在前门直到送达。你真正想做的是继续做其他事情，忽略披萨，直到它送达。这就是 `await` 发挥作用的地方。

当你等待一个异步操作时，你是在说“我现在只能走到这里了。当操作完成时再继续。”但如果你不打算阻塞线程，你能做什么呢？很简单，你可以在此时此地直接返回。你将自身也变为异步继续。如果你想让调用者知道你的异步方法何时完成，你会将一个令牌传回给调用者，调用者可以根据需要选择阻塞等待它，或者（更可能地）将其用于另一个延续。通常，你最终会得到一整个相互调用的异步方法栈；这几乎就像是代码的某个部分进入了“异步模式”。语言本身并未规定必须这样做，但消费异步操作的代码其自身行为也表现为异步操作这一事实，无疑鼓励了这种做法。

## **同步上下文**

前面提到，UI 代码的黄金法则之一是你必须在正确的线程上更新用户界面。在清单 5.1（异步检查网页长度）中，你需要确保 `await` 表达式后的代码在 UI 线程上执行。异步函数通过使用 `SynchronizationContext`（自 .NET 2.0 以来就存在的类，也被 `BackgroundWorker` 等其他组件使用）来回到正确的线程。`SynchronizationContext` 概括了在合适线程上执行委托的思想；它的 `Post`（异步）和 `Send`（同步）方法与 Windows Forms 中的 `Control.BeginInvoke` 和 `Control.Invoke` 类似。

不同的执行环境使用不同的上下文；例如，某个上下文可能允许线程池中的任何线程来执行给定的动作。除了同步上下文外，还存在更多的上下文信息，但如果你开始疑惑异步方法究竟是如何在你想让它们执行的地方执行的，那么你需要关注的就是同步上下文。

关于 `SynchronizationContext` 的更多信息，请阅读 Stephen Cleary 在 MSDN 杂志上的文章。特别是 ASP.NET 开发者需要注意；ASP.NET 上下文很容易诱使粗心的开发者在看似正常的代码中创建死锁。对于 ASP.NET Core，情况略有变化，Stephen 有另一篇博客文章对此进行了介绍。

> **关于示例中使用 `Task.Wait()` 和 `Task.Result` 的说明**
> 我在一些示例代码中使用了 `Task.Wait()` 和 `Task.Result`，因为这使得示例简单。在控制台应用程序中这样做通常是安全的，因为在这种情况下没有同步上下文；异步方法的延续将始终在线程池中执行。
> 在实际应用程序中，使用这些方法时应非常小心。它们都会阻塞直到完成，这意味着如果你从延续需要在其上执行的线程中调用它们，很容易导致应用程序死锁。

理论部分就到这里，让我们更仔细地看看异步方法的具体细节。异步匿名函数符合相同的思维模型，但谈论异步方法要容易得多。





## **对异步方法建模**

我发现按照图5.1所示的方式来思考异步方法很有用。

![image-20260118230157356](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260118230157427.png)



这里有三个代码块（方法）和两个边界类型（方法返回类型）。举个简单的例子，在我们获取页面长度的应用程序的控制台版本中，可能会有如下代码。

**清单5.2 在异步方法中获取页面长度**
```csharp
static readonly HttpClient client = new HttpClient();
static async Task<int> GetPageLengthAsync(string url)
{
    Task<string> fetchTextTask = client.GetStringAsync(url);
    int length = (await fetchTextTask).Length;
    return length;
}
static void PrintPageLength()
{
    Task<int> lengthTask = GetPageLengthAsync("http://csharpindepth.com");
    Console.WriteLine(lengthTask.Result);
}
```
图5.2展示了清单5.2中的具体细节如何映射到图5.1的概念。

![image-20260118230240015](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260118230240056.png)





**图5.2 将清单5.2的细节应用到图5.1的通用模式中**

你主要感兴趣的是 `GetPageLengthAsync` 方法，但我包含了 `PrintPageLength`，以便你了解方法之间如何交互。特别是，你绝对需要了解方法边界的有效类型。我将在本章中以各种形式重复此图。

现在，你终于可以开始学习编写异步方法及其行为方式了。这里涵盖的内容很多，因为你能做什么和你做的时候会发生什么在很大程度上是交织在一起的。

只有两个新的语法：`async` 是声明异步方法时使用的修饰符，`await` 操作符用于消费异步操作。但是，跟踪信息在程序各部分之间传递的方式会很快变得复杂，特别是当你必须考虑出错时会发生什么时。我试图将不同方面分开，但你的代码将同时处理所有问题。如果你在阅读本节时发现自己想问“但是……呢？”，请继续读下去；很可能你的问题很快就会得到解答。

接下来的三节将从三个阶段来看异步方法：
- 声明异步方法
- 使用 `await` 操作符异步等待操作完成
- 在方法完成时返回一个值

图5.3展示了这些部分如何融入我们的概念模型。

![image-20260118230316982](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260118230317031.png)



# **异步方法声明**

让我们从方法声明本身开始；这是最简单的部分。

异步方法声明的语法与任何其他方法完全相同，只是必须包含 `async` 上下文关键字。它可以出现在返回类型之前的任何位置。以下都是有效的：
```csharp
public static async Task<int> FooAsync() { ... }
public async static Task<int> FooAsync() { ... }
async public Task<int> FooAsync() { ... }
public async virtual Task<int> FooAsync() { ... }
```
我的偏好是将 `async` 修饰符紧接在返回类型之前，但没有理由你不应该形成自己的约定。一如既往，与你的团队讨论，并尝试在同一个代码库中保持一致。

现在，`async` 上下文关键字有一个小秘密：语言设计者根本不需要包含它。就像编译器在具有合适返回类型的方法中尝试使用 `yield return` 或 `yield break` 时会进入某种迭代器块模式一样，编译器本可以识别方法内部 `await` 的使用，并用它来进入异步模式。但我很高兴 `async` 是必需的，因为它使得阅读使用异步方法编写的代码容易得多。它立即设定了你的期望，因此你会主动寻找 `await` 表达式，并且可以主动寻找应该转换为异步调用和 `await` 表达式的任何阻塞调用。

不过，`async` 修饰符在生成的代码中没有表示，这一点很重要。就调用方法而言，它是一个恰好返回任务（Task）的普通方法。你可以将现有方法（具有适当签名）更改为使用 `async`，或者你也可以反方向更改；就源代码和二进制兼容性而言，这是一个兼容的更改。`async` 是方法实现细节这一事实意味着你不能使用 `async` 声明抽象方法或接口中的方法。完全可以有一个接口指定返回类型为 `Task<int>` 的方法；该接口的一个实现可以使用 `async/await`，而另一个实现可以使用常规方法。



## **异步方法的返回类型**

调用者和异步方法之间的通信实际上是通过返回值来进行的。在 C# 5 中，异步函数的返回类型仅限于以下三种：

- `void`
- `Task`
- `Task<TResult>`（对于某些类型 `TResult`，它本身可以是类型参数）

在 C# 7 中，此列表扩展为包括任务类型（task types）。我们将在 5.8 节和第六章再次讨论这些。

.NET 4 的 `Task` 和 `Task<TResult>` 类型都表示一个可能尚未完成的操作；`Task<TResult>` 派生自 `Task`。两者之间的区别在于，`Task<TResult>` 表示返回 `TResult` 类型值的操作，而 `Task` 根本不产生结果。不过，返回 `Task` 仍然有用，因为它允许调用代码将自身的延续附加到返回的任务上，检测任务何时失败或完成等。在某些情况下，你可以将 `Task` 视为类似于 `Task<void>` 类型（如果这种类型有效的话）。

> **注意**：此时 F# 开发者可以理直气壮地对 `Unit` 类型感到自得，它类似于 `void` 但却是真正的类型。`Task` 和 `Task<TResult>` 之间的差异可能令人沮丧。如果可以将 `void` 用作类型参数，你也不需要 `Action` 系列的委托；例如，`Action<string>` 等价于 `Func<string, void>`。

异步方法返回 `void` 的能力是为了与事件处理程序兼容而设计的。例如，你可能有一个如下的 UI 按钮点击处理程序：
```csharp
private async void LoadStockPrice(object sender, EventArgs e)
{
    string ticker = tickerInput.Text;
    decimal price = await stockPriceService.FetchPriceAsync(ticker);
    priceDisplay.Text = price.ToString("c");
}
```
这是一个异步方法，但调用代码（按钮的 `OnClick` 方法或引发事件的任何框架代码）并不关心。它不需要知道你已经完成事件处理的时间——即你加载股价并更新 UI 的时间。它只是调用给定的事件处理程序。编译器生成的代码最终会有一个状态机，将延续附加到 `FetchPriceAsync` 返回的任何内容上，这是一个实现细节。

你可以像订阅任何其他事件处理程序一样，用前面的方法订阅事件：
```csharp
loadStockPriceButton.Click += LoadStockPrice;
```
毕竟（是的，我故意强调这一点），就调用代码而言，它只是一个普通的方法。它具有 `void` 返回类型和 `object` 及 `EventArgs` 类型的参数，这使其适合作为 `EventHandler` 委托实例的操作。

> **警告**：事件订阅几乎是我推荐从异步方法返回 `void` 的唯一情况。任何其他你不需要返回特定值的时候，最好声明方法返回 `Task`。这样，调用者能够等待操作完成、检测失败等。

尽管异步方法的返回类型受到相当严格的限制，但大多数其他方面都是正常的：异步方法可以是泛型、静态或非静态的，并且可以指定任何常规的访问修饰符。但是，对可以使用的参数存在限制。

## **异步方法中的参数**

异步方法中的参数不能使用 `out` 或 `ref` 修饰符。这是有道理的，因为这些修饰符用于将信息传递回调用代码；当控制权返回给调用者时，异步方法的部分代码可能尚未运行，因此通过引用传递的参数的值可能尚未设置。实际上，情况可能比这更奇怪：想象一下将局部变量作为 `ref` 参数的实参传递；异步方法可能最终在调用方法已经完成后尝试设置该变量。尝试这样做没有太大意义，因此编译器禁止它。此外，指针类型不能用作异步方法的参数类型。

声明方法后，你可以开始编写方法体并等待其他异步操作。让我们看看可以在何处以及如何使用 `await` 表达式。

# **等待表达式**

使用 `async` 修饰符声明方法的全部意义在于在该方法中使用 `await` 表达式。方法的其他所有方面看起来都很正常：你可以使用各种控制流——循环、异常、`using` 语句等等。那么，你可以在哪里使用 `await` 表达式，它又有什么作用呢？

`await` 表达式的语法很简单：它是 `await` 操作符后跟另一个产生值的表达式。你可以等待方法调用的结果、变量、属性。它也不一定必须是简单表达式。你可以将方法调用链接在一起并等待结果：
```csharp
int result = await foo.Bar().Baz();
```
`await` 操作符的优先级低于点操作符，因此此代码等价于：
```csharp
int result = await (foo.Bar().Baz());
```
## **可等待模式**

不过，限制决定了你可以等待哪些表达式。它们必须是可等待的，这就是可等待模式的用武之地。

可等待模式用于确定可以与 `await` 操作符一起使用的类型。图 5.4 提醒我们，我正在讨论图 5.1 中的第二个边界：异步方法如何与另一个异步操作交互。可等待模式是一种将我们所说的异步操作形式化的方法。

![image-20260118231046920](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260118231046985.png)



你可能期望这通过接口来表达，就像编译器要求类型实现 `IDisposable` 以支持 `using` 语句一样。相反，它基于一种模式。假设你有一个类型为 `T` 的表达式，你想要等待它。编译器执行以下检查：

- `T` 必须有一个无参数的 `GetAwaiter()` 实例方法，或者必须有一个接受单个类型为 `T` 的参数的扩展方法。`GetAwaiter` 方法必须是非 void 的。该方法的返回类型称为等待器类型（awaiter type）。
- 等待器类型必须实现 `System.Runtime.INotifyCompletion` 接口。该接口有一个方法：`void OnCompleted(Action)`。
- 等待器类型必须有一个名为 `IsCompleted`、类型为 `bool` 的可读实例属性。
- 等待器类型必须有一个名为 `GetResult` 的非泛型无参数实例方法。

前面列出的成员不必是公共的，但它们需要可以从你试图等待其值的异步方法中访问。（因此，在某些代码中，你可能能够等待特定类型的值，但并非在所有代码中都可以。不过，这种情况非常少见。）

如果 `T` 通过了所有这些检查，恭喜——你可以等待类型为 `T` 的值！不过，编译器还需要一个信息来确定 `await` 表达式的类型应该是什么。这由等待器类型的 `GetResult` 方法的返回类型决定。它可以是 void 方法，在这种情况下，`await` 表达式被归类为没有结果的表达式，就像直接调用 void 方法的表达式一样。否则，`await` 表达式被归类为产生与 `GetResult` 返回类型相同类型的值。

例如，让我们考虑静态方法 `Task.Yield()`。与 `Task` 上的大多数其他方法不同，`Yield()` 方法本身不返回任务；它返回一个 `YieldAwaitable`。下面是所涉及类型的简化版本：
```csharp
public class Task
{
    public static YieldAwaitable Yield();
}
public struct YieldAwaitable
{
    public YieldAwaiter GetAwaiter();
}
public struct YieldAwaiter : INotifyCompletion
{
    public bool IsCompleted { get; }
    public void OnCompleted(Action continuation);
    public void GetResult();
}
```
如你所见，`YieldAwaitable` 遵循前面描述的可等待模式。因此，这是有效的：
```csharp
public async Task ValidPrintYieldPrint()
{
    Console.WriteLine("Before yielding");
    await Task.Yield();  //有效的
    Console.WriteLine("After yielding");
}
```
但以下代码是无效的，因为它尝试使用等待 `YieldAwaitable` 的结果：
```csharp
public async Task InvalidPrintYieldPrint()
{
    Console.WriteLine("Before yielding");
    var result = await Task.Yield(); // 无效；此 await 表达式不产生值
    Console.WriteLine("After yielding");
}
```
`InvalidPrintYieldPrint` 的中间一行无效的原因与以下写法无效的原因完全相同：
```csharp
var result = Console.WriteLine("WriteLine is a void method");
```
没有产生结果，因此你不能将其赋值给变量。

毫不奇怪，`Task` 的等待器类型有一个返回类型为 `void` 的 `GetResult` 方法，而 `Task<TResult>` 的等待器类型有一个返回 `TResult` 的 `GetResult` 方法。

> **扩展方法的历史重要性**
> `GetAwaiter` 可以是扩展方法这一事实，其历史重要性大于当代重要性。C# 5 与 .NET 4.5 在同一时间框架内发布，后者在 `Task` 和 `Task<TResult>` 中引入了 `GetAwaiter` 方法。如果 `GetAwaiter` 必须是真正的实例方法，那么那些绑定到 .NET 4.0 的开发者就会束手无策。但有了对扩展方法的支持，`Task` 和 `Task<TResult>` 可以通过使用 NuGet 包单独提供这些扩展方法来实现异步/等待。这也意味着社区可以在不测试 .NET 4.5 预发布版本的情况下测试 C# 5 编译器的预发布版本。
> 在面向所有相关 `GetAwaiter` 方法都已存在的现代框架的代码中，你很少需要通过扩展方法使现有类型可等待。

在 5.6 节中，当你考虑异步方法的执行流程时，你将看到有关可等待模式中成员如何使用的更多细节。不过，关于 `await` 表达式，我们还没有完全结束；存在一些限制。



## **对 `await` 表达式的限制**

与 `yield return` 类似，使用 `await` 表达式的位置也受到限制。最明显的限制是只能在异步方法或异步匿名函数中使用（后者将在 5.7 节讨论）。即使在异步方法内部，也不能在匿名函数中使用 `await` 操作符，除非该匿名函数也是异步的。

`await` 操作符还不允许在不安全上下文中使用。这并不意味着在异步方法中不能使用不安全代码；只是不能在那一部分使用 `await` 操作符。清单 5.3 展示了一个人为设计的例子，其中使用指针遍历字符串中的字符以计算 UTF-16 代码单元的总和。它没有做任何真正有用的事情，但演示了在异步方法中使用不安全上下文。

**清单 5.3 在异步方法中使用不安全代码**
```csharp
static async Task DelayWithResultOfUnsafeCode(string text)
{
    int total = 0;
    unsafe // 在异步方法中使用不安全上下文是可以的。
    {
        fixed (char* textPointer = text)
        {
            char* p = textPointer;
            while (*p != 0)
            {
                total += *p;
                p++;
            }
        }
    }
    Console.WriteLine("Delaying for " + total + "ms");
    await Task.Delay(total); // 但是，await 表达式不能在里面。
    Console.WriteLine("Delay complete");
}
```

你也不能在 `lock` 语句中使用 `await` 操作符。如果你发现自己想在异步操作完成期间持有锁，你应该重新设计代码。不要通过手动调用 `Monitor.TryEnter` 和 `Monitor.Exit` 并使用 `try/finally` 块来绕过编译器的限制；应该更改你的代码，以便在操作期间不需要锁。如果这种情况确实非常棘手，考虑使用 `SemaphoreSlim` 及其 `WaitAsync` 方法代替。

`lock` 语句使用的监视器只能由最初获取它的同一线程释放，这与 `await` 表达式前后的代码可能在不同线程上执行的情况相悖。即使使用同一线程（例如，因为你在 GUI 同步上下文中），在异步操作的开始和结束之间，很可能有其他代码在同一线程上执行，而这些其他代码可能能够为同一监视器进入 `lock` 语句，这几乎肯定不是你的本意。基本上，`lock` 语句和异步性不能很好地共存。

最后一组上下文在 C# 5 中不允许使用 `await` 操作符，但从 C# 6 开始允许：
- 任何带有 `catch` 块的 `try` 块
- 任何 `catch` 块
- 任何 `finally` 块

在只有 `finally` 块的 `try` 块中使用 `await` 操作符一直是允许的，这意味着在 `using` 语句中使用 `await` 一直是允许的。C# 设计团队在 C# 5 发布之前没有弄清楚如何安全可靠地在上述上下文中包含 `await` 表达式。这偶尔会带来不便，团队在实现 C# 6 时研究出了如何构建适当的状态机，因此该限制在 C# 6 中被取消。

现在你已经知道如何声明异步方法以及如何在其中使用 `await` 操作符。那么当你完成工作后呢？让我们看看值是如何返回给调用代码的。

# **返回值的包装**

我们已经研究了如何声明调用代码和异步方法之间的边界，以及如何等待异步方法内部的异步操作。现在是时候看看如何从异步方法返回一个值了。这分为两个部分：实际返回的值以及方法签名如何影响它。

![image-20260118231928633](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260118231928695.png)

异步方法返回的值总是被包装在一个 `Task<TResult>` 或 `Task` 中（对于返回 `void` 的异步方法，则完全不返回任务）。在方法内部，你编写代码就像在返回一个普通值一样。编译器会处理将其包装在任务中的细节。

让我们看几个例子。首先，一个返回 `Task<int>` 的异步方法：
```csharp
static async Task<int> GetPageLengthAsync(string url)
{
    HttpClient client = new HttpClient();
    string text = await client.GetStringAsync(url);
    return text.Length; // 返回一个 int，但被包装在 Task<int> 中
}
```

注意，`return` 语句返回的是 `int` 值，但方法的返回类型是 `Task<int>`。编译器会处理转换。如果方法没有返回值，你可以使用一个简单的 `return` 语句，或者让执行流到达方法的末尾：
```csharp
static async Task DoSomethingAsync()
{
    await Task.Delay(1000);
    // 没有返回语句，但返回一个 Task
}
```

对于返回 `void` 的异步方法，你不能使用 `return` 语句返回一个值（因为 `void` 方法不能返回值），但你可以使用 `return;` 来提前退出：
```csharp
static async void DoSomethingAsync()
{
    await Task.Delay(1000);
    return; // 可选，因为 void 方法
}
```

返回值的包装是异步方法的一个重要方面，因为它使得调用代码可以轻松消费异步方法的结果，从而可以通过许多小模块构建复杂系统。你可以将其视为有点像 LINQ：在 LINQ 中，你对序列中的每个元素编写操作，而包装和解包装意味着你可以将这些操作应用于序列并得到序列返回。在异步世界中，你很少需要显式处理任务；相反，你通过 `await` 任务来消费它，并作为异步方法机制的一部分自动产生一个结果任务。现在你知道了异步方法的样子，更容易举例演示执行流程了。





# **异步方法执行流程**

你可以从多个层面来思考 async/await：

- 你可以简单地期望 `await` 会按你的意愿执行，而无需准确定义这意味着什么。
- 你可以从执行时机和线程的角度来推理代码将如何执行，但无需理解这是如何实现的。
- 你可以深入探究实现这一切的基础设施。

到目前为止，我们主要在第一层面思考，偶尔涉足第二层面。本节将聚焦于第二层面，有效地审视语言承诺了什么。我们将把第三点留到下一章，在那里你将看到编译器在幕后做了什么。（即便如此，你总是可以更深入；本书不讨论 IL 层面以下的内容。我们不涉及操作系统或硬件对异步和线程的支持。）

在开发的大部分时间里，根据你的上下文，在前两个层面之间切换是没有问题的。除非我编写的代码需要协调多个操作，否则我很少需要去思考第二层面的细节。大多数时候，我很乐意让事情顺其自然地工作。重要的是，当你需要时，你能够思考这些细节。

## **等待什么以及何时等待？**

让我们先从稍微简化的情况开始。有时 `await` 用于链式方法调用的结果，偶尔也用于属性，如下所示：

```csharp
string pageText = await new HttpClient().GetStringAsync(url);
```
这看起来好像 `await` 可以修改整个表达式的含义。实际上，`await` 总是只对单个值进行操作。上一行代码等价于：
```csharp
Task<string> task = new HttpClient().GetStringAsync(url);
string pageText = await task;
```
同样，`await` 表达式的结果可以用作方法参数或在另一个表达式中使用。同样，如果你能在脑海中将特定于 `await` 的部分与其他部分分开，会有所帮助。

假设你有两个方法，`GetHourlyRateAsync()` 和 `GetHoursWorkedAsync()`，分别返回 `Task<decimal>` 和 `Task<int>`。你可能有这样复杂的语句：
```csharp
AddPayment(await employee.GetHourlyRateAsync() *
           await timeSheet.GetHoursWorkedAsync(employee.Id));
```
C# 表达式求值的常规规则适用，`*` 运算符的左操作数必须在右操作数求值之前完全求值，因此前面的语句可以展开如下：
```csharp
Task<decimal> hourlyRateTask = employee.GetHourlyRateAsync();
decimal hourlyRate = await hourlyRateTask;
Task<int> hoursWorkedTask = timeSheet.GetHoursWorkedAsync(employee.Id);
int hoursWorked = await hoursWorkedTask;
AddPayment(hourlyRate * hoursWorked);
```
你如何编写代码是另一回事。如果你觉得单语句版本更容易阅读，那没问题；如果你想全部展开，你最终会得到更多代码，但可能更容易理解和调试。你可以决定使用第三种形式，看起来类似但又不完全相同：
```csharp
Task<decimal> hourlyRateTask = employee.GetHourlyRateAsync();
Task<int> hoursWorkedTask = timeSheet.GetHoursWorkedAsync(employee.Id);
AddPayment(await hourlyRateTask * await hoursWorkedTask);
```
我发现这是最易读的形式，并且也有潜在的性能优势。我们将在 5.10.2 节再次讨论这个例子。

本节的关键要点是，你需要能够弄清楚正在等待的是什么以及何时等待。在这个例子中，等待的是从 `GetHourlyRateAsync` 和 `GetHoursWorkedAsync` 返回的任务。在每种情况下，它们都在执行 `AddPayment` 调用之前被等待，这是合理的，因为你需要中间结果以便将它们相乘，并将乘法的结果作为参数传递。如果这是使用同步调用，所有这些都会很明显；我的目的是揭开等待部分的神秘面纱。现在你知道了如何将复杂代码简化为你正在等待的值以及何时等待它，你可以继续了解当你在等待部分本身时会发生什么。

## **`await` 表达式的求值**

当执行到达 `await` 表达式时，有两种可能：要么你正在等待的异步操作已经完成，要么还没有。如果操作已经完成，执行流程很简单：继续执行。如果操作失败并捕获了表示该失败的异常，则抛出该异常。否则，获取操作的任何结果（例如，从 `Task<string>` 中提取字符串）并继续执行程序的下一部分。所有这些都是在没有任何线程上下文切换或向任何东西附加延续的情况下完成的。

在更有趣的场景中，异步操作仍在进行中。在这种情况下，方法会异步等待操作完成，然后在适当的上下文中继续。这种异步等待实际上意味着该方法完全不在执行。将一个延续附加到异步操作上，然后方法返回。异步基础设施确保延续在正确的线程上执行：通常，要么是线程池线程（使用哪个线程并不重要），要么是 UI 线程（这在 UI 线程上有意义）。这取决于同步上下文（在 5.2.2 节中讨论），并且也可以使用 `Task.ConfigureAwait` 来控制，我们将在 5.10.1 节中讨论。

> **返回与完成**
> 描述异步行为最困难的部分可能是谈论方法何时返回（无论是返回给原始调用者还是返回给调用延续的任何东西）以及方法何时完成。与大多数方法不同，异步方法可以返回多次——实际上，是在它暂时没有更多工作可做时。
>
> 回到我们之前的外卖披萨的类比，如果你有一个 `EatPizzaAsync` 方法，包括打电话给披萨公司下单、迎接送货员、等待披萨稍微冷却，最后吃掉它，那么该方法可能在前三个部分的每个部分之后返回，但直到披萨被吃掉才算完成。
>

从开发者的角度来看，这感觉就像方法在异步操作完成时被暂停了。编译器确保方法中使用的所有局部变量在延续之前具有与之前相同的值，就像它对迭代器块所做的那样。

让我们看一个两种情况下的例子，使用一个等待两个任务的简单控制台应用程序。`Task.FromResult` 总是返回一个已完成的任务，而 `Task.Delay` 返回一个在指定延迟后完成的任务。

**清单 5.4 等待已完成和未完成的任务**
```csharp
static void Main()
{
    Task task = DemoCompletedAsync(); // 调用异步方法
    Console.WriteLine("Method returned");
    task.Wait(); // 阻塞直到任务完成
    Console.WriteLine("Task completed");
}
static async Task DemoCompletedAsync()
{
    Console.WriteLine("Before first await");
    await Task.FromResult(10); // 等待已完成的任务
    Console.WriteLine("Between awaits");
    await Task.Delay(1000); // 等待未完成的任务
    Console.WriteLine("After second await");
}
```
清单 5.4 的输出如下：
```
Before first await
Between awaits
Method returned
After second await
Task completed
```
顺序的重要方面如下：
- 异步方法在等待已完成的任务时不会返回；方法继续同步执行。这就是为什么你看到前两行之间没有任何内容。
- 异步方法在等待延迟任务时确实返回了。这就是为什么第三行是 `Method returned`，在 `Main` 方法中打印。异步方法可以判断它正在等待的操作（延迟任务）尚未完成，因此它返回以避免阻塞。
- 从异步方法返回的任务只有在方法完成时才完成。这就是为什么 `Task completed` 在 `After second await` 之后打印。

我已经尝试在图 5.6 中捕捉 `await` 表达式的流程，尽管传统的流程图并非真正为异步行为而设计。

![image-20260118232448310](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260118232448402.png)

**图 5.6 用户可见的 await 处理模型**

你可以将虚线视为作为替代方案进入流程图顶部的另一条线。注意我假设 `await` 表达式的目标有一个结果。如果你正在等待一个普通的 `Task` 或类似的东西，获取结果实际上意味着检查操作是否成功完成。

值得停下来简要思考一下从异步方法返回意味着什么。同样，存在两种可能性：
1. 这是你第一次不得不等待的 `await` 表达式，所以你的堆栈中某处仍有原始调用者。（记住，在你真正需要等待之前，方法是同步执行的。）
2. 你已经等待了其他尚未完成的东西，所以你正处于被某个东西调用的延续中。你的调用堆栈几乎肯定与你首次进入方法时看到的堆栈有了显著变化。

在第一种情况下，你通常最终会返回一个 `Task` 或 `Task<TResult>` 给调用者。显然，你还不知道方法的结果；即使没有这样的值需要返回，你也不知道方法是否会无异常地完成。因此，你将要返回的任务必须是一个未完成的任务。

在后一种情况下，回调给你的东西取决于你的上下文。例如，在 Windows Forms UI 中，如果你在 UI 线程上启动异步方法并且没有故意切换离开它，那么整个方法都将在 UI 线程上执行。对于方法的第一部分，你将在某个事件处理程序或其他什么东西中——无论是什么启动了异步方法。然而，稍后，你将被 Windows Forms 内部机制（通常称为消息泵）相当直接地回调，就像你使用 `Control.BeginInvoke(continuation)` 一样。这里，调用代码——无论是 Windows Forms 消息泵、线程池机制的一部分还是其他东西——并不关心你的任务。

提醒一下，在你遇到第一个真正异步的 `await` 表达式之前，方法完全是同步执行的。调用异步方法并不像是在单独线程中启动新任务，你需要确保始终编写异步方法以便它们快速返回。诚然，这取决于你编写代码的上下文，但通常应避免在异步方法中执行长时间运行的阻塞工作。将其分离到另一个方法中，然后为该另一个方法创建任务。

我想简要回顾一下你正在等待的值已经完成的情况。你可能想知道为什么一个立即完成的操作会首先用异步来表示。这有点像在 LINQ 中调用序列上的 `Count()` 方法：在一般情况下，你可能需要遍历序列中的每个项目，但在某些情况下（例如当序列结果是 `List<T>` 时），有一个简单的优化可用。拥有一个涵盖两种场景的单一抽象是很有用的，但无需支付执行时间的代价。

作为异步 API 案例中的一个真实例子，考虑从与磁盘上的文件关联的流中异步读取。你可能想要读取的所有数据可能已经从磁盘获取到内存中，也许是作为之前 `ReadAsync` 调用请求的一部分，因此立即使用它而不经过所有其他异步机制是有意义的。另一个例子，你可能在你的架构中有一个缓存；如果你有一个异步操作从内存缓存中获取值（返回已完成的任务）或访问存储（返回一个未完成的任务，当存储调用完成时它将完成），那么这可以是透明的。既然你了解了流程的基础知识，你就可以看到可等待模式如何融入这个拼图。

## **可等待模式成员的使用**

在 5.4.1 节中，我描述了类型必须实现的可等待模式，以便你能够等待该类型的表达式。现在，你可以将模式的不同部分映射到你试图实现的行为上。图 5.7 与图 5.6 相同，但稍微扩展并用可等待模式的术语重新表述，取代了通用描述。

![image-20260118232954944](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260118232955038.png)

以这种方式书写时，你可能会想，这有什么大不了的；为什么值得语言支持呢？然而，附加延续比你想象的要复杂。在简单情况下，当控制流完全线性时（做一些工作，等待某物，做更多工作，等待其他东西），即使不愉快，你也可以很容易地想象延续可能看起来像 lambda 表达式。然而，一旦代码包含循环或条件，并且你希望将代码保留在一个方法内，生活就会变得复杂得多。这正是 async/await 真正发挥优势的地方。尽管你可以争辩说编译器只是在应用语法糖，但手动创建延续和让编译器为你这样做之间，在可读性上存在巨大差异。

到目前为止，我已经描述了所有我们等待的值都成功完成的愉快路径。失败时会发生什么？





## **异常解包**

在 .NET 中，表示失败的惯用方式是通过异常。与向调用者返回值一样，异常处理也需要语言的额外支持。

当你等待一个已经失败的异步操作时，它可能很久以前就在一个完全不同的线程上失败了。将异常沿堆栈向上传播的常规同步方式不会自然发生。相反，async/await 基础设施采取措施，使处理异步失败的体验尽可能接近同步失败。如果你把失败看作是另一种结果，那么异常和返回值以类似方式处理就说得通了。你将在 5.6.5 节看到异常是如何从异步方法传播出去的，但在此之前，你将看到当你等待一个失败的操作时会发生什么。

就像等待器的 `GetResult()` 方法用于在有返回值时获取返回值一样，它也负责将异步操作中的任何异常传播回方法。这并不像听起来那么简单，因为在异步世界中，单个 `Task` 可以表示多个操作，从而导致多个失败。尽管还有其他可等待模式的实现可用，但值得特别考虑 `Task` 和 `Task<TResult>`，因为它们是你绝大部分时间可能等待的类型。

`Task` 和 `Task<TResult>` 通过多种方式指示失败：
- 当异步操作失败时，任务的 `Status` 变为 `Faulted`（并且 `IsFaulted` 返回 `true`）。
- `Exception` 属性返回一个 `AggregateException`，其中包含导致任务失败的所有（可能多个）异常；如果任务未出错，则返回 `null`。
- `Wait()` 方法在任务最终处于故障状态时抛出 `AggregateException`。
- `Task<TResult>` 的 `Result` 属性（同样会等待完成）同样抛出 `AggregateException`。

此外，任务通过 `CancellationTokenSource` 和 `CancellationToken` 支持取消的概念。如果任务被取消，`Wait()` 方法和 `Result` 属性将抛出包含 `OperationCanceledException`（实际上是从 `OperationCanceledException` 派生的 `TaskCanceledException`）的 `AggregateException`，但状态会变为 `Canceled` 而不是 `Faulted`。

当你等待一个任务时，如果它出错或被取消，将抛出一个异常，但不是 `AggregateException`。为了方便（在大多数情况下），将抛出 `AggregateException` 中的第一个异常。在大多数情况下，这正是你想要的。这符合异步功能的宗旨，即允许你编写看起来很像你原本会编写的同步代码的异步代码。例如，考虑以下清单，它尝试一次获取一个 URL，直到其中一个成功或尝试完所有 URL。

**清单 5.5 获取网页时捕获异常**
```csharp
async Task<string> FetchFirstSuccessfulAsync(IEnumerable<string> urls)
{
    var client = new HttpClient();
    foreach (string url in urls)
    {
        try
        {
            return await client.GetStringAsync(url); // 成功则返回字符串
        }
        catch (HttpRequestException exception) // 否则捕获并显示失败
        {
            Console.WriteLine("Failed to fetch {0}: {1}",
                              url, exception.Message);
        }
    }
    throw new HttpRequestException("No URLs succeeded");
}
```
暂时忽略你丢失了所有原始异常并且你是顺序获取所有页面的事实。我想说明的一点是，在这里你期望捕获 `HttpRequestException`；你正在用 `HttpClient` 尝试一个异步操作，如果某些地方失败，它会抛出一个 `HttpRequestException`。你想捕获并处理它，对吧？这当然感觉像是你想做的事——但 `GetStringAsync()` 调用不能因为服务器超时等错误而抛出 `HttpRequestException`，因为该方法只启动操作。当它发现错误时，方法已经返回。它所能做的就是返回一个最终出错并包含 `HttpRequestException` 的任务。如果你只是在任务上调用 `Wait()`，将会抛出一个包含 `HttpRequestException` 的 `AggregateException`。而任务等待器的 `GetResult` 方法会改为抛出 `HttpRequestException`，并像平常一样被 `catch` 块捕获。

当然，这可能会丢失信息。如果一个故障任务中有多个异常，`GetResult` 只能抛出其中一个，并且它任意使用第一个。你可能想重写前面的代码，以便在失败时，调用者可以捕获一个 `AggregateException` 并检查失败的所有原因。重要的是，一些框架方法确实这样做。例如，`Task.WhenAll()` 是一个异步等待多个任务（在方法调用中指定）完成的方法。如果其中任何一个失败，结果就是一个包含所有故障任务异常的失败。但如果你只等待 `WhenAll()` 返回的任务，你将只看到第一个异常。通常，如果你想详细检查异常，最简单的方法是对每个原始任务使用 `Task.Exception`。

总而言之，你知道等待器类型的 `GetResult()` 方法用于在等待时传播成功的结果和异常。对于 `Task` 和 `Task<TResult>`，`GetResult()` 会解包故障任务的 `AggregateException`，抛出其第一个内部异常。这解释了一个异步方法如何消费另一个异步操作——但它如何将自己的结果传播给调用代码呢？

## **方法完成**

让我们回顾几点：
- 异步方法通常在完成之前返回。
- 一旦遇到 `await` 表达式，且正在等待的操作尚未完成，它就会返回。
- 假设它不是返回 `void` 的方法（这种情况下调用者没有简单的方法来了解情况），方法返回的值将是某种任务类型：C# 7 之前是 `Task` 或 `Task<TResult>`，C# 7 及之后可以选择自定义任务类型（将在 5.8 节解释）。为简单起见，我们暂时假设它是 `Task<TResult>`。
- 该任务负责指示异步方法何时以及如何完成。如果方法正常完成，任务状态变为 `RanToCompletion`，`Result` 属性持有返回值。如果方法体抛出异常，任务状态变为 `Faulted`（或根据异常变为 `Canceled`），并且异常被包装进任务的 `Exception` 属性的 `AggregateException` 中。
- 当任务状态变为这些最终状态中的任何一种时，与之关联的任何延续（例如等待该任务的任何异步方法中的代码）都可以被调度运行。

**是的，这听起来像是重复**
你可能想知道你是否不小心跳回了几页并读了两遍。你刚才在等待某些东西时不是看过相同的概念吗？

完全正确。我所做的只是展示异步方法如何指示其完成，而不是 `await` 表达式如何检查其他东西是否已完成。如果这两者感觉不一样，那就奇怪了，因为异步方法通常是链式调用的：你在一个异步方法中等待的值很可能就是另一个异步方法返回的值。用更专业的术语来说，异步操作易于组合。

所有这些都是编译器在大量基础设施的帮助下为你完成的。下一章你会看到其中的一些细节（尽管不是每一个角落；即使我也有极限）。本章更多是关于你可以在代码中依赖的行为。

**成功返回**
成功的情况是最简单的：如果方法声明为返回 `Task<TResult>`，`return` 语句必须提供类型为 `T`（或可以转换为 `TResult` 的类型）的值，并且异步基础设施会将其传播给任务。如果返回类型是 `Task` 或 `void`，任何 `return` 语句必须是 `return;` 的形式（不带值），或者让执行到达方法末尾也是可以的，就像非异步的 `void` 方法一样。在这两种情况下，都没有要传播的值，但任务的状态会相应地改变。

**延迟的异常与参数验证**
关于异常最重要的一点是，异步方法从不直接抛出异常。即使方法体做的第一件事就是抛出异常，它也会返回一个故障任务。（在这种情况下，任务会立即变为故障状态。）这在参数验证方面有点麻烦。假设你想在验证参数非空后，在异步方法中执行一些工作。如果你像在正常同步代码中一样验证参数，调用者将无法获得任何问题指示，直到任务被等待。下面的清单给出了一个例子。

**清单 5.6 异步方法中无效的参数验证**
```csharp
static async Task MainAsync()
{
    Task<int> task = ComputeLengthAsync(null); // 故意传递错误参数
    Console.WriteLine("Fetched the task");
    int length = await task; // 等待结果
    Console.WriteLine("Length: {0}", length);
}
static async Task<int> ComputeLengthAsync(string text)
{
    if (text == null)
    {
        throw new ArgumentNullException("text"); // 尽早抛出异常
    }
    await Task.Delay(500); // 模拟真正的异步工作
    return text.Length;
}
```
输出显示在失败之前打印了 `Fetched the task`。异常在写入该输出之前就已经同步抛出，因为在验证之前没有 `await` 表达式，但调用代码在等待返回的任务之前不会看到它。

一些参数验证可以明智地提前完成，而无需花费很长时间（或引发其他异步操作）。在这些情况下，最好在系统陷入更多麻烦之前立即报告失败。例如，`HttpClient.GetStringAsync` 如果传递给它一个空引用，会立即抛出异常。

> **注意**：如果你曾经写过需要验证参数的迭代器方法，这听起来可能很熟悉。不完全一样，但有类似的效果。在迭代器块中，方法中的任何代码，包括参数验证，在第一次调用方法返回的序列的 `MoveNext()` 之前根本不会执行。在异步情况下，参数验证会立即发生，但在你等待结果之前，异常不会显现出来。

你可能不太担心这个。在许多情况下，积极的参数验证可能被认为是一个锦上添花的功能。出于实用主义，我在自己的代码中对此确实不那么挑剔了；在大多数情况下，时间上的差异并不十分重要。但如果你确实想从一个返回任务的方法中同步抛出异常，你有三个选项，它们都是同一主题的变体。

思路是编写一个非异步方法，它返回一个任务，并通过验证参数然后调用一个单独的异步函数来实现，该异步函数假设参数已经过验证。三种变体在于异步函数的表示方式：
- 你可以使用一个单独的异步方法。
- 你可以使用一个异步匿名函数（将在下一节看到）。
- 在 C# 7 及以上版本中，你可以使用一个本地异步函数。

我倾向于最后一种；它的优点是不必在类中引入另一个方法，也没有创建委托的缺点。清单 5.7 显示了第一个选项，因为它不依赖于我们尚未涵盖的任何内容，但其他选项的代码是类似的（在本书的可下载代码中）。这只是 `ComputeLengthAsync` 方法；调用代码不需要更改。

**清单 5.7 使用单独方法进行积极的参数验证**
```csharp
static Task<int> ComputeLengthAsync(string text)
{
    if (text == null)
    {
        throw new ArgumentNullException("text"); // 非异步方法，所以异常不会包装在任务中
    }
    return ComputeLengthAsyncImpl(text); // 验证后，委托给实现方法
}
static async Task<int> ComputeLengthAsyncImpl(string text) // 实现异步方法假设输入已验证
{
    await Task.Delay(500);
    return text.Length;
}
```
现在，当使用空参数调用 `ComputeLengthAsync` 时，异常会同步抛出，而不是返回一个故障任务。

在继续讨论异步匿名函数之前，让我们简要回顾一下取消。我已经顺便提过几次，但值得更详细地考虑一下。

**处理取消**
任务并行库（TPL）在 .NET 4 中使用两种类型引入了统一的取消模型：`CancellationTokenSource` 和 `CancellationToken`。思路是你可以创建一个 `CancellationTokenSource`，然后从中获取一个 `CancellationToken`，并传递给异步操作。你只能对源执行取消操作，但会反映到令牌上。（因此，你可以将同一个令牌传递给多个操作，而不必担心它们相互干扰。）使用取消令牌有多种方式，但最惯用的方法是调用 `ThrowIfCancellationRequested`，如果令牌已被取消，它会抛出 `OperationCanceledException`，否则什么都不做。如果同步调用（如 `Task.Wait`）被取消，也会抛出同样的异常。

这与异步方法的交互在 C# 规范中未作规定。根据规范，如果异步方法体抛出任何异常，该方法返回的任务将处于故障状态。“故障”的确切含义是实现定义的，但实际上，如果异步方法抛出 `OperationCanceledException`（或派生异常类型，如 `TaskCanceledException`），返回的任务最终将处于 `Canceled` 状态。你可以通过直接抛出 `OperationCanceledException` 而不使用任何取消令牌来证明，是异常的类型决定了任务状态。

**清单 5.8 通过抛出 OperationCanceledException 创建取消的任务**
```csharp
static async Task ThrowCancellationException()
{
    throw new OperationCanceledException();
}
...
Task task = ThrowCancellationException();
Console.WriteLine(task.Status); // 输出 Canceled
```
这会输出 `Canceled`，而不是你可能根据规范预期的 `Faulted`。如果你在任务上调用 `Wait()` 或请求其结果（对于 `Task<TResult>`），异常仍然会在 `AggregateException` 中抛出，因此你不需要对你使用的每个任务都显式开始检查取消。

**竞争条件？**
你可能想知道清单 5.8 中是否存在竞争条件。毕竟，你调用了一个异步方法，然后立即期望状态是固定的。如果这段代码启动了一个新线程，那将是危险的——但它没有。请记住，在第一个 `await` 表达式之前，异步方法是同步运行的。它仍然执行结果和异常的包装，但它在异步方法中这一事实并不意味着必然涉及更多线程。`ThrowCancellationException` 方法不包含任何 `await` 表达式，因此整个方法同步运行；你知道在它返回时我们将得到一个结果。Visual Studio 会对任何不包含 `await` 表达式的异步函数发出警告，但在这种情况下，这正是你想要的。

重要的是，如果你等待一个被取消的操作，会抛出原始的 `OperationCanceledException`。因此，除非你采取任何直接行动，从异步方法返回的任务也将被取消；取消会以自然的方式传播。

祝贺你坚持到这里。你现在已经学完了本章大部分困难的内容。你还有几个特性要学习，但它们比前面的部分容易理解得多。下一章当我们剖析编译器在幕后的工作时，又会变得艰难，但现在你可以享受相对的简单了。

下面是**完整、技术准确、偏教材风格**的中文翻译。我保持了原有章节结构、编号、说明文字和代码含义，便于你对照原文学习。

# **异步匿名函数（Asynchronous Anonymous Functions）**

我不会在异步匿名函数上花太多时间。正如你所预期的那样，它们是两种特性的结合：**匿名函数**（lambda 表达式和匿名方法）与 **异步函数**（可以包含 `await` 表达式的代码）。它们允许你创建表示异步操作的委托。到目前为止你学到的所有关于异步方法的内容，同样适用于异步匿名函数。

> **注意**
> 如果你在好奇：你**不能**使用异步匿名函数来创建表达式树（expression trees）。

你可以像创建普通匿名方法或 lambda 表达式那样创建异步匿名函数，只需要在开头加上 `async` 修饰符即可。下面是一个示例：

```csharp
Func<Task> lambda = async () => await Task.Delay(1000);

Func<Task<int>> anonMethod = async delegate()
{
    Console.WriteLine("Started");
    await Task.Delay(1000);
    Console.WriteLine("Finished");
    return 10;
};
```

你创建的委托，其签名必须具有**适合作为异步方法返回类型**的返回值类型（在 C# 5 和 C# 6 中是 `void`、`Task` 或 `Task<TResult>`，在 C# 7 中还可以是自定义任务类型）。

和其他匿名函数一样，你可以捕获变量、添加参数。另外，**异步操作并不会在创建委托时启动，而是在调用该委托时才开始**；多次调用同一个委托会创建多个异步操作。

需要注意的是：**调用委托本身就会启动异步操作**。就像调用一个 `async` 方法一样，并不是 `await` 任务才启动操作的，而且你完全可以不对异步匿名函数的返回结果使用 `await`。下面的代码清单展示了一个稍微完整一点（但依然没什么实际用途）的示例。

**代码清单 5.9**
*使用 lambda 表达式创建并调用异步函数*

```csharp
Func<int, Task<int>> function = async x =>
{
    Console.WriteLine("Starting... x={0}", x);
    await Task.Delay(x * 1000);
    Console.WriteLine("Finished... x={0}", x);
    return x * 2;
};

Task<int> first = function(5);
Task<int> second = function(3);

Console.WriteLine("First result: {0}", first.Result);
Console.WriteLine("Second result: {0}", second.Result);
```

我特意选择了这些参数值，使得第二个操作比第一个更快完成。但由于你在打印结果之前先等待第一个任务完成（使用了 `Result` 属性，它会阻塞直到任务完成——再次提醒，要小心在什么地方这样做），所以输出结果如下：

```
Starting... x=5
Starting... x=3
Finished... x=3
Finished... x=5
First result: 10
Second result: 6
```

这一切的行为都和你把这些异步代码放进一个异步方法中**完全一致**。

我写过的 `async` 方法远多于异步匿名函数，但它们在某些场景下确实很有用，尤其是和 **LINQ** 一起使用时。你不能在 LINQ 查询表达式中使用它们，但可以直接调用等价的方法。不过它们也有局限性：因为异步函数**永远不能返回 `bool`**，例如你就无法在 `Where` 中使用异步函数。

我最常见的用法是使用 `Select`，将一种类型的任务序列转换为另一种类型的任务序列。接下来，我将介绍一个我已经多次提到的特性：**C# 7 引入的额外一层泛化能力**。



# **C# 7 中的自定义任务类型（Custom Task Types）**

在 C# 5 和 C# 6 中，异步函数（即 `async` 方法和异步匿名函数）只能返回 `void`、`Task` 或 `Task<TResult>`。C# 7 稍微放宽了这一限制，允许任何**按照特定方式进行修饰的类型**作为异步函数的返回类型。

需要提醒的是，`async/await` 一直都允许我们 `await` 符合 *awaitable 模式* 的自定义类型。这里的新特性在于：**你现在可以编写返回自定义类型的 async 方法**。

这件事既复杂又简单。复杂之处在于：如果你真的想创建自己的任务类型，需要做不少繁琐的工作，这并不适合胆小者。简单之处在于：你几乎肯定不会在实验之外的场景中这么做——你真正想用的类型是 `ValueTask<TResult>`。下面我们就来看它。

## **99.9% 的使用场景：`ValueTask<TResult>`**

在本文写作时，`System.Threading.ValueTask<TResult>` 类型仅在 `netcoreapp2.0` 框架中原生提供，但它也可以通过 NuGet 上的 `System.Threading.Tasks.Extensions` 包获得，这使它适用于更广泛的平台（最重要的是，该包包含对 `netstandard1.0` 的支持）。

`ValueTask<TResult>` 的概念非常简单：它就像 `Task<TResult>`，但它是一个**值类型（struct）**。它提供了一个 `AsTask` 方法，在你需要时可以将其转换为普通的 `Task`（例如作为 `Task.WhenAll` 或 `Task.WhenAny` 的一个元素）。但在大多数情况下，你会像 `Task` 一样直接 `await` 它。

那么，`ValueTask<TResult>` 相比 `Task<TResult>` 的优势是什么呢？**关键在于堆分配和垃圾回收（GC）**。

`Task<TResult>` 是一个类。尽管在某些情况下，异步基础设施会重用已完成的 `Task<TResult>` 对象，但大多数异步方法仍然需要创建新的 `Task<TResult>`。在 .NET 中，分配对象的成本已经相对较低，在很多场景下你不需要关心；但如果你频繁这样做，或者在性能要求极高的环境中，就需要尽可能避免这些分配。

如果一个 async 方法在 `await` 一个尚未完成的操作，那么对象分配是不可避免的。方法会立即返回，但它必须安排一个延续（continuation），在被等待的操作完成后继续执行方法的其余部分。在大多数 async 方法中，这正是常见情况——你通常不会指望被等待的操作在 `await` 之前就完成。在这些场景下，`ValueTask<TResult>` 并没有任何优势，甚至可能略微更昂贵。

但在少数情况下，**“已经完成”的情况才是最常见的**，这正是 `ValueTask<TResult>` 发挥作用的地方。为了说明这一点，我们来看一个简化的真实世界示例。

假设你希望从 `System.IO.Stream` 中**一次读取一个字节**，并且是异步读取。你可以在底层流之上加一层缓冲，以避免频繁调用 `ReadAsync`，但你仍然希望有一个 async 方法，在必要时从流中填充缓冲区，然后返回下一个字节。你可以使用 `byte?`，并用 `null` 表示已到达数据末尾。

这个方法本身并不难写，但如果每次调用都要分配一个新的 `Task<byte?>`，就会对垃圾回收器造成很大的压力。使用 `ValueTask<TResult>` 后，只有在极少数需要从流中重新填充缓冲区的情况下，才需要进行堆分配。

下面的代码清单展示了这个包装类型（`ByteStream`）以及一个使用示例。

**代码清单 5.10**
*为高效的异步逐字节访问而包装流*

```csharp
public sealed class ByteStream : IDisposable
{
    private readonly Stream stream;
    private readonly byte[] buffer;

    // 下一个要返回的缓冲区索引
    private int position;

    // 缓冲区中已读取的字节数
    private int bufferedBytes;

    public ByteStream(Stream stream)
    {
        this.stream = stream;
        // 一个 8KB 的缓冲区意味着你很少需要 await
        buffer = new byte[1024 * 8];
    }

    public async ValueTask<byte?> ReadByteAsync()
    {
        // 如有必要，重新填充缓冲区
        if (position == bufferedBytes)
        {
            position = 0;
            bufferedBytes = await stream
                .ReadAsync(buffer, 0, buffer.Length)
                .ConfigureAwait(false);

            if (bufferedBytes == 0)
            {
                // 表示已到达流末尾
                return null;
            }
        }

        // 从缓冲区返回下一个字节
        return buffer[position++];
    }

    public void Dispose()
    {
        stream.Dispose();
    }
}
```

**示例用法：**

```csharp
using (var stream = new ByteStream(File.OpenRead("file.dat")))
{
    while ((nextByte = await stream.ReadByteAsync()).HasValue)
    {
        ConsumeByte(nextByte.Value);
        // 以某种方式使用该字节
    }
}
```

目前你可以先忽略 `ReadByteAsync` 中的 `ConfigureAwait` 调用；在第 5.10 节中，当你学习如何高效使用 `async/await` 时会再回到这个问题。其余代码都很直观，而且完全可以在不使用 `ValueTask<TResult>` 的情况下实现——只是效率会低得多。

在这个例子中，大多数对 `ReadByteAsync` 的调用甚至不会真正使用 `await`，因为缓冲区中通常仍然有数据可返回；如果你在等待的是一个通常会立即完成的操作，这种模式同样适用。正如我在 5.6.2 节中解释过的那样，当你 `await` 一个已经完成的操作时，执行会同步继续进行，这意味着不需要安排延续，也就可以避免对象分配。

这是 Google.Protobuf 包中 `CodedInputStream` 类（Google Protocol Buffers 的 .NET 实现）的一个简化原型。现实中有多个方法，每次同步或异步地读取少量数据。反序列化一个包含大量整数字段的消息时，可能会涉及大量方法调用，如果每次异步方法都返回一个新的 `Task<TResult>`，其性能开销将是难以接受的。

> **注意**
> 你可能会想：如果一个 async 方法不返回值（通常返回 `Task`），但仍然属于“无需调度延续即可完成”的情况，该怎么办？
> 在这种情况下，你可以继续返回 `Task`：`async/await` 基础设施会缓存一个任务实例，用于任何同步完成且未抛出异常的 `Task` 返回方法。如果方法同步完成但抛出了异常，那么分配一个 `Task` 的成本相比异常本身的开销也就微不足道了。

对大多数人来说，**能够将 `ValueTask<TResult>` 作为 async 方法的返回类型**，才是 C# 7 在异步方面带来的真正收益。不过，这一特性是以一种通用方式实现的，也就是说，你也可以为 async 方法创建**完全自定义的返回类型**。

## **0.1% 的使用场景：构建你自己的自定义任务类型**

我想再次强调，你**几乎肯定永远不会需要**这些信息。我甚至不会尝试给出一个除 `ValueTask<TResult>` 之外的使用场景，因为我能想到的任何例子都非常冷门。话虽如此，如果不展示编译器用来判断某个类型是否为“任务类型”的模式，这本书就不完整了。在下一章中，当你查看为 `async` 方法生成的代码时，我会展示编译器是如何使用这一模式的具体细节。

显然，一个自定义任务类型必须实现 *awaitable* 模式，但这只是其中的一小部分。要创建一个自定义任务类型，你还必须编写一个**对应的 builder 类型**，并使用
`System.Runtime.CompilerServices.AsyncMethodBuilderAttribute`
来告诉编译器这两个类型之间的关系。

这是一个新引入的特性，该特性所在的特性类与 `ValueTask<TResult>` 位于同一个 NuGet 包中。不过，如果你不想引入额外的依赖，也可以自己声明这个特性（放在正确的命名空间中，并提供合适的 `BuilderType` 属性）。编译器会接受这种方式，将其作为装饰任务类型的合法手段。

任务类型可以是**带一个泛型类型参数的泛型类型**，也可以是**非泛型类型**。如果是泛型类型，那么这个类型参数必须与 awaiter 类型中 `GetResult` 方法的返回类型一致；如果是非泛型类型，那么 `GetResult` 必须返回 `void`。²
builder 类型必须在是否泛型这一点上与任务类型保持一致。

builder 类型是编译器在编译返回你自定义任务类型的方法时，与用户代码进行交互的部分。编译器需要知道如何创建你的自定义任务、如何传播完成或异常、如何在延续（continuation）之后恢复执行，等等。你需要提供的方法和属性集合**要比 awaitable 模式复杂得多**。最简单的方式就是展示一个完整的示例，只列出你需要提供的成员，而不关心它们的具体实现。

**代码清单 5.11**
*泛型任务类型所需成员的骨架结构*

```csharp
[AsyncMethodBuilder(typeof(CustomTaskBuilder<>))]
public class CustomTask<T>
{
    public CustomTaskAwaiter<T> GetAwaiter();
}

public class CustomTaskAwaiter<T> : INotifyCompletion
{
    public bool IsCompleted { get; }
    public T GetResult();
    public void OnCompleted(Action continuation);
}

public class CustomTaskBuilder<T>
{
    public static CustomTaskBuilder<T> Create();

    public void Start<TStateMachine>(ref TStateMachine stateMachine)
        where TStateMachine : IAsyncStateMachine;

    public void SetStateMachine(IAsyncStateMachine stateMachine);
    public void SetException(Exception exception);
    public void SetResult(T result);

    public void AwaitOnCompleted<TAwaiter, TStateMachine>
        (ref TAwaiter awaiter, ref TStateMachine stateMachine)
        where TAwaiter : INotifyCompletion
        where TStateMachine : IAsyncStateMachine;

    public void AwaitUnsafeOnCompleted<TAwaiter, TStateMachine>
        (ref TAwaiter awaiter, ref TStateMachine stateMachine)
        where TAwaiter : INotifyCompletion
        where TStateMachine : IAsyncStateMachine;

    public CustomTask<T> Task { get; }
}
```

这段代码展示的是一个**泛型**自定义任务类型。对于非泛型类型，builder 中唯一的区别是：`SetResult` 会变成一个**无参数的方法**。

其中一个有趣的要求是 `AwaitUnsafeOnCompleted` 方法。正如你将在下一章看到的那样，编译器区分“安全等待（safe awaiting）”和“不安全等待（unsafe awaiting）”，后者依赖 awaitable 类型自身来处理上下文传播。自定义任务 builder 类型必须能够处理这两种恢复执行的方式。

> **注意**
> 这里的 *unsafe* 一词并不直接等同于 `unsafe` 关键字，尽管在含义上有相似之处——都暗示着“前方有风险，请小心”。

最后再重复一次：**除了出于兴趣，你几乎肯定不应该去做这件事**。我不指望自己会在生产代码中实现一个自定义任务类型，但我肯定会使用 `ValueTask<TResult>`，所以我仍然很庆幸这个特性存在。

说到有用的新特性，C# 7.1 还引入了一个值得一提的功能。幸运的是，它要比自定义任务类型简单得多。



# **C# 7.1 中的异步 `Main` 方法**

在很长一段时间里，C# 对程序入口点的要求一直是：

- 必须是一个名为 `Main` 的方法
- 必须是 `static`
- 返回类型必须是 `void` 或 `int`
- 必须是无参数，或只有一个 `string[]` 参数（不能是 `ref` 或 `out`）
- 必须是非泛型方法，并且声明在非泛型类型中（如果是嵌套类型，外层类型也必须是非泛型）
- 不能是没有实现的 `partial` 方法
- **不能使用 `async` 修饰符**

在 C# 7.1 中，最后一条限制被移除了，但对返回类型提出了稍微不同的要求。在 C# 7.1 中，你可以编写一个异步入口点（仍然叫 `Main`，而不是 `MainAsync`），但它的返回类型必须是 `Task` 或 `Task<int>`，分别对应同步情况下的 `void` 或 `int`。

与大多数 async 方法不同的是，**异步入口点不能返回 `void`，也不能使用自定义任务类型**。

除此之外，它就是一个普通的 async 方法。例如，下面的代码清单展示了一个异步入口点，它在两次输出之间加入了一个延迟。

**代码清单 5.12**
*一个简单的异步入口点*

```csharp
static async Task Main()
{
    Console.WriteLine("Before delay");
    await Task.Delay(1000);
    Console.WriteLine("After delay");
}
```

编译器通过生成一个**同步包装方法**来处理异步入口点，并将该包装方法标记为程序集的真正入口点。这个包装方法要么无参数，要么带有一个 `string[]` 参数，并且返回 `void` 或 `int`，具体取决于异步入口点的参数和返回类型。

包装方法会调用真正的代码，然后对返回的任务调用 `GetAwaiter()`，再对 awaiter 调用 `GetResult()`。例如，为代码清单 5.12 生成的包装方法大致如下：

```csharp
static void <Main>()
{
    Main().GetAwaiter().GetResult();
}
```

> 方法名在 C# 中是非法的，但在 IL 中是合法的。

异步入口点在编写小型工具或探索性代码时非常有用，尤其是当你使用诸如 **Roslyn** 这样的、以异步 API 为核心的库时。





# **使用建议（Usage Tips）**

本节不可能成为一份关于如何高效使用异步编程的完整指南——仅这一主题本身就足以写成一整本书。我们已经接近一个本就很长的章节的结尾，因此我克制自己，只给出**基于个人经验最重要的一些建议**。

我强烈建议你阅读其他开发者的观点。尤其是 **Stephen Cleary** 和 **Stephen Toub**，他们写了大量博客文章和技术文稿，对异步编程的许多方面进行了非常深入的探讨。本节不按重要性排序，提供的是我能在合理篇幅内给出的最有价值的建议。

## **在合适的场景下使用 `ConfigureAwait` 避免捕获上下文**

在第 5.2.2 和 5.6.2 节中，我介绍了**同步上下文（Synchronization Context）**以及它们对 `await` 运算符的影响。

例如，在 WPF 或 WinForms 的 UI 线程上运行时，如果你 `await` 一个异步操作，UI 同步上下文和 async 基础设施会确保 `await` 之后继续执行的代码仍然运行在**同一个 UI 线程**上。这正是 UI 代码所需要的行为，因为这样你就可以在 `await` 之后安全地访问 UI。

但当你在编写**库代码**——或者应用程序中**不直接操作 UI 的代码**时，你通常**不希望**回到 UI 线程，即便最初是在 UI 线程上运行的。一般来说，在 UI 线程上执行的代码越少越好，这样可以让界面更加流畅，避免 UI 线程成为性能瓶颈。

当然，如果你正在编写的是 **UI 库**，那你很可能确实希望回到 UI 线程；但大多数库——比如业务逻辑、Web 服务、数据库访问等——都不需要这样做。

`ConfigureAwait` 方法正是为此而设计的。它接受一个参数，用来决定在 `await` 时是否捕获当前上下文。实践中，我几乎只见过传入 `false` 的用法。

在库代码中，你**不应该**像下面这样写获取页面内容长度的代码：

```csharp
static async Task<int> GetPageLengthAsync(string url)
{
    var fetchTextTask = client.GetStringAsync(url);
    int length = (await fetchTextTask).Length;
    // 想象这里还有更多代码
    return length;
}
```

相反，你应该在 `client.GetStringAsync(url)` 返回的任务上调用 `ConfigureAwait(false)`，然后再 `await` 结果：

```csharp
static async Task<int> GetPageLengthAsync(string url)
{
    var fetchTextTask = client.GetStringAsync(url).ConfigureAwait(false);
    int length = (await fetchTextTask).Length;
    // 同样的额外代码
    return length;
}
```

我在这里稍微“作弊”了一下，使用了隐式类型推断。第一个例子中，`fetchTextTask` 的类型是 `Task<string>`；第二个例子中，它是 `ConfiguredTaskAwaitable<string>`。不过在我见过的大多数代码中，都会直接 `await` 结果，例如：

```csharp
string text = await client
    .GetStringAsync(url)
    .ConfigureAwait(false);
```

调用 `ConfigureAwait(false)` 的结果是：**后续的 continuation 不会被调度回原来的同步上下文**，而是在线程池线程上执行。需要注意的是：只有当任务在 `await` 时尚未完成，这种行为差异才会体现出来；如果任务已经完成，那么即使使用了 `ConfigureAwait(false)`，方法仍会同步继续执行。

因此，在**库代码中，你 `await` 的每一个任务都应该进行这种配置**。你不能只在 async 方法中对第一个任务调用 `ConfigureAwait(false)`，然后指望后面的代码自动都在线程池线程上执行。

这意味着你在编写库代码时必须非常小心。我预计未来可能会出现更好的解决方案（例如为整个程序集设置默认行为），但在目前阶段，你需要保持高度警惕。我建议使用 **Roslyn 分析器**来检测是否有遗漏 `ConfigureAwait(false)` 的地方。我个人对 `ConfigureAwaitChecker.Analyzer` 这个 NuGet 包有不错的使用体验，当然也有其他选择。

如果你担心这会对调用方造成影响，其实完全不必。假设调用方在 UI 代码中 `await GetPageLengthAsync`，然后更新界面来显示结果。即便 `GetPageLengthAsync` 内部的 continuation 在线程池线程上运行，UI 代码中的 `await` 仍然会捕获 UI 上下文，并将它自己的 continuation 调度回 UI 线程，因此 UI 依然可以安全更新。

## **通过启动多个相互独立的任务来实现并行性**

在第 5.6.1 节中，你看到过多段代码，它们的目标相同：根据员工的时薪和工作时长计算应付薪酬。最后两段代码如下：

```csharp
Task<decimal> hourlyRateTask = employee.GetHourlyRateAsync();
decimal hourlyRate = await hourlyRateTask;

Task<int> hoursWorkedTask = timeSheet.GetHoursWorkedAsync(employee.Id);
int hoursWorked = await hoursWorkedTask;

AddPayment(hourlyRate * hoursWorked);
```

以及：

```csharp
Task<decimal> hourlyRateTask = employee.GetHourlyRateAsync();
Task<int> hoursWorkedTask = timeSheet.GetHoursWorkedAsync(employee.Id);

AddPayment(await hourlyRateTask * await hoursWorkedTask);
```

除了更简洁之外，第二段代码**引入了并行性**。这两个任务可以独立启动，因为你并不需要第二个任务的输出作为第一个任务的输入。

这并不意味着 async 基础设施创建了更多线程。例如，如果这两个异步操作是 Web 服务调用，那么两个请求可以同时在网络中进行，而不会有任何线程被阻塞等待结果。

代码更短只是次要的。如果你想保留中间变量，同时又获得并行性，也完全没问题：

```csharp
Task<decimal> hourlyRateTask = employee.GetHourlyRateAsync();
Task<int> hoursWorkedTask = timeSheet.GetHoursWorkedAsync(employee.Id);

decimal hourlyRate = await hourlyRateTask;
int hoursWorked = await hoursWorkedTask;

AddPayment(hourlyRate * hoursWorked);
```

这段代码与最初版本的唯一区别在于：我交换了第二和第三行的位置。你不再是先 `await hourlyRateTask` 再启动 `hoursWorkedTask`，而是**先启动两个任务，再分别等待它们完成**。

在大多数情况下，如果你能并行执行相互独立的工作，就应该这样做。需要注意的是：如果 `hourlyRateTask` 失败了，你将无法观察到 `hoursWorkedTask` 的结果，包括其中可能发生的异常。如果你需要记录所有任务的失败情况，那么你可能应该使用 `Task.WhenAll`。

当然，这种并行化的前提是：这些任务在逻辑上确实是独立的。有时依赖关系并不那么明显。例如，如果一个任务用于用户认证，另一个任务代表以该用户身份执行某个操作，那么即便技术上可以并行执行，你也应该在确认认证成功之后再启动后续操作。`async/await` 无法替你做出这些业务决策，但在你已经决定合适的前提下，它能让并行执行异步操作变得非常简单。

## **避免混合同步代码和异步代码**

尽管异步编程并非完全“非黑即白”，但当你的代码中一部分是同步的，另一部分是异步的时，**正确实现的难度会急剧上升**。在这两种模式之间切换充满了陷阱——有些很隐蔽，有些则不那么隐蔽。

如果你有一个只暴露同步操作的网络库，那么为这些操作编写安全的异步封装是非常困难的；反过来也是如此。

尤其要警惕使用 `Task<TResult>.Result` 属性和 `Task.Wait()` 方法来**同步获取异步操作结果**的做法。这非常容易导致**死锁**。在最常见的情况下，异步操作需要在某个线程上执行 continuation，而该线程却正被阻塞着等待该操作完成。

Stephen Toub 针对这个主题写过两篇非常出色、细节丰富的博客文章：

- *Should I expose synchronous wrappers for asynchronous methods?*
- *Should I expose asynchronous wrappers for synchronous methods?*

（剧透一下：两个问题的答案都是“否”，你大概也已经猜到了。）

和所有规则一样，这里也有例外，但我强烈建议：**在你完全理解这条规则之前，不要试图打破它。

## **尽可能支持取消（Cancellation）**

取消是一个在同步代码中几乎没有强力等价物的领域——在同步代码中，你通常只能等方法返回后才能继续执行。能够取消异步操作是一种**极其强大的能力**，但它依赖于调用链中各层代码的配合。

如果你调用的某个方法不允许你传入 `CancellationToken`，那你几乎无能为力。你可以写一些相当复杂的代码，让你的 async 方法以“已取消”的状态完成，并忽略那个不可取消任务的最终结果，但这远非理想方案。你真正想要的是**停止正在进行的工作**，而且你也不希望在该异步方法最终完成时，还要担心它返回的可释放资源（`IDisposable`）该如何处理。

幸运的是，大多数底层异步 API 都提供了 `CancellationToken` 参数。你需要做的只是遵循相同的模式：通常是将你方法参数中接收到的 cancellation token 传递给你调用的所有异步方法。

即使你当前并没有明确的取消需求，我也建议你**从一开始就一致性地提供取消支持**，因为事后再补充这一能力往往非常痛苦。

同样地，Stephen Toub 也写过一篇非常优秀的博客文章，深入探讨了尝试绕过不可取消异步操作时的各种微妙问题。你可以搜索 **“How do I cancel non-cancelable async operations?”** 来找到它。



## 测试异步代码

测试异步代码可能会非常棘手，尤其是当你希望测试“异步行为本身”时更是如此。（例如回答这样的问题：“如果在该方法内部第二个和第三个异步调用之间取消操作，会发生什么？”——这类测试通常需要相当复杂的工作。）

这并非不可能，但如果你想要进行全面测试，就要做好打一场艰难仗的准备。在我撰写本书第三版时，我曾希望到 2019 年能出现一些健壮的框架，使这一切变得相对简单。不幸的是，我对此感到失望。

不过，大多数单元测试框架确实都支持异步测试，而这种支持几乎是编写异步方法测试所必需的，原因正如之前提到的：同步代码与异步代码混用会带来诸多困难。通常，编写一个异步测试就像编写一个带有 `async` 修饰符、返回 `Task` 而不是 `void` 的测试方法一样简单：

```csharp
[Test]
public async Task FooAsync()
{
}
```

（此处编写用于测试你生产代码中 `FooAsync` 方法的逻辑。）

测试框架通常还会提供 `Assert.ThrowsAsync` 方法，用于验证某个异步方法调用返回的任务最终会进入失败（faulted）状态。

在测试异步代码时，你往往需要创建一个**已经完成的任务**，并且该任务具有特定的结果或异常。此时，`Task.FromResult`、`Task.FromException` 和 `Task.FromCanceled` 这些方法就非常有用。

如果你需要更大的灵活性，可以使用 `TaskCompletionSource<TResult>`。该类型被 .NET 框架中的大量异步基础设施所使用。它本质上允许你创建一个表示“正在进行中的操作”的任务，然后在之后的某个时刻设置其结果（包括异常或取消），一旦设置完成，该任务就会结束。

当你需要从一个被模拟（mock）的依赖中返回一个任务，但又希望这个任务在测试过程中稍后才完成时，`TaskCompletionSource<TResult>` 会显得极其有用。

关于 `TaskCompletionSource<TResult>` 有一个需要特别注意的地方：**当你设置结果时，附加在该任务上的 continuation 可能会在同一线程上同步执行**。continuation 具体如何运行，取决于线程、同步上下文等多方面因素。一旦你意识到“同步执行 continuation 是有可能发生的”，就相对容易在设计测试时将这一点考虑进去。现在你已经知道这一点了，希望你能避免像我当初那样因为这个问题而感到困惑、浪费大量时间。

这只是我在过去四年左右编写异步代码过程中所学到经验的一个不完整总结，但我不想偏离本书的主题（C# 语言本身，而不是异步编程）。你已经从开发者视角看到了 `async/await` 特性的作用。虽然你还没有深入了解其底层实现细节，但 *awaitable pattern* 已经提供了一些线索。

如果你还没有真正动手使用过 `async/await`，我强烈建议你现在就去尝试一下，再继续阅读下一章。下一章将讨论异步的实现细节，这些内容本身就已经不太容易理解；如果你缺乏实际使用经验，那么理解起来会更加困难。如果你目前还没有这些经验，或者暂时不想投入时间去实践，我建议你先跳过下一章。那一章只涉及异步的实现细节；我保证你不会错过其他重要内容。



**总结**

- 异步的核心在于：启动一个操作，然后在该操作完成之后继续执行，而不是在中间阻塞等待。
- `async/await` 允许你编写看起来很熟悉、但行为是异步的代码。
- `async/await` 能够处理同步上下文，使得 UI 代码可以启动异步操作，并在操作完成后继续在 UI 线程上执行。
- 成功结果和异常都会沿着异步操作链进行传播。
- 对 `await` 运算符的使用位置存在一些限制，但 C# 6（及之后版本）的限制比 C# 5 少得多。
- 编译器通过 *awaitable pattern* 来判断哪些类型是可以被 `await` 的。
- C# 7 允许你创建自定义任务类型，但你几乎肯定只需要使用 `ValueTask<TResult>`。
- C# 7.1 允许你将 `async Main` 方法作为程序的入口点。



