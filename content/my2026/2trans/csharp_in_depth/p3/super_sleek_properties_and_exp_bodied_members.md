---
weight: 10
title: "超简洁的属性和表达式体成员"
---

**本章涵盖**
- 自动实现只读属性
- 在声明处初始化自动实现的属性
- 使用表达式体成员消除不必要的样板代码

某些版本的 C# 有一个统一的、几乎所有其他特性都为之服务的大特性。例如，C# 3 引入了 LINQ，C# 5 引入了异步。C# 6 则不同，但它有一个总的主题：几乎所有特性都有助于使代码更清晰、更简单、更具可读性。C# 6 不在于做更多的事，而在于用更少的代码做相同的工作。

本章要介绍的特性是关于属性和其他简单代码的。当不涉及太多逻辑时，去除即使是最小的样板代码（例如大括号和 return 语句）也能产生很大的不同。尽管这里的特性听起来可能不那么令人印象深刻，但我对它们在真实代码中的影响感到惊讶。我们将从属性开始，然后讨论方法、索引器和运算符。

# **属性简史**

C# 从第一个版本开始就有属性。尽管其核心功能没有改变，但在源代码中表达它们的方式已逐渐变得更简单、更通用。属性允许你在 API 中区分状态的访问和操作方式与状态的实现方式。

例如，假设你想表示二维空间中的一个点。你可以使用公共字段轻松表示，如下所示。

**清单 8.1 具有公共字段的 Point 类**
```csharp
public sealed class Point
{
    public double X;
    public double Y;
}
```

乍一看，这似乎还不错，但类的能力（“我可以访问其 X 和 Y 值”）与实现（“我将使用两个 double 字段”）紧密耦合。但此时，实现已经失去了控制。只要类状态直接通过字段暴露，你就无法做到以下事情：
- 在设置新值时执行验证（例如，防止 X 和 Y 坐标为无穷大或非数值）
- 在获取值时执行计算（例如，如果你想以不同的格式存储字段——对于点来说不太可能，但在其他情况下完全可行）

你可能会说，当你发现需要类似这样的功能时，你以后总可以将字段更改为属性，但这是一个破坏性更改，你可能希望避免。（它破坏了源代码兼容性、二进制兼容性和反射兼容性。仅仅为了避免从一开始就使用属性，这是一个很大的风险。）

在 C# 1 中，语言几乎没有为属性提供任何帮助。基于属性的清单 8.1 版本需要手动声明后备字段以及每个属性的 getter 和 setter，如下所示。

**清单 8.2 C# 1 中具有属性的 Point 类**
```csharp
public sealed class Point
{
    private double x, y;
    public double X { get { return x; } set { x = value; } }
    public double Y { get { return y; } set { y = value; } }
}
```

你可能会说，许多属性一开始只是简单地读取和写入字段，没有额外的验证、计算或其他任何东西，并且在代码的整个历史中都保持这种状态。像这样的属性本可以作为字段公开，但很难预测哪些属性以后可能需要额外的代码。即使你能准确预测，也会感觉自己在没有理由的情况下操作两个抽象级别。对我来说，属性充当了类型提供的契约的一部分：其宣传的功能。字段只是实现细节；它们是盒子内部的机制，在绝大多数情况下用户不需要知道。我倾向于在几乎所有情况下都将字段设为私有。

> **注意**：像所有好的经验法则一样，也有例外。在某些情况下，直接暴露字段是有意义的。在第 11 章中，当你看到 C# 7 提供的元组时，你会看到一个有趣的案例。

C# 2 中对属性的唯一改进是允许为 getter 和 setter 设置不同的访问修饰符——例如，公共的 getter 和私有的 setter。（这不是唯一可用的组合，但它是迄今为止最常见的组合。）

然后，C# 3 添加了自动实现的属性，允许以更简单的方式重写清单 8.2，如下所示。

**清单 8.3 C# 3 中具有属性的 Point 类**
```csharp
public sealed class Point
{
    public double X { get; set; }
    public double Y { get; set; }
}
```

这段代码几乎与清单 8.2 中的代码完全相同，只是无法直接访问后备字段。它们被赋予了不可言说的名称，这些名称不是有效的 C# 标识符，但在运行时看来是可以的。

重要的是，C# 3 只允许自动实现读/写属性。这里我不打算深入探讨不变性的所有好处（和陷阱），但有很多原因可能希望你的 Point 类是不可变的。为了使你的属性真正只读，你需要回到手动编写代码。

**清单 8.4 在 C# 3 中通过手动实现具有只读属性的 Point 类**
```csharp
public sealed class Point
{
    private readonly double x, y; // 声明只读字段
    public double X { get { return x; } } // 声明返回字段值的只读属性
    public double Y { get { return y; } }
    public Point(double x, double y)
    {
        this.x = x; // 在构造时初始化字段
        this.y = y;
    }
}
```

这至少是令人恼火的。许多开发者——包括我——有时会作弊。如果我们想要只读属性，我们会使用带有私有 setter 的自动实现的属性，如下所示。

**清单 8.5 在 C# 3 中通过带有私有 setter 的自动实现具有公共只读属性的 Point 类**
```csharp
public sealed class Point
{
    public double X { get; private set; }
    public double Y { get; private set; }
    public Point(double x, double y)
    {
        X = x;
        Y = y;
    }
}
```

这可行，但不令人满意。它没有表达你想要的东西。它允许你在类中更改属性的值，即使你不想这样做；你想要一个可以在构造函数中设置但之后永远不会更改的属性，并且你希望它以简单的方式由字段支持。在 C# 5 及之前的版本中，语言迫使你在实现的简洁性和意图的清晰性之间做出选择，每种选择都牺牲了另一个。自 C# 6 起，你不再需要妥协；你可以编写简洁的代码来清晰地表达你的意图。

# **自动实现属性的升级**

C# 6 为自动实现的属性引入了两个新特性。两者都易于解释和使用。在上一节中，我重点介绍了暴露属性而不是公共字段的重要性，以及简洁地实现不可变类型的困难。你可能已经猜到我们 C# 6 的第一个新特性是如何工作的，但另外两个限制也被取消了。

## **只读的自动实现属性**

C# 6 允许以一种简单的方式表达由只读字段支持的真正只读属性。只需要一个空的 getter 而没有 setter，如下所示。

**清单 8.6 使用只读自动实现属性的 Point 类**
```csharp
public sealed class Point
{
    public double X { get; } // 声明只读的自动实现属性
    public double Y { get; }
    public Point(double x, double y)
    {
        X = x; // 在构造时初始化属性
        Y = y;
    }
}
```

与清单 8.5 相比，唯一改变的部分是 X 和 Y 属性的声明；它们完全没有了 setter。由于没有 setter，你可能想知道如何在构造函数中初始化属性。它的发生方式与清单 8.4 中手动实现时完全相同：自动实现的属性声明的字段是只读的，并且对属性的任何赋值都会由编译器转换为直接字段赋值。任何在构造函数之外的代码中设置属性的尝试都会导致编译时错误。

作为一个不变性的爱好者，这对我来说是一个真正的进步。它让你可以用少量的代码表达理想的结果。懒惰现在不再是代码健康的障碍，至少在这一小方面是这样。

C# 6 中取消的下一个限制与初始化有关。到目前为止，我展示的属性要么根本没有显式初始化，要么在构造函数中初始化。但是，如果你想像字段一样初始化一个属性呢？

## **初始化自动实现的属性**

在 C# 6 之前，任何自动实现属性的初始化都必须在构造函数中；你不能在声明点初始化属性。例如，假设在 C# 2 中有一个 Person 类，如下所示。

**清单 8.7 C# 2 中具有手动属性的 Person 类**
```csharp
public class Person
{
    private List<Person> friends = new List<Person>(); // 声明并初始化字段
    public List<Person> Friends // 暴露一个属性以读写该字段
    {
        get { return friends; }
        set { friends = value; }
    }
}
```

如果你想将此代码更改为使用自动实现的属性，则必须将初始化移动到构造函数中，而之前你根本没有显式声明任何构造函数。你最终会得到如下代码。

**清单 8.8 C# 3 中具有自动实现属性的 Person 类**
```csharp
public class Person
{
    public List<Person> Friends { get; set; } // 声明属性；不允许初始化器
    public Person()
    {
        Friends = new List<Person>(); // 在构造函数中初始化属性
    }
}
```

这几乎和以前一样冗长！在 C# 6 中，这个限制被取消了。你可以在属性声明点进行初始化，如下所示。

**清单 8.9 C# 6 中具有自动实现的读/写属性的 Person 类**
```csharp
public class Person
{
    public List<Person> Friends { get; set; } = new List<Person>(); // 声明并初始化一个读/写的自动实现属性
}
```

当然，你也可以将此特性与只读的自动实现属性一起使用。一个常见的模式是使用一个只读属性公开一个可变集合，这样调用者可以向集合中添加或删除项，但永远不能将属性更改为引用不同的集合（或将其设置为 null 引用）。正如你所期望的，这只是去掉 setter 的问题。

**清单 8.10 C# 6 中具有自动实现的只读属性的 Person 类**
```csharp
public class Person
{
    public List<Person> Friends { get; } = new List<Person>(); // 声明并初始化一个只读的自动实现属性
}
```

我很少发现 C# 早期版本的这个特定限制是一个大问题，因为通常我想根据构造函数参数来初始化属性，但这个变化肯定是一个受欢迎的补充。下一个被取消的限制在与只读的自动实现属性结合使用时变得更重要。



## **结构体中的自动实现属性**

在 C# 6 之前，我总觉得在结构体中使用自动实现属性有点问题。原因有两个：
- 我总是编写不可变的结构体，因此缺少只读的自动实现属性一直是个痛点。
- 由于关于明确赋值的规则，我只能在构造函数中通过链式调用另一个构造函数之后，才能为自动实现的属性赋值。

> **注意**：一般来说，明确赋值规则是指编译器跟踪在代码特定位置哪些变量将被赋值，无论你如何到达该位置。这些规则主要与局部变量相关，以确保你不会尝试读取尚未赋值的局部变量。在这里，我们讨论的是相同规则的略有不同的用法。

下面的清单在我们之前 Point 类的结构体版本中演示了这两点。仅仅是把它打出来就让我有点坐立不安。

**清单 8.11 在 C# 5 中使用自动实现属性的 Point 结构体**
```csharp
public struct Point
{
    public double X { get; private set; } // 具有公共 getter 和私有 setter 的属性
    public double Y { get; private set; }
    public Point(double x, double y) : this() // 链式调用默认构造函数
    {
        X = x; // 属性初始化
        Y = y;
    }
}
```

这不是我会包含在实际代码库中的代码。自动实现属性的好处被其丑陋所掩盖。你已经熟悉了属性的只读方面，但为什么你需要在我们的构造函数初始化器中调用默认构造函数？

答案在于围绕结构体中字段赋值的微妙规则。这里有两个规则在起作用：
- 在编译器认为所有字段都已明确赋值之前，你不能在结构体中使用任何属性、方法、索引器或事件。
- 每个结构体构造函数在将控制权返回给调用者之前，必须为所有字段赋值。

在 C# 5 中，如果不调用默认构造函数，你就违反了这两条规则。设置 X 和 Y 属性仍然算作使用结构体的值，因此你是不允许这样做的。设置属性不算作给字段赋值，所以你无论如何也不能从构造函数返回。链式调用默认构造函数是一种变通方法，因为它会在你的构造函数体执行之前赋值所有字段。然后你可以设置属性并在最后返回，因为编译器很满意所有字段无论如何都被设置了。

在 C# 6 中，语言和编译器对自动实现的属性与其所基于的字段之间的关系有了更紧密的理解：
- 允许在所有字段初始化之前设置自动实现的属性。
- 设置自动实现的属性算作初始化字段。
- 只要事先设置过，允许在其他字段初始化之前读取自动实现的属性。另一种思考方式是，在构造函数内部，自动实现的属性被视为字段。

有了这些新规则和真正的只读自动实现属性，C# 6 中的 Point 结构体版本（如下所示）与清单 8.6 中的类版本相同，除了声明为结构体而不是密封类。

**清单 8.12 在 C# 6 中使用自动实现属性的 Point 结构体**
```csharp
public struct Point
{
    public double X { get; }
    public double Y { get; }
    public Point(double x, double y)
    {
        X = x;
        Y = y;
    }
}
```

结果干净、简洁，正是你想要的。

> **注意**：你可能会问 Point 是否应该是一个结构体。在这种情况下，我持中立态度。点确实感觉像相当自然的值类型，但我通常仍然默认创建类。除了 Noda Time（它大量使用结构体）之外，我很少编写自己的结构体。这个示例当然不是建议你应该开始更广泛地使用结构体，但如果你确实编写自己的结构体，那么语言比过去更有帮助。

到目前为止，你所看到的一切都使自动实现的属性使用起来更清晰，这通常减少了样板代码的数量。然而，并非所有属性都是自动实现的。从代码中去除混乱的任务并未止步于此。

# **表达式体成员**

我绝不打算规定一种特定的 C# 编码风格。除此之外，不同的问题领域适合不同的方法。但我肯定遇到过许多具有大量简单方法和属性的类型。C# 6 通过表达式体成员在这方面为你提供帮助。我们将从属性开始，因为你之前在上一节看过它们，然后看看同样的想法如何应用于其他函数成员。

## **更简单的只读计算属性**

有些属性是简单的：如果基于字段的实现与类型的逻辑状态匹配，属性可以直接返回字段值。这就是自动实现属性的用途。其他属性涉及基于其他字段或属性的计算。为了演示 C# 6 所解决的问题，下面的清单为我们的 Point 类添加了另一个属性：`DistanceFromOrigin`，它以简单的方式使用勾股定理返回该点距离原点的距离。

> **注意**：如果这里的数学不熟悉也不用担心。细节并不重要，重要的是它是一个使用 X 和 Y 的只读属性这一事实。

**清单 8.13 向 Point 添加 DistanceFromOrigin 属性**
```csharp
public sealed class Point
{
    public double X { get; }
    public double Y { get; }
    public Point(double x, double y)
    {
        X = x;
        Y = y;
    }
    public double DistanceFromOrigin // 用于计算距离的只读属性
    {
        get { return Math.Sqrt(X * X + Y * Y); }
    }
}
```

![image-20260122004250597](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260122004250683.png)



我并不打算说这段代码特别难以阅读，但它确实包含了许多我称之为“仪式性”的语法：这些语法存在仅仅是为了让编译器理解有意义的代码是如何组合在一起的。图 8.1 展示了同一个属性，但加上了注释以突出有用的部分；那些仪式性的部分（花括号、return 语句和分号）则用较浅的阴影表示。

**图 8.1 带注释的属性声明，展示了重要方面**

C# 6 允许你以一种更简洁的方式来表达：

```c#
public double DistanceFromOrigin => Math.Sqrt(X * X + Y * Y);
```

在这里，`=>` 用于表示一个**表达式主体成员**——在本例中是一个只读属性。不再需要花括号，不再需要关键字。只读属性这一事实，以及表达式被用来返回值，都是隐式的。将此与图 8.1 进行比较，你会发现表达式主体形式包含了所有有用的部分（以一种不同的方式表明它是只读属性），没有任何多余的东西。完美！

> **不，这不是 Lambda 表达式**
>
> 是的，你以前见过这种语法元素。Lambda 表达式在 C# 3 中引入，作为声明委托和表达式树的一种简洁方式。例如：
> Func<string, int> stringLength = text => text.Length;
>
> 表达式主体成员使用 `=>` 语法，但它们**不是** Lambda 表达式。前面 `DistanceFromOrigin` 的声明不涉及任何委托或表达式树；它只是指示编译器创建一个只读属性，该属性计算给定的表达式并返回结果。
>

大声读出这种语法时，我通常将 `=>` 描述为“粗箭头”。

你可能想知道这在现实世界中是否实用，而不仅仅是本书虚构示例中的摆设。为了向你展示具体例子，我将使用 Noda Time。

**传递或委托属性**

我们将简要地看一下 Noda Time 中的三种类型：

*   **LocalDate** — 仅仅是特定日历中的一个日期，没有时间部分。
*   **LocalTime** — 一天中的某个时间，没有日期部分。
*   **LocalDateTime** — 日期和时间的组合。

不要担心初始化的细节等；只需考虑你会期望从这三种类型中得到什么。显然，一个日期会有年、月、日的属性，一个时间会有时、分、秒等属性。那么组合了这两者的类型呢？能够分别获取日期和时间部分是很方便的，但通常你也需要日期和时间的子组件。`LocalDate` 和 `LocalTime` 的每个实现都经过精心优化，我不希望在 `LocalDateTime` 中重复这些逻辑，所以子组件属性是传递给日期或时间组件属性的“传递属性”。下面清单中展示的实现现在变得非常简洁。

**清单 8.14 Noda Time 中的委托属性**
```csharp
public struct LocalDateTime
{
    // 日期组件的属性
    public LocalDate Date { get; }
    // 委托给日期子组件的属性
    public int Year => Date.Year;
    public int Month => Date.Month;
    public int Day => Date.Day;

    // 时间组件的属性
    public LocalTime TimeOfDay { get; }
    // 委托给时间子组件的属性
    public int Hour => TimeOfDay.Hour;
    public int Minute => TimeOfDay.Minute;
    public int Second => TimeOfDay.Second;

    // 初始化、其他属性和成员...
}
```
许多属性都是这样的；从每个属性中移除 `{ get { return ... } }` 部分真是一件乐事，并且让代码清晰得多。

**在另一个状态上执行简单逻辑**

在 `LocalTime` 内部，只有一个状态：一天内的纳秒数。所有其他属性都基于该值计算。例如，计算以纳秒为单位的亚秒值的代码是一个简单的取余操作：
```csharp
public int NanosecondOfSecond =>
    (int) (NanosecondOfDay % NodaConstants.NanosecondsPerSecond);
```
这段代码在第 10 章中会变得更简单，但现在，你可以享受表达式主体属性的简洁性。

**重要提醒**

表达式主体属性有一个缺点：只读属性和公共读写字段之间只有一个字符的差异。在大多数情况下，如果你犯了错误，由于在字段初始化器中使用了其他属性或字段，会发生编译时错误。但对于静态属性或返回常量值的属性来说，这是一个容易犯的错误。考虑以下声明之间的区别：
```csharp
// 声明一个只读属性
public int Foo => 0;
// 声明一个公共读写字段
public int Foo = 0;
```
这让我栽过几次跟头，但一旦你意识到这个问题，检查起来就容易多了。确保你的代码审查人员也知道这一点，你就不太可能中招了。

到目前为止，我们主要将属性作为自然地承接其他与属性相关新特性的部分来讨论。然而，正如你可能从本节标题猜到的，其他类型的成员也可以拥有表达式主体。





## **表达式主体方法、索引器和运符**  

除了表达式主体属性，您还可以编写表达式主体方法、只读索引器和运算符（包括用户定义的转换）。`=>` 的使用方式相同，表达式周围没有花括号，也不需要显式的 `return` 语句。

例如，一个简单的 `Add` 方法及其对应的运算符（用于将具有明显 `X` 和 `Y` 属性的 `Vector` 添加到 `Point`）在 C# 5 中可能如下所示：

**清单 8.15 C# 5 中的简单方法和运算符**
```csharp
public static Point Add(Point left, Vector right)
{
    return left + right; // 直接委托给运算符
}
public static Point operator +(Point left, Vector right)
{
    return new Point(left.X + right.X, // 简单的构造函数调用以实现加法
                     left.Y + right.Y);
}
```

在 C# 6 中，通过使用表达式主体成员实现，代码可以更简洁，如下一个清单所示。

**清单 8.16 C# 6 中的表达式主体方法和运算符**
```csharp
public static Point Add(Point left, Vector right) => left + right;
public static Point operator +(Point left, Vector right) =>
    new Point(left.X + right.X, left.Y + right.Y);
```

请注意我在 `operator+` 中使用的格式；将所有内容放在一行会使行过长。通常，我将 `=>` 放在声明部分的末尾，并像往常一样缩进主体部分。代码格式完全由您决定，但我发现这种约定适用于所有类型的表达式主体成员。

您也可以为返回 `void` 的方法使用表达式主体。在这种情况下，没有需要省略的 `return` 语句，只需移除花括号。

**注意：** 这与 lambda 表达式的工作方式一致。提醒一下，表达式主体成员不是 lambda 表达式，但它们在这一方面是相同的。

例如，考虑一个简单的日志方法：
```csharp
public static void Log(string text)
{
    Console.WriteLine("{0:o}: {1}", DateTime.UtcNow, text);
}
```
可以改用表达式主体方法重写：
```csharp
public static void Log(string text) =>
    Console.WriteLine("{0:o}: {1}", DateTime.UtcNow, text);
```
这里的好处确实较小，但对于声明和主体能放在一行的方法来说，仍然值得这样做。在第 9 章中，您将看到使用内插字符串字面量使代码更简洁的方法。

对于方法、属性和索引器的最后一个示例，假设您想创建自己的 `IReadOnlyList<T>` 实现，为任何 `IList<T>` 提供只读视图。当然，`ReadOnlyCollection<T>` 已经做到了这一点，但它也实现了可变接口（`IList<T>`、`ICollection<T>`）。有时，您可能希望准确通过实现的接口来限定集合允许的操作。使用表达式主体成员，这样的包装器实现确实很短。

**清单 8.17 使用表达式主体成员的 `IReadOnlyList<T>` 实现**
```csharp
public sealed class ReadOnlyListView<T> : IReadOnlyList<T>
{
    private readonly IList<T> list;

    public ReadOnlyListView(IList<T> list)
    {
        this.list = list;
    }

    public T this[int index] => list[index]; // 索引器委托给列表索引器
    public int Count => list.Count; // 属性委托给列表属性
    public IEnumerator<T> GetEnumerator() =>
        list.GetEnumerator(); // 方法委托给列表方法
    IEnumerator IEnumerable.GetEnumerator() =>
        GetEnumerator(); // 方法委托给另一个 GetEnumerator 方法
}
```

这里展示的唯一新特性是表达式主体索引器的语法，我希望它与其它类型成员的语法足够相似，以至于您甚至没有注意到它是新的。

不过，有什么让您觉得突出吗？有什么让您感到惊讶吗？那个构造函数看起来有点丑，不是吗？

## **C# 6 中表达式主体成员的限制**
通常，在刚刚指出一段代码多么冗长之后，我会揭示 C# 实现的另一个特性来改善它。但这次恐怕不行——至少在 C# 6 中不行。

尽管构造函数只有一条语句，但在 C# 6 中没有表达式主体构造函数。它并不孤单。您不能拥有表达式主体的：
*   静态构造函数
*   终结器
*   实例构造函数
*   读/写或只写属性
*   读/写或只写索引器
*   事件

这些并不会让我夜不能寐，但这种不一致性显然让 C# 团队感到困扰，以至于 C# 7 允许所有这些都成为表达式主体成员。它们通常不会节省任何可打印字符，但格式化约定允许它们节省垂直空间，并且仍然提示这是一个简单的成员。它们都使用您已经习惯的相同语法，清单 8.18 提供了一个完整的示例，纯粹是为了展示语法。这段代码除了作为示例外并不实用，对于事件处理程序，与简单的类似字段的事件相比，它是危险的线程不安全。

**清单 8.18 C# 7 中额外的表达式主体成员**
```csharp
public class Demo
{
    static Demo() =>
        Console.WriteLine("静态构造函数被调用"); // 静态构造函数
    ~Demo() => Console.WriteLine("终结器被调用"); // 终结器

    private string name;
    private readonly int[] values = new int[10];
    public Demo(string name) => this.name = name; // 构造函数

    private PropertyChangedEventHandler handler;
    public event PropertyChangedEventHandler PropertyChanged // 具有自定义访问器的事件
    {
        add => handler += value;
        remove => handler -= value;
    }

    public int this[int index] // 读/写索引器
    {
        get => values[index];
        set => values[index] = value;
    }

    public string Name // 读/写属性
    {
        get => name;
        set => name = value;
    }
}
```

一个好处是，即使 `set` 访问器不是表达式主体的，`get` 访问器也可以是表达式主体的，反之亦然。例如，假设您希望索引器的 `setter` 验证新值不为负数。您仍然可以保留表达式主体的 `getter`：
```csharp
public int this[int index]
{
    get => values[index];
    set
    {
        if (value < 0)
        {
            throw new ArgumentOutOfRangeException();
        }
        Values[index] = value;
    }
}
```
我预计这在将来会相当普遍。根据我的经验，`setter` 往往有验证逻辑，而 `getter` 通常很简单。

**提示：** 如果您发现自己在 `getter` 中写了很多逻辑，值得考虑它是否应该是一个方法。有时界限可能很模糊。

考虑到表达式主体成员的所有优点，它们还有其他缺点吗？在尽可能将所有内容转换为使用它们时，应该有多积极？

## **使用表达式主体成员的指南**

根据我的经验，表达式主体成员对于运算符、转换、比较、相等性检查和 `ToString` 方法特别有用。这些通常由简单的代码组成，但对于某些类型，可能有大量的这些成员，可读性的差异可能是显著的。

与一些较为小众的特性不同，表达式主体成员几乎可以在我遇到的每个代码库中产生显著效果。当我将 Noda Time 转换为使用 C# 6 时，我删除了代码中大约 50% 的 `return` 语句。这是一个巨大的差异，并且随着我逐渐利用 C# 7 提供的额外机会，这一差异只会增加。

请注意，表达式主体成员不仅仅关乎可读性。我发现它们产生了一种心理效应：感觉我比以前更大程度地进行函数式编程。这反过来让我感觉自己更聪明。是的，这听起来很傻，但确实让人感到满足。当然，您可能比我更理性。

危险，一如既往，在于过度使用。在某些情况下，您不能使用表达式主体成员，因为您的代码包含 `for` 语句或类似的东西。在许多情况下，可以将常规方法转换为表达式主体成员，但您真的不应该这样做。我发现有两类这样的成员：
*   执行前置条件检查的成员
*   使用解释性变量的成员

作为第一类的一个例子，我有一个名为 `Preconditions` 的类，它有一个通用的 `CheckNotNull` 方法，它接受一个引用和一个参数名。如果引用为 `null`，则使用参数名抛出 `ArgumentNullException`；否则，返回值。这允许在构造函数等地方方便地组合检查和赋值语句。

这也允许（但肯定不强制）您将结果同时用作方法调用的目标或参数。问题是，如果不小心，理解正在发生的事情会变得困难。以下是我之前描述的 `LocalDateTime` 结构体中的一个方法：
```csharp
public ZonedDateTime InZone(
    DateTimeZone zone,
    ZoneLocalMappingResolver resolver)
{
    Preconditions.CheckNotNull(zone);
    Preconditions.CheckNotNull(resolver);
    return zone.ResolveLocal(this, resolver);
}
```
这读起来简单明了：检查参数是否有效，然后通过委托给另一个方法来完成工作。这可以写成表达式主体成员，像这样：
```csharp
public ZonedDateTime InZone(
    DateTimeZone zone,
    ZoneLocalMappingResolver resolver) =>
    Preconditions.CheckNotNull(zone)
        .ResolveLocal(
            this,
            Preconditions.CheckNotNull(resolver);
```
这将具有完全相同的效果，但可读性差得多。根据我的经验，一个验证检查就会使一个方法处于是否使用表达式主体成员的边界线上；有两个验证检查，就太痛苦了。

对于解释性变量，我之前提供的 `NanosecondOfSecond` 示例只是 `LocalTime` 上众多属性之一。其中大约一半使用表达式主体，但相当一部分有两个语句，像这样：
```csharp
public int Minute
{
    get
    {
        int minuteOfDay = (int) NanosecondOfDay / NanosecondsPerMinute;
        return minuteOfDay % MinutesPerHour;
    }
}
```
通过内联 `minuteOfDay` 变量，可以很容易地将其写成表达式主体属性：
```csharp
public int Minute =>
    ((int) NanosecondOfDay / NodaConstants.NanosecondsPerMinute) %
    NodaConstants.MinutesPerHour;
```
同样，代码实现了完全相同的目标，但在原始版本中，`minuteOfDay` 变量增加了关于子表达式含义的信息，使代码更易于阅读。

在任何一天，我可能得出不同的结论。但在更复杂的情况下，遵循一系列步骤并命名结果，当您六个月后回头查看代码时，会产生巨大的差异。如果您需要在调试器中单步执行代码，它也会对您有所帮助，因为您可以轻松地一次执行一个语句并检查结果是否符合预期。

好消息是，您可以随时尝试并改变主意。表达式主体成员纯粹是语法糖，所以如果您的品味随时间改变，您总是可以转换更多代码来使用它们，或者恢复那些过于急切地使用了表达式主体的代码。







**总结**

*   自动实现的属性现在可以是只读的，并由只读字段支持。
*   自动实现的属性现在可以有初始化器，而不必在构造函数中初始化非默认值。
*   结构体可以使用自动实现的属性，而无需将构造函数链接在一起。
*   表达式主体成员允许以更简洁的方式编写简单的（单个表达式的）代码。
*   尽管 C# 6 中的限制限定了哪些成员可以用表达式主体编写，但这些限制在 C# 7 中被取消了。