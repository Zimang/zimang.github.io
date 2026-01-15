---
weight: 1
title: "迭代器块实现细节：自动生成的状态机"
---

迭代器块实现细节：自动生成的状态机



# 引言

自 .NET 首次发布以来，迭代器就已经存在，但 C# 2 通过迭代器块让它们变得易于实现。本文将深入探讨微软 C# 编译器如何将迭代器块转换为状态机的细节。如果你对 `IEnumerable` 和 `IEnumerator` 接口（及其泛型对应物）或迭代器块的基础知识不太确定，建议在继续阅读本文之前，先阅读详细介绍这些内容的文章（或《C# in Depth》的第 6 章，这是免费样章）。

本文几乎所有的内容都是实现特定的。例如，Mono 编译器可能以略有不同的方式处理事情，但很可能非常相似；当不确定可以依赖什么时，请查阅语言规范（C# 3.0 规范的第 10.14 节涵盖了迭代器块）。

# 高级概述：模式是什么？

这篇文章最初是因为一位读者询问了用于返回 `IEnumerator` 的方法与返回 `IEnumerable` 的方法使用迭代器块的区别而产生的。此外，还存在非泛型接口和泛型接口之间的选择。我们将从为四种可能性中的每一种使用相同的迭代器块开始，并查看生成代码的差异。在整篇文章中，我将展示编译器生成的代码的 C# 等效形式。显然，编译器实际上并不生成 C#，但我使用 Reflector 将代码反编译为 C#。

我们的第一个示例将按顺序产生数字 0 到 9。最初我们将方法声明为返回非泛型 `IEnumerator`。以下是完整的代码：

```csharp
using System;
using System.Collections;

class Test
{
    static IEnumerator GetCounter()
    {
        for (int count = 0; count < 10; count++)
        {
            yield return count;
        }
    }
}
```

很简单，对吧？那么，让我们看看编译后它变成了什么。屏住呼吸...

```csharp
internal class Test
{
    // 注意这里没有执行我们原始代码的任何部分
    private static IEnumerator GetCounter()
    {
        return new <GetCounter>d__0(0);
    }

    // 编译器自动创建的嵌套类型，用于实现迭代器
    [CompilerGenerated]
    private sealed class <GetCounter>d__0 : IEnumerator<object>, IEnumerator, IDisposable
    {
        // 字段：总会有一个 "state" 和 "current"，但 "count" 来自我们迭代器块中的局部变量
        private int <>1__state;
        private object <>2__current;
        public int <count>5__1;

        [DebuggerHidden]
        public <GetCounter>d__0(int <>1__state)
        {
            this.<>1__state = <>1__state;
        }

        // 几乎所有实际工作都在这里发生
        private bool MoveNext()
        {
            switch (this.<>1__state)
            {
                case 0:
                    this.<>1__state = -1;
                    this.<count>5__1 = 0;
                    while (this.<count>5__1 < 10)
                    {
                        this.<>2__current = this.<count>5__1;
                        this.<>1__state = 1;
                        return true;
                    Label_004B:
                        this.<>1__state = -1;
                        this.<count>5__1++;
                    }
                    break;

                case 1:
                    goto Label_004B;
            }
            return false;
        }

        [DebuggerHidden]
        void IEnumerator.Reset()
        {
            throw new NotSupportedException();
        }

        void IDisposable.Dispose()
        {
        }

        object IEnumerator<object>.Current
        {
            [DebuggerHidden]
            get
            {
                return this.<>2__current;
            }
        }

        object IEnumerator.Current
        {
            [DebuggerHidden]
            get
            {
                return this.<>2__current;
            }
        }
    }
}
```

我们将从较高层次开始了解发生了什么，解释每个成员。真正的工作是在 `MoveNext()` 中完成的，所以我们先略过这一部分，当我们对代码的其他部分满意后，再深入探讨这个方法。从上到下查看代码，以下是连续注释：

1.  上面显示的源代码不是完全有效的 C#——它使用了在普通 C# 中非法的名称。在生成的代码中使用非法名称是很常见的；这可以防止与普通名称发生任何冲突，同时阻止手动编写的 C# 显式使用生成的代码。
2.  调用 `GetCounter()` 只是调用 `<GetCounter>d__0` 的构造函数，这是实现迭代器的类型。构造函数又只是设置迭代器的状态。我们原始类中的任何代码都尚未执行。
3.  代码的各种元素都装饰有 `CompilerGenerated` 和 `DebuggerHidden` 属性。`CompilerGenerated` 几乎无关紧要——其 MSDN 文档提到它允许 SQL Server 额外访问，但 99.9% 的开发者真的不需要知道这一点。但为了清晰起见，看到它在那里还是很好的。`DebuggerHidden` 只是阻止调试器（那些选择遵循此属性的调试器）进入或在代码中中断。从现在开始，我不再展示这些属性。
4.  `<GetCounter>d__0`（以下简称“迭代器类型”）实现了三个接口：`IEnumerator<object>`、`IEnumerator` 和 `IDisposable`。实际上，第一个接口已经隐含了其他两个，但我在列表中保留了它们以使内容更清晰。有趣的是，即使我们原始的 `GetCounter()` 方法使用了非泛型形式，也实现了泛型接口。这可能只是为了保持一致性——我们将看到，当我们告诉编译器使用泛型版本时，变化非常小。
5.  实现 `IDisposable` 非常重要（除了为实现 `IEnumerator<object>` 所必需之外）。C# 中的 `foreach` 语句会调用任何实现了 `IDisposable` 的迭代器上的 `Dispose`；调用位于 `finally` 块中，就像 `using` 语句一样。在极少数情况下，你手动使用迭代器而不是使用 `foreach` 循环时，应使用 `using` 语句以确保调用 `Dispose`。我们稍后会看到原因。
6.  迭代器类型是嵌套的，这意味着它可以访问封闭类的所有私有成员。如果你的迭代器块调用其他私有方法或访问私有变量（当然或其他私有成员），这一点很重要。我们稍后会看到这方面的示例。
7.  迭代器类型有三个字段——两个私有和一个公有。两个私有变量跟踪要从 `Current` 属性返回的值以及迭代器所处的状态。我们稍后将更详细地研究这些。老实说，我不知道为什么与 `count` 关联的变量是公有的。当我们查看如果原始的迭代器块方法接受参数会发生什么时，我们将看到公有字段的使用——尽管不同的设计决策可能使所有内容保持私有。
8.  构造函数只设置迭代器的状态。在这种情况下，唯一调用构造函数的代码是 `GetCounter()` 方法，它总是传递 0 作为初始状态，但我们稍后将看到，在其他情况下，将状态作为参数传递是有真正原因的。
9.  `MoveNext()` 基本上只是一个大的 `switch` 语句。它总是这样，并且它切换的值是状态机中的一个状态，因此它知道接下来要执行哪部分代码。我们将花很多时间研究这一点。
10. `IEnumerator` 中的 `Reset` 方法总是抛出 `System.NotSupportedException`。这不仅仅是一个实现决策——它是 C# 规范强制要求的。
11. 在第一个简单示例中，`Dispose` 方法什么也不做。我们将以此作为一个单独的主题进行探讨。
12. `IEnumerator` 和 `IEnumerator<object>` 的 `Current` 实现都只是返回 `<>2__current` 变量的值。C# 语言规范明确规定了在“奇怪”情况下（例如在第一次调用 `MoveNext()` 之前访问它，或当 `MoveNext()` 已返回 `false` 时）`Current` 属性的行为是未定义的。

# 泛型与非泛型接口

我们已经看到了声明一个返回 `IEnumerator` 的方法如何导致迭代器同时实现 `IEnumerator<object>`。现在让我们稍微更改一下代码，使方法显式返回 `IEnumerator<int>`：

```csharp
using System;
using System.Collections.Generic;

class Test
{
    static IEnumerator<int> GetCounter()
    {
        for (int count = 0; count < 10; count++)
        {
            yield return count;
        }
    }
}
```

我不会完整发布生成的代码，但以下是一些更改。首先，也是最明显的，`GetCounter()` 方法更改了返回类型，但其他都没有改变：

```csharp
private static IEnumerator<int> GetCounter()
{
    return new <GetCounter>d__0(0);
}
```

同样，迭代器现在实现 `IEnumerator<int>` 而不是 `IEnumerator<object>`。这里涉及的类型称为迭代器的**产出类型**。每个 `yield return` 语句都必须返回可以隐式转换为产出类型的值。正如我们所见，当使用非泛型接口时，产出类型是 `object`。以下是新的类签名：

```csharp
private sealed class <GetCounter>d__0 : IEnumerator<int>, IEnumerator, IDisposable
```

类似地，实现泛型接口的 `Current` 属性和后备变量更改为 `int`：

```csharp
private int <>2__current;

int IEnumerator<int>.Current
{
    get
    {
        return this.<>2__current;
    }
}
```

除了这些微小的更改之外，该类看起来与之前相同。

# 返回 IEnumerable

如果我们将原始代码更改为返回 `IEnumerable` 或其泛型对应物，而不是 `IEnumerator`，会有更显著的变化。我们将代码更改为返回 `IEnumerable<int>` 并从现在开始坚持使用泛型接口，因为我们看到它们几乎没有区别。那么，以下是源代码：

```csharp
using System;
using System.Collections.Generic;

class Test
{
    static IEnumerable<int> GetCounter()
    {
        for (int count = 0; count < 10; count++)
        {
            yield return count;
        }
    }
}
```

……以及生成的代码（为简洁起见，去除了属性）：

```csharp
internal class Test
{
    private static IEnumerable<int> GetCounter()
    {
        return new <GetCounter>d__0(-2);
    }

    private sealed class <GetCounter>d__0 : IEnumerable<int>, IEnumerable, IEnumerator<int>, IEnumerator, IDisposable
    {
        // 字段
        private int <>1__state;
        private int <>2__current;
        private int <>l__initialThreadId;
        public int <count>5__1;

        public <GetCounter>d__0(int <>1__state)
        {
            this.<>1__state = <>1__state;
            this.<>l__initialThreadId = Thread.CurrentThread.ManagedThreadId;
        }

        private bool MoveNext()
        {
            switch (this.<>1__state)
            {
                case 0:
                    this.<>1__state = -1;
                    this.<count>5__1 = 0;
                    while (this.<count>5__1 < 10)
                    {
                        this.<>2__current = this.<count>5__1;
                        this.<>1__state = 1;
                        return true;
                    Label_0046:
                        this.<>1__state = -1;
                        this.<count>5__1++;
                    }
                    break;

                case 1:
                    goto Label_0046;
            }
            return false;
        }

        IEnumerator<int> IEnumerable<int>.GetEnumerator()
        {
            if ((Thread.CurrentThread.ManagedThreadId == this.<>l__initialThreadId) && (this.<>1__state == -2))
            {
                this.<>1__state = 0;
                return this;
            }
            return new Test.<GetCounter>d__0(0);
        }

        IEnumerator IEnumerable.GetEnumerator()
        {
            return ((IEnumerable<Int32>) this).GetEnumerator();
        }

        void IEnumerator.Reset()
        {
            throw new NotSupportedException();
        }

        void IDisposable.Dispose()
        {
        }

        int IEnumerator<int>.Current
        {
            get
            {
                return this.<>2__current;
            }
        }

        object IEnumerator.Current
        {
            get
            {
                return this.<>2__current;
            }
        }
    }
}
```

我再次展示了整个代码，以便我们可以轻松地看到差异：

1.  显然，迭代器类型现在实现了 `IEnumerable<int>`，但它仍然实现了 `IEnumerator<int>`。同时是可迭代对象和迭代器是非常奇怪的。这是对常见情况的优化，我们稍后会看到。
2.  `IEnumerator<int>` 的实现完全保持不变——`Reset` 仍然抛出异常，`Current` 仍然只返回当前值，`MoveNext()` 中的逻辑也相同。
3.  创建迭代器实例的方法向构造函数传递初始状态 -2 而不是 0。
4.  我们有一个额外的私有变量 `<>l__initialThreadId`，它在构造函数中设置为反映创建实例的线程。
5.  `GetEnumerator()` 要么将状态设置为 0 并返回 `this`，要么创建一个新的迭代器实例，该实例从状态 0 开始。

那么，这是怎么回事呢？嗯，最常见的使用情况（到目前为止）是创建一个 `IEnumerable<T>` 的实例，然后某些东西（如 `foreach` 语句）从同一线程调用 `GetEnumerator()`，遍历数据，并在最后处理掉 `IEnumerator<T>`。原始的 `IEnumerable<T>` 在初始调用 `IEnumerator<T>` 之后永远不会再被使用。考虑到这种模式的普遍性，C# 编译器选择一种优化这种情况的模式是有道理的。当是这种行为时，即使我们使用它来实现两个不同的实例，我们也只创建一个对象。状态 -2 用于表示“尚未调用 `GetEnumerator()`”，而状态 0 用于表示“我准备开始迭代，尽管 `MoveNext()` 尚未被调用”。

但是，如果你尝试从不同线程调用 `GetEnumerator()`，或者当它不处于 -2 状态时，代码必须创建一个新实例以跟踪不同的状态。在后一种情况下，你基本上有两个独立的计数器，因此它们需要独立的数据存储。`GetEnumerator()` 处理新迭代器的初始化，然后返回它准备行动。线程安全方面是为了防止两个独立的线程同时独立调用 `GetEnumerator()`，并最终得到相同的迭代器（即 `this`）。

这就是实现 `IEnumerable<T>` 时的基本模式：编译器在同一个类中实现所有接口，并且代码在必须时惰性地创建额外的迭代器。我们将看到，当涉及参数时，还有更多工作要做，但基本原则是相同的。

# 选择返回的接口

通常，`IEnumerable<T>` 是返回的最灵活的接口。如果你的迭代器块不改变任何内容，并且你的类本身没有实现 `IEnumerable<T>`（当然，在这种情况下，你必须从 `GetEnumerator()` 方法返回 `IEnumerator<T>`），这是一个不错的选择。它允许客户端使用 `foreach`、多次迭代、使用 LINQ to Objects 以及获得普遍的好处。绝对值得使用泛型接口而不是非泛型接口。从现在开始，我将在文本中仅提及非泛型接口，但每次我都指两种形式（换句话说，`IEnumerable` 和 `IEnumerator` 之间存在重要区别，但从这一点开始，我不会区分 `IEnumerable` 和 `IEnumerable<T>`）。

# 状态管理

迭代器类型需要跟踪多达 x 种状态：

1.  它的“虚拟指令指针”（即它到达的位置）
2.  局部变量
3.  参数初始值和 `this`
4.  创建线程（如上所示，并且仅在 `IEnumerable` 的情况下；我不会进一步讨论这个）
5.  最后产出的值（即 `Current`；这足够简单，不需要单独关注）

我们将依次查看前三个。

# 跟踪我们到达的位置

我们状态机中的第一个状态是跟踪我们从原始源代码中执行了多少代码。如果你想到一个正常的状态机图（有圆圈和线条），这就是我们当前所处的圆圈。在许多情况下，它只被称为状态——确实在我们迄今为止的示例反编译输出中，我们已将其视为 `<>1__state`。（这是不幸的，因为所有其余的数据也是状态，但没关系……）规范提到了之前、运行、暂停和之后的状态，但正如我们将看到的，暂停需要更多细节——并且我们需要一个额外的状态用于 `IEnumerable` 实现。

在我继续之前，值得记住的是，迭代器块并不仅仅从开始运行到结束。当最初调用方法时，只创建迭代器。只有当调用 `MoveNext()` 时（如果我们使用 `IEnumerable`，则在调用 `GetEnumerator()` 之后），才开始执行。此时，执行像往常一样从方法的顶部开始，并一直进行到第一个 `yield return` 或 `yield break` 语句，或方法的结尾。此时，返回一个布尔值以指示块是否已完成迭代。如果/当再次调用 `MoveNext()` 时，方法从紧接 `yield return` 语句之后继续执行。（如果之前的调用因任何其他原因结束，则我们已完成迭代，不会发生任何事情。）在不查看生成代码的情况下，让我们编写一个小程序来逐步执行一个简单的迭代器。以下是代码：

```csharp
using System;
using System.Collections.Generic;

class Test
{
    static readonly string Padding = new string(' ', 30);
    
    static IEnumerator<int> GetNumbers()
    {
        Console.WriteLine(Padding + "First line of GetNumbers()");
        Console.WriteLine(Padding + "Just before yield return 0");
        yield return 10;
        Console.WriteLine(Padding + "Just after yield return 0");

        Console.WriteLine(Padding + "Just before yield return 1");
        yield return 20;
        Console.WriteLine(Padding + "Just after yield return 1");
    }
    
    static void Main()
    {
        Console.WriteLine("Calling GetNumbers()");
        IEnumerator<int> iterator = GetNumbers();
        Console.WriteLine("Calling MoveNext()...");
        bool more = iterator.MoveNext();
        Console.WriteLine("Result={0}; Current={1}", more, iterator.Current);
        
        Console.WriteLine("Calling MoveNext() again...");
        more = iterator.MoveNext();
        Console.WriteLine("Result={0}; Current={1}", more, iterator.Current);

        Console.WriteLine("Calling MoveNext() again...");
        more = iterator.MoveNext();
        Console.WriteLine("Result={0} (stopping)", more);
    }
}
```

我为迭代器块中创建的输出包含了一些填充，以使结果更清晰。左边的行在调用代码中；右边的行在迭代器块中：

```
Calling GetNumbers()
Calling MoveNext()...
                              First line of GetNumbers()
                              Just before yield return 0
Result=True; Current=10
Calling MoveNext() again...
                              Just after yield return 0
                              Just before yield return 1
Result=True; Current=20
Calling MoveNext() again...
                              Just after yield return 1
Result=False (stopping)
```

现在让我们介绍 `<>1__state` 可以取的值及其含义：

*   **-2**：（仅限 `IEnumerable`）在创建线程第一次调用 `GetEnumerator()` 之前
*   **-1**：“运行中”——迭代器当前正在执行代码；也用于“之后”——迭代器已完成，无论是通过到达方法结尾还是通过遇到 `yield break`
*   **0**：“之前”——`MoveNext()` 尚未被调用
*   **任何正数**：指示从哪里恢复；它已经产出了至少一个值，并且可能还有更多。当代码仍在运行但处于具有相应 `finally` 块的 `try` 块中时，也使用正数状态。我们稍后会看到原因。

有趣的是，生成的代码并不区分“运行中”和“之后”。确实没有理由区分：如果你在迭代器处于该状态时调用 `MoveNext()`（可能是由于它在不同线程中运行），那么 `MoveNext()` 将立即返回 `false`。此状态也是我们在未捕获异常之后结束的状态。

现在我们知道状态的含义，让我们看看上述迭代器的 `MoveNext()` 是什么样子。它基本上是一个 `switch` 语句，根据状态从代码中的特定位置开始执行。对于 `MoveNext()` 来说总是这样，唯一的例外是迭代器主体仅由 `yield break` 组成。

```csharp
private bool MoveNext()
{
    switch (this.<>1__state)
    {
        case 0:
            this.<>1__state = -1;
            Console.WriteLine(Test.Padding + "First line of GetNumbers()");
            Console.WriteLine(Test.Padding + "Just before yield return 0");
            this.<>2__current = 10;
            this.<>1__state = 1;
            return true;

        case 1:
            this.<>1__state = -1;
            Console.WriteLine(Test.Padding + "Just after yield return 0");
            Console.WriteLine(Test.Padding + "Just before yield return 1");
            this.<>2__current = 20;
            this.<>1__state = 2;
            return true;

        case 2:
            this.<>1__state = -1;
            Console.WriteLine(Test.Padding + "Just after yield return 1");
            break;
    }
    return false;
}
```

这当然是一个非常简单的例子——我们只是从状态 0 开始，然后是 1，然后是 2，然后是 -1（尽管我们在每次执行代码时短暂地处于 -1 状态）。让我们再看一下我们的第一个例子。以下是原始代码：

```csharp
using System;
using System.Collections;

class Test
{
    static IEnumerator GetCounter()
    {
        for (int count = 0; count < 10; count++)
        {
            yield return count;
        }
    }
}
```

以下是 `MoveNext()` 方法，正如我们之前看到的：

```csharp
private bool MoveNext()
{
    switch (this.<>1__state)
    {
        case 0:
            this.<>1__state = -1;
            this.<count>5__1 = 0;
            while (this.<count>5__1 < 10)
            {
                this.<>2__current = this.<count>5__1;
                this.<>1__state = 1;
                return true;
            Label_004B:
                this.<>1__state = -1;
                this.<count>5__1++;
            }
            break;
         case 1:
            goto Label_004B;
    }
    return false;
}
```

是的，它使用 `goto` 语句从一个 `case` 跳到另一个 `case` 的一半处。哎呀。但别忘了这只是生成的代码。当你仔细想想，`for` 循环和 `while` 循环实际上只是比较和跳转的良好包装。我们并不真正关心这段代码在可读性方面有多糟糕，只要它工作并且性能良好。C# 编译器团队实现这两个目标的最简单方法是使用 `switch` 和 `goto` 对其进行建模。

我不打算解释所有不同的转换是如何发生的。稍后我将研究 `finally` 块的处理，有趣的是，你不能从具有 `catch` 块的 `try` 块中，或从任何 `catch` 或 `finally` 块中 `yield`（无论是返回还是中断）。重要的是，你可以从只有 `finally` 块的 `try` 块中 `yield`。这意味着你仍然可以使用 `using` 语句，这确实非常方便。尝试用不同的方式使代码分支和循环，然后查看生成的 `MoveNext()` 方法发生了什么变化是相当有趣的。然而，我无法详尽地做到这一点，而你可以非常轻松地进行实验。上面的简单示例展示了原理以及涉及的状态。让我们继续下一部分状态。

# **本地变量**

在迭代器块中，常规的局部变量处理起来非常简单。它们会成为迭代器类型中的实例变量，并且其赋值（以及赋值时机）与在常规代码中的初始化方式相同。当然，由于它们是实例变量，就不再存在“明确赋值”这一有意义的说法，但常规的编译规则会阻止你看到它们的默认值（除非你故意使用反射来干扰上述状态）。所有这些都可以在前面 `<count>5__1` 变量的示例中看到。

局部变量的特性意味着，创建迭代器实例不需要关于变量本身的任何额外信息——任何初始值都将在代码执行过程中设置。当然，这个初始值可能会依赖于非本地变量，这就引出了最后一种状态类型。

# **参数与 this**

使用迭代器块实现的方法可以接收参数，并且如果它们是实例方法，也可以使用 `this`。任何对包含迭代器块的类型的实例变量的引用，本质上就是使用 `this`，然后从该引用定位到变量。下面是一个同时包含方法参数（`max`）和对实例变量（`min`）引用的例子——我对实例变量加了 `this.` 前缀以便更清晰。

```c#
using System;
using System.Collections.Generic;

class Test
{
    int min;
    
    public Test(int min)
    {
        this.min = min;
    }
    
    public IEnumerator<int> GetCounter(int max)
    {
        for (int count = this.min; count < max; count++)
        {
            yield return count;
        }
    }
}
```

请注意，它返回的是 `IEnumerator<int>` 而不是 `IEnumerable<int>`。正如我们很快就会看到的，在使用参数和 `this` 时，这会产生更大的差异。以下是生成代码中的有趣部分：

```c#
internal class Test
{
    private int min;

    public Test(int min)
    {
        this.min = min;
    }

    public IEnumerator<int> GetCounter(int max)
    {
        <GetCounter>d__0 d__ = new <GetCounter>d__0(0);
        d__.<>4__this = this;
        d__.max = max;
        return d__;
    }

    private sealed class <GetCounter>d__0 : IEnumerator<int>, IEnumerator, IDisposable
    {
        private int <>1__state;
        private int <>2__current;
        public Test <>4__this;
        public int <count>5__1;
        public int max;

        public <GetCounter>d__0(int <>1__state)
        {
            this.<>1__state = <>1__state;
        }

        private bool MoveNext()
        {
            switch (this.<>1__state)
            {
                case 0:
                    this.<>1__state = -1;
                    this.<count>5__1 = this.<>4__this.min;
                    while (this.<count>5__1 < this.max)
                    {
                        this.<>2__current = this.<count>5__1;
                        this.<>1__state = 1;
                        return true;
                    Label_0050:
                        this.<>1__state = -1;
                        this.<count>5__1++;
                    }
                    break;

                case 1:
                    goto Label_0050;
            }
            return false;
        }

        // Other methods as normal
    }
}
```

我们在迭代器中增加了两个额外的字段。一个就叫 `max`，另一个是 `<>4__this`。在原始代码访问 `min` 的地方，生成的代码访问 `<>4__this.min`——尽管 `Test.min` 是私有的，但由于这是在一个嵌套类型中，所以可以这样访问。

有趣且（在我看来）有些违反直觉的部分是这些额外字段的初始化方式。个人而言，我会将它们作为构造函数的参数添加进去，使它们都成为私有的，并且 `<>4__this` 也是只读的。C# 团队的 Eric Lippert 解释过，负责此部分的代码与从闭包中提升捕获变量的代码是同一份——而那些变量确实需要是公共的，以便原始方法仍然可以访问它们。所以，基本上这是一个代码重用问题，而不是有什么隐蔽的原因导致我偏好的方式无法实现。这里并没有真正的危害，但我发现这类事情很迷人 :)

碰巧，我们的迭代器块没有改变 `max` 的值——但它是可以改变的。现在假设我们要返回 `IEnumerable` 而不是 `IEnumerator`。考虑到我们希望每次调用 `GetEnumerator()` 生成的迭代器都使用 `max` 的原始值，编译器是如何确保这一点的呢？以下是生成代码的有趣部分子集（源代码与之前相同，只是返回类型改变了）。

```c#
internal class Test
{
    private int min;

    public IEnumerable<int> GetCounter(int max)
    {
        <GetCounter>d__0 d__ = new <GetCounter>d__0(-2);
        d__.<>4__this = this;
        d__.<>3__max = max;
        return d__;
    }

    private sealed class <GetCounter>d__0 : IEnumerable<int>, IEnumerable, IEnumerator<int>, IEnumerator, IDisposable
    {
        private int <>1__state;
        private int <>2__current;
        public int <>3__max;
        public Test <>4__this;
        private int <>l__initialThreadId;
        public int <count>5__1;
        public int max;

        public <GetCounter>d__0(int <>1__state)
        {
            this.<>1__state = <>1__state;
            this.<>l__initialThreadId = Thread.CurrentThread.ManagedThreadId;
        }

        private bool MoveNext()
        {
            switch (this.<>1__state)
            {
                case 0:
                    this.<>1__state = -1;
                    this.<count>5__1 = this.<>4__this.min;
                    while (this.<count>5__1 < this.max)
                    {
                        this.<>2__current = this.<count>5__1;
                        this.<>1__state = 1;
                        return true;
                    Label_0050:
                        this.<>1__state = -1;
                        this.<count>5__1++;
                    }
                    break;

                case 1:
                    goto Label_0050;
            }
            return false;
        }

        IEnumerator<int> IEnumerable<int>.GetEnumerator()
        {
            Test.<GetCounter>d__0 d__;
            if ((Thread.CurrentThread.ManagedThreadId == this.<>l__initialThreadId) && (this.<>1__state == -2))
            {
                this.<>1__state = 0;
                d__ = this;
            }
            else
            {
                d__ = new Test.<GetCounter>d__0(0);
                d__.<>4__this = this.<>4__this;
            }
            d__.max = this.<>3__max;
            return d__;
        }
    }
}
```

现在我们又多了一个额外的字段（`<>3__max`）来表示 `max` 参数的原始值。每当我们开始将一个实例用作 `IEnumerator` 时（即状态变为 0，无论是否在构造时），我们都用那个值初始化 `max` 字段。注意，我们不需要为 `<>4__this` 设置额外的字段，因为 `this` 对于引用类型是只读的，所以原始代码不可能更改它。（但在结构体中使用且引用了 `this` 的迭代器块，确实会有一个额外的字段来存储 `this` 的初始值，因为迭代器块代码可能会更改它。）

可以说，编译器可以检查是否没有任何东西改变参数的值，从而避免复制，但我怀疑这付出的努力可能得不偿失，并且会遇到奇怪的边界情况。当前的系统允许你以各种奇妙的方式改变参数值，而不会对从同一个 `IEnumerable` 实例创建的任何其他迭代器造成问题。

这就涵盖了我们需要跟踪的所有状态，但还有一个问题需要处理：finally 块。

**还有 finally...**

迭代器带来了一个棘手的问题。与整个方法在执行完毕前栈帧才被弹出不同，每次产生一个值时，执行实际上是暂停的。无法保证调用者会以任何方式、形态或形式再次使用迭代器。如果你需要在某个值产生之后执行一些代码，那就麻烦了：你不能保证这会发生。长话短说，在 finally 块中的代码，通常几乎在所有情况下离开方法前都会被执行，但现在就没那么可靠了。

值得记住的是，代码中的大多数 finally 块并不是在 C# 中显式编写的——它们是编译器作为 `lock` 和 `using` 语句的一部分生成的。`lock` 在迭代器块中尤其危险——任何时候，只要你在 `lock` 块内有一个 `yield return` 语句，你就有一个等待发生的线程问题。即使已经产生了下一个值，你的代码仍会持有锁——谁知道客户端要过多久才会调用 `MoveNext()` 或 `Dispose()`？同样，任何用于关键事务（如安全性）的 `try/finally` 块也不应出现在迭代器块中：如果客户端不再需要更多的值，他们可以故意阻止 finally 块的执行。

不过，状态机的构建确保了当迭代器被正确使用时，finally 块会被执行。这是因为 `IEnumerator<T>` 实现了 `IDisposable`，并且 C# 的 `foreach` 循环会调用迭代器的 `Dispose`（即使是那些实现了 `IDisposable` 的非泛型 `IEnumerator`）。生成的迭代器中的 `IDisposable` 实现会根据当前位置（总是基于状态）计算出哪些 finally 块是相关的，并执行适当的代码。

这次我不会给出另一个不切实际的例子，而是展示我最喜欢的迭代器块的用途之一。这样一个简单的类，但真的很方便。它让你可以遍历文本文件（或任何其他 `TextReader`），并在迭代器完成或被释放时关闭读取器。在 MiscUtil 中有一个功能稍全的版本，但只是就构造实例的方式而言。

```c#
using System;
using System.Collections;
using System.Collections.Generic;
using System.IO;

public sealed class LineReader : IEnumerable<string>
{
    readonly Func<TextReader> dataSource;

    public LineReader(string filename)
        : this(() => File.OpenText(filename))
    {
    }

    public LineReader(Func<TextReader> dataSource)
    {
        this.dataSource = dataSource;
    }

    public IEnumerator<string> GetEnumerator()
    {
        using (TextReader reader = dataSource())
        {
            string line;
            while ((line = reader.ReadLine()) != null)
            {
                yield return line;
            }
        }
    }


    IEnumerator IEnumerable.GetEnumerator()
    {
        return GetEnumerator();
    }
}
```

有了 `LineReader`，逐行转储文本文件就变得如此简单：

```c#
foreach (string line in new LineReader(filename))
  {
      Console.WriteLine(line);
  }
```

此外，你可以使用 LINQ 标准查询操作符进行过滤、投影等操作。然而，重要的是在我们完成迭代后尽快关闭文件——无论是因为我们到达了文件末尾、抛出了异常，还是仅仅决定读够了。以下是 `LineReader` 生成的迭代器类的有趣部分：

```c#
private sealed class <GetEnumerator>d__3 : IEnumerator<string>, IEnumerator, IDisposable
{
    private int <>1__state;
    private string <>2__current;
    public LineReader <>4__this;
    public string <line>5__5;
    public TextReader <reader>5__4;

    private void <>m__Finally6()
    {
        this.<>1__state = -1;
        if (this.<reader>5__4 != null)
        {
            this.<reader>5__4.Dispose();
        }
    }

    private bool MoveNext()
    {
        try
        {
            switch (this.<>1__state)
            {
                case 0:
                    this.<>1__state = -1;
                    this.<reader>5__4 = this.<>4__this.dataSource();
                    this.<>1__state = 1;
                    while ((this.<line>5__5 = this.<reader>5__4.ReadLine()) != null)
                    {
                        this.<>2__current = this.<line>5__5;
                        this.<>1__state = 2;
                        return true;
                    Label_0061:
                        this.<>1__state = 1;
                    }
                    this.<>m__Finally6();
                    break;

                case 2:
                    goto Label_0061;
            }
            return false;
        }
        // Note "fault" not "finally"
        fault
        {
            this.System.IDisposable.Dispose();
        }
    }

    void IDisposable.Dispose()
    {
        switch (this.<>1__state)
        {
            case 1:
            case 2:
                break;

            // Very strange! Reflector bug? See below.
            default:
                break;
                try
                {
                }
                finally
                {
                    this.<>m__Finally6();
                }
                break;
        }
    }
    
    // Other stuff: Reset, Current, constructor etc.
}
```

这里有几个有趣的要点需要注意：

*   我们有了一个额外的方法，`<>m__Finally6`，它从 `Dispose()` 和 `MoveNext()` 中被调用。它包含了来自原始源代码 `using` 语句的 finally 块中的代码，并将状态设置为“运行中/之后”。编译器跟踪了如果迭代器在任何特定状态下被释放，哪些 finally 块需要被调用，并在 `Dispose()` 方法中调用它们。
*   为迭代器块本身抛出异常的情况提供了额外的保护。在这种情况下，迭代器被释放并实际上变得无效。这是由生成代码中的 `fault` 块处理的。就像那些“不可言说”的生成名称一样，这是在真正的 C# 中无法做到的，但它是 IL 中的一个故障处理程序。这类似于 `finally`，但在对应的 `try` 块正常进入时不会触发。你可以把它看作有点像通用的 `catch` 块，在最后重新抛出异常。
*   存在一个正值状态，即使我们在运行时也会使用。经过一些实验，看起来每个包含 `yield return` 语句的 `try` 块都会生成一个额外的状态。我不完全确定这些额外状态的用途，但大概是为了让在任何特定情况下做正确的事情变得更容易。`yield break` 语句会导致在方法返回前执行 `Dispose()`。
*   `Dispose` 中的 `try/finally` 块确实很奇怪。我怀疑这是 Reflector 的一个 bug，并且 `break` 实际上应该在该块内。我不太明白为什么不直接调用，但可能有很好的理由。

同样，如果你进行实验，会发现更多东西，但你可以看到编译器尽最大努力确保原始迭代器块中的 finally 块尽可能忠实地被执行。

**结论**

呼！这篇文章比我预期的要长得多。迭代器块中有许多不同的可能性和怪癖需要考虑，我很高兴是 C# 团队而不是我来正确地处理它们。正如你所知，Reflector 是探究底层机制的一个极佳助手，但你需要意识到 C# 编译器会生成一些没有直接对应 C# 代码的代码，比如最后一个例子中使用的 `fault` 处理程序。

手动编写迭代器和使用迭代器块编写迭代器之间的差异是巨大的。使用迭代器块实现 LINQ to Objects 的很多功能都相当容易，尽管不可否认在适当时机检查参数是棘手的。手动编写相同的功能将极其痛苦且容易出错。我认为，毫不夸张地说，没有迭代器块，像 LINQBridge 这样的产品可能就不会存在；我确信 MiscUtil 中许多与迭代器相关的代码也不会存在。

所以，向 C# 团队表示“巨大的感谢”，我期待看到他们能为我消除哪些其他繁琐的任务。