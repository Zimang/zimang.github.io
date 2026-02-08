---
weight: 8
title: "精简代码的功能集"
---


本章涵盖：

*   避免引用静态成员时的代码混乱
*   在导入扩展方法时更具选择性
*   在集合初始化器中使用扩展方法
*   在对象初始化器中使用索引器
*   编写更少的显式空值检查
*   仅捕获你真正感兴趣的异常

本章是各种功能的集合。除了以日益精炼的方式表达代码意图之外，没有贯穿始终的特定主题。本章中的功能是在所有显而易见的功能分组方式都被使用后剩下的那些。然而，这丝毫不会削弱它们的实用性。

# **使用静态指令**

我们将看到的第一个功能提供了一种更简单的方式来引用类型的静态成员，包括扩展方法。

## **导入静态成员**

此功能的典型例子是 `System.Math`，它是一个静态类，因此只有静态成员。你将编写一个方法，将极坐标（一个角度和一个距离）转换为笛卡尔坐标（熟悉的 (x, y) 模型），并使用更人性化的度数而不是弧度来表示角度。图 10.1 给出了一个具体示例，展示了一个点如何在两种坐标系中表示。如果你对数学部分不太熟悉也不用担心；这只是一个在短代码中使用大量静态成员的例子。
（图注：100 30° 极坐标 (100, 30°) 笛卡尔坐标：(86.6, 50) 图 10.1 极坐标和笛卡尔坐标示例）

假设你已经有一个简单表示笛卡尔坐标的 `Point` 类型。转换本身是相当简单的三角学：
*   将角度从度转换为弧度，乘以 π / 180。常数 π 可通过 `Math.PI` 获得。
*   使用 `Math.Cos` 和 `Math.Sin` 方法计算出模长为 1 的点的 x 和 y 分量，然后进行乘法放大。

下面的清单展示了完整的方法，其中 `System.Math` 的使用以粗体显示。为方便起见，我省略了类声明。它可以在一个 `CoordinateConverter` 类中，也可以是 `Point` 类型本身的工厂方法。

**清单 10.1 C# 5 中的极坐标到笛卡尔坐标转换**
```csharp
using System;
...
static Point PolarToCartesian(double degrees, double magnitude)
{
    double radians = degrees * Math.PI / 180; // 将度转换为弧度
    return new Point(
        Math.Cos(radians) * magnitude, // 用于完成转换的三角学计算
        Math.Sin(radians) * magnitude);
}
```

虽然这段代码读起来并不太难，但你可以想象，当你编写更多数学相关代码时，重复的 `Math.` 会使代码变得相当混乱。

C# 6 引入了 `using static` 指令来简化这类代码。以下清单与清单 10.1 等效，但导入了 `System.Math` 的所有静态成员。

**清单 10.2 C# 6 中的极坐标到笛卡尔坐标转换**
```csharp
using static System.Math;
...
static Point PolarToCartesian(double degrees, double magnitude)
{
    double radians = degrees * PI / 180; // 将度转换为弧度
    return new Point(
        Cos(radians) * magnitude, // 用于完成转换的三角学计算
        Sin(radians) * magnitude);
}
```

如你所见，`using static` 指令的语法很简单：
```csharp
using static type-name-or-alias;
```
设置好后，以下所有成员都可以直接使用它们的简单名称来访问，而不需要用类型名来限定：
*   静态字段和属性
*   静态方法
*   枚举值
*   嵌套类型

直接使用枚举值的能力在 `switch` 语句和组合枚举值的任何地方都特别有用。下面的并列示例展示了如何使用反射检索类型的所有字段。粗体文本突出显示了可以通过适当的 `using static` 指令删除的代码。

![image-20260208133550838](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260208133550952.png)

**C# 5 代码**
```csharp
using System.Reflection;
...
var fields = type.GetFields(
    BindingFlags.Instance |
    BindingFlags.Static |
    BindingFlags.Public |
    BindingFlags.NonPublic);
```
**在 C# 6 中使用 `using static`**
```csharp
using static System.Reflection.BindingFlags;
...
var fields = type.GetFields(
    Instance | Static | Public | NonPublic);
```



类似地，通过避免在每个 `case` 标签中重复枚举类型名称，可以使响应特定 HTTP 状态码的 `switch` 语句更简单：

![image-20260208133621046](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260208133621120.png)

**C# 5 代码**

```csharp
using System.Net;
...
switch (response.StatusCode)
{
    case HttpStatusCode.OK:
        ...
    case HttpStatusCode.TemporaryRedirect:
    case HttpStatusCode.Redirect:
    case HttpStatusCode.RedirectMethod:
        ...
    case HttpStatusCode.NotFound:
        ...
    default:
        ...
}
```
**在 C# 6 中使用 `using static`**
```csharp
using static System.Net.HttpStatusCode;
...
switch (response.StatusCode)
{
    case OK:
        ...
    case TemporaryRedirect:
    case Redirect:
    case RedirectMethod:
        ...
    case NotFound:
        ...
    default:
        ...
}
```

嵌套类型在手写代码中相对少见，但在生成代码中更常见。如果你哪怕偶尔使用它们，在 C# 6 中直接导入它们的能力也能显著简化你的代码。例如，我将 Google Protocol Buffers 序列化框架实现到 C# 时，会生成嵌套类型来表示原始 `.proto` 文件中声明的嵌套消息。一个特点是，C# 嵌套类型是双重嵌套的，以避免命名冲突。假设你有一个原始的 `.proto` 文件，其中包含如下消息：
```
message Outer {
    message Inner {
        string text = 1;
    }
    Inner inner = 1;
}
```
生成的代码具有以下结构（当然还有其他更多成员）：
```csharp
public class Outer
{
    public static class Types
    {
        public class Inner
        {
            public string Text { get; set; }
        }
    }
    public Types.Inner Inner { get; set; }
}
```
要在 C# 5 中从代码中引用 `Inner`，你必须使用 `Outer.Types.Inner`，这很麻烦。双重嵌套在 C# 6 中变得不那么不方便，因为它可以简化为单个 `using static` 指令：
```csharp
using static Outer.Types;
...
Outer outer = new Outer { Inner = new Inner { Text = "Some text here" } };
```

在所有这些情况下，通过静态导入可用的成员仅在考虑了其他成员之后才在成员查找期间被考虑。例如，如果你静态导入了 `System.Math`，但你在自己的类中也声明了一个 `Sin` 方法，那么调用 `Sin()` 将会找到你的 `Sin` 方法，而不是 `Math` 中的那个。

> **导入的类型不必须是静态的**
> `using static` 中的 `static` 部分并不意味着你导入的类型必须是静态的。目前展示的例子都是静态的，但你也可以导入常规类型。这使你可以无限制地访问这些类型的静态成员：
> ```csharp
> using static System.String;
> ...
> string[] elements = { "a", "b" };
> Console.WriteLine(Join(" ", elements)); // 通过其简单名称访问 String.Join
> ```
> 我发现这个功能不如前面的例子有用，但如果你需要，它是可用的。任何嵌套类型也可以通过它们的简单名称使用。对于通过 `using static` 指令导入的静态成员集合，有一个例外情况不那么直接了当，那就是扩展方法。
>

## **扩展方法和 `using static`**

C# 3 中我从未喜欢的一个方面是发现扩展方法的方式。导入命名空间和导入扩展方法都是通过单个 `using` 指令执行的；没有办法只做其中一个而不做另一个，也没有办法从单个类型导入扩展方法。C# 6 改善了这种情况，尽管我不喜欢的一些方面在不破坏向后兼容性的情况下无法修复。

在 C# 6 中，扩展方法与 `using static` 指令交互的两种重要方式很容易说明，但具有微妙的影响：
*   可以使用针对该类型的 `using static` 指令导入来自单个类型的扩展方法，而无需导入命名空间其余部分的任何扩展方法。
*   从类型导入的扩展方法不能像调用 `Math.Sin` 这样的常规静态方法那样使用。相反，你必须像在扩展类型上调用实例方法那样调用它们。

我将使用整个 .NET 中最常用的扩展方法集，即 LINQ 的扩展方法，来演示第一点。`System.Linq.Queryable` 类包含接受表达式树的 `IQueryable<T>` 扩展方法，而 `System.Linq.Enumerable` 类包含接受委托的 `IEnumerable<T>` 扩展方法。因为 `IQueryable<T>` 继承自 `IEnumerable<T>`，如果使用常规的 `using` 指令导入 `System.Linq`，你可以在 `IQueryable<T>` 上使用接受委托的扩展方法，尽管你通常不希望这样做。下面的清单展示了仅针对 `System.Linq.Queryable` 的 `using static` 指令如何意味着不会拾取 `System.Linq.Enumerable` 中的扩展方法。

**清单 10.3 选择性导入扩展方法**
```csharp
using static System.Linq.Queryable;
...
var query = new[] { "a", "bc", "d" }.AsQueryable(); // 创建一个 IQueryable<string>
Expression<Func<string, bool>> expr = x => x.Length > 1; // 创建一个委托
Func<string, bool> del = x => x.Length > 1; // 和表达式树
var valid = query.Where(expr); // 有效：使用 Queryable.Where
var invalid = query.Where(del); // 无效：没有接受委托的在作用域内的 Where 方法
```

值得注意的一点是，如果你不小心使用常规的 `using` 指令导入了 `System.Linq`（例如为了允许 `query` 被显式类型化），那将默默地使最后一行代码变得有效。

库作者应仔细考虑此更改的影响。如果你希望包含一些扩展方法但允许用户明确选择使用它们，我鼓励为此目的使用单独的命名空间。好消息是，你现在可以确信任何用户——至少是那些使用 C# 6 的用户——可以选择导入哪些扩展方法，而无需你创建许多命名空间。例如，在 Noda Time 2.0 中，我引入了一个 `NodaTime.Extensions` 命名空间，其中包含针对许多类型的扩展方法。我预计一些用户只希望导入这些扩展方法的一个子集，因此我将方法声明拆分为几个类，每个类包含扩展单一类型的方法。在其他情况下，你可能希望沿着不同的思路拆分你的扩展方法。重要的是你应该仔细考虑你的选择。

扩展方法不能像常规静态方法那样调用的事实也很容易用 LINQ 来证明。清单 10.4 展示了通过调用 `Enumerable.Count` 方法来实现这一点：一次是作为扩展方法的有效方式，就像它是 `IEnumerable<T>` 中声明的实例方法一样，一次是尝试将其用作常规静态方法。

**清单 10.4 尝试以两种方式调用 Enumerable.Count**
```csharp
using System.Collections.Generic;
using static System.Linq.Enumerable;
...
IEnumerable<string> strings = new[] { "a", "b", "c" };
int valid = strings.Count(); // 有效：调用 Count 就像它是一个实例方法
int invalid = Count(strings); // 无效：扩展方法不作为常规静态方法导入
```

实际上，这门语言正在鼓励你将扩展方法视为与其他静态方法不同，而以前并非如此。同样，这对库开发人员有影响：将静态类中已存在的方法转换为扩展方法（通过向第一个参数添加 `this` 修饰符）过去是一个非破坏性更改。从 C# 6 开始，这变成了一个破坏性更改：使用 `using static` 指令导入该方法的调用者会发现，在该方法变为扩展方法后，他们的代码将无法再编译。

**注意**：通过静态导入发现的扩展方法并不优先于通过命名空间导入发现的扩展方法。如果你进行的方法调用不是由常规方法调用处理的，但通过导入的命名空间或类有多个扩展方法适用，则正常应用重载解析。

就像扩展方法一样，对象和集合初始化器主要是作为更大功能 LINQ 的一部分添加到语言中的。也就像扩展方法一样，它们在 C# 6 中进行了调整，使其功能稍微更强大。

> 本章集中展示了C#语言通过语法层面改进追求“精简表达”的设计哲学。核心在于通过`using static`指令减少命名重复，它不仅适用于数学库，更在处理枚举、反射标志位或复杂嵌套类型时显著提升可读性。然而，其最大革新在于扩展方法的“选择性导入”，这解决了长期存在的命名空间污染问题，允许开发者精确控制方法集，尤其对库作者意义重大——他们现在可以更灵活地组织扩展方法，而用户能避免因导入整个命名空间导致的重载解析意外或性能陷阱（如在`IQueryable`上误用`Enumerable`方法）。这标志着C#从“便利性”向“精确控制”的演进，开发者需更审慎地设计API，因为一个方法从静态方法改为扩展方法，对使用`using static`的调用者而言将变成破坏性变更。





# **对象和集合初始化器的增强**

提醒一下，对象和集合初始化器是在 C# 3 中引入的。对象初始化器用于设置新创建对象的属性（或者，较少见的情况下是字段）；集合初始化器用于通过集合类型支持的 `Add` 方法向新创建的集合添加元素。以下简单示例展示了初始化一个具有文本和背景颜色的 Windows 窗体 `Button` 以及初始化一个包含三个值的 `List<int>`：

```csharp
Button button = new Button { Text = "Go", BackColor = Color.Red };
List<int> numbers = new List<int> { 5, 10, 20 };
```

C# 6 增强了这两个功能，使它们稍微更灵活。这些增强不像 C# 6 中的其他一些功能那样具有全局实用性，但它们仍然是受欢迎的补充。在这两种情况下，初始化器都扩展到了包括以前不能在那里使用的成员：对象初始化器现在可以使用索引器，而集合初始化器现在可以使用扩展方法。

## **对象初始化器中的索引器**

在 C# 6 之前，对象初始化器只能调用属性设置器或直接设置字段。C# 6 还允许使用常规代码中用于调用它们的 `[index] = value` 语法来调用索引器设置器。

为了简单演示这一点，我将使用 `StringBuilder`。这将是一种相当不寻常的用法，但我们很快会讨论最佳实践。该示例从现有字符串（"This text needs truncating"）初始化一个 `StringBuilder`，将构建器截断到设定的长度，并将最后一个字符修改为 Unicode 省略号（…）。当打印到控制台时，结果是 "This text..."。在 C# 6 之前，你不能在初始化器中修改最后一个字符，所以你最终会得到类似这样的代码：

```csharp
string text = "This text needs truncating";
StringBuilder builder = new StringBuilder(text)
{
    Length = 10 // 设置 Length 属性以截断构建器
};
builder[9] = '\u2026'; // 将最终字符修改为“…”
Console.OutputEncoding = Encoding.UTF8; // 确保控制台支持 Unicode
Console.WriteLine(builder); // 打印出构建器内容
```

考虑到初始化器给你的好处如此之少（只有一个属性），我至少会考虑在单独的语句中设置长度。C# 6 允许你在单个表达式中执行所有需要的初始化，因为你可以在对象初始化器中使用索引器。以下清单以一种稍显刻意的方式展示了这一点。

**清单 10.5 在 StringBuilder 对象初始化器中使用索引器**

```csharp
string text = "This text needs truncating";
StringBuilder builder = new StringBuilder(text)
{
    Length = 10, // 设置 Length 属性以截断构建器
    [9] = '\u2026' // 将最终字符修改为“…”
};
Console.OutputEncoding = Encoding.UTF8; // 确保控制台支持 Unicode
Console.WriteLine(builder); // 打印出构建器内容
```

我特意选择在这里使用 `StringBuilder`，并不是因为它是最明显的包含索引器的类型，而是为了清楚地表明这是一个对象初始化器而不是集合初始化器。

你可能期望我使用某种 `Dictionary<,>` 来代替，但这里有一个隐藏的危险。如果你的代码是正确的，它会如你期望的那样工作，但在大多数情况下，我建议坚持使用集合初始化器。为了了解原因，让我们看一个初始化两个字典的示例：一个在对象初始化器中使用索引器，一个使用集合初始化器。

**清单 10.6 初始化字典的两种方法**

```csharp
var collectionInitializer = new Dictionary<string, int>
{
    { "A", 20 },
    { "B", 30 },
    { "B", 40 } // C# 3 以来的常规集合初始化器
};

var objectInitializer = new Dictionary<string, int>
{
    ["A"] = 20,
    ["B"] = 30,
    ["B"] = 40 // C# 6 中使用索引器的对象初始化器
};
```

表面上，这些可能看起来是等价的。当你没有重复键时，它们是等价的，我甚至更喜欢对象初始化器的外观。但是字典索引器设置器会用相同的键覆盖任何现有条目，而 `Add` 方法在键已存在时会抛出异常。

清单 10.6 故意包含了两次 "B" 键。这是一个容易犯的错误，通常是复制粘贴一行然后忘记修改键部分的结果。在这两种情况下，编译时都不会捕获错误，但至少使用集合初始化器时，它不会默默地做错事。如果你有任何执行这段代码的单元测试——即使它们没有明确检查字典的内容——你很可能会很快发现这个错误。

**Roslyn 来救援？**

当然，能够在编译时发现这个错误会更好。编写一个分析器来为集合和对象初始化器发现这个问题应该是可能的。对于使用索引器的对象初始化器，很难想象有多少情况你会合法地想要多次指定相同的常量索引器键，因此弹出警告似乎完全合理。

我目前还不知道有这样的分析器，但我希望将来会有。清除了这个危险后，就没有理由不在字典中使用索引器了。

那么，什么时候应该在对象初始化器中使用索引器而不是集合初始化器？在以下几种相当明显的情况下，你应该这样做：

*   如果你不能使用集合初始化器，因为该类型没有实现 `IEnumerable` 或者没有合适的 `Add` 方法。（但是，正如你将在下一节中看到的，你可以引入自己的 `Add` 方法作为扩展方法。）例如，`ConcurrentDictionary<,>` 没有 `Add` 方法，但有索引器。它有 `TryAdd` 和 `AddOrUpdate` 方法，但集合初始化器不使用这些方法。你不需要担心在对象初始化器期间对字典的并发更新，因为只有初始化线程知道新字典。
*   如果索引器和 `Add` 方法将以相同的方式处理重复键。仅仅因为字典遵循“添加时抛出异常，索引器中覆盖”的模式，并不意味着所有类型都这样做。
*   如果你确实是在替换元素而不是添加它们。例如，你可能正在基于另一个字典创建一个字典，然后替换与特定键对应的值。

也存在一些不那么明确的情况，你需要在可读性和上述错误可能性之间取得平衡。清单 10.7 展示了一个无模式实体类型的示例，它有两个常规属性，但除此之外允许其数据有任意的键/值对。然后你将研究如何初始化一个实例的选项。

**清单 10.7 具有键属性的无模式实体类型**

```csharp
public sealed class SchemalessEntity : IEnumerable<KeyValuePair<string, object>>
{
    private readonly IDictionary<string, object> properties = new Dictionary<string, object>();

    public string Key { get; set; }
    public string ParentKey { get; set; }

    public object this[string propertyKey]
    {
        get { return properties[propertyKey]; }
        set { properties[propertyKey] = value; }
    }

    public void Add(string propertyKey, object value)
    {
        properties.Add(propertyKey, value);
    }

    public IEnumerator<KeyValuePair<string, object>> GetEnumerator() => properties.GetEnumerator();
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

让我们考虑两种初始化实体的方法，你想要为该实体指定父键、新实体的键和两个属性（名称和位置，只是简单的字符串）。你可以使用集合初始化器，但随后再设置其他属性，或者使用对象初始化器完成所有操作，但冒键拼写错误的风险。以下清单演示了这两种选项。

**清单 10.8 初始化 SchemalessEntity 的两种方法**

```csharp
SchemalessEntity parent = new SchemalessEntity { Key = "parent-key" };

SchemalessEntity child1 = new SchemalessEntity
{
    { "name", "Jon Skeet" }, // 使用集合初始化器指定数据属性
    { "location", "Reading, UK" }
};
child1.Key = "child-key"; // 单独指定键属性
child1.ParentKey = parent.Key;

SchemalessEntity child2 = new SchemalessEntity
{
    Key = "child-key", // 在对象初始化器中指定键属性
    ParentKey = parent.Key,
    ["name"] = "Jon Skeet", // 使用索引器指定数据属性
    ["location"] = "Reading, UK"
};
```

这些方法中哪个更好？第二种对我来说看起来更清晰。无论如何，我通常会将名称和位置键提取到字符串常量中，这样至少减少了意外使用重复键的风险。

如果你控制这样的类型，你可以添加额外的成员来允许你使用集合初始化器。你可以添加一个 `Properties` 属性，它直接暴露字典或暴露其上的视图。在这一点上，你可以在对象初始化器中使用集合初始化器来初始化 `Properties`，该对象初始化器还设置 `Key` 和 `ParentKey`。或者，你可以提供一个接受键和父键的构造函数，在这一点上，你可以用这些值进行显式的构造函数调用，然后使用集合初始化器指定名称和位置属性。

对于选择在对象初始化器中使用索引器还是使用以前版本的集合初始化器，这感觉像是大量的细节。关键是选择权在你手中：没有一本书能给你简单的规则，让你在所有情况下都能得到最佳答案。了解利弊，并运用你自己的判断。

## **在集合初始化器中使用扩展方法**

C# 6 中与对象和集合初始化器相关的第二个变化涉及集合初始化器中可用的方法。提醒一下，为了对类型使用集合初始化器，必须满足两个条件：

*   该类型必须实现 `IEnumerable`。我发现这是一个恼人的限制；有时我实现 `IEnumerable` 仅仅是为了能在集合初始化器中使用该类型。但事实就是如此。这个限制在 C# 6 中没有改变。
*   对于集合初始化器中的每个元素，都必须有一个合适的 `Add` 方法。任何不在花括号中的元素都被假定为对应于单参数调用 `Add` 方法。当需要多个参数时，它们必须放在花括号中。

有时，这可能有点限制。有时你希望以集合本身不支持的 `Add` 方法的方式轻松创建集合。在 C# 6 中，上述条件仍然成立，但第二个条件中“合适”的定义现在包括扩展方法。在某些方面，这简化了转换。下面是一个使用集合初始化器的声明：

```csharp
List<string> strings = new List<string>
{
    10,
    "hello",
    { 20, 3 }
};
```

该声明本质上等同于：

```csharp
List<string> strings = new List<string>();
strings.Add(10);
strings.Add("hello");
strings.Add(20, 3);
```

应用正常的重载解析来确定每个方法调用的含义。如果失败，集合初始化器将无法编译。仅使用常规的 `List<T>`，前面的代码将无法编译，但如果你添加一个扩展方法，它就会编译：

```csharp
public static class StringListExtensions
{
    public static void Add(this List<string> list, int value, int count = 1)
    {
        list.AddRange(Enumerable.Repeat(value.ToString(), count));
    }
}
```

有了这个扩展方法，我们之前代码中对 `Add` 的第一次和最后一次调用最终会调用扩展方法。列表最终有五个元素（"10", "hello", "20", "20", "20"），因为最后一次 `Add` 调用添加了三个元素。这是一个不寻常的扩展方法，但它有助于说明三点：

*   扩展方法可以在集合初始化器中使用，这是本书这一部分的全部要点。
*   这不是一个泛型扩展方法；它只适用于 `List<string>`。这是一种无法在 `List<T>` 本身中执行的专门化。（泛型扩展方法也可以，只要可以推断类型参数。）
*   可选参数可以在扩展方法中使用；我们对 `Add` 的第一次调用将有效地编译为 `Add(10, 1)`，因为第二个参数有默认值。

现在你知道了你可以做什么，让我们更仔细地看看在哪些地方使用这个功能是合理的。

**创建其他通用的 Add 签名**

我在 Protocol Buffers 工作中发现有用的一种技术是创建接受集合的 `Add` 方法。这个过程类似于使用 `AddRange`，但它可以在集合初始化器中使用。这在对象初始化器中特别有用，其中你正在初始化的属性是只读的，但你想添加 LINQ 查询的结果。

例如，考虑一个 `Person` 类，它具有一个只读的 `Contacts` 属性，你想用来自另一个列表的所有居住在 Reading 的联系人来填充它。在 Protocol Buffers 中，`Contacts` 属性将是 `RepeatedField<Person>` 类型，而 `RepeatedField<T>` 有适当的 `Add` 方法，允许你使用集合初始化器：

```csharp
Person jon = new Person
{
    Name = "Jon",
    Contacts = { allContacts.Where(c => c.Town == "Reading") }
};
```

可能需要一点时间来习惯，但然后它非常有用，并且肯定胜过必须单独调用 `jon.Contacts.AddRange(...)`。但是，如果你没有使用 Protocol Buffers，并且 `Contacts` 仅作为 `List<Person>` 暴露呢？在 C# 6 中，这不是问题：你可以为 `List<T>` 创建一个扩展方法，添加一个接受 `IEnumerable<T>` 的 `Add` 重载，并用它调用 `AddRange`，如下面的清单所示。

**清单 10.9 通过扩展方法暴露显式接口实现**

```csharp
static class ListExtensions
{
    public static void Add<T>(this List<T> list, IEnumerable<T> collection)
    {
        list.AddRange(collection);
    }
}
```

有了这个扩展方法，即使使用 `List<T>`，前面的代码也能正常工作。如果你想更广泛，你可以针对 `IList<T>` 编写扩展方法，尽管如果你走这条路，你需要在方法体内编写循环，因为 `IList<T>` 没有 `AddRange` 方法。

**创建专门的 Add 签名**

假设你有一个 `Person` 类，如前所述，具有 `Name` 属性，并且在代码的某个区域中，你做了很多 `Dictionary<string, Person>` 对象的工作，总是按名称索引 `Person` 对象。用简单的 `dictionary.Add(person)` 调用向字典添加条目可能很方便，但 `Dictionary<string, Person>` 作为一个类型，不知道你是按名称索引的。你有什么选择？

你可以创建一个继承自 `Dictionary<string, Person>` 的类，并向其添加一个 `Add(Person)` 方法。这对我没有吸引力，因为你没有以任何有意义的方式专门化字典的行为；你只是让它更方便使用。

你可以创建一个更通用的类，实现 `IDictionary<TKey, TValue>`，它接受一个解释从 `TValue` 到 `TKey` 映射的委托，并通过组合来实现。这可能有用，但对于这一项任务来说可能太过了。最后，你可以为这一种特定情况创建一个扩展方法，如下面的清单所示。

**清单 10.10 为字典添加类型参数特定的 Add 方法**

```csharp
static class PersonDictionaryExtensions
{
    public static void Add(this Dictionary<string, Person> dictionary, Person person)
    {
        dictionary.Add(person.Name, person);
    }
}
```

在 C# 6 之前，这已经是一个不错的选择，但是结合使用 `using static` 功能来限制扩展方法的导入方式以及在集合初始化器中使用扩展方法，使其更具吸引力。然后你可以在不重复名称的情况下初始化字典：

```csharp
var dictionary = new Dictionary<string, Person>
{
    { new Person { Name = "Jon" } },
    { new Person { Name = "Holly" } }
};
```

这里重要的一点是你如何为 `Dictionary<,>` 的一个特定类型参数组合专门化了 API，但没有改变你创建的对象的类型。其他代码不需要知道这里的专门化，因为它只是表面的；它只是为了我们的方便而存在，而不是对象固有行为的一部分。

**注意**：这种方法也有缺点，其中之一是没有什么能阻止使用人名以外的其他方式添加条目。一如既往，我鼓励你自己权衡利弊；不要盲目相信我的建议或任何其他人的建议。

**重新暴露被显式接口实现“隐藏”的现有方法**

在 10.2.1 节中，我使用 `ConcurrentDictionary<,>` 作为你可能想要使用索引器而不是集合初始化器的例子。没有任何额外帮助，你不能使用集合初始化器，因为没有暴露 `Add` 方法。但是 `ConcurrentDictionary<,>` 确实有一个 `Add` 方法；它只是使用显式接口实现来实现 `IDictionary<,>.Add`。通常，如果你想访问使用显式接口实现的成员，你必须强制转换为接口——但你不能在集合初始化器中这样做。相反，你可以暴露一个扩展方法，如下面的清单所示。

**清单 10.11 通过扩展方法暴露显式接口实现**

```csharp
public static class DictionaryExtensions
{
    public static void Add<TKey, TValue>(this IDictionary<TKey, TValue> dictionary, TKey key, TValue value)
    {
        dictionary.Add(key, value);
    }
}
```

乍一看，这看起来完全没有意义。这是一个调用具有完全相同签名的方法的扩展方法。但这有效地绕过了显式接口实现，使 `Add` 方法始终可用，包括在集合初始化器中。你现在可以为 `ConcurrentDictionary<,>` 使用集合初始化器：

```csharp
var dictionary = new ConcurrentDictionary<string, int>
{
    { "x", 10 },
    { "y", 20 }
};
```

当然，这应该谨慎使用。当一个方法被显式接口实现隐藏时，通常是为了阻止你在没有一定注意的情况下调用它。这就是使用 `using static` 选择性导入扩展方法的能力有用的地方：你可以有一个静态类的命名空间，其中包含只打算选择性使用的扩展方法，并在每种情况下只导入相关类。不幸的是，它仍然向同一类中的其他代码暴露了 `Add` 方法，但同样你需要权衡这是否比替代方案更糟。

清单 10.11 中的扩展方法是广泛的，扩展了所有字典。你可以决定只针对 `ConcurrentDictionary<,>` 以避免无意中使用来自其他字典类型的显式实现的 `Add` 方法。

## **测试代码与生产代码**

你可能已经注意到本节中有很多注意事项。关于这些功能，很少有明确的情况可以让你“肯定在这里使用”。但我注意到的大多数缺点都是指该功能在一个代码中很方便，但你不想让它感染其他地方。

根据我的经验，对象和集合初始化器通常用在两个地方：

*   类型初始化后永远不会修改的集合的静态初始化器
*   测试代码

关于暴露和正确性的担忧仍然适用于静态初始化器，但对于测试代码来说则少得多。如果你决定在你的测试程序集中拥有 `Add` 扩展方法来使集合初始化器更简单，那很好。它根本不会影响你的生产代码。同样，如果你在测试的集合初始化器中使用索引器，并意外地设置了相同的键两次，你的测试很可能会失败。同样，缺点被最小化了。

这并不是只影响这一对功能的区别。测试代码仍然应该具有高质量，但你如何衡量这种质量以及做出任何特定权衡的影响对于测试代码与生产代码是不同的，特别是对于公共 API。

作为 LINQ 一部分的扩展方法的添加鼓励了更流畅的方式来组合多个操作。在许多情况下，现在惯用的做法是在单个语句中将多个方法调用链接在一起，而不是使用多个语句。这就是 LINQ 查询一直所做的，但随着像 LINQ to XML 这样的 API，它成为一种更惯用的模式。这可能导致我们长期以来在链接属性访问时遇到的相同问题：一旦遇到空值，一切都会中断。C# 6 允许你在此时安全地终止其中一个链，而不是代码抛出异常。



> 本章节详细探讨了C# 6对对象和集合初始化器的两项重要增强：在对象初始化器中使用索引器，以及在集合初始化器中使用扩展方法。
>
> 第一项增强（索引器）提升了表达灵活性，尤其适合初始化`ConcurrentDictionary`这类没有传统`Add`方法的类型，或在统一语法中混合设置属性和键值对。但需警惕字典等类型中索引器与`Add`方法语义不同（覆盖vs异常）可能引入的隐藏Bug。
>
> 第二项增强（扩展方法）极大地扩展了集合初始化的可能性。它允许通过自定义扩展方法，为现有集合类型（如`List<T>`）添加新的`Add`重载，从而支持接受`IEnumerable<T>`或特定类型参数的初始化。更重要的是，它能“暴露”被显式接口实现隐藏的`Add`方法（如`ConcurrentDictionary`），使其能在初始化器中使用。结合`using static`的选择性导入，可以精细控制这些扩展方法的可见范围。
>
> 作者明智地指出，这些增强在**测试代码**中价值更高，因为可以为了编写便利而引入扩展，且测试失败能快速暴露问题，而对生产代码则需更慎重地权衡便利性与设计纯洁性。这些特性共同体现了C#语言在提供强大表达力的同时，赋予开发者更多选择和设计责任。





# **空条件运算符**

我不打算讨论空值的利弊，但我们经常不得不与之共存，同时还有具有多层深度的属性的复杂对象模型。C# 语言团队长期以来一直在思考如何让处理空值更容易。其中一些工作仍在进行中，但 C# 6 已经迈出了一步。同样，它可以使你的代码更简短、更简单，表达你希望如何处理空值，而无需在各处重复表达式。

## **简单且安全的属性解引用**

作为一个实际例子，假设你有一个 `Customer` 类型，它有一个 `Profile` 属性，该属性有一个 `DefaultShippingAddress` 属性，而该属性又有一个 `Town` 属性。现在，假设你想在一个集合中找出所有默认收货地址的城镇名称为 Reading 的客户。如果不考虑空值，你可以这样写：

```csharp
var readingCustomers = allCustomers
    .Where(c => c.Profile.DefaultShippingAddress.Town == "Reading");
```

如果你知道每个客户都有个人资料，每个个人资料都有默认收货地址，并且每个地址都有城镇，那么这段代码工作正常。但是如果其中任何一个为 `null` 呢？你可能会得到一个 `NullReferenceException`，而你很可能只想将该客户排除在结果之外。以前，你不得不将其重写为非常糟糕的代码，使用短路 `&&` 运算符逐一检查每个属性是否为空：

```csharp
var readingCustomers = allCustomers
    .Where(c => c.Profile != null &&
                c.Profile.DefaultShippingAddress != null &&
                c.Profile.DefaultShippingAddress.Town == "Reading");
```

哎呀，重复太多了。如果你需要在最后调用一个方法而不是使用 `==`（`==` 已经能正确处理空值，至少对于引用类型是这样；关于可能的意外，请参见第 10.3.3 节），情况会更糟。那么 C# 6 是如何改进的呢？它引入了空条件运算符 `?.`，这是一个短路运算符，如果表达式计算结果为 `null` 则停止。该查询的一个空安全版本如下：

```csharp
var readingCustomers = allCustomers
    .Where(c => c.Profile?.DefaultShippingAddress?.Town == "Reading");
```

这与我们的第一个版本完全相同，只是使用了两次空条件运算符。如果 `c.Profile` 或 `c.Profile.DefaultShippingAddress` 为 `null`，则 `==` 左侧的整个表达式的计算结果为 `null`。你可能会问自己，为什么只用了两次空条件运算符，而实际上有四样东西可能为空：
*   `c`
*   `c.Profile`
*   `c.Profile.DefaultShippingAddress`
*   `c.Profile.DefaultShippingAddress.Town`

我假设 `allCustomers` 的所有元素都是非空引用。如果你需要处理那里可能存在空元素的情况，你可以改用 `c?.Profile` 开头。这涵盖了第一项；`==` 运算符已经能处理空操作数，所以你不需要担心最后一项。

## **更详细地了解空条件运算符**

这个简短的例子只展示了属性，但空条件运算符也可以用于访问方法、字段和索引器。基本规则是，当遇到空条件运算符时，编译器会在 `?.` 左侧的值上注入一个空值检查。如果该值为 `null`，则计算停止，整个表达式的结果为 `null`。否则，计算将继续进行 `?.` 右侧的属性、方法、字段或索引访问，而不会重新计算表达式的第一部分。如果整个表达式的类型在没有空条件运算符的情况下是不可空值类型，那么在序列中任何地方涉及空条件运算符时，它会变成其对应的可空类型。

这里的整个表达式——如果在遇到空值时计算会突然停止的部分——基本上就是所涉及的属性、字段、索引器和方法访问的序列。其他运算符，例如比较运算符，会由于优先级规则而中断序列。为了演示这一点，让我们更仔细地看看第 10.3.1 节中 `Where` 方法的条件。我们的 Lambda 表达式如下：

```csharp
c => c.Profile?.DefaultShippingAddress?.Town == "Reading"
```

编译器处理它大致就像你写了这样：

```csharp
string result;
var tmp1 = c.Profile;
if (tmp1 == null)
{
    result = null;
}
else
{
    var tmp2 = tmp1.DefaultShippingAddress;
    if (tmp2 == null)
    {
        result = null;
    }
    else
    {
        result = tmp2.Town;
    }
}
return result == "Reading";
```

注意每个属性访问（我已用粗体突出显示）只发生一次。在我们检查空值的 C# 6 之前的版本中，你可能会计算 `c.Profile` 三次，计算 `c.Profile.DefaultShippingAddress` 两次。如果这些计算依赖于其他线程突变的数据，你可能会遇到麻烦：你可能通过了前两个空值测试，但仍然会因 `NullReferenceException` 而失败。C# 代码更安全、更高效，因为你只计算一次所有内容。

## **处理布尔比较**

目前，你仍然在最后使用 `==` 运算符进行比较；如果任何部分为 `null`，这不会被短路掉。假设你想改用 `Equals` 方法并这样写：

```csharp
c => c.Profile?.DefaultShippingAddress?.Town?.Equals("Reading")
```

不幸的是，这无法编译。你添加了第三个空条件运算符，这样如果有一个收货地址的 `Town` 属性为 `null`，你就不会调用 `Equals`。但现在整体结果是 `Nullable<bool>` 而不是 `bool`，这意味着我们的 Lambda 表达式还不适合用于 `Where` 方法。

这是使用空条件运算符时相当常见的情况。任何时候你在任何类型的条件中使用空条件运算符，都需要考虑三种可能性：
*   表达式的每个部分都被计算，结果为 `true`。
*   表达式的每个部分都被计算，结果为 `false`。
*   由于空值，表达式短路，结果为 `null`。

通常，你想通过将第三种情况映射为 `true` 或 `false` 结果，将这三种可能性归结为两种。有两种常见的方法：与 `bool` 常量比较，或者使用空合并 `??` 运算符。

**可空布尔比较的语言设计选择**
`bool?` 在与不可空值比较时的行为在 C# 2 时期就引起了语言设计者的关注。如果 `x` 是一个 `bool?` 变量，`x == true` 和 `x != false` 都是有效的，但含义不同，这可能相当令人惊讶。（如果 `x` 为 `null`，`x == true` 计算结果为 `false`，而 `x != false` 计算结果为 `true`。）

这是正确的设计选择吗？也许。通常所有可用的选择在某一方面或其他方面都不尽如人意。不过现在不会改变了，所以最好意识到这一点，并为可能不太清楚的读者尽可能清晰地编写代码。

为了简化我们的示例，假设你已有一个名为 `name` 的变量，其中包含相关的字符串值，但它可能为 `null`。你想写一个 `if` 语句，并根据 `Equals` 方法，如果城镇为 X 则执行语句的主体。这是演示条件的最简单方法：在实际生活中，你可能需要条件性地访问一个布尔属性。表 10.1 根据你是否也希望在 `name` 为 `null` 时进入语句主体，展示了你可以使用的选项。

**表 10.1 使用空条件运算符执行布尔比较的选项**

| 你不想在 `name` 为 `null` 时进入主体 | `if (name?.Equals("X") ?? false)` 或 `if (name?.Equals("X") == true)` |
| :----------------------------------- | :----------------------------------------------------------- |
| 你想在 `name` 为 `null` 时进入主体   | `if (name?.Equals("X") ?? true)` 或 `if (name?.Equals("X") != false)` |

我更喜欢空合并运算符方法；我把它读作“尝试执行比较，但如果必须提前停止，则默认使用 `??` 之后的值”。一旦你理解了表达式的类型（本例中是 `name?.Equals("X")`）是 `Nullable<bool>`，这里就没有什么新内容了。只是碰巧，在空条件运算符出现之后，你更有可能遇到这种情况。

## **索引器和空条件运算符**

正如我前面提到的，空条件运算符不仅适用于字段、属性和方法，也适用于索引器。语法同样是添加一个问号，但这次是在开方括号之前。这适用于数组访问和用户定义的索引器，同样，如果结果类型原本是不可空值类型，则结果类型会变成可空类型。下面是一个简单的例子：

```csharp
int[] array = null;
int? firstElement = array?[0];
```

关于空条件运算符如何与索引器一起工作，没有太多要说的；就这么简单。我发现这远不如处理属性和方法有用，但仍然值得知道它的存在，这更多是为了保持一致性。

## **有效地使用空条件运算符**

你已经看到空条件运算符在处理可能为空的属性对象模型时很有用，但还有其他引人注目的用例。我们在这里看其中两个，但这并不是一个详尽的列表，你自己可能还会想出其他新颖的用法。

**安全便捷的事件触发**
即使在面对多线程的情况下安全地触发事件的模式也广为人知很多年了。例如，要触发一个类型为 `EventHandler`、类似字段的 `Click` 事件，你会这样写代码：

```csharp
EventHandler handler = Click;
if (handler != null)
{
    handler(this, EventArgs.Empty);
}
```

这里有两个方面很重要：
*   你不是直接调用 `Click(this, EventArgs.Empty)`，因为 `Click` 可能为 `null`。（如果没有处理程序订阅该事件，就会是这种情况。）
*   你首先将 `Click` 字段的值复制到一个局部变量，这样即使它在另一个线程中在你检查空值之后发生了变化，你仍然有一个非空引用。你可能会调用一个“稍旧”（刚刚取消订阅的）事件处理程序，但这是一个合理的竞态条件。

到目前为止，一切都好——但是太冗长了。然而，空条件运算符来救援了。它不能用于 `handler(...)` 这种简写的委托调用方式，但你可以用它来有条件地调用 `Invoke` 方法，并且全部在一行内完成：

```csharp
Click?.Invoke(this, EventArgs.Empty);
```

如果这是你的方法（`OnClick` 或类似方法）中唯一的一行，那么这还有一个额外的好处，即现在它是一个单表达式主体，因此可以写成一个表达式主体方法。它与之前的模式一样安全，但简洁得多。

**充分利用返回 null 的 API**
在第 9 章中，我讨论了日志记录以及内插字符串字面量在性能方面没有帮助。但是，如果你有一个考虑到这种模式而设计的日志记录 API，它们可以与空条件运算符干净地结合。例如，假设你有一个类似于下个清单所示的日志记录器 API。

**清单 10.12 一个面向空条件的日志记录 API 草图**

```csharp
public interface ILogger
{
    IActiveLogger Debug { get; } // 当日志禁用时返回 null 的属性
    IActiveLogger Info { get; }
    IActiveLogger Warning { get; }
    IActiveLogger Error { get; }
}

public interface IActiveLogger // 表示启用的日志接收器的接口
{
    void Log(string message);
}
```

这只是一个草图；完整的日志记录 API 会有更多内容。但是，通过将获取特定日志级别的活动日志记录器的步骤与执行日志记录的步骤分开，你可以编写高效且信息丰富的日志记录：

```csharp
logger.Debug?.Log($"Received request for URL {request.Url}");
```

如果调试日志记录被禁用，你永远不会走到格式化内插字符串字面量那一步，并且你可以在不创建任何对象的情况下确定这一点。如果调试日志记录已启用，则内插字符串字面量将被计算并像往常一样传递给 `Log` 方法。不说过多的溢美之词，这就是让我热爱 C# 发展方式的那种东西。

当然，你需要日志记录 API 首先以适当的方式处理这一点。如果你正在使用的任何日志记录 API 没有类似这样的东西，扩展方法可能会帮助你。

许多反射 API 会在适当的时候返回 `null`，LINQ 的 `FirstOrDefault`（及类似）方法可以很好地与空条件运算符配合使用。同样，LINQ to XML 有许多方法在找不到你要找的东西时返回 `null`。例如，假设你有一个 XML 元素，它有一个可选的 `<author>` 元素，该元素可能有一个 `name` 属性，也可能没有。你可以使用以下任一语句轻松检索作者姓名：

```csharp
string authorName = book.Element("author")?.Attribute("name")?.Value;
string authorName = (string) book.Element("author")?.Attribute("name");
```

第一个使用了两次空条件运算符：一次用于访问元素的属性，一次用于访问属性的值。第二种方法利用了 LINQ to XML 在其显式转换运算符中已经接受空值的方式。

## **空条件运算符的限制**

除了偶尔需要处理可空值类型，而以前你只使用不可空值类型之外，空条件运算符几乎没有什么令人不快的意外。唯一可能让你惊讶的是，表达式的结果总是被归类为一个值，而不是一个变量。这样做的结果是，你不能将空条件运算符用作赋值的左侧。例如，以下所有代码都是无效的：

```csharp
person?.Name = "";
stats?.RequestCount++;
array?[index] = 10;
```

在这些情况下，你需要使用老式的 `if` 语句。根据我的经验，这个限制很少成为问题。

空条件运算符对于避免 `NullReferenceException` 非常有用，但有时异常的发生有更合理的原因，你需要能够处理它们。异常过滤器代表了自 C# 首次引入以来对 `catch` 块结构的第一次改变。



> 空条件运算符（`?.`）是 C# 6 为处理空值引入的革命性语法糖。其核心价值在于**安全地缩短访问链**，并**将`null`作为合法的、可传播的返回值**，从而在语法层面大幅减少因空引用导致的运行时异常和重复的防御性代码。
>
> 它不仅是简单的编译时代码替换（确保中间表达式只求值一次，保证线程安全），更改变了 API 的设计思路。如事件触发和日志 API 的例子所示，通过设计返回`null`的成员（如`ILogger.Debug`），开发者可以结合`?.`与内插字符串，实现**零开销的条件化昂贵操作**（字符串格式化），这是此前难以优雅实现的模式。
>
> 需注意，它会使表达式类型“升格”为可空类型（如`bool?`），常需配合空合并运算符（`??`）或与`true`/`false`比较来得到最终布尔值。同时，它只能作为取值操作，不能用于赋值左侧。
>
> 总体而言，`?.` 通过将 `null` 检查无缝嵌入成员访问语法，使代码在表达安全意图时更简洁、更专注，是向更健壮、更声明式编程风格迈进的关键一步。





# **异常过滤器**

本章的最后一个特性有点令人尴尬：这是 C# 在追赶 VB。是的，VB 一直都有异常过滤器，但它们是直到 C# 6 才被引入的。这是另一个你可能很少使用的特性，但它能让你有趣地窥探 CLR 的内部机制。基本前提是，你现在可以编写 `catch` 块，使其仅有时捕获异常，这取决于过滤器表达式返回 `true` 还是 `false`。如果它返回 `true`，则捕获异常。如果它返回 `false`，则忽略该 `catch` 块。

作为一个例子，假设你正在执行一个 Web 操作，并且知道你所连接的服务器有时会离线。如果你无法连接到它，你还有另一个选择，但任何其他类型的失败都应该以正常方式导致异常向上冒泡。在 C# 6 之前，你必须捕获异常，并在其状态不正确时重新抛出：

```csharp
try
{
    ... // 尝试进行 Web 操作
}
catch (WebException e)
{
    if (e.Status != WebExceptionStatus.ConnectFailure)
    {
        throw; // 如果不是连接失败，则重新抛出
    }
    ... // 处理连接失败
}
```

有了异常过滤器，如果你不想处理一个异常，你就不捕获它；你从一开始就用过滤器把它从你的 `catch` 块中过滤掉：

```csharp
try
{
    ... // 尝试进行 Web 操作
}
catch (WebException e) when (e.Status == WebExceptionStatus.ConnectFailure)
{
    ... // 处理连接失败
}
```

除了像这样的特定情况，我还能看到异常过滤器在两个通用用例中很有用：重试和日志记录。在重试循环中，通常只有当你要重试操作时才想捕获异常（如果它满足某些条件并且你还没有用完尝试次数）；在日志记录场景中，你可能永远不想捕获异常，但可以说，在它“飞行”过程中记录它。在深入了解具体用例的更多细节之前，让我们看看这个特性在代码中是什么样子以及它的行为如何。

**异常过滤器的语法和语义**
我们的第一个完整示例如下面的清单所示，很简单：它遍历一组消息，并为每条消息抛出一个异常。你有一个异常过滤器，只有当消息包含单词“catch”时才会捕获异常。异常过滤器用粗体标出。

**清单 10.13 抛出三个异常并捕获其中两个**

```csharp
string[] messages =
{
    "You can catch this",
    "You can catch this too",
    "This won't be caught"
};
foreach (string message in messages) // 每条消息执行一次外部循环
{
    try
    {
        throw new Exception(message); // 每次抛出带有不同消息的异常
    }
    catch (Exception e) when (e.Message.Contains("catch")) // 仅当消息包含“catch”时才捕获异常
    {
        Console.WriteLine($"Caught '{e.Message}'"); // 输出被捕获异常的消息
    }
}
```

输出是被捕获的两个异常的两行信息：
```
Caught 'You can catch this'
Caught 'You can catch this too'
```

未捕获异常的输出是一条消息“This won't be caught”。（具体显示取决于你如何运行代码，但这属于正常的未处理异常。）

从语法上讲，异常过滤器就这些内容：上下文关键字 `when` 后面跟着括号内的一个表达式，该表达式可以使用在 `catch` 子句中声明的异常变量，并且必须计算为布尔值。然而，其语义可能不完全符合你的预期。

**两阶段异常模型**
你可能已经习惯了 CLR 在异常“冒泡”直到被捕获时展开堆栈的概念。更令人惊讶的是这个过程具体是如何发生的。这个过程比你想象的要复杂，它使用一个两阶段模型。该模型使用以下步骤：
1.  异常被抛出，第一阶段开始。
2.  CLR 沿着堆栈向下搜索，尝试找到哪个 `catch` 块将处理该异常。（我们简称之为处理 `catch` 块，但这不是官方术语。）
3.  仅考虑具有兼容异常类型的 `catch` 块。
4.  如果一个 `catch` 块有异常过滤器，则执行该过滤器；如果过滤器返回 `false`，则此 `catch` 块不会处理该异常。
5.  没有异常过滤器的 `catch` 块等同于一个返回 `true` 的异常过滤器。
6.  一旦确定了处理 `catch` 块，第二阶段开始：
7.  CLR 从抛出异常的位置开始，将堆栈展开到已确定的 `catch` 块所在的位置。
8.  在展开堆栈过程中遇到的任何 `finally` 块都将被执行。（这不包括与处理 `catch` 块关联的任何 `finally` 块。）
9.  执行处理 `catch` 块。
10. 执行与处理 `catch` 块关联的 `finally` 语句（如果有的话）。

清单 10.14 通过三个重要方法展示了一个具体的例子：`Bottom`、`Middle` 和 `Top`。`Bottom` 调用 `Middle`，`Middle` 调用 `Top`，因此堆栈最终是自描述的。`Main` 方法调用 `Bottom` 来启动整个过程。请不要被这段代码的长度吓到；它并没有做什么非常复杂的事情。同样，异常过滤器用粗体标出。`LogAndReturn` 方法只是一种方便跟踪执行的方式。异常过滤器使用它来记录特定方法，然后返回指定值以指示是否应捕获异常。

**清单 10.14 异常过滤的三层演示**

```csharp
static bool LogAndReturn(string message, bool result)
{
    Console.WriteLine(message);
    return result;
}
// 异常过滤器调用的便捷方法

static void Top()
{
    try
    {
        throw new Exception();
    }
    finally
    {
        Console.WriteLine("Top finally"); // 第二阶段执行的 finally 块（无 catch）
    }
}

static void Middle()
{
    try
    {
        Top();
    }
    catch (Exception e) when (LogAndReturn("Middle filter", false)) // 永远不捕获的异常过滤器
    {
        Console.WriteLine("Caught in middle"); // 因为过滤器返回 false，所以这行永远不会打印。
    }
    finally
    {
        Console.WriteLine("Middle finally"); // 第二阶段执行的 finally 块
    }
}

static void Bottom()
{
    try
    {
        Middle();
    }
    catch (IOException e) when (LogAndReturn("Never called", true)) // 永远不会被调用的异常过滤器——异常类型不匹配
    {
    }
    catch (Exception e) when (LogAndReturn("Bottom filter", true)) // 总是捕获的异常过滤器
    {
        Console.WriteLine("Caught in Bottom"); // 因为在这里捕获了异常，所以这行会打印。
    }
}

static void Main()
{
    Bottom();
}
```

喘口气！有了前面的描述和清单中的注释，你已经有足够的信息来推断输出结果是什么。我们将逐步分析以确保完全清楚。首先，看看打印了什么：
```
Middle filter
Bottom filter
Top finally
Middle finally
Caught in Bottom
```
图 10.2 展示了这个过程。在每个步骤中，左侧显示堆栈（忽略 `Main`），中间部分描述正在发生的事情，右侧显示该步骤的任何输出。

![image-20260208140159366](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260208140159507.png)

> **两阶段模型的安全影响**
> `finally` 块的执行时机也影响着 `using` 和 `lock` 语句。这有一个重要的含义，即如果你在可能包含恶意代码的环境中编写可能被执行的代码，你可以使用 `try/finally` 或 `using` 来做什么。如果你的方法可能被你不信任的代码调用，并且你允许异常从该方法逃逸，那么调用者可以使用异常过滤器在你的 `finally` 块执行之前执行代码。
>
> 所有这一切意味着你不应该将 `finally` 用于任何安全敏感的用途。例如，如果你的 `try` 块进入了一个更高权限的状态，并且你依赖一个 `finally` 块来返回到较低权限的状态，那么在你仍然处于该高权限状态时，其他代码可能会执行。许多代码不需要担心这类事情——它总是在友好条件下运行——但你绝对应该意识到这一点。如果你担心，你可以使用一个空的 `catch` 块，搭配一个移除权限并返回 `false`（这样异常就不会被捕获）的过滤器，但这不是我想经常做的事情。
>

**多次捕获同一异常类型**
过去，在同一个 `try` 块的多个 `catch` 块中指定相同的异常类型总是一个错误。这没有任何意义，因为第二个块永远不会被访问到。有了异常过滤器，这就合理多了。

为了演示这一点，让我们扩展最初的 `WebException` 示例。假设你正在基于用户提供的 URL 获取 Web 内容。你可能希望以某种方式处理连接失败，以不同的方式处理名称解析失败，并让任何其他类型的异常冒泡到更高级别的 `catch` 块。使用异常过滤器，你可以简单地做到：

```csharp
try
{
    ... // 尝试进行 Web 操作
}
catch (WebException e) when (e.Status == WebExceptionStatus.ConnectFailure)
{
    ... // 处理连接失败
}
catch (WebException e) when (e.Status == WebExceptionStatus.NameResolutionFailure)
{
    ... // 处理名称解析失败
}
```

如果你想在同一级别处理所有其他 `WebException`，在两个特定状态的 `catch` 块之后，再添加一个没有异常过滤器的通用 `catch (WebException e) { ... }` 块也是有效的。

现在你知道了异常过滤器的工作原理，让我们回到我之前给出的两个通用示例。这些并不是唯一的用途，但它们应该能帮助你识别其他类似的情况。让我们从重试开始。

## **重试操作**

随着云计算变得越来越普遍，我们通常越来越意识到操作可能失败，并需要思考我们希望这种失败对我们的代码产生什么影响。对于远程操作——例如 Web 服务调用和数据库操作——有时存在瞬态故障，重试是完全安全的。

**跟踪你的重试策略**
尽管能够像这样重试很有用，但要注意代码中可能尝试重试失败操作的每一层。如果你有多个抽象层，每层都试图友好且透明地重试可能是瞬态的失败，你最终可能会将记录真实故障的时间推迟很久。简而言之，这是一种自身组合性不好的模式。

如果你控制着应用程序的整个堆栈，你应该思考希望重试发生在哪里。如果你只负责其中的一个方面，你应该考虑使重试可配置，以便控制整个堆栈的开发人员可以确定是否希望在你这一层进行重试。

生产环境的重试处理有些复杂。你可能需要复杂的启发式方法来确定何时重试以及重试多长时间，并且在尝试之间的延迟中加入随机性，以避免重试的客户端彼此同步。清单 10.15 提供了一个高度简化的版本，以避免让你分心而忽略异常过滤器的方面。

你的代码只需要知道以下两点：
*   你试图执行什么操作
*   你愿意尝试多少次

此时，使用异常过滤器来仅在你将要重试操作时才捕获异常，代码就变得简单明了。

**清单 10.15 一个简单的重试循环**

```csharp
static T Retry<T>(Func<T> operation, int attempts)
{
    while (true)
    {
        try
        {
            attempts--;
            return operation();
        }
        catch (Exception e) when (attempts > 0)
        {
            Console.WriteLine($"Failed: {e}");
            Console.WriteLine($"Attempts left: {attempts}");
            Thread.Sleep(5000);
        }
    }
}
```

尽管 `while(true)` 循环很少是个好主意，但这个有意义。你可以编写一个基于 `retryCount` 的条件循环，但异常过滤器实际上已经提供了这个条件，所以那样会误导人。而且，从编译器的角度来看，循环的末尾将是可达的，因此在方法的末尾没有 `return` 或 `throw` 语句将无法编译。

有了这个方法之后，调用它来实现重试很简单：

```csharp
Func<DateTime> temporamentalCall = () =>
{
    DateTime utcNow = DateTime.UtcNow;
    if (utcNow.Second < 20)
    {
        throw new Exception("I don't like the start of a minute");
    }
    return utcNow;
};
var result = Retry(temporamentalCall, 3);
Console.WriteLine(result);
```

通常，这会立即返回结果。有时，如果你在每分钟大约 10 秒的时候执行它，它会失败几次然后成功。有时，如果你在每分钟刚开始的时候执行它，它会失败几次，捕获异常并记录它，然后第三次失败，此时异常将不会被捕获。

## **日志记录作为副作用**

我们的第二个示例是一种在“飞行中”记录异常的方法。我意识到我已经使用日志记录来演示许多 C# 6 的特性，但这只是巧合。我不相信 C# 团队决定专门针对这个版本以日志记录为目标；这只是作为一个熟悉的场景效果很好。

关于究竟如何以及在何处记录异常才有意义，是一个备受争议的话题，我不打算在这里参与这场辩论。相反，我将断言，至少有时，在一个方法调用中记录异常是有用的，即使它将在堆栈的更下方某个地方被捕获（并且可能被第二次记录）。

你可以使用异常过滤器来记录异常，而不会以任何其他方式干扰执行流程。你所需要的只是一个异常过滤器，它调用一个方法来记录异常，然后返回 `false` 以指示你并不真的想捕获该异常。以下清单在一个 `Main` 方法中演示了这一点，该方法仍将导致进程以错误代码完成，但仅在它用时间戳记录了异常之后。

**清单 10.16 在过滤器中记录日志**

```csharp
static void Main()
{
    try
    {
        UnreliableMethod();
    }
    catch (Exception e) when (Log(e))
    {
    }
}

static void UnreliableMethod()
{
    throw new Exception("Bang!");
}

static bool Log(Exception e)
{
    Console.WriteLine($"{DateTime.UtcNow}: {e.GetType()} {e.Message}");
    return false;
}
```

这个清单在很多方面只是清单 10.14 的一个变体，在那里我们使用日志记录来研究两阶段异常系统的语义。在这种情况下，你永远不会在过滤器中捕获异常；整个 `try/catch` 和过滤器的存在仅仅是为了记录日志的副作用。

## **个别、特定于案例的异常过滤器**

除了那些通用示例外，特定的业务逻辑有时要求捕获某些异常，而让其他异常进一步传播。如果你怀疑这是否有用，考虑一下你是否总是捕获 `Exception`，或者你是否倾向于捕获特定的异常类型，如 `IOException` 或 `SqlException`。考虑下面的块：

```csharp
catch (IOException e)
{
    ...
}
```

你可以认为这个块大致等同于：

```csharp
catch (Exception tmp) when (tmp is IOException)
{
    IOException e = (IOException) tmp;
    ...
}
```

C# 6 中的异常过滤器是这种方式的泛化。通常，相关信息并不在类型中，而是通过其他方式暴露。以 `SqlException` 为例；它有一个 `Number` 属性，对应着根本原因。以不同的方式处理某些 SQL 故障和其他故障是相当合理的。

由于 API 的原因，从 `WebException` 获取底层的 HTTP 状态码有点棘手，但同样，你可能希望以不同于 500（内部错误）的方式处理 404（未找到）响应。

一句警告：我强烈建议你不要基于异常消息进行过滤（除了像我在清单 10.13 中那样用于实验目的）。异常消息通常不被视为必须在版本之间保持稳定，并且根据来源，它们很可能是本地化的。基于特定异常消息而表现不同的代码是脆弱的。

## **为什么不直接重新抛出？**

你可能想知道这有什么大不了的。毕竟，我们一直都能够重新抛出异常。像这样使用异常过滤器的代码：

```csharp
catch (Exception e) when (condition)
{
    ...
}
```

与下面的代码并没有太大不同：

```csharp
catch (Exception e)
{
    if (!condition)
    {
        throw;
    }
    ...
}
```

这真的达到了一个新语言特性的高标准吗？这是有争议的。

这两段代码之间存在差异：你已经看到 `condition` 的评估时机相对于调用堆栈中更高位置的任何 `finally` 块发生了变化。此外，虽然简单的 `throw` 语句在很大程度上保留了原始的堆栈跟踪，但可能存在细微的差异，特别是在捕获和重新抛出异常的堆栈帧中。这当然可能使诊断错误变得简单或痛苦。

我怀疑异常过滤器会极大地改变许多开发人员的生活。当我不得不在 C# 5 代码库上工作时，我并不怀念它们，不像表达式体成员和内插字符串字面量那样，但它们仍然是个好东西。

在本章描述的特性中，`using static` 和空条件运算符无疑是我最常使用的。它们适用于广泛的情况，有时可以使代码的可读性大大提高。（特别是，如果你的代码处理大量在其他地方定义的常量，`using static` 在可读性方面可以产生天壤之别。）

空条件运算符和对象/集合初始化器改进的一个共同点是能够在一个表达式中表达复杂的操作。这加强了对象/集合初始化器早在 C# 3 中引入的好处：它允许将表达式用于字段初始化或方法参数，而这些可能原本不得不单独计算且不太方便。

**总结**
*   `using static` 指令允许你的代码引用静态类型成员（通常是常量或方法）而无需再次指定类型名称。
*   `using static` 还从指定类型导入所有扩展方法，因此你无需从命名空间导入所有扩展方法。
*   扩展方法导入方式的改变意味着，将常规静态方法转换为扩展方法在所有情况下不再是向后兼容的更改。
*   集合初始化器现在可以使用 `Add` 扩展方法，以及正在初始化的集合类型上定义的那些方法。
*   对象初始化器现在可以使用索引器，但在使用索引器和集合初始化器之间存在权衡。
*   空条件 `?.` 运算符使得处理链式操作更加容易，其中链中的一个元素可以返回 `null`。
*   异常过滤器允许基于异常的数据而不仅仅是其类型，更精确地控制捕获哪些异常。



> 异常过滤器是C# 6引入的精细控制异常处理流程的特性。其核心价值在于**在不中断异常传播的前提下，执行特定逻辑（如日志记录）或基于异常状态（非仅类型）进行更精确的捕获**。
>
> 关键洞察在于其底层**两阶段模型**：先评估过滤器（第一阶段），再执行匹配`catch`块及其外层`finally`块（第二阶段）。这带来了重要影响：1) **性能优化**：避免因捕获后再`throw`重新计算堆栈的开销；2) **安全警示**：恶意代码可利用过滤器在`finally`（如权限还原）执行前插入代码，因此`finally`块不应用于安全敏感操作。
>
> 主要应用场景包括：
> 1.  **无干扰日志记录**：在过滤器内记录异常并返回`false`，异常继续向上传播，实现了“仅记录不处理”。
> 2.  **清晰的重试逻辑**：如`catch (...) when (retryCount-- > 0)`，将重试条件直接表达在捕获点，代码意图更明确。
> 3.  **基于状态的捕获**：可对同一异常类型（如`SqlException`）使用多个`catch`块，通过过滤器（如`e.Number == 1205`）区分处理不同错误码，比在`catch`块内用`if-else`判断更清晰。
>
> 需注意：应避免依赖易变的**异常消息**进行过滤。此特性虽非每日必用，但在需要上述精细控制的场景下，它提供了更优雅、高效的解决方案。

