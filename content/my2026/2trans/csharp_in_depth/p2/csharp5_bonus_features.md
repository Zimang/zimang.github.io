---
weight: 1
title: "C# 5 bonus features"

---



**本章涵盖**

- 修复了 `foreach` 循环中变量捕获的问题
- 新增调用方信息特性



如果 C# 是为本书作者着想而设计的，那本章本不该存在，或者长度会更标准些。我可以说，在 C# 5 的异步大餐之后、C# 6 的甜点之前，我想加入一个简短的章节来“清洁味蕾”，但现实是 C# 5 中还有两个需要涵盖的变化无法归入异步章节。第一个与其说是一个新特性，不如说是对早期语言设计错误的一个修正。

# **在 `foreach` 循环中捕获变量**

在 C# 5 之前，语言规范将 `foreach` 循环描述为：**每个循环声明一个单一的迭代变量**。该变量在原代码中是只读的，但在循环的每次迭代中会获得不同的值。例如，在 C# 3 中，对一个 `List<string>` 的 `foreach` 循环如下：

```csharp
foreach (string name in names)
{
    Console.WriteLine(name);
}
```

大致等价于：

```csharp
string name; // 声明一个单一的迭代变量
using (var iterator = names.GetEnumerator()) // 不可见的迭代器变量
{
    while (iterator.MoveNext())
    {
        name = iterator.Current; // 每次迭代为迭代变量赋新值
        Console.WriteLine(name); // foreach 循环的原主体
    }
}
```

> **注意**：规范围绕集合和元素可能的转换还有许多其他细节，但与此变更无关。此外，迭代变量的作用域仅限于循环内部；你可以想象在整个代码周围加一对额外的大括号。

在 C# 1 中，这没有问题。但在 C# 2 引入匿名方法时，就开始出问题了。那是第一次变量可以被**捕获**，显著改变了它的生命周期。当一个变量在匿名函数中被使用时，它就被捕获了，编译器必须在幕后做工作，使其使用起来感觉自然。尽管 C# 2 的匿名方法很有用，但我的印象是，C# 3 及其 lambda 表达式和 LINQ 才真正鼓励开发者更广泛地使用委托。

我们之前只用单一迭代变量展开 `foreach` 循环有什么问题？如果那个迭代变量在一个用于委托的匿名函数中被捕获，那么每当该委托被调用时，委托将使用该单一变量的**当前值**。下面的清单展示了一个具体例子。

**清单 7.1 在 `foreach` 循环中捕获迭代变量**

```csharp
List<string> names = new List<string> { "x", "y", "z" };
var actions = new List<Action>();
foreach (string name in names) // 遍历名字列表
{
    actions.Add(() => Console.WriteLine(name)); // 创建一个捕获 name 的委托
}
foreach (Action action in actions)
{
    action(); // 执行所有委托
}
```

如果我没有让你注意这个问题，你期望输出什么？大多数开发者会期望它打印 `x`，然后 `y`，然后 `z`。这是有用的行为。但实际上，在使用版本 5 之前的 C# 编译器时，它会打印三遍 `z`，这真的没什么帮助。

从 C# 5 开始，`foreach` 循环的规范已被更改，以便在循环的**每次迭代中引入一个新变量**。在 C# 5 及以后版本中，完全相同的代码会产生预期的结果：`x`, `y`, `z`。

请注意，此更改仅影响 `foreach` 循环。如果你改用常规的 `for` 循环，你仍然只会捕获一个变量。下面的清单与清单 7.1 相同，仅更改了加粗显示的部分。

**清单 7.2 在 `for` 循环中捕获迭代变量**

```csharp
List<string> names = new List<string> { "x", "y", "z" };
var actions = new List<Action>();
for (int i = 0; i < names.Count; i++) // 遍历列表的索引
{
    actions.Add(() => Console.WriteLine(names[i])); // 创建一个捕获 names 和 i 的委托
}
foreach (Action action in actions)
{
    action(); // 执行所有委托
}
```

这不会打印最后一个名字三遍；它会因 `ArgumentOutOfRangeException` 而失败，因为当你开始执行委托时，`i` 的值已经是 `3` 了。

这不是 C# 设计团队的疏忽。只是当 `for` 循环初始化器声明一个局部变量时，它是为整个循环周期声明一次。循环的语法使这个模型易于理解，而 `foreach` 的语法则鼓励了“每次迭代一个变量”的心智模型。

接下来介绍 C# 5 的最后一个特性：调用方信息特性。

 

# **调用方信息特性**

有些特性是通用的，如 lambda 表达式、隐式类型局部变量、泛型等。另一些则更具针对性：LINQ 旨在处理某种形式的数据查询，尽管其目标是泛化到许多数据源。C# 5 的最后一个特性目标极其明确：有两个重要的用例（一个明显，一个稍不明显），我预计它不会在这些情况之外被广泛使用。

## **基本行为**

.NET 4.5 引入了三个新特性：

- `CallerFilePathAttribute`
- `CallerLineNumberAttribute`
- `CallerMemberNameAttribute`

它们都位于 `System.Runtime.CompilerServices` 命名空间中。与其他特性一样，使用它们时可以省略 `Attribute` 后缀。因为这是使用特性的最常见方式，在本书的剩余部分我将适当地缩写这些名称。

所有三个特性都只能应用于参数，并且只有当它们应用于具有适当类型的可选参数时才有用。其思想很简单：如果调用站点不提供实参，编译器将使用当前文件、行号或成员名称来填充实参，而不是采用常规的默认值。如果调用方确实提供了实参，编译器将保留它。

> **注意**：在正常使用中，参数类型几乎总是 `int` 或 `string`。如果存在适当的转换，它们也可以是其他类型。如果你有兴趣，请参阅规范了解细节，但如果你真需要知道，我会感到惊讶。

下面的清单展示了所有三个特性的例子，以及编译器指定值和用户指定值的混合。

**清单 7.3 调用方成员特性的基本演示**

```csharp
static void ShowInfo(
    [CallerFilePath] string file = null,
    [CallerLineNumber] int line = 0,
    [CallerMemberName] string member = null)
{
    Console.WriteLine("{0}:{1} - {2}", file, line, member);
}
static void Main()
{
    ShowInfo(); // 编译器根据上下文提供所有三个实参
    ShowInfo("LiesAndDamnedLies.java", -10); // 编译器仅根据上下文提供成员名称
}
```

在我的机器上，清单 7.3 的输出如下：

```
C:\Users\jon\Projects\CSharpInDepth\Chapter07\CallerInfoDemo.cs:20 - Main
LiesAndDamnedLies.java:-10 - Main
```

你通常不会为这些参数提供假值，但能够显式传递值是有用的，特别是如果你想使用相同的特性记录当前方法的调用方时。

成员名称通常以显而易见的方式适用于所有成员。这些特性的默认值通常无关紧要，但我们将在 7.2.4 节回到一些有趣的边界情况。首先，我们来看看前面提到的两个常见用例。其中最普遍的是日志记录。

## **日志记录**

调用方信息最有用的场景是写入日志文件。以前记录日志时，你通常会构造堆栈跟踪（例如使用 `System.Diagnostics.StackTrace`）来找出日志调用来自何处。这通常在日志框架中被隐藏，但它仍然存在——而且丑陋。这在性能方面可能是一个问题，并且在面对 JIT 编译器内联时很脆弱。

很容易看出日志框架如何利用这个新特性来廉价地记录调用方信息，即使构建时剥离了调试信息甚至在混淆之后，也能保留行号和成员名称。当然，当你想记录完整的堆栈跟踪时，这没有帮助，但它也没有剥夺你这样做能力。

根据 2017 年底进行的一项快速抽样，似乎这个功能尚未被特别广泛地使用。特别是，我没有看到它被常用于 ASP.NET Core 的 `ILogger` 接口中。但为你自己的 `ILogger` 扩展方法编写使用这些特性并创建适当状态对象以供记录是完全合理的。

项目包含自己的原始日志框架并不少见，这些框架也可能适合使用这些特性。项目特定的日志框架也不大需要担心面向不包含这些特性的框架。

> **注意**：缺乏一个高效的系统级日志框架是一个棘手的问题。对于希望提供日志功能但不想添加第三方依赖项且不知道其用户将面向哪些日志框架的类库开发者来说，尤其如此。

尽管日志记录用例需要框架方面的特定思考，但我们的第二个用例集成起来要简单得多。

## **简化 INotifyPropertyChanged 实现**

如果你恰好频繁实现 `INotifyPropertyChanged`，那么只使用其中一个特性 `[CallerMemberName]` 的较不明显用途对你来说可能很明显。如果你不熟悉 `INotifyPropertyChanged` 接口，它通常用于富客户端应用程序（相对于 Web 应用程序），允许用户界面响应模型或视图模型的变化。它位于 `System.ComponentModel` 命名空间中，因此不依赖于任何特定的 UI 技术。例如，它在 Windows Forms、WPF 和 Xamarin Forms 中都有使用。该接口很简单；它是一个类型为 `PropertyChangedEventHandler` 的单个事件。这是一个具有以下签名的委托类型：

```csharp
public delegate void PropertyChangedEventHandler(object sender, PropertyChangedEventArgs e)
```

`PropertyChangedEventArgs` 反过来有一个构造函数：

```csharp
public PropertyChangedEventArgs(string propertyName)
```

在 C# 5 之前，`INotifyPropertyChanged` 的典型实现可能如下面的清单所示。

**清单 7.4 以旧方式实现 INotifyPropertyChanged**

```csharp
class OldPropertyNotifier : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler PropertyChanged;
    private int firstValue;
    public int FirstValue
    {
        get { return firstValue; }
        set
        {
            if (value != firstValue)
            {
                firstValue = value;
                NotifyPropertyChanged("FirstValue"); // 使用字符串字面量
            }
        }
    }
    // (其他属性遵循相同模式)
    private void NotifyPropertyChanged(string propertyName)
    {
        PropertyChangedEventHandler handler = PropertyChanged;
        if (handler != null)
        {
            handler(this, new PropertyChangedEventArgs(propertyName));
        }
    }
}
```

辅助方法的目的是避免在每个属性中都进行空值检查。你可以很容易地将其设为扩展方法，以避免在每个实现中重复。

这不仅冗长（这一点没有改变），而且脆弱。问题在于属性的名称（`FirstValue`）是作为字符串字面量指定的，如果你将属性名称重构为其他名称，很容易忘记更改字符串字面量。如果你幸运，你的工具和测试会帮助你发现错误，但这仍然很丑陋。你将在第 9 章看到，C# 6 引入的 `nameof` 操作符将使此代码更易于重构，但它仍然容易受到复制粘贴错误的影响。使用调用方信息特性，大部分代码保持不变，但你可以让编译器通过辅助方法中的 `CallerMemberName` 来填充属性名称，如下面的清单所示。

**清单 7.5 使用调用方信息实现 INotifyPropertyChanged**

```csharp
// 属性 setter 内的更改
if (value != firstValue)
{
    firstValue = value;
    NotifyPropertyChanged(); // 不再需要传递属性名称
}
// 辅助方法
void NotifyPropertyChanged([CallerMemberName] string propertyName = null)
{
    // 方法体与之前相同
    PropertyChangedEventHandler handler = PropertyChanged;
    if (handler != null)
    {
        handler(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

我只展示了已更改的代码部分；就是这么简单。现在当你更改属性名称时，编译器将使用新名称。这不是惊天动地的改进，但仍然更好了。

与日志记录不同，这种模式已经被提供视图模型和模型基类的模型-视图-视图模型框架所采用。例如，在 Xamarin Forms 中，`BindableObject` 类有一个使用 `CallerMemberName` 的 `OnPropertyChanged` 方法。同样，Caliburn Micro MVVM 框架有一个带有 `NotifyOfPropertyChange` 方法的 `PropertyChangedBase` 类。

关于调用方信息特性，你可能需要了解的就这些了，但存在一些有趣的奇特之处，特别是关于调用方成员名称。



## **调用方信息特性的边界情况**

在几乎所有情况下，编译器应为调用方信息特性提供哪个值是显而易见的。不过，看看那些不明显的情况也很有趣。我需要强调，这主要是出于好奇，并聚焦于语言设计的选择，而不是会影响常规开发的问题。首先，是一个小限制。

**动态调用的成员**
在许多方面，围绕动态类型的基础设施都努力在运行时应用与常规编译器在编译时相同的规则。但调用方信息为此目的并未被保留。如果被调用的成员包含一个带有调用方信息特性的可选参数，但调用没有包含相应的实参，则将使用参数中指定的默认值，就好像该特性不存在一样。

除了其他原因，编译器将不得不为每个动态调用的成员嵌入所有行号信息，以防万一需要，这样会在 99.9% 的情况下无益地增加程序集大小。此外，还需要在运行时进行额外分析来检查是否需要调用方信息，这也可能干扰缓存。我怀疑，如果 C# 设计团队认为这是一个常见且重要的场景，他们会找到一种方法使其工作，但我也认为他们决定将时间花在更有价值的功能上是完全合理的。基本上，你只需要了解并接受这种行为即可。不过，在某些情况下存在变通方法。

如果你传递的方法实参碰巧是动态的，但你又不需要它是动态的，可以将其转换为适当的类型。那时，方法调用将是一个常规调用，不涉及任何动态类型。如果你确实需要动态行为，但知道你调用的成员使用了调用方信息特性，你可以显式调用一个使用调用方信息特性返回值的辅助方法。这有点丑陋，但无论如何这是一个边界情况。下面的清单展示了问题以及两种变通方法。

**清单 7.6 调用方信息特性和动态类型**

```csharp
static void ShowLine(string message,
                     [CallerLineNumber] int line = 0)
{
    Console.WriteLine("{0}: {1}", line, message); // 尝试调用的使用行号的方法
}
static int GetLineNumber( // 变通方法 2 的辅助方法
    [CallerLineNumber] int line = 0)
{
    return line;
}
static void Main()
{
    dynamic message = "Some message";
    ShowLine(message); // 简单的动态调用；行号将报告为 0。
    ShowLine((string) message); // 变通方法 1：转换值以移除动态类型
    ShowLine(message, GetLineNumber()); // 变通方法 2：使用辅助方法显式提供行号
}
```

清单 7.6 对第一个调用打印行号 0，但对两种变通方法都打印正确的行号。这是在拥有简单代码和保留更多信息之间的权衡。当然，当你需要使用动态重载解析，并且某些重载需要调用方信息而某些不需要时，这两种变通方法都不合适。在我看来，这个限制相当合理。接下来，让我们思考一些不寻常的名称。

**不明显的成员名称**
当调用方成员名称由编译器提供且该调用方是一个方法时，名称很明显：就是方法的名称。但并非所有东西都是方法。这里有一些需要考虑的情况：

- 从实例构造函数中调用
- 从静态构造函数中调用
- 从终结器中调用
- 从运算符中调用
- 作为字段、事件或属性初始化器的一部分调用
- 从索引器中调用

前四种被规定为依赖于实现；由编译器决定如何处理它们。第五种（初始化器）根本没有规定，最后一种（索引器）被规定使用名称 `Item`，除非已对索引器应用了 `IndexerNameAttribute`。

Roslyn 编译器对前四种使用 IL 中存在的名称：`.ctor`、`.cctor`、`Finalize` 和运算符名称，如 `op_Addition`。对于初始化器，它使用被初始化的字段、事件或属性的名称。

可下载的代码包含一个展示所有这些的完整示例；我没有在此包含代码，因为结果比代码本身更有趣。所有的名称都是最明显、最应该选择的，如果看到不同的编译器选择不同的选项，我会感到惊讶。然而，我在另一个方面发现了编译器之间的差异：确定编译器何时应填充调用方信息特性。

**隐式构造函数调用**
C# 5 语言规范要求仅当函数在源代码中显式调用时才使用调用方信息，但被视为语法扩展的查询表达式除外。其他基于模式的 C# 语言结构无论如何不适用于带有可选参数的方法，但构造函数初始化器绝对适用。（解构是 C# 7 的功能，在 12.2 节描述。）语言规范以构造函数为例，指出除非调用是显式的，否则编译器不提供调用方成员信息。下面的清单展示了一个使用调用方成员信息的构造函数的抽象基类以及三个派生类。

**清单 7.7 构造函数中的调用方信息**

```csharp
public abstract class BaseClass
{
    protected BaseClass( // 基类构造函数使用调用方信息特性
        [CallerFilePath] string file = "Unspecified file",
        [CallerLineNumber] int line = -1,
        [CallerMemberName] string member = "Unspecified member")
    {
        Console.WriteLine("{0}:{1} - {2}", file, line, member);
    }
}
public class Derived1 : BaseClass { } // 隐式添加无参数构造函数
public class Derived2 : BaseClass
{
    public Derived2() { } // 带有隐式调用 base() 的构造函数
}
public class Derived3 : BaseClass
{
    public Derived3() : base() { } // 显式调用基类的构造函数
}
```

对于 Roslyn 编译器，只有 `Derived3` 会显示真实的调用方信息。`Derived1` 和 `Derived2` 中对 `BaseClass` 构造函数的调用是隐式的，它们使用参数中指定的默认值，而不是提供文件名、行号和成员名称。

这符合 C# 5 规范，但我认为这是一个设计缺陷。我相信大多数开发者会期望这三个派生类完全等价。有趣的是，Mono 编译器目前为这些派生类中的每一个打印相同的输出。我们必须等待，看看是语言规范改变，还是 Mono 编译器改变，或者这种不兼容性持续到未来。

**查询表达式调用**
正如我之前提到的，语言规范指出查询表达式是编译器提供调用方信息的地方之一，即使调用是隐式的。我怀疑这会被经常使用，但我在可下载的源代码中提供了一个完整的例子。它需要的代码比在这里包含的合理量要多，但其用法如下面的清单所示。

**清单 7.8 查询表达式中的调用方信息**

```csharp
string[] source =
{
    "the", "quick", "brown", "fox",
    "jumped", "over", "the", "lazy", "dog"
};
var query = from word in source
            where word.Length > 3
            select word.ToUpperInvariant(); // 使用捕获调用方信息的方法的查询表达式
Console.WriteLine("Data:");
Console.WriteLine(string.Join(", ", query)); // 记录数据
Console.WriteLine("CallerInfo:");
Console.WriteLine(string.Join(
    Environment.NewLine, query.CallerInfo)); // 记录查询的调用方信息
```

虽然它包含一个常规查询表达式，但我引入了新的扩展方法（与示例在同一命名空间中，因此在 `System.Linq` 之前被找到），这些方法包含调用方信息特性。输出显示调用方信息与数据本身一起被捕获在查询中：

```
Data:
QUICK, BROWN, JUMPED, OVER, LAZY
CallerInfo:
CallerInfoLinq.cs:91 - Main
CallerInfoLinq.cs:92 – Main
```

这有用吗？老实说，可能没有。但这确实凸显了语言设计者在引入该功能时，必须仔细考虑许多情况。如果有人为查询表达式中的调用方信息找到了一个好的用途，但规范没有明确规定应该发生什么，那将会很烦人。我们还需要考虑最后一种成员调用，对我来说，这甚至比构造函数初始化器和查询表达式更微妙：特性的实例化。

**带有调用方信息特性的特性**
我倾向于认为应用特性只是指定额外的数据。它感觉不像是在调用任何东西，但特性也是代码，当特性对象被构造时（通常是从反射调用返回），会调用构造函数和属性设置器。如果你创建一个在其构造函数中使用调用方信息特性的特性，那么什么算是调用方呢？让我们来找出答案。

首先，你需要一个特性类。这部分很简单，如下面的清单所示。

**清单 7.9 捕获调用方信息的特性类**

```csharp
[AttributeUsage(AttributeTargets.All)]
public class MemberDescriptionAttribute : Attribute
{
    public MemberDescriptionAttribute(
        [CallerFilePath] string file = "Unspecified file",
        [CallerLineNumber] int line = 0,
        [CallerMemberName] string member = "Unspecified member")
    {
        File = file;
        Line = line;
        Member = member;
    }
    public string File { get; }
    public int Line { get; }
    public string Member { get; }
    public override string ToString() =>
        $"{Path.GetFileName(File)}:{Line} - {Member}";
}
```

为简洁起见，这个类使用了 C# 6 的一些功能，但目前有趣的是构造函数参数使用了调用方信息特性。

当你应用我们新的 `MemberDescriptionAttribute` 时会发生什么？在下一个清单中，让我们将它应用到一个类和一个方法的各个方面，然后看看你得到了什么。

**清单 7.10 将特性应用到一个类和一个方法**

```csharp
using MDA = MemberDescriptionAttribute; // 有助于保持反射代码简短
[MemberDescription] // 将特性应用到一个类
class CallerNameInAttribute
{
    [MemberDescription] // 以各种方式将特性应用到一个方法
    public void Method<[MemberDescription] T>(
        [MemberDescription] int parameter) { }
}
static void Main()
{
    var typeInfo = typeof(CallerNameInAttribute).GetTypeInfo();
    var methodInfo = typeInfo.GetDeclaredMethod("Method");
    var paramInfo = methodInfo.GetParameters()[0];
    var typeParamInfo =
        methodInfo.GetGenericArguments()[0].GetTypeInfo();
    Console.WriteLine(typeInfo.GetCustomAttribute<MDA>());
    Console.WriteLine(methodInfo.GetCustomAttribute<MDA>());
    Console.WriteLine(paramInfo.GetCustomAttribute<MDA>());
    Console.WriteLine(typeParamInfo.GetCustomAttribute<MDA>());
}
```

`Main` 方法使用反射从所有应用了该特性的地方获取它。你可以将 `MemberDescriptionAttribute` 应用到其他地方：字段、属性、索引器等。随意用可下载的代码进行实验，以找出到底发生了什么。我觉得有趣的是，编译器在所有情况下都非常乐意捕获行号和文件路径，但它不使用类名作为成员名，因此输出如下：

```
CallerNameInAttribute.cs:36 - Unspecified member
CallerNameInAttribute.cs:39 - Method
CallerNameInAttribute.cs:40 - Method
CallerNameInAttribute.cs:40 – Method
```

同样，这在 C# 5 规范中有所规定，即它规定了当特性应用于函数成员（方法、属性、事件等）时的行为，但不包括应用于类型的情况。也许将类型也包括在这里会更有用。类型被定义为命名空间的成员，因此将成员名称映射到类型名称并非不合理。

重申一下，我包含本节的原因不仅仅是出于完整性。它突出了一些有趣的语言选择。什么时候语言设计可以接受限制以避免实现成本？什么时候语言设计选择与用户期望相冲突是合理的？什么时候将决策明确转变为实现选择是有意义的？在元层面上，语言设计团队应该花多少时间来为一个相对次要的功能指定边界情况？在本章结束之前，还有一个实用的细节：在不支持这些特性的旧框架上启用此功能。

## **在 .NET 旧版本中使用调用方信息特性**

希望现在大多数读者都将目标定为 .NET 4.5+ 或 .NET Standard 1.0+，两者都包含调用方信息特性。但在某些情况下，你仍然可以使用现代编译器，但需要以旧框架为目标。

在这些情况下，你仍然可以使用调用方信息特性，但需要使这些特性对编译器可用。最简单的方法是使用 Microsoft.Bcl NuGet 包，它提供了这些特性以及框架后续版本提供的许多其他功能。

如果由于某种原因无法使用 NuGet 包，你可以自己提供这些特性。它们是简单的特性，没有参数或属性，因此你可以直接从 API 文档中复制声明。它们仍然需要在 `System.Runtime.CompilerServices` 命名空间中。为了避免类型冲突，你需要确保这些仅在系统提供的特性不可用时才可用。这可能很棘手（因为所有版本控制往往如此），并且细节超出了本书的范围。

当我开始写这一章时，我没想到最后会写这么多关于调用方信息特性的内容。我不能说我在日常工作中经常使用这个功能，但我觉得设计方面很有趣。这不是因为它是一个次要功能；正是因为它是一个次要功能。你会期望主要功能——动态类型、泛型、async/await——需要大量的语言设计工作，但次要功能也可能有各种各样的边界情况。功能之间经常相互作用，因此引入新功能的一个危险是，它可能会使未来的功能更难设计或实现。

**总结**

- 在 C# 5 中，捕获的 `foreach` 迭代变量更有用。
- 你可以使用调用方信息特性要求编译器根据调用方的源文件、行号和成员名称来填充参数。
- 调用方信息特性展示了语言设计通常需要的细节水平。

> 本节深入探讨了调用方信息特性在多种边界情况下的行为：动态调用会丢失信息；非常规成员（构造函数、运算符等）的名称由编译器以可预测但实现定义的方式提供；隐式构造函数调用和显式调用在信息捕获上存在差异（这被部分开发者视为设计缺陷）；查询表达式和特性实例化等特殊场景也遵循特定规则。这些细节体现了语言设计的复杂性，即使对于一个小特性，也需要仔细考虑众多场景以平衡实用性、一致性和实现成本。
>

