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



## **动态类型的限制与意外情况**

将动态类型集成到从一开始就设计为静态类型的语言中是困难的。在少数地方，两者不能很好地协同工作并不奇怪。我整理了一些动态类型的方面，包括在运行时可能遇到的限制或潜在意外情况。这个列表并不详尽，但涵盖了最常见的问题。

**动态类型与泛型**
将动态类型与泛型结合使用可能会很有趣。编译时应用关于在哪里可以使用 `dynamic` 的规则：
- 一个类型不能指定它实现在类型参数中任何地方使用 `dynamic` 的接口。
- 你不能在类型约束中的任何地方使用 `dynamic`。
- 一个类可以指定一个在类型参数中使用 `dynamic` 的基类，甚至作为接口类型参数的一部分。
- 你可以将 `dynamic` 用作变量的接口类型参数。

以下是一些无效代码的示例：
```csharp
class DynamicSequence : IEnumerable<dynamic>
class DynamicListSequence : IEnumerable<List<dynamic>>
class DynamicConstraint1<T> : IEnumerable<T> where T : dynamic
class DynamicConstraint2<T> : IEnumerable<T> where T : List<dynamic>
```

但这些都是有效的：
```csharp
class DynamicList : List<dynamic>
class ListOfDynamicSequences : List<IEnumerable<dynamic>>
IEnumerable<dynamic> x = new List<dynamic> { 1, 0.5 }.Select(x => x * 2);
```

**扩展方法**
运行时绑定器不解析扩展方法。理论上可以这样做，但需要在每个方法调用站点保留每个相关 `using` 指令的额外信息。需要注意的是，这不会影响静态绑定的调用，即使这些调用恰好在类型参数中的某个地方使用了动态类型。因此，例如，以下清单编译和运行都没有问题。

清单 4.10 对动态值列表的 LINQ 查询
```csharp
List<dynamic> source = new List<dynamic>
{
    5,
    2.75,
    TimeSpan.FromSeconds(45)
};

IEnumerable<dynamic> query = source.Select(x => x * 2);
foreach (dynamic value in query)
{
    Console.WriteLine(value);
}
```

这里唯一的动态操作是乘法（`x * 2`）和 `Console.WriteLine` 中的重载解析。对 `Select` 的调用在编译时正常绑定。作为将失败的一个例子，让我们尝试使源本身动态化，并将你使用的 LINQ 操作简化为 `Any()`。（如果你像之前一样继续使用 `Select`，你会遇到另一个问题，稍后会看到。）以下清单显示了更改。

清单 4.11 尝试在动态目标上调用扩展方法
```csharp
dynamic source = new List<dynamic>
{
    5,
    2.75,
    TimeSpan.FromSeconds(45)
};
bool result = source.Any();
```

我没有包含输出部分，因为执行没有到达那里。相反，它会失败并抛出 `RuntimeBinderException`，因为 `List<T>` 不包含名为 `Any` 的方法。

如果你想调用扩展方法，就好像它的目标是动态值一样，你需要将其作为常规静态方法调用来进行。例如，你可以将清单 4.11 的最后一行重写为：
```csharp
bool result = Enumerable.Any(source);
```

调用仍然会在运行时绑定，但仅就重载解析而言。

**匿名函数**
匿名函数有三个限制。为了简单起见，我将使用 lambda 表达式展示它们。

首先，匿名方法不能赋值给 `dynamic` 类型的变量，因为编译器不知道要创建哪种委托。如果你进行强制转换或使用中间静态类型变量（然后复制值），这是可以的，你也可以动态调用委托。例如，这是无效的：
```csharp
dynamic function = x => x * 2;
Console.WriteLine(function(0.75));
```

但这是可以的，并打印 1.5：
```csharp
dynamic function = (Func<dynamic, dynamic>)(x => x * 2);
Console.WriteLine(function(0.75));
```

其次，出于相同的基本原因，lambda 表达式不能出现在动态绑定操作中。这就是我在清单 4.11 中没有使用 `Select` 来演示扩展方法问题的原因。否则，清单 4.11 将如下所示：
```csharp
dynamic source = new List<dynamic>
{
    5,
    2.75,
    TimeSpan.FromSeconds(45)
};
dynamic result = source.Select(x => x * 2);
```

你知道这在运行时不会工作，因为它找不到 `Select` 扩展方法，但由于使用了 lambda 表达式，它甚至无法编译。编译时问题的解决方法与之前相同：只需将 lambda 表达式强制转换为委托类型，或先将其赋值给静态类型变量。对于像 `Select` 这样的扩展方法，这在运行时仍然会失败，但如果你调用的是常规方法，例如 `List<T>.Find`，这是可以的。

最后，转换为表达式树的 lambda 表达式不得包含任何动态操作。考虑到 DLR 在内部使用表达式树的方式，这可能听起来有点奇怪，但在实践中很少成为问题。在表达式树有用的大多数情况下，动态类型的含义不清楚，或者可能无法实现。

作为一个例子，你可以尝试调整清单 4.10（使用静态类型的源变量）以使用 `IQueryable<T>`，如下面的清单所示。

清单 4.12 尝试在 IQueryable<T> 中使用动态元素类型
```csharp
List<dynamic> source = new List<dynamic>
{
    5,
    2.75,
    TimeSpan.FromSeconds(45)
};
IEnumerable<dynamic> query = source
    .AsQueryable()      // 这一行现在编译失败
    .Select(x => x * 2);
```

`AsQueryable()` 调用的结果是 `IQueryable<dynamic>`。这是静态类型的，但其 `Select` 方法接受表达式树而不是委托。这意味着 lambda 表达式 `(x => x * 2)` 必须转换为表达式树，但它执行的是动态操作，因此编译失败。

**匿名类型**
我在第一次介绍匿名类型时提到了这个问题，但值得重复：匿名类型在 IL 中由 C# 编译器生成为常规类。它们具有内部访问权限，因此在其声明的程序集外部无法使用它们。通常这不是问题，因为每个匿名类型通常仅在单个方法中使用。使用动态类型，你可以读取匿名类型实例的属性，但前提是该代码有权访问生成的类。以下清单展示了一个有效的示例。

清单 4.13 动态访问匿名类型的属性
```csharp
static void PrintName(dynamic obj)
{
    Console.WriteLine(obj.Name);
}

static void Main()
{
    var x = new { Name = "Abc" };
    var y = new { Name = "Def", Score = 10 };
    PrintName(x);
    PrintName(y);
}
```

这个清单中有两个匿名类型，但绑定过程并不关心它是否绑定到匿名类型。不过，它确实会检查它是否有权访问找到的属性。如果你将此代码拆分到两个程序集中，那将导致问题；绑定器会发现匿名类型在创建它的程序集内部，并抛出 `RuntimeBinderException`。如果你遇到这个问题，并且可以使用 `[InternalsVisibleTo]` 来允许执行动态绑定的程序集有权访问创建匿名类型的程序集，这是一个合理的解决方法。

**显式接口实现**
运行时绑定器使用任何动态值的运行时类型，然后以与你在变量编译时类型中写入该类型相同的方式绑定。不幸的是，这与现有的 C# 显式接口实现特性不能很好地配合。当你使用显式接口实现时，实际上意味着所实现的成员仅在你使用对象的接口视图而不是类型本身时才可用。

展示这一点比解释更容易。以下清单以 `List<T>` 为例。

清单 4.14 显式接口实现示例
```csharp
List<int> list1 = new List<int>();
Console.WriteLine(list1.IsFixedSize); // 编译时错误

IList list2 = list1;
Console.WriteLine(list2.IsFixedSize); // 成功；打印 False

dynamic list3 = list1;
Console.WriteLine(list3.IsFixedSize); // 运行时错误
```

`List<T>` 实现了 `IList` 接口。该接口有一个名为 `IsFixedSize` 的属性，但 `List<T>` 类显式实现了它。任何通过静态类型为 `List<T>` 的表达式访问它的尝试都将在编译时失败。你可以通过静态类型为 `IList` 的表达式访问它，并且它将始终返回 false。但是动态访问它呢？绑定器将始终使用动态值的具体类型，因此它找不到该属性，并抛出 `RuntimeBinderException`。这里的解决方法是将动态值转换回接口（通过强制转换或单独的变量），如果你知道要使用接口成员。

我相信任何经常使用动态类型的人都能够向你讲述一长串越来越晦涩的边界情况，但上述内容应该能让你不会太频繁地感到惊讶。我们将通过一些关于何时以及如何使用动态类型的指导来结束对动态类型的介绍。

>
> **探索路线**：
>
> 1. 探究动态类型与泛型类型系统的深层冲突 → 分析CLR对动态类型在泛型中的支持限制。
> 2. 研究扩展方法动态解析的可行性 → 探讨DLR能否通过引入编译上下文信息实现扩展方法解析。
> 3. 比较动态类型与`object`类型在反射场景下的性能差异 → 了解运行时绑定与编译时绑定的代价对比。
> 4. 探索现代C#中动态类型的改进方向 → 如C# 10对调用站点缓存的优化。





## **使用建议**

我在此直言不讳：我通常不是动态类型的拥趸。我不记得上次在生产代码中使用它是什么时候了，即使使用，我也会非常谨慎，并在进行大量正确性和性能测试后才使用。

我钟情于静态类型。根据我的经验，静态类型有四个显著好处：
1. 当我犯错时，我可能更早发现它们——在编译时而非运行时。这对于可能难以全面测试的代码路径尤为重要。
2. 编辑器可以提供代码补全。这在打字速度方面并不特别重要，但作为一种探索下一步可能操作的方式，尤其是在使用我不熟悉的类型时，非常有用。如今动态语言的编辑器可以提供出色的代码补全功能，但它们永远不会像静态类型语言那样精确，因为可用的信息量不够。
3. 它促使我思考所提供的 API，包括参数、返回类型等。在我决定了接受和返回哪些类型后，这些就构成了现成的文档：我只需为任何不明显的内容（如可接受值的范围）添加注释。
4. 通过在编译时而非运行时完成工作，静态类型代码通常比动态类型代码具有性能优势。我不想过分强调这一点，因为现代运行时可以做惊人的事情，但这确实值得考虑。

我相信动态类型的爱好者也能列出一系列动态类型的绝佳好处，但我不是合适的人选。我怀疑这些好处在从一开始就设计为动态类型的语言中更容易实现。C# 主要是一种静态类型语言，其传统是明确的，这就是为什么我之前列出的边界情况会存在。话虽如此，以下是关于何时可能想使用动态类型的几个建议。

**简化反射**
假设你发现自己使用反射来访问属性或方法；你在编译时知道名称，但由于某种原因无法引用静态类型。使用动态类型让运行时绑定器执行该访问，比直接使用反射 API 要简单得多。如果你原本需要执行多个反射步骤，这种好处会更大。例如，考虑如下代码片段：
```csharp
dynamic value = ...;
value.SomeProperty.SomeMethod();
```
涉及的反射步骤如下：
1. 基于初始值的类型获取 PropertyInfo。
2. 获取该属性的值并记住它。
3. 基于属性结果的类型获取 MethodInfo。
4. 在属性结果上执行方法。
当你添加了验证以检查属性和方法是否都存在时，你将看到多行代码。结果不会比前面展示的动态方法更安全，但可读性会差很多。

**无共同接口的公共成员**
有时你确实预先知道值的所有可能类型，并且希望在所有类型上使用同名成员。如果这些类型实现了共同接口或共享声明该成员的共同基类，那很好，但这并不总是发生。如果每个类型都独立声明了该成员（并且你无法更改），你将面临不愉快的选择。

这次，你不需要使用反射，但可能需要执行多次重复的步骤：检查类型、强制转换、访问成员。C# 7 的模式使这大大简化，但仍然可能重复。相反，你可以使用动态类型来有效地表达“相信我，我知道这个成员会存在，即使我无法以静态类型的方式表达它。” 我会在测试中放心地这样做（如果出错，代价只是测试失败），但在生产代码中我会更加谨慎。

**使用为动态类型构建的库**
.NET 生态系统相当丰富，并且一直在变得更好。开发者正在创建各种有趣的库，我怀疑有些库可能采用动态类型。例如，我可以想象一个库设计用于轻松原型化基于 REST 或 RPC 的 API，无需代码生成。这在开发的初始阶段可能有用，此时一切都在变化，然后再生成用于后续开发的静态类型库。

这类似于你之前看到的 Json.NET 示例。在数据模型定义良好后，你可能希望编写类来表示数据模型，但在原型设计时，更改 JSON 然后动态访问它的代码可能更简单。同样，稍后你将看到 COM 改进如何意味着你通常可以最终使用动态类型，而不是执行大量强制转换。

简而言之，我认为在简单的情况下使用静态类型仍然有意义，但你应该接受动态类型作为某些情况下潜在有用的工具。我鼓励你在每种情况下权衡利弊。例如，对于原型甚至测试代码可接受的代码，可能不适合生产代码。

除了出于专业目的编写的代码外，通过使用 `DynamicObject` 或 `IDynamicMetaObjectProvider` 来响应动态行为的能力，当然为有趣的开发提供了很大空间。尽管我自己可能回避动态类型，但它在 C# 中设计良好、实现出色，并提供了丰富的探索途径。

我们的下一个特性有些不同，尽管两者将在你查看 COM 互操作性时结合。我们将回到静态类型及其一个特定方面：为参数提供参数。



# **可选参数和命名实参**

可选参数和命名实参的使用范围有限：给定一个要调用的方法、构造函数、索引器或委托，你如何为调用提供实参？可选参数允许调用者完全省略某个实参，而命名实参允许调用者向编译器和任何人类读者清楚地说明实参对应的形参。

让我们从一个简单的例子开始，然后深入细节。在整个本节中，我将只考虑方法。相同的规则适用于所有其他可以有形参的成员。

## **带默认值的形参和带名称的实参**

以下清单显示了一个具有三个形参的简单方法，其中两个是可选的。对该方法的多次调用展示了不同的特性。

清单 4.15 调用具有可选参数的方法

![image-20260118152656223](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260118152656372.png)

```csharp
static void Method(int x, int y = 5, int z = 10)
{
    Console.WriteLine("x={0}; y={1}; z={2}", x, y, z);
}

// 调用示例
Method(1, 2, 3);        // x=1; y=2; z=3
Method(x: 1, y: 2, z: 3); // x=1; y=2; z=3
Method(z: 3, y: 2, x: 1); // x=1; y=2; z=3
Method(1, 2);           // x=1; y=2; z=10
Method(1, y: 2);        // x=1; y=2; z=10
Method(1, z: 3);        // x=1; y=5; z=3
Method(1);              // x=1; y=5; z=10
Method(x: 1);           // x=1; y=5; z=10
```

图4.2展示了相同的方法声明和一个方法调用，以使术语清晰。

语法很简单：
- 形参可以在其名称后使用等号指定默认值。任何有默认值的形参都是可选的；任何没有默认值的形参是必需的。带有 `ref` 或 `out` 修饰符的形参不允许有默认值。
- 实参可以在值之前用冒号指定名称。没有名称的实参称为位置实参。

形参的默认值必须是以下表达式之一：
- 编译时常量，例如数字或字符串字面量，或 `null` 字面量。
- 默认表达式，例如 `default(CancellationToken)`。正如你将在第14.5节中看到的，C# 7.1 引入了默认字面量，因此你可以写 `default` 而不是 `default(CancellationToken)`。
- 新的表达式，例如 `new Guid()` 或 `new CancellationToken()`。这仅适用于值类型。

所有可选形参必须出现在所有必需形参之后，参数数组（带有 `params` 修饰符的形参）除外。

**警告**：即使你可以声明一个带有可选形参后跟参数数组的方法，但调用时会令人困惑。我强烈建议你避免这种情况，并且我不会讨论如何解析对此类方法的调用。

使形参成为可选的目的是允许调用者在要提供的值与默认值相同时省略它。让我们看看编译器如何处理涉及默认参数和/或命名实参的方法调用。

**确定方法调用的含义**
如果你阅读规范，你会发现确定哪个实参对应哪个形参的过程是重载解析的一部分，并且与类型推断交织在一起。这比你预期的要复杂，因此我将在此简化。我们将专注于单个方法签名，假设它已经被重载解析选中，然后从这里开始。

规则列出起来相当简单：
- 所有位置实参必须出现在所有命名实参之前。此规则在 C# 7.2 中略有放宽，如第14.6节所示。
- 位置实参总是对应方法签名中相同位置的形参。第一个位置实参对应第一个形参，第二个位置实参对应第二个形参，依此类推。
- 命名实参通过名称而不是位置匹配：名为 `x` 的实参对应名为 `x` 的形参。命名实参可以按任意顺序指定。
- 任何形参只能有一个对应的实参。你不能在两个命名实参中指定相同的名称，也不能对已经有对应位置实参的形参使用命名实参。
- 每个必需形参必须有一个对应的实参来提供值。
- 可选形参允许没有对应的实参，在这种情况下，编译器将提供默认值作为实参。

为了查看这些规则的实际效果，让我们考虑我们原始的简单方法签名：
```csharp
static void Method(int x, int y = 5, int z = 10)
```

你可以看到 `x` 是必需形参，因为它没有默认值，但 `y` 和 `z` 是可选形参。表4.1显示了几个有效的调用及其结果。

表4.1 命名实参和可选参数的有效方法调用示例
| 调用                       | 结果实参         | 说明                                                         |
| -------------------------- | ---------------- | ------------------------------------------------------------ |
| `Method(1, 2, 3)`          | `x=1; y=2; z=3`  | 所有位置实参。C# 4 之前的常规调用。                          |
| `Method(1)`                | `x=1; y=5; z=10` | 编译器为 `y` 和 `z` 提供值，因为没有对应的实参。             |
| `Method()`                 | 无效             | 无效：没有实参对应 `x`。                                     |
| `Method(y: 2)`             | 无效             | 无效：没有实参对应 `x`。                                     |
| `Method(1, z: 3)`          | `x=1; y=5; z=3`  | 编译器为 `y` 提供值，因为没有对应的实参。通过使用命名实参 `z` 跳过了它。 |
| `Method(1, x: 2, z: 3)`    | 无效             | 无效：两个实参对应 `x`。                                     |
| `Method(1, y: 2, y: 2)`    | 无效             | 无效：两个实参对应 `y`。                                     |
| `Method(z: 3, y: 2, x: 1)` | `x=1; y=2; z=3`  | 命名实参可以按任意顺序排列，只要每个形参恰好对应一个实参。   |

在计算方法调用时，还有两个重要方面需要注意。首先，实参按照它们在方法调用的源代码中出现的顺序从左到右求值。在大多数情况下，这无关紧要，但如果实参求值有副作用，则可能很重要。例如，考虑对我们的示例方法的以下两个调用：

```csharp
int tmp1 = 0;
Method(x: tmp1++, y: tmp1++, z: tmp1++); // x=0; y=1; z=2

int tmp2 = 0;
Method(z: tmp2++, y: tmp2++, x: tmp2++); // x=2; y=1; z=0
```

这两个调用仅在命名实参的顺序上不同，但这影响了传递给方法的值。在这两种情况下，代码都比可能的更难阅读。当实参求值的副作用很重要时，我鼓励你将它们作为单独的语句求值，并赋值给新的局部变量，然后将这些局部变量作为实参直接传递给方法，如下所示：

```csharp
int tmp3 = 0;
int argX = tmp3++;
int argY = tmp3++;
int argZ = tmp3++;
Method(x: argX, y: argY, z: argZ);
```

此时，是否命名实参不会改变行为；你可以选择你认为最易读的形式。在我看来，将实参求值与方法调用分离使实参求值的顺序更易于理解。

需要注意的第二点是，如果编译器必须为形参指定任何默认值，这些值将嵌入到调用代码的 IL 中。编译器无法说“我没有这个形参的值；请使用你拥有的任何默认值。”这就是为什么默认值必须是编译时常量，也是可选参数影响版本控制的方式之一。



## **确定方法调用的含义**

如果你阅读规范，你会发现确定哪个实参对应哪个形参的过程是重载解析的一部分，并且与类型推断交织在一起。这比你预期的要复杂，因此我将在此简化。我们将专注于单个方法签名，假设它已经被重载解析选中，然后从这里开始。

规则列出起来相当简单：
- 所有位置实参必须出现在所有命名实参之前。此规则在 C# 7.2 中略有放宽，如第14.6节所示。
- 位置实参总是对应方法签名中相同位置的形参。第一个位置实参对应第一个形参，第二个位置实参对应第二个形参，依此类推。
- 命名实参通过名称而不是位置匹配：名为 `x` 的实参对应名为 `x` 的形参。命名实参可以按任意顺序指定。
- 任何形参只能有一个对应的实参。你不能在两个命名实参中指定相同的名称，也不能对已经有对应位置实参的形参使用命名实参。
- 每个必需形参必须有一个对应的实参来提供值。
- 可选形参允许没有对应的实参，在这种情况下，编译器将提供默认值作为实参。

为了查看这些规则的实际效果，让我们考虑我们原始的简单方法签名：
```csharp
static void Method(int x, int y = 5, int z = 10)
```

你可以看到 `x` 是必需形参，因为它没有默认值，但 `y` 和 `z` 是可选形参。表4.1显示了几个有效的调用及其结果。

表4.1 命名实参和可选参数的有效方法调用示例
| 调用                       | 结果实参         | 说明                                                         |
| -------------------------- | ---------------- | ------------------------------------------------------------ |
| `Method(1, 2, 3)`          | `x=1; y=2; z=3`  | 所有位置实参。C# 4 之前的常规调用。                          |
| `Method(1)`                | `x=1; y=5; z=10` | 编译器为 `y` 和 `z` 提供值，因为没有对应的实参。             |
| `Method()`                 | 无效             | 无效：没有实参对应 `x`。                                     |
| `Method(y: 2)`             | 无效             | 无效：没有实参对应 `x`。                                     |
| `Method(1, z: 3)`          | `x=1; y=5; z=3`  | 编译器为 `y` 提供值，因为没有对应的实参。通过使用命名实参 `z` 跳过了它。 |
| `Method(1, x: 2, z: 3)`    | 无效             | 无效：两个实参对应 `x`。                                     |
| `Method(1, y: 2, y: 2)`    | 无效             | 无效：两个实参对应 `y`。                                     |
| `Method(z: 3, y: 2, x: 1)` | `x=1; y=2; z=3`  | 命名实参可以按任意顺序排列，只要每个形参恰好对应一个实参。   |

在计算方法调用时，还有两个重要方面需要注意。首先，实参按照它们在方法调用的源代码中出现的顺序从左到右求值。在大多数情况下，这无关紧要，但如果实参求值有副作用，则可能很重要。例如，考虑对我们的示例方法的以下两个调用：

```csharp
int tmp1 = 0;
Method(x: tmp1++, y: tmp1++, z: tmp1++); // x=0; y=1; z=2

int tmp2 = 0;
Method(z: tmp2++, y: tmp2++, x: tmp2++); // x=2; y=1; z=0
```

这两个调用仅在命名实参的顺序上不同，但这影响了传递给方法的值。在这两种情况下，代码都比可能的更难阅读。当实参求值的副作用很重要时，我鼓励你将它们作为单独的语句求值，并赋值给新的局部变量，然后将这些局部变量作为实参直接传递给方法，如下所示：

```csharp
int tmp3 = 0;
int argX = tmp3++;
int argY = tmp3++;
int argZ = tmp3++;
Method(x: argX, y: argY, z: argZ);
```

此时，是否命名实参不会改变行为；你可以选择你认为最易读的形式。在我看来，将实参求值与方法调用分离使实参求值的顺序更易于理解。

需要注意的第二点是，如果编译器必须为形参指定任何默认值，这些值将嵌入到调用代码的 IL 中。编译器无法说“我没有这个形参的值；请使用你拥有的任何默认值。”这就是为什么默认值必须是编译时常量，也是可选参数影响版本控制的方式之一。

## **对版本控制的影响**

库中公共 API 的版本控制是一个难题。它真的很难，而且远不如我们喜欢假装的那样清晰。虽然语义版本控制规定任何破坏性更改都意味着你需要升级到新的主版本，但几乎任何更改都可能破坏某些依赖该库的代码，如果你愿意包含晦涩的情况。也就是说，可选参数和命名实参对版本控制尤其棘手。让我们看看各种因素。

**参数名称更改是破坏性的**
假设你有一个包含之前看过的方法的库，但它是公开的：
```csharp
public static Method(int x, int y = 5, int z = 10)
```

现在假设你想在新版本中将其更改为：
```csharp
public static Method(int a, int b = 5, int c = 10)
```

这是一个破坏性更改；任何在调用方法时使用命名实参的代码都将被破坏，因为他们之前指定的名称不再存在。检查你的参数名称时要像检查你的类型和成员名称一样仔细！

**默认值更改至少会令人惊讶**
正如我所指出的，默认值被编译到调用代码的 IL 中。当这发生在同一程序集内时，更改默认值不会导致问题。当它位于不同程序集中时，对默认值的更改只有在调用代码重新编译时才会可见。

这并不总是一个问题，如果你预料到可能想更改默认值，在方法文档中明确说明这一点并非完全不合理。但它肯定会让一些使用你代码的开发者感到惊讶，特别是如果涉及复杂的依赖链。避免这种情况的一种方法是使用专用的默认值，该值始终表示“让方法在运行时选择”。例如，如果你有一个通常具有 `int` 参数的方法，你可以改用 `Nullable<int>`，默认值为 `null` 表示“方法将选择”。以后你可以更改方法的实现以做出不同的选择，每个使用新版本的调用者都会获得新行为，无论他们是否重新编译。

**添加重载很麻烦**
如果你认为在单一版本场景中重载解析很棘手，那么当你尝试添加重载而不破坏任何人时，情况会变得更糟。所有原始方法签名都必须出现在新版本中，以避免破坏二进制兼容性，并且对原始方法的所有调用应该在新版本中解析为相同的调用，或至少等效的调用。参数是必需的还是可选的不是方法签名本身的一部分；通过将可选参数更改为必需参数，或反之亦然，你不会破坏二进制兼容性。但你可能会破坏源代码兼容性。如果你不小心，很容易通过添加一个带有更多可选参数的新方法来引入重载解析的歧义。

如果两个方法在重载解析中都适用（两者在调用方面都有意义）并且就所涉及的实参到形参的转换而言，没有一个比另一个更好，那么可以使用默认参数作为决胜局。没有对应实参的可选参数的方法比至少有一个没有对应实参的可选参数的方法“更好”。但有一个未填充参数的方法并不比有两个此类参数的方法更好。

如果可能的话，我强烈建议你在涉及可选参数时尽量避免向方法添加重载——并且，理想情况下，从一开始就记住这一点。对于可能有很多选项的方法，一个考虑的模式是创建一个代表所有这些选项的类，然后将其作为方法调用中的可选参数。然后，你可以通过向选项类添加属性来添加新选项，而完全不必更改方法签名。

尽管有这些注意事项，我仍然赞成在合理的情况下使用可选参数来简化常见情况的调用代码，并且我非常喜欢能够命名实参以澄清调用代码。当多个相同类型的参数可能相互混淆时，这一点尤其相关。例如，当我需要调用 Windows Forms 的 `MessageBox.Show` 方法时，我总是使用它们。我永远记不住消息框的标题和文本哪个在前。IntelliSense 可以在编写代码时帮助我，但在阅读代码时就不那么明显了，除非我使用命名实参：

```csharp
MessageBox.Show(text: "This is text", caption: "This is the title");
```

我们的下一个主题是许多读者可能不需要，而其他读者将每天使用。尽管 COM 在许多上下文中是一种遗留技术，但仍有大量代码在使用它。





# **COM互操作性改进**

在C# 4之前，如果你想要与COM组件进行互操作，VB显然是更好的选择。它一直是一种更宽松的语言（至少如果你这样要求的话），并且从一开始就具有命名参数和可选参数。C# 4使COM开发者的生活变得更加简单。也就是说，如果你不使用COM，跳过本节也不会错过任何重要内容。我在这里介绍的所有特性在COM之外都不相关。

**注意**：COM是微软于1993年引入的组件对象模型，作为Windows上跨语言的互操作形式。完整描述超出了本书的范围，但如果你需要了解，你很可能已经知道它。最常用的COM库可能是Microsoft Office的那些库。

让我们从一个超越语言的特性开始。它主要关乎部署，尽管也影响了操作的公开方式。

## **链接主互操作程序集**

当你针对COM类型进行编码时，你使用为该组件库生成的程序集。通常，你使用由组件发布者生成的主互操作程序集（PIA）。你可以使用类型库导入工具（tlbimp）为自己的COM库生成此程序集。

在C# 4之前，完整的PIA必须存在于最终运行代码的机器上，并且必须与你编译时使用的版本相同。这意味着要么随应用程序一起分发PIA，要么信任正确版本已经安装。

从C# 4和Visual Studio 2010开始，你可以选择链接PIA而不是引用它。在Visual Studio中，在引用的属性页中，这是“嵌入互操作类型”选项。当此选项设置为True时，PIA的相关部分直接嵌入到你的程序集中。只有应用程序中使用的部分被包含在内。当代码运行时，只要客户端机器具有应用程序所需的一切，是否存在用于编译的完全相同的组件版本并不重要。

![image-20260118155052282](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260118155052421.png)

图4.3显示了在代码运行方式上引用（旧方式）和链接（新方式）之间的区别。

除了部署更改之外，链接PIA还会影响COM类型中VARIANT类型的处理方式。当PIA被引用时，任何返回VARIANT值的操作都会在C#中使用`object`类型公开。然后你必须将其强制转换为适当的类型以使用其方法和属性。

当PIA被链接时，返回的是`dynamic`而不是`object`。正如你之前看到的，从`dynamic`类型的表达式到任何非指针类型存在隐式转换，然后在运行时进行检查。以下清单展示了打开Excel并填充范围中20个单元格的示例。

清单4.16 使用隐式动态转换在Excel中设置值范围
```csharp
var app = new Application { Visible = true };
app.Workbooks.Add();
Worksheet sheet = app.ActiveSheet;
Range start = sheet.Cells[1, 1];
Range end = sheet.Cells[1, 20];
sheet.Range[start, end].Value = Enumerable.Range(1, 20).ToArray();
```

清单4.16悄悄地使用了一些后面将要介绍的特性，但此刻请专注于对`sheet`、`start`和`end`的赋值。通常每个都需要强制转换，因为被赋值的值将是`object`类型。你不必为变量指定静态类型；如果你对变量类型使用`var`或`dynamic`，你将在更多操作中使用动态类型。我更喜欢在知道期望类型的地方指定静态类型，部分是为了进行隐式验证，部分是为了在后续代码中启用IntelliSense。

对于广泛使用VARIANT的COM库，这是动态类型最重要的好处之一。下一个COM特性也建立在C# 4的新特性之上，并将可选参数提升到了一个新的水平。





## **COM 中的可选参数**

一些 COM 方法有许多参数，而且通常都是 `ref` 参数。这意味着在 C# 4 之前，像在 Word 中保存文件这样的简单操作可能非常麻烦。

清单 4.17 C# 4 之前创建并保存 Word 文档
```csharp
object missing = Type.Missing; // 用于 ref 参数的占位变量

Application app = new Application { Visible = true }; // 启动 Word
Document doc = app.Documents.Add(                    // 创建并填充文档
    ref missing, ref missing, ref missing, ref missing);

Paragraph para = doc.Paragraphs.Add(ref missing);
para.Range.Text = "Awkward old code";

object fileName = "demo1.docx";
doc.SaveAs2(ref fileName, ref missing, ref missing, ref missing, ref missing, // 保存文档
            ref missing, ref missing, ref missing, ref missing, ref missing,
            ref missing, ref missing, ref missing, ref missing, ref missing,
            ref missing, ref missing, ref missing, ref missing, ref missing);

doc.Close(ref missing, ref missing, ref missing);     // 关闭文档
app.Application.Quit(ref missing, ref missing, ref missing); // 关闭 Word
```

仅仅为了创建和保存文档就需要大量代码，包括 20 处 `ref missing`。在你不关心的大量参数中，很难看到代码的有用部分。

C# 4 提供了协同工作的特性，使这一切变得简单得多：
- 命名实参可用于明确哪个实参应对应哪个形参（正如你已看到的）。
- 仅针对 COM 库，可以直接指定值作为 `ref` 参数的实参。编译器将在背后创建一个局部变量，并通过引用传递它。
- 仅针对 COM 库，`ref` 参数可以是可选的，然后在调用代码中省略。`Type.Missing` 被用作默认值。

利用所有这些特性，你可以将清单 4.17 转换为更短、更清晰的代码。

清单 4.18 使用 C# 4 创建并保存 Word 文档
```csharp
Application app = new Application { Visible = true };
Document doc = app.Documents.Add();          // 各处省略可选参数
Paragraph para = doc.Paragraphs.Add();
para.Range.Text = "Simple new code";
doc.SaveAs2(FileName: "demo2.docx");         // 使用命名实参以使意图清晰
doc.Close();
app.Application.Quit();
```

这在可读性上是一个巨大的转变。所有 20 处 `ref missing` 以及变量本身都消失了。实际上，传递给 `SaveAs2` 的实参对应于该方法的第一个形参。你可以使用位置实参代替命名实参，但指定名称可以增加清晰度。如果你想为后面的形参指定一个值，可以通过名称指定，而无需为中间的所有其他形参提供值。

`SaveAs2` 的该实参也演示了隐式 `ref` 特性。就我们的源代码而言，无需在 `demo2.docx` 的初始值中声明变量，然后通过引用传递它，而是可以直接传递该值。编译器会为你将其转换为 `ref` 参数。最后一个与 COM 相关的特性暴露了 VB 比 C# 稍丰富的另一个方面。

**命名索引器**
索引器自 C# 诞生以来就存在。它们主要用于集合：例如，通过索引从列表中检索元素，或通过键从字典中检索值。但 C# 索引器在源代码中从不命名。你只能编写该类型的默认索引器。你可以使用特性指定一个名称，该名称将被其他语言使用，但 C# 不允许你通过名称区分索引器。至少，在 C# 4 之前是这样。

其他语言允许你编写和使用带名称的索引器，因此你可以通过索引使用名称访问对象的不同方面，以明确你想要什么。C# 对于常规的 .NET 代码仍然不这样做，但专门为 COM 类型做了一个例外。一个例子将使这更清晰。

Word 中的 `Application` 类型公开了一个名为 `SynonymInfo` 的命名索引器。它的声明如下：
```
SynonymInfo SynonymInfo[string Word, ref object LanguageId = Type.Missing]
```

在 C# 4 之前，你可以像调用名为 `get_SynonymInfo` 的方法一样调用该索引器，但这有些笨拙。在 C# 4 中，你可以通过名称访问它，如下面的清单所示。

清单 4.19 访问命名索引器
```csharp
Application app = new Application { Visible = false };
object missing = Type.Missing;

SynonymInfo info = app.get_SynonymInfo("method", ref missing); // C# 4 之前访问同义词
Console.WriteLine("'method' has {0} meanings", info.MeaningCount);

info = app.SynonymInfo["index"]; // 使用命名索引器的更简单代码
Console.WriteLine("'index' has {0} meanings", info.MeaningCount);
```

清单 4.19 展示了可选参数如何在命名索引器以及常规方法调用中使用。C# 4 之前的代码必须声明一个变量并通过引用传递给名称笨拙的方法。使用 C# 4，你可以通过名称使用索引器，并且可以省略第二个形参的实参。

这是对 C# 4 中 COM 相关特性的简要介绍，但我希望其好处是显而易见的。尽管我不经常使用 COM，但如果将来需要，这里展示的更改将使我减少很多沮丧。好处的程度取决于你使用的 COM 库的结构。例如，如果它使用大量 `ref` 参数和 `VARIANT` 返回类型，那么差异将比具有少量参数和具体返回类型的库更显著。但仅仅链接 PIA 的选项就可使部署大大简化。

我们即将结束 C# 4 的介绍。最后一个特性可能有点难以理解，但也是一个你可能在不经意间就会使用的特性。





# **泛型变体**

泛型变体通过示例展示比描述更容易。它涉及基于类型参数在泛型类型之间安全转换，并特别注意数据流动的方向。

## **泛型变体的简单示例**

我们从一个熟悉的接口 `IEnumerable<T>` 开始，它表示类型 `T` 的元素序列。任何字符串序列也是对象序列，这是合理的，而变体允许这种转换：

```csharp
IEnumerable<string> strings = new List<string> { "a", "b", "c" };
IEnumerable<object> objects = strings;
```

这看起来非常自然，如果编译失败反而会让人惊讶，但这正是在 C# 4 之前会发生的情况。

**注意**：在这些示例中，我一致使用 `string` 和 `object`，因为它们是所有 C# 开发者都了解的类，且不绑定于任何特定上下文。其他具有相同基类/派生类关系的类也同样适用。

可能会有更多令人惊讶的地方；即使使用 C# 4，并非所有听起来应该可行的转换都能工作。例如，你可能会尝试将对序列的推理扩展到列表。任何字符串列表都是对象列表吗？你可能认为是，但事实并非如此：

```csharp
IList<string> strings = new List<string> { "a", "b", "c" };
IList<object> objects = strings; // 无效：无法从 IList<string> 转换为 IList<object>
```

`IEnumerable<T>` 和 `IList<T>` 有什么区别？为什么不允许这种转换？答案是这并不安全，因为 `IList<T>` 中的方法允许类型 `T` 的值作为输入和输出。使用 `IEnumerable<T>` 的每种方式最终都会返回 `T` 值作为输出，但 `IList<T>` 有像 `Add` 这样的方法，接受 `T` 值作为输入。这将使允许变体变得危险。如果你尝试稍微扩展我们无效的示例，可以看到这一点：

```csharp
IList<string> strings = new List<string> { "a", "b", "c" };
IList<object> objects = strings; // 假设允许
objects.Add(new object());        // 向 IList<object> 添加对象
string element = strings[3];      // 作为字符串检索它
```

除了第二行，其他每一行单独看都有意义。向 `IList<object>` 添加对象引用是没问题的，从 `IList<string>` 获取字符串引用也是没问题的。但如果你能将字符串列表视为对象列表，这两种能力就会冲突。使第二行无效的语言规则有效地保护了其余代码。

到目前为止，你已经看到值作为输出返回（`IEnumerable<T>`）和值同时作为输入和输出使用（`IList<T>`）。在一些 API 中，值始终仅用作输入。最简单的例子是 `Action<T>` 委托，在调用委托时传入类型 `T` 的值。变体在这里仍然适用，但方向相反。这可能一开始会令人困惑。

如果你有一个 `Action<object>` 委托，它可以接受任何对象引用。它肯定可以接受字符串引用，语言规则允许你从 `Action<object>` 转换为 `Action<string>`：

```csharp
Action<object> objectAction = obj => Console.WriteLine(obj);
Action<string> stringAction = objectAction;
stringAction("Print me");
```

有了这些例子，我可以定义一些术语：
- **协变**：当值仅作为输出返回时发生。
- **逆变**：当值仅作为输入接受时发生。
- **不变**：当值同时作为输入和输出使用时。

目前这些定义故意有点模糊。它们更多是关于一般概念，而不是关于 C#。在你查看了 C# 用于指定变体的语法后，我们可以进一步明确它们。

## **接口和委托声明中的变体语法**

关于 C# 中变体的第一件事是，它只能为接口和委托指定。例如，你不能使类或结构协变。其次，变体是为每个类型参数单独定义的。虽然你可能会 loosely 说“`IEnumerable<T>` 是协变的”，但更准确的说法是“`IEnumerable<T>` 在 `T` 上是协变的”。这导致了接口和委托声明的语法，其中每个类型参数都有单独的修饰符。以下是 `IEnumerable<T>`、`IList<T>` 接口和 `Action<T>` 委托的声明：

```csharp
public interface IEnumerable<out T>
public delegate void Action<in T>
public interface IList<T>
```

如你所见，修饰符 `in` 和 `out` 用于指定类型参数的变体：
- 带有 `out` 修饰符的类型参数是协变的。
- 带有 `in` 修饰符的类型参数是逆变的。
- 没有修饰符的类型参数是不变的。

编译器根据声明的其余部分检查你使用的修饰符是否合适。例如，以下委托声明是无效的，因为协变类型参数被用作输入：

```csharp
public delegate void InvalidCovariant<out T>(T input)
```

以下接口声明是无效的，因为逆变类型参数被用作输出：

```csharp
public interface IInvalidContravariant<in T>
{
    T GetValue();
}
```

任何单个类型参数只能有其中一个修饰符，但同一声明中的两个类型参数可以有不同的修饰符。例如，考虑 `Func<T, TResult>` 委托。它接受类型 `T` 的值并返回类型 `TResult` 的值。很自然地，`T` 应该是逆变的，`TResult` 应该是协变的。委托声明如下：

```csharp
public TResult Func<in T, out TResult>(T arg)
```

## **使用变体的限制**

重复之前的一点，变体只能在接口和委托中声明。这种变体不会被实现接口的类或结构继承；类和结构始终是不变的。例如，假设你创建这样一个类：

```csharp
public class SimpleEnumerable<T> : IEnumerable<T>
{
    // 实现
}
// 此处不允许 out 修饰符
```

你仍然不能从 `SimpleEnumerable<string>` 转换为 `SimpleEnumerable<object>`。但你可以使用 `IEnumerable<T>` 的协变，从 `SimpleEnumerable<string>` 转换为 `IEnumerable<object>`。

假设你正在处理具有一些协变或逆变类型参数的委托或接口。有哪些转换可用？你需要一些定义来解释规则：
- 涉及变体的转换称为**变体转换**。
- 变体转换是**引用转换**的一个例子。引用转换是不改变所涉及值（始终是引用）的转换；它只改变编译时类型。
- **恒等转换**是从一个类型到 CLR 视为相同类型的转换。从 C# 的角度来看，这可能是相同的类型（例如从 `string` 到 `string`），也可能是仅在 C# 语言层面不同的类型之间的转换，例如从 `object` 到 `dynamic`。

假设你想将 `IEnumerable<A>` 转换为 `IEnumerable<B>`，其中 `A` 和 `B` 是某些类型参数。如果存在从 `A` 到 `B` 的恒等或隐式引用转换，则转换有效。例如，以下转换是有效的：
- `IEnumerable<string>` 到 `IEnumerable<object>`：存在从类到其基类（或其基类的基类等）的隐式引用转换。
- `IEnumerable<string>` 到 `IEnumerable<IConvertible>`：存在从类到其实现的任何接口的隐式引用转换。
- `IEnumerable<IDisposable>` 到 `IEnumerable<object>`：存在从任何引用类型到 `object` 或 `dynamic` 的隐式引用转换。

以下转换无效：
- `IEnumerable<object>` 到 `IEnumerable<string>`：存在从 `object` 到 `string` 的显式引用转换，但没有隐式转换。
- `IEnumerable<string>` 到 `IEnumerable<Stream>`：`string` 和 `Stream` 类无关。
- `IEnumerable<int>` 到 `IEnumerable<IConvertible>`：存在从 `int` 到 `IConvertible` 的隐式转换，但它是装箱转换，而不是引用转换。
- `IEnumerable<int>` 到 `IEnumerable<long>`：存在从 `int` 到 `long` 的隐式转换，但它是数值转换，而不是引用转换。

如你所见，类型参数之间的转换必须是引用或恒等转换的要求，可能会以令人惊讶的方式影响值类型。

使用 `IEnumerable<T>` 的示例只涉及单个类型参数。当你有多个类型参数时怎么办？实际上，它们从转换的源到目标是成对检查的，确保每个转换对于所涉及的类型参数都是合适的。

更正式地说，考虑具有 n 个类型参数的泛型类型声明：`T<X1, ..., Xn>`。从 `T<A1, ..., An>` 到 `T<B1, ..., Bn>` 的转换是根据每个类型参数和类型参数对依次考虑的。对于每个 i 从 1 到 n：
- 如果 `Xi` 是协变的，则必须存在从 `Ai` 到 `Bi` 的恒等或隐式引用转换。
- 如果 `Xi` 是逆变的，则必须存在从 `Bi` 到 `Ai` 的恒等或隐式引用转换。
- 如果 `Xi` 是不变的，则必须存在从 `Ai` 到 `Bi` 的恒等转换。

用一个具体例子来说明，考虑 `Func<in T, out TResult>`。规则意味着：
- 存在从 `Func<object, int>` 到 `Func<string, int>` 的有效转换，因为：
  - 第一个类型参数是逆变的，并且存在从 `string` 到 `object` 的隐式引用转换。
  - 第二个类型参数是协变的，并且存在从 `int` 到 `int` 的恒等转换。
- 存在从 `Func<dynamic, string>` 到 `Func<object, IConvertible>` 的有效转换，因为：
  - 第一个类型参数是逆变的，并且存在从 `dynamic` 到 `object` 的恒等转换。
  - 第二个类型参数是协变的，并且存在从 `string` 到 `IConvertible` 的隐式引用转换。
- 不存在从 `Func<string, int>` 到 `Func<object, int>` 的转换，因为：
  - 第一个类型参数是逆变的，并且不存在从 `object` 到 `string` 的隐式引用转换。
  - 第二个类型参数无关紧要；由于第一个类型参数，转换已经无效。

如果这一切有点 overwhelming，不要担心；99% 的时间你甚至不会注意到你正在使用泛型变体。我提供这些细节是为了在你遇到编译时错误而不理解原因时提供帮助。让我们通过几个泛型变体有用的例子来结束本章。







## **泛型变型（Generic Variance）的实际应用**

在很多情况下，你可能在**毫无察觉**的情况下就已经使用了泛型变型，因为事情往往会“按你期望的方式正常工作”。实际上，你并不一定需要意识到自己正在使用泛型变型，但我会指出几个它非常有用的例子。

首先，来看一下 **LINQ 和 `IEnumerable<T>`**。假设你有一组字符串想要执行查询，但你希望最终得到的是一个 `List<object>`，而不是 `List<string>`。例如，你可能需要在之后向这个列表中添加其他类型的对象。

下面的代码清单展示了在**协变（covariance）出现之前**，实现这一目的最简单的方法是使用一次额外的 `Cast` 调用。

**代码清单 4.20**
*在没有变型的情况下，从字符串查询创建 `List<object>`*

```csharp
IEnumerable<string> strings = new[] { "a", "b", "cdefg", "hij" };
List<object> list = strings
    .Where(x => x.Length > 1)
    .Cast<object>()
    .ToList();
```

对我来说，这种写法感觉很别扭。为什么只是为了在一个**必然安全的类型转换**上，就要在查询管道中多加一步呢？

有了变型之后，你可以直接在 `ToList()` 调用中指定类型参数，以明确你想要的列表类型，如下面的代码清单所示。

**代码清单 4.21**
*通过使用变型，从字符串查询创建 `List<object>`*

```csharp
IEnumerable<string> strings = new[] { "a", "b", "cdefg", "hij" };
List<object> list = strings
    .Where(x => x.Length > 1)
    .ToList<object>();
```

这之所以可行，是因为 `Where` 调用的输出类型是 `IEnumerable<string>`，而你要求编译器将 `ToList()` 的输入视为 `IEnumerable<object>`。由于**泛型变型**的存在，这种转换是安全的。

------

我发现**逆变（contravariance）**在配合 `IComparer<T>` 使用时非常有用。`IComparer<T>` 是一个用于对某种类型进行排序比较的接口。

举个例子，假设你有一个基类 `Shape`，其中包含一个 `Area` 属性，然后派生出 `Circle` 和 `Rectangle` 类。你可以编写一个实现了 `IComparer<Shape>` 的 `AreaComparer`，这样就可以通过 `List<T>.Sort()` 方法对 `List<Shape>` 进行原地排序。

但如果你有的是 `List<Circle>` 或 `List<Rectangle>`，该如何排序呢？

在泛型变型出现之前，通常需要各种变通方案；而现在，这件事变得非常简单，如下所示。

**代码清单 4.22**
*使用 `IComparer<Shape>` 对 `List<Circle>` 进行排序*

```csharp
List<Circle> circles = new List<Circle>
{
    new Circle(5.3),
    new Circle(2),
    new Circle(10.5)
};
circles.Sort(new AreaComparer());
foreach (Circle circle in circles)
{
    Console.WriteLine(circle.Radius);
}
```

代码清单 4.22 中所使用的完整类型定义可以在可下载的示例代码中找到，它们都和你预期的一样简单。关键点在于：**你可以将 `AreaComparer` 转换为 `IComparer<Circle>`，以供 `Sort` 方法调用**。在 C# 4 之前，这是做不到的。

------

如果你自己声明泛型接口或委托，**总是值得考虑一下这些类型参数是否可以是协变或逆变的**。通常情况下，如果这种变型关系不能自然成立，我不会刻意去强求；但花一点时间思考它是值得的。

使用一个本可以支持变型类型参数、却因为开发者没有考虑到而无法使用的接口，往往会让人感到非常恼火。

------

# 总结（Summary）

- C# 4 支持**动态类型（dynamic typing）**，将绑定从编译期推迟到运行期。
- 动态类型可以通过 `IDynamicMetaObjectProvider` 接口和 `DynamicObject` 类来自定义行为。
- 动态类型的实现同时依赖于**编译器和框架**的支持，框架通过大量优化和缓存来保证合理的性能。
- C# 4 允许参数指定**默认值**。任何带有默认值的参数都是可选参数，调用方可以省略。
- C# 4 允许在调用时通过**参数名**来指定实参，这与可选参数结合使用时，可以只为部分参数提供值。
- C# 4 允许 **COM 主互操作程序集（PIA）** 以链接方式而非引用方式使用，从而简化部署模型。
- 链接的 PIA 通过动态类型暴露变型值，避免了大量的显式类型转换。
- COM 库中的可选参数机制被扩展，使 `ref` 参数也可以是可选的。
- COM 库中的 `ref` 参数可以按值传递。
- **泛型变型**允许基于“类型参数作为输入还是输出”的角色，对泛型接口和委托进行安全的类型转换。
