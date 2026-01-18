---
weight: 1
title: "C#4"
---



**本章涵盖内容**
- 使用动态类型实现互操作性和简化反射
- 为参数提供默认值，使调用方无需指定它们
- 为参数指定名称以使调用更清晰
- 以更简化的方式编写针对 COM 库的代码
- 通过泛型变体在泛型类型之间进行转换

C# 4 是一个有趣的版本。最引人注目的变化是引入了带有 `dynamic` 类型的动态类型。这一特性使 C# 在同一语言中既是静态类型化的（对于大多数代码），又是动态类型化的（当使用 `dynamic` 时）。这在编程语言中是罕见的。

动态类型的引入是为了互操作性，但事实证明，这对于许多开发人员的日常工作中并不常见。其他版本中的主要特性（泛型、LINQ、async/await）已成为大多数 C# 开发人员工具包的自然组成部分，但动态类型仍然相对较少使用。我相信它对需要它的人来说是有用的，至少它是一个有趣的特性。

C# 4 中的其他特性也改善了互操作性，特别是与 COM 的互操作。有些改进是特定于 COM 的，例如命名索引器、隐式 `ref` 参数和嵌入式互操作类型。可选参数和命名参数对于 COM 很有用，但它们也可以在纯托管代码中使用。这两个是我日常使用的 C# 4 特性。

最后，C# 4 公开了从 CLR v2（第一个包含泛型的运行时版本）就已存在的泛型特性。泛型变体既简单又复杂。乍一看，它似乎很明显：例如，一个字符串序列显然是一个对象序列。但随后我们发现，字符串列表并不是对象列表，这打破了一些开发人员的期望。这是一个有用的特性，但当你仔细研究它时，很容易让人头痛。大多数时候，你甚至可以在没有意识到的情况下利用它。希望本章的介绍能让你在代码未按预期工作而需要更深入研究时，能够很好地解决问题而不感到困惑。我们将从动态类型开始。

# 动态类型
有些特性伴随着大量新语法，但解释完语法后，就没有太多可说的了。动态类型则恰恰相反：语法极其简单，但我可以几乎无休止地详细说明其影响和实现。本节将向你展示基础知识，然后介绍一些细节，最后就如何使用和何时使用动态类型提出一些建议。

## 动态类型简介
让我们从一个例子开始。以下清单展示了两种从文本中获取子字符串的尝试。目前，我并不打算解释为什么要使用动态类型，只说明它的作用。

清单 4.1 使用动态类型获取子字符串
```csharp
dynamic text = "hello world"; // 声明一个 dynamic 类型的变量
string world = text.Substring(6); // 调用 Substring 方法；这可行。
Console.WriteLine(world);
string broken = text.SUBSTR(6); // 尝试调用 SUBSTR；这将抛出异常。
Console.WriteLine(broken);
```

对于如此少量的代码，这里发生了很多事情。最重要的方面是它竟然能够编译。如果你将第一行改为使用 `string` 类型声明 `text`，那么对 `SUBSTR` 的调用将在编译时失败。相反，编译器很乐意编译它，甚至不查找名为 `SUBSTR` 的方法。它也不查找 `Substring`。相反，这两个查找都在运行时执行。

在运行时，第二行将查找一个名为 `Substring` 且可以使用参数 6 调用的方法。找到该方法并返回一个字符串，然后你将其赋值给 `world` 变量并以常规方式打印。当代码查找一个名为 `SUBSTR` 且可以使用参数 6 调用的方法时，它没有找到任何此类方法，代码将失败并抛出 `RuntimeBinderException`。

如第 3 章所述，在特定上下文中查找名称含义的过程称为绑定。动态类型就是将绑定的发生时间从编译时更改为运行时。编译器不会生成调用在编译时确定精确签名的方法的 IL，而是生成执行绑定然后根据结果采取行动的 IL。所有这些都是通过使用 `dynamic` 类型触发的。

**什么是 dynamic 类型？**
清单 4.1 将 `text` 变量声明为 `dynamic` 类型：
```csharp
dynamic text = "hello world";
```
`dynamic` 类型是什么？它不同于你在 C# 中看到的其他类型，因为它仅在 C# 语言层面存在。没有与之关联的 `System.Type`，CLR 根本不知道它。每当你在 C# 中使用 `dynamic` 时，如果需要，IL 会使用用 `[Dynamic]` 修饰的 `object`。

**注意**：如果 `dynamic` 类型用于方法签名中，编译器需要使该信息可供针对其编译的代码使用。对于局部变量则不需要这样做。

`dynamic` 类型的基本规则很简单：
1. 从任何非指针类型到 `dynamic` 都存在隐式转换。
2. 从 `dynamic` 类型的表达式到任何非指针类型都存在隐式转换。
3. 涉及 `dynamic` 类型值的表达式通常在运行时绑定。
4. 大多数涉及 `dynamic` 类型值的表达式的编译时类型也是 `dynamic`。

你很快就会看到最后两点的例外情况。使用这个规则列表，你可以用新的眼光再次查看清单 4.1。让我们考虑前两行：
```csharp
dynamic text = "hello world";
string world = text.Substring(6);
```
在第一行中，你正在从 `string` 转换为 `dynamic`，根据规则 1，这是可以的。第二行演示了其他三个规则：
- `text.Substring(6)` 在运行时绑定（规则 3）。
- 该表达式的编译时类型是 `dynamic`（规则 4）。
- 从该表达式到 `string` 存在隐式转换（规则 2）。

从 `dynamic` 类型的表达式到非动态类型的转换也是动态绑定的。如果你将 `world` 变量声明为 `int` 类型，这将编译，但在运行时失败并抛出 `RuntimeBinderException`。如果你将其声明为 `XNamespace` 类型，这将编译，然后在运行时绑定器将使用从 `string` 到 `XNamespace` 的用户定义的隐式转换。

考虑到这一点，让我们看看更多动态绑定的例子。



**在各种上下文中应用动态绑定**
到目前为止，你已经看到了基于方法调用的动态目标然后进行转换的动态绑定，但几乎执行的任何方面都可能是动态的。以下清单在加法运算符的上下文中展示了这一点，并根据运行时动态值的类型执行了三种加法。

清单 4.2 动态值的加法
```csharp
static void Add(dynamic d)
{
    Console.WriteLine(d + d); // 根据运行时类型执行加法
}

Add("text"); // 以不同值调用方法
Add(10);
Add(TimeSpan.FromMinutes(45));
```

清单 4.2 的结果如下：
```
texttext
20
01:30:00
```

每种加法对所涉及的类型都有意义，但在静态类型化上下文中，它们看起来会不同。作为最后一个示例，以下清单展示了方法重载在动态方法参数下的行为。

清单 4.3 动态方法重载解析
```csharp
static void SampleMethod(int value)
{
    Console.WriteLine("Method with int parameter");
}

static void SampleMethod(decimal value)
{
    Console.WriteLine("Method with decimal parameter");
}

static void SampleMethod(object value)
{
    Console.WriteLine("Method with object parameter");
}

static void CallMethod(dynamic d)
{
    SampleMethod(d); // 动态调用 SampleMethod
}

CallMethod(10); // 间接以不同类型调用 SampleMethod
CallMethod(10.5m);
CallMethod(10L);
CallMethod("text");
```

清单 4.3 的输出如下：
```
Method with int parameter
Method with decimal parameter
Method with decimal parameter
Method with object parameter
```

输出的第三行和第四行特别有趣。它们表明，运行时的重载解析仍然知道转换。第三行中，`long` 值被转换为 `decimal` 而不是 `int`，尽管它在 `int` 的范围内。第四行中，`string` 值被转换为 `object`。目的是尽可能使运行时的绑定行为与编译时的行为相同，只是使用在运行时发现的动态值的类型。

> **仅动态值被视为动态的**
> 编译器努力确保在运行时可用正确的信息。当绑定涉及多个值时，对于任何静态类型化的值，使用编译时类型；但对于任何 `dynamic` 类型的值，使用运行时类型。大多数情况下，这种细微差别无关紧要，但我已在可下载源代码中提供了一个带注释的示例。

任何动态绑定方法调用的结果都具有 `dynamic` 的编译时类型。当绑定发生时，如果所选方法具有 `void` 返回类型并且使用了方法的结果（例如，赋值给变量），则绑定失败。大多数动态绑定操作都是这种情况：编译器对动态操作将涉及什么知之甚少。该规则有一些例外情况。

**在动态绑定上下文中编译器可以检查什么？**
如果方法调用的上下文在编译时已知，编译器能够检查是否存在具有指定名称的方法。如果在运行时不可能匹配任何方法，仍然会报告编译时错误。这适用于以下情况：

- 目标不是动态值的实例方法和索引器
- 静态方法
- 构造函数

以下清单展示了各种使用动态值但在编译时失败的调用示例。

清单 4.4 涉及动态值的编译时失败示例
```csharp
dynamic d = new object();
int invalid1 = "text".Substring(0, 1, 2, d); // 没有接受四个参数的 String.Substring 方法
bool invalid2 = string.Equals<int>("foo", d); // 没有泛型 String.Equals 方法
string invalid3 = new string(d, "broken"); // 没有接受字符串作为第二个参数的、带两个参数的 String 构造函数
char invalid4 = "text"[d, d]; // 没有带两个参数的 String 索引器
```

仅仅因为编译器能够判断这些特定示例肯定有问题，并不意味着它总是能够这样做。除非你对所涉及的值非常小心，否则动态绑定总是有点冒险。

我给出的示例如果能够编译，仍然会使用动态绑定。只有少数情况不是这样。

**涉及动态值的哪些操作不是动态绑定的？**
几乎所有使用动态值的操作都涉及某种绑定，并找到正确的方法调用、属性、转换、运算符等。只有少数事情编译器不需要生成任何绑定代码：
- 对 `object` 或 `dynamic` 类型的变量赋值。不需要转换，因此编译器只需复制现有引用。
- 将参数传递给具有相应 `object` 或 `dynamic` 类型参数的方法。这类似于变量赋值，但变量是参数。
- 使用 `is` 运算符测试值的类型。
- 使用 `as` 运算符尝试转换值。

虽然如果你通过强制转换将动态值转换为特定类型或隐式执行此操作，运行时绑定基础结构很乐意查找用户定义的转换，但 `is` 和 `as` 运算符从不使用用户定义的转换，因此不需要绑定。类似地，几乎所有涉及动态值的操作的结果也是动态的。

**涉及动态值的哪些操作仍具有静态类型？**
同样，编译器希望尽可能提供帮助。如果表达式始终只能是特定类型，编译器乐于将其作为表达式的编译时类型。例如，如果 `d` 是 `dynamic` 类型的变量，则以下情况成立：
- 表达式 `new SomeType(d)` 具有 `SomeType` 的编译时类型，即使构造函数在运行时是动态绑定的。
- 表达式 `d is SomeType` 具有 `bool` 的编译时类型。
- 表达式 `d as SomeType` 具有 `SomeType` 的编译时类型。

这就是本次介绍所需的所有细节。在第 4.1.4 节中，你将看到在编译时和运行时的意外转折。但现在你已经了解了动态类型的风格，可以看看它在执行常规运行时绑定之外的一些强大功能。



## **超越反射的动态行为**

动态类型的一个用途是，让编译器和框架基于类型中通常声明的成员，为你执行反射操作。虽然这是一个完全合理的用途，但动态类型更具扩展性。引入它的部分原因是为了更好地与允许动态绑定的动态语言进行互操作。许多动态语言允许在运行时拦截调用。这可以用于透明缓存和日志记录，或者使看起来从未在源代码中声明的函数和字段成为可能。

**数据库访问的假想示例**
作为一个（未实现的）示例，想象你有一个包含书籍表（包括作者）的数据库。动态类型可以使以下代码成为可能：
```csharp
dynamic database = new Database(connectionString);
var books = database.Books.SearchByAuthor("Holly Webb");
foreach (var book in books)
{
    Console.WriteLine(book.Title);
}
```
这将涉及以下动态操作：
- `Database` 类将通过查询数据库模式中名为 `Books` 的表来响应 `Books` 属性的请求，并返回某种表对象。
- 该表对象将通过发现方法名以 `SearchBy` 开头，并在模式中查找名为 `Author` 的列，来响应 `SearchByAuthor` 方法调用。然后，它将使用提供的参数生成按该列查询的 SQL，并返回行对象列表。
- 每个行对象将通过返回 `Title` 列的值来响应 `Title` 属性。

如果你熟悉 Entity Framework 或类似的对象关系映射（ORM），这可能听起来并不新鲜。你可以相当容易地编写类来实现相同的查询代码，或者从模式生成这些类。这里的区别在于一切都是动态的：没有 `Book` 或 `BooksTable` 类。一切都在运行时发生。在第 4.1.5 节中，我将讨论这通常是好事还是坏事，但我希望你至少能看到它在某些情况下如何有用。

在介绍允许所有这些发生的类型之前，让我们看两个已实现的示例。首先，你将查看框架中的一个类型，然后是 Json.NET。

**ExpandoObject：动态的数据和方法包**
.NET 框架在 `System.Dynamic` 命名空间中提供了一个名为 `ExpandoObject` 的类型。它根据你是否将其用作动态值以两种模式运行。以下清单提供了一个简短的示例，以帮助你理解随后的描述。

清单 4.5 在 ExpandoObject 中存储和检索项
```csharp
dynamic expando = new ExpandoObject();
expando.SomeData = "Some data"; // 将数据赋值给属性

Action<string> action = input => Console.WriteLine("The input was '{0}'", input);
expando.FakeMethod = action; // 将委托赋值给属性

Console.WriteLine(expando.SomeData); // 动态访问数据和委托
expando.FakeMethod("hello");

IDictionary<string, object> dictionary = expando; // 将 ExpandoObject 视为字典以打印键
Console.WriteLine("Keys: {0}", string.Join(", ", dictionary.Keys));

dictionary["OtherData"] = "other"; // 使用静态上下文填充数据并从动态值获取
Console.WriteLine(expando.OtherData);
```

当 `ExpandoObject` 在静态类型上下文中使用时，它是一个名称/值对的字典，并且如你所期望的那样实现了 `IDictionary<string, object>`。你可以这样使用它，查找在运行时提供的键等。

更重要的是，它还实现了 `IDynamicMetaObjectProvider`。这是动态行为的入口点。稍后我们将介绍接口本身，但 `ExpandoObject` 实现了它，因此你可以在代码中按名称访问字典键。当你在动态上下文中调用 `ExpandoObject` 上的方法时，它将在字典中查找方法名称作为键。如果与该键关联的值是具有适当参数的委托，则执行委托，委托的结果用作方法调用的结果。

清单 4.5 仅存储了一个数据值和一个委托，但你可以存储任意多个具有任意名称的数据和委托。它只是一个可以动态访问的字典。你可以使用 `ExpandoObject` 实现前面数据库示例的大部分内容。你可以创建一个来表示 `Books` 表，然后也用单独的 `ExpandoObject` 表示每本书。该表将有一个 `SearchByAuthor` 键，其值为合适的委托来执行查询。每本书将有一个存储书名的 `Title` 键，等等。但在实践中，你可能希望直接实现 `IDynamicMetaObjectProvider` 或使用 `DynamicObject`。在深入研究这些类型之前，让我们看看另一个实现：动态访问 JSON 数据。

**JSON.NET 的动态视图**
如今 JSON 无处不在，Json.NET 是最受欢迎的用于消费和创建 JSON 的库之一。它提供了多种处理 JSON 的方式，包括直接解析为用户提供的类以及解析为更接近 LINQ to XML 的对象模型。后者称为 LINQ to JSON，包含 `JObject`、`JArray` 和 `JProperty` 等类型。它可以像 LINQ to XML 一样使用，通过字符串访问，也可以动态使用。以下清单展示了同一 JSON 的两种方法。

清单 4.6 动态使用 JSON 数据
```csharp
string json = @"
{
    'name': 'Jon Skeet',
    'address': {
        'town': 'Reading',
        'country': 'UK'
    }
}".Replace('\'', '"'); // 硬编码的示例 JSON

JObject obj1 = JObject.Parse(json); // 将 JSON 解析为 JObject
Console.WriteLine(obj1["address"]["town"]); // 使用静态类型视图

dynamic obj2 = obj1;
Console.WriteLine(obj2.address.town); // 使用动态类型视图
```

这个 JSON 很简单，但包含嵌套对象。代码的后半部分展示了如何通过使用 LINQ to JSON 中的索引器或使用它提供的动态视图来访问它。

你更喜欢哪种方法？每种方法都有支持和反对的论据。两者都容易拼写错误，无论是在字符串字面量中还是在动态属性访问中。静态类型视图适合将属性名提取为常量以便重用，而动态类型视图在原型设计时更易读。我将在第 4.1.5 节中提出一些关于何时何地使用动态类型的建议，但在此之前，值得反思你的最初反应。接下来，我们将快速了解如何自己实现所有这些。

**在自己的代码中实现动态行为**
**动态行为很复杂。让我们一开始就承认这一点。请不要期望在本节之后就能为你任何惊人的想法编写生产就绪的优化实现。这只是一个起点。也就是说，它应该足以让你探索和实验，以便决定你希望投入多少精力来学习所有细节。**

当我介绍 `ExpandoObject` 时，我提到它实现了 `IDynamicMetaObjectProvider` 接口。这个接口表示对象实现了自己的动态行为，而不是仅仅让基于反射的基础设施以正常方式工作。作为一个接口，它看起来简单得令人误解：
```csharp
public interface IDynamicMetaObjectProvider
{
    DynamicMetaObject GetMetaObject(Expression parameter);
}
```

复杂性在于 `DynamicMetaObject`，它是驱动其他一切的类。其官方文档给出了一个线索，表明在使用它时需要思考的层次：

> 表示参与动态绑定的对象的动态绑定和绑定逻辑。

即使使用过这个类，我也不想声称我完全理解这个句子，也无法写出更好的描述。通常，你会创建一个继承自 `DynamicMetaObject` 的类，并覆盖它提供的一些虚方法。例如，如果你想动态处理方法调用，你可以覆盖这个方法：
```csharp
public virtual DynamicMetaObject BindInvokeMember
    (InvokeMemberBinder binder, DynamicMetaObject[] args);
```

`binder` 参数提供诸如被调用方法的名称以及调用者是否期望区分大小写执行绑定等信息。`args` 参数以更多 `DynamicMetaObject` 值的形式提供调用者提供的参数。结果是另一个 `DynamicMetaObject`，表示应如何处理方法调用。它不会立即执行调用，而是创建一个表示调用将执行的操作的表达式树。

所有这些都极其复杂，但允许高效处理复杂情况。幸运的是，你不必自己实现 `IDynamicMetaObjectProvider`，我也不打算尝试这样做。相反，我将给出一个使用更友好的类型的示例：`DynamicObject`。

`DynamicObject` 类作为希望尽可能简单地实现动态行为的类型的基类。结果可能不如你自己直接实现 `IDynamicMetaObjectProvider` 高效，但更容易理解。

作为一个简单的例子，你将创建一个类（`SimpleDynamicExample`），具有以下动态行为：
- 调用其上的任何方法都会在控制台打印一条消息，包括方法名和参数。
- 获取属性通常返回该属性名并带有前缀，以表明你确实调用了动态行为。

以下清单展示了如何使用该类。

清单 4.7 动态行为的预期使用示例
```csharp
dynamic example = new SimpleDynamicExample();
example.CallSomeMethod("x", 10);
Console.WriteLine(example.SomeProperty);
```

输出应如下：
```
Invoked: CallSomeMethod(x, 10)
Fetched: SomeProperty
```

`CallSomeMethod` 和 `SomeProperty` 这两个名称并没有什么特别之处，但如果你愿意，你可以以不同的方式对特定名称做出反应。即使到目前为止描述的简单行为，使用低级接口也很难正确实现，但以下清单展示了使用 `DynamicObject` 是多么容易。

清单 4.8 实现 SimpleDynamicExample
```csharp
class SimpleDynamicExample : DynamicObject
{
    public override bool TryInvokeMember(  // 处理方法调用
        InvokeMemberBinder binder,
        object[] args,
        out object result)
    {
        Console.WriteLine("Invoked: {0}({1})",
            binder.Name, string.Join(", ", args));
        result = null;
        return true;
    }

    public override bool TryGetMember(  // 处理属性访问
        GetMemberBinder binder,
        out object result)
    {
        result = "Fetched: " + binder.Name;
        return true;
    }
}
```

与 `DynamicMetaObject` 上的方法一样，在覆盖 `DynamicObject` 中的方法时，你仍然会收到绑定器，但你不再需要担心表达式树或其他 `DynamicMetaObject` 值。每个方法的返回值指示动态对象是否成功处理了操作。如果返回 `false`，将抛出 `RuntimeBinderException`。

这就是我要展示的关于实现动态行为的内容，但我希望清单 4.8 的简洁性能鼓励你尝试 `DynamicObject`。即使你从未在生产中使用它，玩弄它也会很有趣。如果你想尝试但没有具体的想法，可以尝试实现我在本节开头给出的数据库示例。提醒一下，以下是你试图实现的代码：
```csharp
dynamic database = new Database(connectionString);
var books = database.Books.SearchByAuthor("Holly Webb");
foreach (var book in books)
{
    Console.WriteLine(book.Title);
}
```

接下来，你将查看 C# 编译器在遇到动态值时生成的代码。

## **幕后简析**

你现在可能已经意识到，我喜欢研究 C# 编译器为实现其各种特性而使用的 IL。你已经了解了 lambda 表达式中的捕获变量如何导致生成额外的类，以及转换为表达式树的 lambda 表达式如何导致对 `Expression` 类中方法的调用。动态类型在创建源代码的数据表示方面有点像表达式树，但规模更大。

本节将比上一节更不详细。虽然细节很有趣，但你几乎肯定不需要了解它们。好消息是，这一切都是开源的，所以如果你对这个主题的简要介绍感兴趣，可以深入研究到任意程度。我们首先考虑哪个子系统负责动态类型的哪个方面。

**谁负责什么？**
通常，当你考虑 C# 特性时，很自然地将职责分为三个领域：
- C# 编译器
- CLR
- 框架库

有些特性纯粹是 C# 编译器的领域。隐式类型就是这样一个例子。框架不需要提供任何类型来支持 `var`，运行时也完全不知道你使用的是隐式类型还是显式类型。

另一个极端是泛型，它需要编译器、运行时以及反射 API 方面的框架提供大量支持。LINQ 介于两者之间：编译器提供了你在第 3 章中看到的各种特性，而框架不仅提供了 LINQ to Objects 的实现，还提供了表达式树的 API。另一方面，运行时不需要改变。对于动态类型，情况稍微复杂一些。图 4.1 给出了所涉及元素的图形表示。

![image-20260118150126663](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260118150126768.png)

CLR 不需要改变，尽管我相信从 v2 到 v4 有一些优化是由这项工作推动的。编译器显然参与了生成不同的 IL，我们稍后将看一个例子。对于框架/库支持，有两个方面。第一个是动态语言运行时（DLR），它提供了语言无关的基础设施，如 `DynamicMetaObject`。它负责执行所有动态行为。但第二个库不是核心框架本身的一部分：Microsoft.CSharp.dll。

**注意**：这个库随框架一起提供，但本身不是系统框架库的一部分。我发现将其视为第三方依赖很有帮助，而这个第三方恰好是微软。另一方面，Microsoft C# 编译器与它耦合得相当紧密。它并不完全符合任何一个盒子。

这个库负责任何 C# 特定的内容。例如，如果你进行一个方法调用，其中一个参数是动态值，那么就是这个库在运行时执行重载解析。它是 C# 编译器负责绑定的部分的副本，但它是在所有动态 API 的上下文中进行的。

如果你曾在项目中看到对 Microsoft.CSharp.dll 的引用并想知道它是干什么的，这就是原因。如果你在任何地方都不使用动态类型，你可以安全地删除这个引用。如果你使用动态类型但删除了引用，你会得到一个编译时错误，因为 C# 编译器会生成对该程序集的调用。说到 C# 编译器生成的代码，我们现在来看一些。

**为动态类型生成的 IL**
我们将回到最初的动态类型示例，但使其更短。以下是我向你展示的前两行动态代码：
```csharp
dynamic text = "hello world";
string world = text.Substring(6);
```
很简单，对吧？这里有两个动态操作：
- 对 `Substring` 方法的调用
- 从结果到 `string` 的转换

以下清单是从这两行生成的代码的反编译版本。为了清晰起见，我包含了类声明和 `Main` 方法的周围上下文。

清单 4.9 两个简单动态操作的反编译结果
```csharp
using Microsoft.CSharp.RuntimeBinder;
using System;
using System.Runtime.CompilerServices;

class DynamicTypingDecompiled
{
    // 调用站点的缓存
    private static class CallSites
    {
        public static CallSite<Func<CallSite, object, int, object>> method;
        public static CallSite<Func<CallSite, object, string>> conversion;
    }

    static void Main()
    {
        object text = "hello world";

        // 必要时为方法调用创建调用站点
        if (CallSites.method == null)
        {
            CSharpArgumentInfo[] argumentInfo = new[]
            {
                CSharpArgumentInfo.Create(
                    CSharpArgumentInfoFlags.None, null),
                CSharpArgumentInfo.Create(
                    CSharpArgumentInfoFlags.Constant |
                    CSharpArgumentInfoFlags.UseCompileTimeType,
                    null)
            };

            CallSiteBinder binder =
                Binder.InvokeMember(CSharpBinderFlags.None, "Substring",
                    null, typeof(DynamicTypingDecompiled), argumentInfo);
            CallSites.method =
                CallSite<Func<CallSite, object, int, object>>.Create(binder);
        }

        // 必要时为转换创建调用站点
        if (CallSites.conversion == null)
        {
            CallSiteBinder binder =
                Binder.Convert(CSharpBinderFlags.None, typeof(string),
                    typeof(DynamicTypingDecompiled));
            CallSites.conversion =
                CallSite<Func<CallSite, object, string>>.Create(binder);
        }

        // 调用方法调用站点
        object result = CallSites.method.Target(CallSites.method, text, 6);

        // 调用转换调用站点
        string str = CallSites.conversion.Target(CallSites.conversion, result);
    }
}
```

我为这个格式道歉。我已经尽力使其可读，但这段代码涉及很多很长的名称。好消息是，除了出于兴趣，你几乎肯定不需要看这样的代码。需要注意的一点是，`CallSite` 位于 `System.Runtime.CompilerServices` 命名空间中，因为它是语言中立的，而使用的 `Binder` 类来自 `Microsoft.CSharp.RuntimeBinder`。

正如你所看到的，涉及很多调用站点。生成的代码缓存了每个调用站点，DLR 内部也有多级缓存。绑定是一个相当复杂的过程。调用站点内的缓存通过存储每个绑定操作的结果来提高性能，以避免冗余工作，同时意识到如果某些上下文在调用之间发生变化，相同的调用可能最终得到不同的绑定结果。

所有这些努力的结果是一个非常高效的系统。它的性能不如静态类型代码，但出奇地接近。我预计在大多数情况下，如果动态类型由于其他原因是合适的选择，其性能不会成为限制因素。为了结束对动态类型的介绍，我将解释你可能遇到的一些限制，然后就在何时以及如何有效选择动态类型提供一些指导。





