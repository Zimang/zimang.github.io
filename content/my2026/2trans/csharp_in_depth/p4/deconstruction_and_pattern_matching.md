---
weight: 8
title: "解构与模式匹配"
---

本章涵盖：
*   将元组解构为多个变量
*   解构非元组类型
*   在 C# 7 中应用模式匹配
*   使用 C# 7 引入的三种模式

在第 11 章中，您了解到元组允许您简单地组合数据，而无需创建新类型，并允许一个变量充当其他变量的包。当您使用元组时——例如，打印整数序列的最小值，然后打印最大值——您需要逐个从元组中提取值。

这当然可行，并且在许多情况下这就是您所需要的。但在很多情况下，您会希望将复合值分解为单独的变量。此操作称为**解构**。该复合值可以是元组，也可以是其他类型——例如 `KeyValuePair`。C# 7 提供了简单的语法，允许在单个语句中声明或初始化多个变量。

解构以无条件的方式发生，就像一系列赋值一样。模式匹配类似，但处于更动态的上下文中；输入值必须匹配模式才能执行其后的代码。C# 7 在几种上下文中引入了模式匹配和几种模式，未来版本可能会有更多。我们将从解构您刚刚创建的元组开始，构建在第 11 章的基础之上。

# **元组的解构**

C# 7 提供两种解构方式：一种用于元组，一种用于其他所有情况。它们遵循相同的语法并具有相同的通用特性，但抽象地讨论它们可能会令人困惑。我们将首先讨论元组，并指出任何特定于元组的细节。在 12.2 节中，您将看到相同的想法如何应用于其他类型。为了让您了解即将介绍的内容，下面的清单展示了解构的几个特性，每个特性您都将更详细地研究。

**清单 12.1 使用元组解构的概述**
```csharp
var tuple = (10, "text"); // 创建一个 (int, string) 类型的元组
var (a, b) = tuple; // 隐式地将元组解构到新变量 a, b
(int c, string d) = tuple; // 显式地将元组解构到新变量 c, d

int e;
string f;
(e, f) = tuple; // 将元组解构到现有变量 e, f

Console.WriteLine($"a: {a}; b: {b}");
Console.WriteLine($"c: {c}; d: {d}");
Console.WriteLine($"e: {e}; f: {f}"); // 证明解构有效
```

我猜，如果给您看这段代码并告诉您它能编译，您可能已经能猜到输出，即使您之前没有阅读过任何关于元组或解构的内容：
```
a: 10; b: text
c: 10; d: text
e: 10; f: text
```
您所做的就是以新的方式声明并初始化了六个变量 a, b, c, d, e, f，这种方式比以前的代码更简洁。这并不是要贬低该特性的实用性，但这次需要深入探讨的微妙之处相对较少。在所有情况下，操作都像将值从元组复制到变量中一样简单。它不会将变量与元组关联；稍后更改变量不会更改元组，反之亦然。

**元组声明与解构语法**
语言规范将解构视为与其他元组特性密切相关。解构语法被描述为一种元组表达式，即使您不是在解构元组（您将在 12.2 节看到）。您可能不需要太担心这个，但您应该意识到潜在的混淆原因。考虑这两个语句：
```csharp
(int c, string d) = tuple; // 解构，声明两个变量 c 和 d
(int c, string d) x = tuple; // 声明一个类型为 (int c, string d) 的单个变量 x
```
第一个使用解构来声明两个变量（c 和 d）；第二个是声明一个元组类型 `(int c, string d)` 的单个变量 x。我不认为这种相似性是设计错误，但就像表达式体成员看起来像 Lambda 表达式一样，需要一点时间来适应。

让我们首先更详细地查看示例的前两部分，您在其中用一条语句声明并初始化变量。

## **解构到新变量**

在单个语句中声明多个变量一直是可行的，但仅限于它们类型相同时。为了可读性，我通常坚持每个语句一个声明。但是，当您可以在单个语句中声明和初始化多个变量，并且初始值都具有相同的来源时，这就很简洁了。特别是，如果该来源是函数调用的结果，您可以避免仅仅为了避免多次调用而声明额外的变量。

可能最容易理解的语法是每个变量都显式类型的语法——与参数列表或元组类型相同的语法。为了阐明我前面关于额外变量的观点，下面的清单展示了一个作为方法调用结果的元组被解构为三个新变量。

**清单 12.2 调用方法并将结果解构为三个变量**
```csharp
static (int x, int y, string text) MethodReturningTuple() => (1, 2, "t");
static void Main()
{
    (int a, int b, string name) = MethodReturningTuple();
    Console.WriteLine($"a: {a}; b: {b}; name: {name}");
}
```
直到您考虑不使用解构的等效代码时，其好处才那么明显。以下是编译器将上述代码转换成的形式：
```csharp
static void Main()
{
    var tmp = MethodReturningTuple();
    int a = tmp.x;
    int b = tmp.y;
    string name = tmp.text;
    Console.WriteLine($"a: {a}; b: {b}; name: {name}");
}
```
尽管我欣赏原始代码的简洁性，但这三个声明语句对我来说并不太麻烦，但 `tmp` 变量确实让我有点困扰。正如其名称所示，它只是临时存在；其唯一目的是记住方法调用的结果，以便用于初始化您真正想要的三个变量：a、b 和 name。尽管您只在那一小段代码中需要 tmp，但它与其他变量具有相同的作用域，这让我感觉混乱。如果您想对某些变量使用隐式类型，但对其他变量使用显式类型，那也没问题，如图 12.1 所示。

（图注：`(int a, int b, var name) = MethodReturningTuple();` // 显式声明与隐式声明混合）

当您希望为原始元组中的某些元素使用隐式转换指定与元素类型不同的类型时，这尤其有用；参见图 12.2。

（图注：`(long a, var b, XNamespace name) = MethodReturningTuple();` // 涉及隐式转换的解构）

如果您乐意对所有变量使用隐式类型，C# 7 提供了简写形式使其变得简单；只需在名称列表前使用 `var`：
```csharp
var (a, b, name) = MethodReturningTuple();
```
这相当于在参数列表内为每个变量使用 `var`，而这又相当于基于被赋值值的类型显式指定推断出的类型。就像常规的隐式类型变量声明一样，使用 `var` 不会使您的代码变成动态类型；它只是让编译器推断类型。

虽然您可以在括号内的类型指定上混合使用隐式和显式类型，但不能在变量列表前使用 `var`，然后为某些变量提供类型：
```csharp
var (a, long b, name) = MethodReturningTuple(); // 无效：“内部和外部”声明混合
```

**特殊标识符：`_` 弃元**
C# 7 有三个特性允许在引入局部变量的新位置使用：
*   解构（本节和 12.2）
*   模式（12.3 至 12.7 节）
*   out 变量（14.2 节）

在所有情况下，指定一个变量名为 `_`（单个下划线）具有特殊含义。它是一个**弃元**，意思是“我不关心这个结果。我甚至根本不希望它是一个变量——直接丢弃它。”当使用弃元时，它不会在作用域中引入新变量。您可以使用多个弃元，而不是为多个不关心的变量指定不同的变量名。

以下是元组解构中使用弃元的示例：
```csharp
var tuple = (1, 2, 3, 4); // 具有四个元素的元组
var (x, y, _, _) = tuple; // 解构元组但只保留前两个元素
Console.WriteLine(_); // 错误 CS0103：当前上下文中不存在名称 '_'
```
如果您的当前作用域中已有一个名为 `_` 的变量（通过常规变量声明声明），您仍然可以在解构到一组新的其他变量时使用弃元，并且现有变量将保持不变。

正如您在原始概述中看到的，您不必声明新变量即可使用解构。解构可以充当一系列赋值。

## **对现有变量和属性的解构赋值**

上一节解释了我们原始概述示例的大部分内容。在本节中，我们将查看代码的以下部分：

```csharp
var tuple = (10, "text");
int e;
string f;
(e, f) = tuple;
```
在这种情况下，编译器不是将解构视为一系列带有相应初始化表达式的声明；相反，它只是一系列赋值。这在避免临时变量方面具有与您在上一节中看到的相同的好处。以下清单给出了一个示例，使用了您之前使用的相同 `MethodReturningTuple()`。

**清单 12.3 使用解构为现有变量赋值**
```csharp
static (int x, int y, string text) MethodReturningTuple() => (1, 2, "t");
static void Main()
{
    int a = 20;
    int b = 30;
    string name = "before"; // 声明、初始化并使用三个变量
    Console.WriteLine($"a: {a}; b: {b}; name: {name}");
    (a, b, name) = MethodReturningTuple(); // 使用解构为所有三个变量赋值
    Console.WriteLine($"a: {a}; b: {b}; name: {name}"); // 显示新值
}
```
到目前为止，一切顺利，但该特性并不止于能够为局部变量赋值。任何作为单独语句有效的赋值也可以使用解构来完成。这可以是对字段、属性或索引器的赋值，包括对数组和其他对象的操作。

**声明或赋值：不能混合**
解构允许您要么声明和初始化变量，要么执行一系列赋值。您不能混合使用两者。例如，这是无效的：
```csharp
int x;
(x, int y) = (1, 2);
```
但是，赋值可以使用各种目标：一些现有的局部变量、一些字段、一些属性等等。

除了常规赋值，您还可以赋值给一个弃元（`_` 标识符），从而有效地丢弃该值（如果作用域中没有名为 `_` 的变量）。如果您的作用域中确实有一个名为 `_` 的变量，解构会像正常情况一样为其赋值。

**在解构中使用 `_`：赋值还是弃元？**
这乍一看有点令人困惑：有时当存在一个同名现有变量时，解构到 `_` 会改变其值，有时则会丢弃它。您可以通过两种方式避免这种混淆。第一种是查看解构的其余部分，看看它是在引入新变量（此时 `_` 是弃元）还是为现有变量赋值（此时 `_` 会像其他变量一样被赋予新值）。
第二种避免混淆的方法是不要将 `_` 用作局部变量名。

实际上，我预计几乎所有赋值解构的目标都是局部变量或 `this` 的字段和属性。事实上，您可以在构造函数中使用一个巧妙的小技巧，使 C# 7 中引入的表达式体构造函数更加有用。许多构造函数根据构造函数参数为属性或字段赋值。如果您首先将参数收集到一个元组字面量中，则可以在一个表达式中执行所有这些赋值，如下一个清单所示。

**清单 12.4 使用解构和元组字面量的简单构造函数赋值**
```csharp
public sealed class Point
{
    public double X { get; }
    public double Y { get; }
    public Point(double x, double y) => (X, Y) = (x, y);
}
```
我真的很喜欢这种简洁性。我喜欢从构造函数参数到属性的映射的清晰性。C# 编译器甚至将其识别为一种模式，并避免构造 `ValueTuple<double, double>`。不幸的是，它仍然需要依赖 `System.ValueTuple.dll` 来构建，这足以让我放弃使用它，除非我还在项目的其他地方使用了元组，或者目标框架已经包含了 `System.ValueTuple`。

**这是地道的 C# 吗？**
正如我所描述的，这个技巧有利有弊。它是构造函数的纯粹实现细节；它甚至不影响类体的其余部分。如果您决定采用这种风格，然后又决定不喜欢它，删除它应该是轻而易举的。现在说这是否会流行起来还为时过早，但我希望如此。不过，一旦元组字面量需要不仅仅是确切的参数值，我就会谨慎起来。即使只添加一个前提条件，根据我的主观意见，天平也会倾向于支持常规的赋值序列。

与声明解构相比，赋值解构在顺序方面有一个额外的复杂性。使用赋值的解构有三个不同的阶段：
1.  评估赋值的目标
2.  评估赋值运算符的右侧
3.  执行赋值

这三个阶段严格按照这个顺序执行。在每个阶段内，评估按通常的从左到右的源代码顺序进行。这种情况很少会造成差异，但这是可能的。

提示：如果您必须担心本节内容才能理解您面前的代码，那么这是一个强烈的代码异味。当您理解后，我敦促您重构它。解构具有在表达式内使用副作用的所有相同注意事项，但因为您在每个阶段要执行多次评估，所以问题被放大了。

我不打算在这个话题上停留太久；一个简单的例子足以说明您可能会看到的问题。然而，这绝不是您可能遇到的最糟糕的例子。您可能会做各种各样的事情来使这更加复杂。以下清单将一个 `(StringBuilder, int)` 元组解构到一个现有的 `StringBuilder` 变量和与该变量关联的 `Length` 属性。

**清单 12.5 评估顺序重要的解构**
```csharp
StringBuilder builder = new StringBuilder("12345");
StringBuilder original = builder; // 保留对原始 builder 的引用用于诊断

(builder, builder.Length) = (new StringBuilder("67890"), 3); // 执行解构赋值

Console.WriteLine(original); // 显示旧的和新的构建器的内容
Console.WriteLine(builder);
```
中间一行是这里棘手的地方。需要考虑的关键问题是哪个 `StringBuilder` 的 `Length` 属性被设置：是 `builder` 最初引用的那个，还是在解构的第一部分中分配的新值？正如我之前描述的，所有赋值目标都会首先被评估，然后才执行任何赋值。下面的清单通过以慢动作手动执行相同代码的分解版本来演示这一点。

**清单 12.6 显示评估顺序的慢动作解构**
```csharp
StringBuilder builder = new StringBuilder("12345");
StringBuilder original = builder;
StringBuilder targetForLength = builder; // 评估赋值目标

(StringBuilder, int) tuple = (new StringBuilder("67890"), 3); // 评估元组字面量

builder = tuple.Item1; // 对目标执行赋值
targetForLength.Length = tuple.Item2;

Console.WriteLine(original);
Console.WriteLine(builder);
```
当目标只是局部变量时，不需要额外的评估；您可以直接赋值给它。但是对变量的属性赋值需要在第一阶段评估该变量值；这就是为什么您有 `targetForLength` 变量。在从字面量构造出元组后，您可以将不同的项分配给您的目标，确保在分配 `Length` 属性时使用 `targetForLength` 而不是 `builder`。`Length` 属性设置在内容为 `12345` 的原始 `StringBuilder` 上，而不是内容为 `67890` 的新 `StringBuilder` 上。这意味着清单 12.5 和 12.6 的输出如下：
```
123
67890
```
了解了这一点后，在继续讨论非元组解构之前，还有最后一个——而且更令人愉快的——元组构造的细节需要讨论。

## **元组字面量解构的细节**

正如我在 11.3.1 节中描述的，并非所有元组字面量都有类型。例如，元组字面量 `(null, x => x * 2)` 没有类型，因为它的两个元素表达式都没有类型。但您知道它可以转换为类型 `(string, Func<int, int>)`，因为每个表达式都有到对应类型的转换。

好消息是元组解构也具有完全相同的“逐元素赋值兼容性”。这适用于声明解构和赋值解构。这里有一个简短的例子：
```csharp
(string text, Func<int, int> func) = (null, x => x * 2); // 声明 text 和 func 的解构
(text, func) = ("text", x => x * 3); // 赋值给 text 和 func 的解构
```
这也适用于需要从表达式到目标类型进行隐式转换的解构。例如，使用我们最爱的“在 `byte` 范围内的 int 常量”例子，以下代码是有效的：
```csharp
(byte x, byte y) = (5, 10);
```
像许多优秀的语言特性一样，这可能是您可能已经隐含期望的东西，但语言需要经过精心设计和规范才能允许它。现在您已经相当广泛地了解了元组解构，非元组类型的解构就相对简单了。

# **非元组类型的解构**

非元组类型的解构使用基于模式的方法，其方式与 `async/await` 和 `foreach` 类似。就像任何具有合适的 `GetAwaiter` 方法或扩展方法的类型都可以被等待一样，任何具有合适的 `Deconstruct` 方法或扩展方法的类型都可以使用与元组相同的语法进行解构。让我们从使用常规实例方法的解构开始。

## **实例解构方法**

使用现在多个示例中使用的 `Point` 类来演示解构是最简单的。您可以像这样向其中添加一个 `Deconstruct` 方法：

```csharp
public void Deconstruct(out double x, out double y)
{
    x = X;
    y = Y;
}
```
然后，您可以将任何 `Point` 解构为两个 `double` 变量，如下面的清单所示。

**清单 12.7 将 Point 解构为两个变量**
```csharp
var point = new Point(1.5, 20); // 构造一个 Point 实例
var (x, y) = point; // 将其解构为两个 double 类型的变量
Console.WriteLine($"x = {x}"); // 显示两个变量值
Console.WriteLine($"y = {y}");
```
`Deconstruct` 方法的任务是用解构的结果填充 `out` 参数。在这种情况下，您只是解构为两个 `double` 值。顾名思义，它就像反向的构造函数。

但是等等；您在构造函数中使用了一个巧妙的技巧，用一个语句将参数值赋给属性。您可以在这里也这样做吗？是的，您可以，而且我个人非常喜欢它。这里是构造函数和 `Deconstruct` 方法，以便您能看到它们的相似之处：
```csharp
public Point(double x, double y) => (X, Y) = (x, y);
public void Deconstruct(out double x, out double y) => (x, y) = (X, Y);
```
这种简洁性非常优美，至少在您习惯了之后。

用于解构的 `Deconstruct` 实例方法的规则非常简单：
*   该方法必须对进行解构的代码可访问。（例如，如果所有内容都在同一个程序集中，`Deconstruct` 可以是 `internal` 方法。）
*   它必须是一个 `void` 方法。
*   必须至少有两个参数。（不能解构为单个值。）
*   它必须是非泛型的。

您可能想知道为什么设计使用 `out` 参数，而不是要求 `Deconstruct` 是无参数的但具有元组返回类型。答案是，能够解构到多组值是有用的，这在有多个方法时是可行的，但您不能仅基于返回类型重载方法。为了更清楚地说明这一点，我将使用一个解构 `DateTime` 的例子，但当然，您不能将自己的实例方法添加到 `DateTime` 中。是时候引入扩展解构方法了。

## **扩展解构方法与重载**

正如我在引言中简要提到的，编译器会查找遵循相关模式的任何 `Deconstruct` 方法，包括扩展方法。您可能可以想象扩展解构方法的样子，但下面的清单给出了一个具体示例，使用 `DateTime`。

**清单 12.8 使用扩展方法解构 DateTime**
```csharp
static void Deconstruct(
    this DateTime dateTime,
    out int year, out int month, out int day) =>
    (year, month, day) =
    (dateTime.Year, dateTime.Month, dateTime.Day); // 解构 DateTime 的扩展方法

static void Main()
{
    DateTime now = DateTime.UtcNow;
    var (year, month, day) = now; // 将当前日期解构为年/月/日
    Console.WriteLine($"{year:0000}-{month:00}-{day:00}"); // 使用三个变量显示日期
}
```
实际上，这是一个声明在同一个（静态）类中的私有扩展方法，但通常它会是 `public` 或 `internal` 的，就像大多数扩展方法一样。

如果您想将 `DateTime` 解构为不仅仅是日期呢？这就是重载有用的地方。您可以有两个参数列表不同的方法，编译器将根据参数数量决定使用哪一个。让我们添加另一个扩展方法，以便也根据时间以及日期来解构 `DateTime`，然后使用我们的两种方法来解构不同的值。

**清单 12.9 使用 Deconstruct 重载**
```csharp
static void Deconstruct(
    this DateTime dateTime,
    out int year, out int month, out int day) =>
    (year, month, day) =
    (dateTime.Year, dateTime.Month, dateTime.Day); // 将日期解构为年/月/日

static void Deconstruct(
    this DateTime dateTime,
    out int year, out int month, out int day,
    out int hour, out int minute, out int second) =>
    (year, month, day, hour, minute, second) =
    (dateTime.Year, dateTime.Month, dateTime.Day,
     dateTime.Hour, dateTime.Minute, dateTime.Second); // 将日期解构为年/月/日/时/分/秒

static void Main()
{
    DateTime birthday = new DateTime(1976, 6, 19);
    DateTime now = DateTime.UtcNow;
    var (year, month, day, hour, minute, second) = now; // 使用六值解构器
    (year, month, day) = birthday; // 使用三值解构器
}
```
您可以对已经具有实例 `Deconstruct` 方法的类型使用扩展 `Deconstruct` 方法，并且在解构时，如果实例方法不适用，它们将被使用，就像普通方法调用一样。

扩展 `Deconstruct` 方法的限制自然遵循实例方法的限制：
*   它对调用代码必须是可访问的。
*   除了第一个参数（扩展方法的目标）外，所有参数都必须是 `out` 参数。
*   必须至少有两个这样的 `out` 参数。
*   该方法可以是泛型的，但只有调用的接收者（第一个参数）可以参与类型推断。

关于方法何时可以以及何时不能是泛型的规则值得仔细研究，特别是因为它们也阐明了为什么在重载 `Deconstruct` 时需要不同数量的参数。关键点在于编译器如何处理 `Deconstruct` 方法。

## **编译器对 Deconstruct 调用的处理**

当一切按预期工作时，您可以不用太考虑编译器如何决定使用哪个 `Deconstruct` 方法。但是，如果您遇到问题，试着设身处地为编译器着想可能会很有用。

您已经看到的元组解构的时机仍然适用于使用方法进行解构，因此我将重点关注方法调用本身。让我们举一个稍微具体的例子，计算编译器在遇到像这样的解构时会做什么：
```csharp
(int x, string y) = target;
```
我说这是一个稍微具体的例子，因为我没有说明 `target` 的类型是什么。这是故意的，因为您只需要知道它不是元组类型。

编译器将此扩展为类似这样的内容：
```csharp
target.Deconstruct(out var tmpX, out var tmpY);
int x = tmpX;
string y = tmpY;
```
然后，它使用所有常规的方法调用规则来尝试找到要调用的正确方法。我意识到使用 `out var` 是您之前没有见过的内容。您将在 14.2 节中更仔细地查看它，但现在您只需要知道它是在使用 `out` 参数的类型来推断类型，从而声明一个隐式类型的变量。

需要注意的重要一点是，您在原始代码中声明的变量类型不作为 `Deconstruct` 调用的一部分使用。这意味着它们无法参与类型推断。这解释了三件事：
*   实例 `Deconstruct` 方法不能是泛型的，因为没有信息供类型推断使用。
*   扩展 `Deconstruct` 方法可以是泛型的，因为编译器可以使用 `target` 推断类型参数，但这是在类型推断方面唯一有用的参数。
*   当重载 `Deconstruct` 方法时，重要的是 `out` 参数的数量，而不是它们的类型。如果您引入多个具有相同数量 `out` 参数的 `Deconstruct` 方法，那只会阻止编译器使用其中任何一个，因为调用代码将无法分辨您的意图。

我就说到这里，因为我不想过多讨论这个。如果您遇到无法理解的问题，请尝试执行前面显示的转换，这很可能会使事情更清晰。

这就是您需要了解的关于解构的一切。本章的其余部分侧重于模式匹配，这是一个理论上与解构完全分离的特性，但在使用现有数据的新方式方面，它有着类似的感觉。



> 本章介绍的**解构**和**模式匹配**是C# 7强化数据导向编程的两大支柱。
>
> **解构**（无论是元组还是通过`Deconstruct`方法）的核心思想是**对称于构造的逆向操作**，它将复合值“拆包”到多个变量中。这极大地简化了多返回值方法的调用、变量组的批量赋值（如在构造函数中的优雅应用）以及临时数据的处理。其设计巧妙利用了模式匹配的基础设施（如弃元`_`）和类型转换规则。
>
> **模式匹配**则是更为强大的概念，它允许在`is`表达式和`switch`语句中根据值的**形状**（而不仅仅是类型）进行条件分支。C# 7引入了三种基本模式：**常量模式**（如`case 10:`）、**类型模式**（`case int i:`，可提取变量）和**`var`模式**（`case var tmp:`，总是匹配）。这改变了编写条件逻辑的方式，使代码更专注于数据的状态，而非冗长的类型检查和属性访问。
>
> 这两者共同推动C#向更声明式、更简洁的数据处理风格演进。解构提供了便捷的数据提取方式，而模式匹配则提供了基于数据形状进行逻辑分支的优雅语法。它们与元组（第11章）结合使用，能显著减少样板代码，提升在数据处理、状态判断等场景下的代码表现力和安全性。





# **模式匹配简介**

像许多其他特性一样，模式匹配对 C# 来说是新的，但在编程语言中并不新鲜。特别是函数式语言经常大量使用模式。C# 7.0 中的模式满足了许多相同的用例，但以符合语言其余语法的方式实现。

模式的基本思想是测试值的某个方面，并使用该测试的结果来执行另一个操作。是的，这听起来就像 `if` 语句，但模式通常用于为条件提供更多上下文，或者基于模式在操作本身中提供更多上下文。同样，这个特性不允许您做任何以前做不到的事情；它只是让您更清晰地表达相同的意图。

我不想不举例就说得太远。如果现在看起来有点奇怪，请不要担心；目的是让您感受一下。假设您有一个定义了抽象 `Area` 属性的抽象类 `Shape`，以及派生类 `Rectangle`、`Circle` 和 `Triangle`。不幸的是，对于您当前的应用程序，您不需要形状的面积；您需要它的周长。您可能无法修改 `Shape` 来添加 `Perimeter` 属性（您可能根本无法控制其源代码），但您知道如何为所有感兴趣的类计算它。在 C# 7 之前，`Perimeter` 方法可能类似于以下清单。

**清单 12.10 不使用模式匹配计算周长**
```csharp
static double Perimeter(Shape shape)
{
    if (shape == null)
        throw new ArgumentNullException(nameof(shape));
    Rectangle rect = shape as Rectangle;
    if (rect != null)
        return 2 * (rect.Height + rect.Width);
    Circle circle = shape as Circle;
    if (circle != null)
        return 2 * PI * circle.Radius;
    Triangle triangle = shape as Triangle;
    if (triangle != null)
        return triangle.SideA + triangle.SideB + triangle.SideC;
    throw new ArgumentException(
        $"Shape type {shape.GetType()} perimeter unknown", nameof(shape));
}
```
注意：如果缺少大括号冒犯了您，我表示歉意。我通常为所有循环、`if` 语句等使用大括号，但在这种情况下，它们最终会使此处和其他一些后来的模式示例中的有用代码相形见绌。为了简洁，我删除了它们。

这很丑。它重复且冗长；“检查形状是否为特定类型，然后使用该类型的属性”这种模式出现了三次。重要的是，即使这里有多个 `if` 语句，每个语句的主体都会返回一个值，因此您总是只选择其中一个来执行。下面的清单展示了如何使用 C# 7 中的模式在 `switch` 语句中编写相同的代码。

**清单 12.11 使用模式匹配计算周长**
```csharp
static double Perimeter(Shape shape)
{
    switch (shape)
    {
        case null: // 处理空值
            throw new ArgumentNullException(nameof(shape));
        case Rectangle rect: // 处理您知道的每种类型
            return 2 * (rect.Height + rect.Width);
        case Circle circle:
            return 2 * PI * circle.Radius;
        case Triangle tri:
            return tri.SideA + tri.SideB + tri.SideC;
        default: // 如果您不知道如何处理，则抛出异常。
            throw new ArgumentException(...);
    }
}
```
这与之前版本 C# 中的 `switch` 语句有很大的不同，之前版本中 `case` 标签都只是常量值。在这里，您有时只关心值匹配（对于 `null` 情况），有时关心值的类型（矩形、圆形和三角形情况）。当您按类型匹配时，该匹配还会引入一个该类型的新变量，您使用它来计算周长。

C# 中的模式主题有两个不同的方面：
*   模式的语法
*   可以使用模式的上下文

起初，可能感觉一切都是新的，区分这两个方面似乎毫无意义。但 C# 7.0 中的模式只是一个开始：C# 设计团队已经明确表示，语法设计为随时间推移提供新的模式。当您知道语言中允许模式的这些位置时，您可以轻松掌握新模式。这有点像先有鸡还是先有蛋——很难在不展示另一部分的情况下演示一部分——但我们将从查看 C# 7.0 中可用的模式类型开始。

# **C# 7.0 中可用的模式**

C# 7.0 引入了三种模式：常量模式、类型模式和 `var` 模式。我将使用 `is` 运算符演示每一种模式，`is` 运算符是使用模式的上下文之一。

每个模式都尝试匹配一个**输入**。这可以是任何非指针表达式。为简单起见，我将在模式描述中将其称为 `input`，就好像它是一个变量一样，但它不必是。

## **常量模式**

常量模式顾名思义：模式完全由编译时常量表达式组成，然后检查其是否与 `input` 相等。如果 `input` 和常量都是整数表达式，则使用 `==` 进行比较。否则，调用静态 `object.Equals` 方法。重要的是调用的是静态方法，因为这使您可以安全地检查 `null` 值。以下清单显示了一个示例，其实际用途甚至比书中大多数其他示例更少，但它确实展示了一些有趣的点。

**清单 12.12 简单的常量匹配**
```csharp
static void Match(object input)
{
    if (input is "hello")
        Console.WriteLine("Input is string hello");
    else if (input is 5L)
        Console.WriteLine("Input is long 5");
    else if (input is 10)
        Console.WriteLine("Input is int 10");
    else
        Console.WriteLine("Input didn't match hello, long 5 or int 10");
}
static void Main()
{
    Match("hello");
    Match(5L);
    Match(7);
    Match(10);
    Match(10L);
}
```
输出大部分是直接的，但您可能会对倒数第二行感到惊讶：
```
Input is string hello
Input is long 5
Input didn't match hello, long 5 or int 10
Input is int 10
Input didn't match hello, long 5 or int 10
```
如果整数使用 `==` 进行比较，为什么最后一个 `Match(10L)` 调用没有匹配？答案是 `input` 的编译时类型不是整数类型，它只是 `object`，因此编译器生成的代码等效于调用 `object.Equals(x, 10)`。当 `x` 的值是装箱的 `Int64` 而不是装箱的 `Int32` 时，如我们对 `Match` 的最后一次调用中那样，它会返回 `false`。对于使用 `==` 的示例，您需要类似这样的东西：
```csharp
long x = 10L;
if (x is 10)
{
    Console.WriteLine("x is 10");
}
```
在像这样的 `is` 表达式中，这并不有用；它更可能用于 `switch` 语句中，您可能有一些整数常量（如预模式匹配的 `switch` 语句）以及其他模式。一种明显更有用的模式是类型模式。

## **类型模式**

类型模式由一个类型和一个标识符组成——有点像变量声明。如果 `input` 是该类型的值，则模式匹配，就像常规的 `is` 运算符一样。为此使用模式的好处是，如果模式匹配，它还会引入一个该类型的新模式变量，并用该值初始化。如果模式不匹配，变量仍然存在；只是没有明确赋值。如果 `input` 为 `null`，它将不匹配任何类型。如 12.1.1 节所述，可以使用下划线标识符 `_`，在这种情况下，它是一个弃元，不会引入变量。以下清单是我们之前的 `as`-后跟-`if` 语句集（清单 12.10）的转换，以使用模式匹配，而没有采取使用 `switch` 语句的更极端步骤。

**清单 12.13 使用类型模式代替 as/if**
```csharp
static double Perimeter(Shape shape)
{
    if (shape == null)
        throw new ArgumentNullException(nameof(shape));
    if (shape is Rectangle rect)
        return 2 * (rect.Height + rect.Width);
    if (shape is Circle circle)
        return 2 * PI * circle.Radius;
    if (shape is Triangle triangle)
        return triangle.SideA + triangle.SideB + triangle.SideC;
    throw new ArgumentException(
        $"Shape type {shape.GetType()} perimeter unknown", nameof(shape));
}
```
在这种情况下，我绝对更喜欢 `switch` 语句选项，但如果只有一个 `as/if` 要替换，那将是大材小用。类型模式通常用于替换 `as/if` 组合，或者替换为 `is` 后跟强制转换的 `if`。当您测试的类型是不可为空的值类型时，后者是必需的。

类型模式中指定的类型不能是可空值类型，但可以是类型参数，并且该类型参数可能在运行时最终成为可空值类型。在这种情况下，仅当值非空时，模式才会匹配。以下清单使用 `int?` 作为方法的类型参数来展示这一点，该方法在类型模式中使用类型参数，即使表达式 `value is int? t` 本不会编译。

**清单 12.14 可空值类型在类型模式中的行为**
```csharp
static void Main()
{
    CheckType<int?>(null);
    CheckType<int?>(5);
    CheckType<int?>("text");
    CheckType<string>(null);
    CheckType<string>(5);
    CheckType<string>("text");
}
static void CheckType<T>(object value)
{
    if (value is T t)
    {
        Console.WriteLine($"Yes! {t} is a {typeof(T)}");
    }
    else
    {
        Console.WriteLine($"No! {value ?? "null"} is not a {typeof(T)}");
    }
}
```
输出如下：
```
No! null is not a System.Nullable`1[System.Int32]
Yes! 5 is a System.Nullable`1[System.Int32]
No! text is not a System.Nullable`1[System.Int32]
No! null is not a System.String
No! 5 is not a System.String
Yes! text is a System.String
```
为了结束本节关于类型模式的讨论，C# 7.0 中有一个问题在 C# 7.1 中得到了解决。这是这样一种情况：如果您的项目已经设置为使用 C# 7.1 或更高版本，您甚至可能不会注意到。我包含这一部分主要是为了让您在将代码从 C# 7.1 项目复制到 C# 7.0 项目时不会感到困惑，如果发现它破坏的话。

在 C# 7.0 中，像这样的类型模式
```csharp
x is SomeType y
```
要求 `x` 的编译时类型可以强制转换为 `SomeType`。这听起来完全合理，直到您开始使用泛型。考虑以下使用模式匹配显示所提供形状详细信息的泛型方法。

**清单 12.15 使用类型模式的泛型方法**
```csharp
static void DisplayShapes<T>(List<T> shapes) where T : Shape
{
    foreach (T shape in shapes) // 变量类型是类型参数 (T)
    {
        switch (shape) // 在该变量上进行切换
        {
            case Circle c: // 尝试使用类型转换为具体形状类型
                Console.WriteLine($"Circle radius {c.Radius}");
                break;
            case Rectangle r:
                Console.WriteLine($"Rectangle {r.Width} x {r.Height}");
                break;
            case Triangle t:
                Console.WriteLine(
                    $"Triangle sides {t.SideA}, {t.SideB}, {t.SideC}");
                break;
        }
    }
}
```
在 C# 7.0 中，此清单无法编译，因为这也不会编译：
```csharp
if (shape is Circle)
{
    Circle c = (Circle) shape;
}
```
`is` 运算符的使用是有效的，但强制转换无效。直接强制转换类型参数的能力长期以来一直是 C# 中的一个烦恼，通常的解决方法是首先强制转换为 `object`：
```csharp
if (shape is Circle)
{
    Circle c = (Circle) (object) shape;
}
```
这在普通强制转换中已经足够笨拙了，但当您试图使用优雅的类型模式时更糟。在清单 12.15 中，可以通过以下方式解决此问题：要么接受 `IEnumerable<Shape>`（利用泛型协变允许将 `List<Circle>` 转换为 `IEnumerable<Shape>`），要么将 `shape` 的类型指定为 `Shape` 而不是 `T`。在其他情况下，解决方法并不那么简单。C# 7.1 通过允许对任何使用 `as` 运算符有效的类型使用类型模式来解决此问题，这使得清单 12.15 有效。

我期望类型模式是 C# 7.0 引入的三种模式中最常用的模式。我们的最终模式听起来几乎不像一个模式。

## **var 模式**

`var` 模式看起来像类型模式，但使用 `var` 作为类型，因此它只是 `var` 后跟一个标识符：

```csharp
someExpression is var x
```
像类型模式一样，它引入一个新变量。但与类型模式不同，它不测试任何内容。它总是匹配，产生一个与 `input` 编译时类型相同的新变量，其值与 `input` 相同。与类型模式不同，即使 `input` 是空引用，`var` 模式仍然匹配。

因为它总是匹配，所以在我为其他模式演示的那种方式的 `if` 语句中使用带有 `is` 运算符的 `var` 模式是相当无意义的。它最适用于带有守卫子句的 `switch` 语句（在 12.6.1 节中描述），尽管如果您想在不将其赋值给变量的情况下切换更复杂的表达式，它有时也可能有用。

只是为了展示不使用守卫子句的 `var` 示例，清单 12.16 显示了一个类似于清单 12.11 中的 `Perimeter` 方法。但这一次，如果 `shape` 参数具有空值，则会创建一个随机形状。如果之后无法计算周长，则使用 `var` 模式报告形状的类型。现在您不再需要值为 `null` 的常量模式，因为您要确保永远不会在空引用上切换。

**清单 12.16 使用 var 模式在出错时引入变量**
```csharp
static double Perimeter(Shape shape)
{
    switch (shape ?? CreateRandomShape())
    {
        case Rectangle rect:
            return 2 * (rect.Height + rect.Width);
        case Circle circle:
            return 2 * PI * circle.Radius;
        case Triangle triangle:
            return triangle.SideA + triangle.SideB + triangle.SideC;
        case var actualShape:
            throw new InvalidOperationException(
                $"Shape type {actualShape.GetType()} perimeter unknown");
    }
}
```
在这种情况下，另一种方法是在 `switch` 语句之前引入 `actualShape` 变量，对该变量进行切换，然后像以前一样使用 `default` 情况。

以上就是 C# 7.0 中可用的所有模式。您已经看到了可以使用它们的两种上下文——与 `is` 运算符一起使用和在 `switch` 语句中使用——但在每种情况下还有更多要说的。

# **与 is 运算符一起使用模式**

`is` 运算符可以在任何地方作为正常表达式的一部分使用。它几乎总是与 `if` 语句一起使用，但肯定不一定。在 C# 7 之前，`is` 运算符的右侧必须只是一个类型，但现在可以是任何模式。虽然这确实允许您使用常量或 `var` 模式，但实际上您几乎总是会使用类型模式。

`var` 模式和类型模式都会引入一个新变量。在 C# 7.3 之前，这带来了一个额外的限制：不能在字段、属性或构造函数初始化器或查询表达式中使用它们。例如，这将是无效的：
```csharp
static int length = GetObject() is string text ? text.Length : -1;
```
我还没有发现这是一个问题，但无论如何，这个限制在 C# 7.3 中被取消了。

这就剩下了模式引入局部变量，这导致了一个明显的问题：新引入变量的作用域是什么？我理解这是 C# 语言团队和社区内部大量讨论的原因，但最终结果是引入变量的作用域是封闭块。

正如您可能从激烈辩论的话题中预料的那样，这有利有弊。关于清单 12.10 中所示的 `as/if` 模式，我从未喜欢的一点是，即使您通常不想在值匹配您测试的类型的情况之外使用它们，您最终也会在作用域内拥有很多变量。不幸的是，在使用类型模式时，情况仍然如此。情况并不完全相同，因为在模式不匹配的分支中，变量不会被明确赋值。

为了比较，在这段代码之后
```csharp
string text = input as string;
if (text != null)
{
    Console.WriteLine(text);
}
```
`text` 变量在作用域内并明确赋值。大致等效的类型模式代码如下：
```csharp
if (input is string text)
{
    Console.WriteLine(text);
}
```
在此之后，`text` 变量在作用域内，但未明确赋值。虽然这确实污染了声明空间，但如果您试图提供获取值的替代方法，它可能很有用。例如：
```csharp
if (input is string text)
{
    Console.WriteLine("Input was already a string; using that");
}
else if (input is StringBuilder builder)
{
    Console.WriteLine("Input was a StringBuilder; using that");
    text = builder.ToString();
}
else
{
    Console.WriteLine(
        $"Unable to use value of type ${input.GetType()}. Enter text:");
    text = Console.ReadLine();
}
Console.WriteLine($"Final result: {text}");
```
在这里，您真的希望 `text` 变量保持在作用域内，因为您想使用它；您以两种方式之一为其赋值。您并不真的希望 `builder` 在中间块之后还在作用域内，但您不能两全其美。

关于明确赋值，更技术性一点地说，在带有引入模式变量的模式的 `is` 表达式之后，变量（用语言规范术语来说）“在真表达式之后明确赋值”。如果您希望 `if` 条件做的不仅仅是测试类型，这可能很重要。例如，假设您想检查提供的值是否是一个大整数。这没问题：
```csharp
if (input is int x && x > 100)
{
    Console.WriteLine($"Input was a large integer: {x}");
}
```
您可以在 `&&` 之后使用 `x`，因为只有当第一个操作数计算结果为 `true` 时，您才会计算该操作数。您也可以在 `if` 语句中使用 `x`，因为只有当两个 `&&` 操作数都计算为 `true` 时，您才会执行 `if` 语句的主体。但是，如果您想处理 `int` 或 `long` 值怎么办？您可以测试该值，但随后您无法分辨哪个条件匹配：
```csharp
if ((input is int x && x > 100) || (input is long y && y > 100))
{
    Console.WriteLine($"Input was a large integer of some kind");
}
```
在这里，`x` 和 `y` 都在 `if` 语句的内部和外部作用域内，即使声明 `y` 的部分看起来可能不会执行。但变量仅在您检查值有多大的非常小的代码片段中明确赋值。

所有这一切在逻辑上都有意义，但第一次看到时可能会有点令人惊讶。本节的两个要点如下：
*   期望在 `is` 表达式中声明的模式变量的作用域是封闭块。
*   如果编译器阻止您使用模式变量，则意味着语言规则无法证明该变量在该点已被赋值。

在本章的最后一部分，我们将看看在 `switch` 语句中使用的模式。

# **使用模式与 switch 语句**

规范通常不是用算法本身来写的，而是用案例来写的。以下是与计算相距甚远的例子：

*   税收和福利——您的税级可能取决于您的收入和一些其他因素。
*   旅行票——可能有团体折扣以及儿童、成人和老年人的单独价格。
*   外卖订餐——如果您的订单符合某些标准，可能会有优惠。

过去，我们有两种方法来判断哪种情况适用于特定输入：`switch` 语句和 `if` 语句，其中 `switch` 语句仅限于简单的常量。我们仍然只有这两种方法，但正如您所见，使用模式的 `if` 语句已经更清晰，而 `switch` 语句则强大得多。

注意：基于模式的 `switch` 语句与过去仅限常量值的 `switch` 语句感觉非常不同。除非您有其他具有类似功能语言的经验，否则您应该预计需要一点时间来适应这种变化。

带有模式的 `switch` 语句在很大程度上等同于一系列 `if/else` 语句，但它们鼓励您更多地从“这种输入导致这种输出”而不是步骤的角度思考。

**所有 switch 语句都可以被视为基于模式的**
在本节中，我将基于常量的 `switch` 语句和基于模式的 `switch` 语句视为不同的。由于常量模式是模式，每个有效的 `switch` 语句都可以被视为基于模式的 `switch` 语句，并且它的行为方式将完全相同。无论如何，您稍后将看到的关于执行顺序和引入新变量的差异并不适用于常量模式。

至少目前我发现，将它们视为恰好使用相同语法的两个独立构造非常有帮助。您可能更愿意不进行这种区分。使用任意思维模型都是安全的；它们都能正确预测代码的行为。

您已经在 12.3 节中看到了 `switch` 语句中模式的示例，其中您使用常量模式匹配 `null`，并使用类型模式匹配不同类型的形状。除了简单地在 `case` 标签中放置模式之外，还有一个新的语法要介绍。

## **守卫子句**

每个 `case` 标签还可以有一个**守卫子句**，它由一个表达式组成：

```csharp
case pattern when expression:
```
该表达式必须计算为布尔值，就像 `if` 语句的条件一样。仅当表达式计算结果为 `true` 时，才会执行 `case` 的主体。表达式可以使用更多模式，从而引入额外的模式变量。

让我们看一个具体的例子，它也将说明我关于规范的观点。考虑斐波那契序列的以下定义：
*   fib(0) = 0
*   fib(1) = 1
*   对于所有 n > 1，fib(n) = fib(n-2) + fib(n-1)

在第 11 章中，您看到了如何使用元组生成斐波那契序列，当将其视为序列时，这是一种简洁的方法。但是，如果仅将其视为函数，则前面的定义导致以下清单：一个使用模式和守卫子句的简单 `switch` 语句。

**清单 12.17 使用模式递归实现斐波那契序列**
```csharp
static int Fib(int n)
{
    switch (n)
    {
        case 0: return 0; // 使用常量模式处理基本情况
        case 1: return 1;
        case var _ when n > 1: return Fib(n - 2) + Fib(n - 1); // 使用 var 模式和守卫子句处理递归情况
        default: throw new ArgumentOutOfRangeException( // 如果您不匹配任何模式，则输入无效。
            nameof(n), "Input must be non-negative");
    }
}
```
这是一个我永远不会在现实生活中使用的极其低效的实现，但它清楚地展示了如何将规范直接转换为代码。

在这个例子中，守卫子句不需要使用模式变量，所以我使用了带有 `_` 标识符的弃元。在许多情况下，如果模式引入了新变量，它将在守卫子句或至少 `case` 主体中使用。

当您使用守卫子句时，同一个模式多次出现是完全合理的，因为第一次模式匹配时，守卫子句可能会计算为 `false`。这是来自 Noda Time 的一个工具示例，用于构建文档：
```csharp
private string GetUid(TypeReference type, bool useTypeArgumentNames)
{
    switch (type)
    {
        case ByReferenceType brt:
            return $"{GetUid(brt.ElementType, useTypeArgumentNames)}@";
        case GenericParameter gp when useTypeArgumentNames:
            return gp.Name;
        case GenericParameter gp when gp.DeclaringType != null:
            return $"`{gp.Position}";
        case GenericParameter gp when gp.DeclaringMethod != null:
            return $"``{gp.Position}";
        case GenericParameter gp:
            throw new InvalidOperationException(
                "Unhandled generic parameter");
        case GenericInstanceType git:
            return "(This part of the real code is long and irrelevant)";
        default:
            return type.FullName.Replace('/', '.');
    }
}
```
我有四个模式根据 `useTypeArgumentNames` 方法参数处理泛型参数，然后根据泛型类型参数是在方法中还是在类型中引入的。抛出异常的情况几乎是泛型参数的 `default` 情况，表明遇到了我尚未想到的情况。我为多个 `case` 使用相同的模式变量名 (`gp`) 这一事实引出了另一个自然问题：`case` 标签中引入的模式变量的作用域是什么？

## **case 标签的模式变量作用域**

如果您在 `case` 主体内直接声明局部变量，则该变量的作用域是整个 `switch` 语句，包括其他 `case` 主体。这仍然成立（并且不幸，在我看来），但不包括在 `case` 标签中声明的变量。这些变量的作用域仅是与该 `case` 标签关联的主体。这适用于由模式声明的模式变量、在守卫子句内声明的模式变量以及在守卫子句中声明的任何 `out` 变量（见 14.2 节）。

这几乎肯定是您想要的，并且它允许您为处理类似情况的多个 `case` 使用相同的模式变量，如 Noda Time 工具代码所示。这里有一个怪癖：就像普通的 `switch` 语句一样，可以有多个具有相同主体的 `case` 标签。在这一点上，该主体所有 `case` 标签内声明的变量必须具有不同的名称（因为它们贡献给同一个声明空间）。但是在 `case` 主体内，这些变量都没有被明确赋值，因为编译器无法判断哪个标签匹配。引入这些变量仍然可能有用，但主要是为了在守卫子句中使用它们。

例如，假设您正在匹配一个对象 `input`，并且您想确保如果它是数字，则它在特定范围内，并且该范围可能因类型而异。您可以为每种数值类型使用一个类型模式，并带有相应的守卫子句。以下清单针对 `int` 和 `long` 展示了这一点，但您可以扩展到其他类型。

**清单 12.18 对单个 case 主体使用带模式的多个 case 标签**
```csharp
static void CheckBounds(object input)
{
    switch (input)
    {
        case int x when x > 1000:
        case long y when y > 10000L:
            Console.WriteLine("Value is too large");
            break;
        case int x when x < -1000:
        case long y when y < -10000L:
            Console.WriteLine("Value is too low");
            break;
        default:
            Console.WriteLine("Value is in range");
            break;
    }
}
```
模式变量在守卫子句内明确赋值，因为只有当模式一开始匹配时，执行才会到达守卫子句，并且它们仍然在主体作用域内，但它们没有被明确赋值。您可以为其分配新值并在之后使用它们，但我觉得这通常不会有太大用处。

除了模式匹配的基本前提是新的和不同的之外，过去的基于常量的 `switch` 语句和新的基于模式的 `switch` 语句之间有一个巨大的区别：`case` 的顺序以一种以前不存在的方式重要。

## **基于模式的 switch 语句的求值顺序**

在几乎所有情况下，基于常量的 `switch` 语句的 `case` 标签可以自由重新排序而不会改变行为。这是因为每个 `case` 标签匹配一个常量值，并且用于任何 `switch` 语句的常量都必须不同，因此任何输入最多只能匹配一个 `case` 标签。对于模式，情况不再如此。

基于模式的 `switch` 语句的逻辑求值顺序可以简单地总结为：
*   每个 `case` 标签按源代码顺序求值。
*   `default` 标签的代码主体仅在所有 `case` 标签都已求值后执行，无论 `default` 标签在 `switch` 语句中的位置如何。

提示：尽管您现在知道，仅当没有任何 `case` 标签匹配时，才会执行与 `default` 标签关联的代码，无论它出现在哪里，但阅读您代码的某些人可能不知道。（实际上，当您下次阅读自己的代码时，您可能已经忘记了。）如果您将 `default` 标签作为 `switch` 语句的最后一部分，行为总是清晰的。

有时这无关紧要。例如，在我们的斐波那契计算方法中，情况只有 0、1 和大于 1，因此它们可以自由重新排序。然而，我们的 Noda Time 工具代码有四个情况，肯定需要按顺序检查：
```csharp
case GenericParameter gp when useTypeArgumentNames:
    return gp.Name;
case GenericParameter gp when gp.DeclaringType != null:
    return $"`{gp.Position}";
case GenericParameter gp when gp.DeclaringMethod != null:
    return $"``{gp.Position}";
case GenericParameter gp:
    throw new InvalidOperationException(...);
```
在这里，无论何时 `useTypeArgumentNames` 为 `true`（第一种情况），您都想使用泛型类型参数名称，而不管其他情况。第二种和第三种情况是互斥的（以您知道但编译器不知道的方式），因此它们的顺序无关紧要。最后一种情况必须在这四种情况下最后出现，因为您希望仅在输入是未以其他方式处理的 `GenericParameter` 时抛出异常。

编译器在这里很有帮助：最后一种情况没有守卫子句，因此如果类型模式匹配，它将始终有效。编译器知道这一点；如果您将该情况放在具有相同模式的其他 `case` 标签之前，它知道这实际上隐藏了它们并报告错误。

只有一个方法可以执行多个 `case` 主体，那就是使用很少使用的 `goto` 语句。这在基于模式的 `switch` 语句中仍然有效，但您只能 `goto` 一个常量值，并且 `case` 标签必须与该值关联而没有守卫子句。例如，您不能 `goto` 一个类型模式，也不能在关联的守卫子句也计算为 `true` 的条件下 `goto` 一个值。实际上，我在 `switch` 语句中看到的 `goto` 语句如此之少，以至于我认为这不是什么大限制。

我之前特意提到了逻辑求值顺序。尽管 C# 编译器可以有效地将每个 `switch` 语句转换为一系列 `if/else` 语句，但它可以比这更高效地行动。例如，如果有多个针对相同类型但具有不同守卫子句的类型模式，它可以一次求值类型模式部分，然后依次检查每个守卫子句。同样，对于没有守卫模式的常量值（它们仍然必须不同，就像 C# 的先前版本一样），编译器可以使用 IL 的 `switch` 指令，可能在执行隐式类型检查之后。编译器执行哪些优化超出了本书的范围，但如果您碰巧查看与 `switch` 语句关联的 IL，并且它与源代码几乎没有相似之处，那么这很可能就是原因。



> 本章详细介绍了C# 7的核心特性之一：模式匹配。它不仅仅是语法糖，而是一种**声明式编程范式的引入**，允许开发者根据数据的“形状”（类型、值、结构）而非仅类型进行条件分支。
>
> 模式匹配通过`is`表达式和增强的`switch`语句实现，提供了三种基本模式：常量模式、类型模式和`var`模式。其核心价值在于**将条件判断与变量提取合二为一**，显著简化了传统的`as`/`if`或`is`/`cast`组合代码。守卫子句（`when`）进一步允许在模式基础上施加额外条件，实现精细控制。
>
> 关键设计决策包括：模式变量的作用域规则（封闭块）、`switch`中`case`的**顺序敏感性**（与常量`switch`不同），以及对可空值类型和泛型的细致处理。这些特性共同使模式匹配尤其适合处理复杂的分层数据（如AST、UI事件流或业务规则引擎），将冗长的过程式条件逻辑转化为清晰、可组合的声明式规则。
>
> 与解构结合，模式匹配推动了C#向更函数式、更注重数据流的方向演进。它鼓励开发者从“如何检查”转向“数据应满足什么条件”，提升了代码的表达力和可维护性。不过，也需注意其适用场景，避免在不必要时过度使用，以免降低简单条件的可读性。



# **关于使用的思考**

本节提供了关于如何最好地使用本章所述功能的初步思考。这两个功能都可能进一步发展，甚至可能与解构模式相结合。其他相关的潜在功能，例如为基于模式匹配的 switch 表达式编写表达式主体方法的语法，也很可能影响这些功能的使用场景。您将在第15章看到一些类似的潜在 C# 8 功能。

模式匹配是一个实现细节问题，这意味着您不必担心以后发现自己过度使用了它。如果发现模式匹配没有带来预期的可读性优势，您可以恢复使用旧的编码风格。在某种程度上，解构也是如此。但是，如果您在整个 API 中添加了公共的 Deconstruct 方法，那么移除它们将是一项破坏性变更。

更重要的是，我认为大多数类型本身并不天然适合解构，就像大多数类型没有天然的 `IComparable<T>` 实现一样。我建议仅在组成部分的顺序明显且明确无误时才添加 Deconstruct 方法。这对于坐标、任何具有层次结构性质的事物（如日期/时间值），甚至存在通用约定的情况（例如将颜色视为 RGB 加可选的 Alpha）来说是可以的。然而，大多数与业务相关的实体可能不属于此类；例如，在线购物篮中的商品具有多个方面，但它们之间没有明显的顺序。

## **发现解构机会**

最简单的解构使用可能与元组有关。如果您调用的方法返回一个元组，并且您不需要将这些值保持在一起，请考虑对它们进行解构。例如，对于第11章中的 MinMax 方法，我几乎总是立即进行解构，而不是将返回值保持为元组：

```csharp
int[] values = { 2, 7, 3, -5, 1, 0, 10 };
var (min, max) = MinMax(values);
Console.WriteLine(min);
Console.WriteLine(max);
```

我怀疑非元组解构的使用会更少见，但如果您处理的是点、颜色、日期/时间值或类似的东西，您可能会发现，如果您本需要通过属性多次引用其组成部分，那么尽早解构该值是值得的。在 C# 7 之前您也可以做到这一点，但是通过解构轻松声明多个局部变量的便利性，可以轻易地改变“不值得做”和“值得做”之间的平衡。

## **发现模式匹配机会**

您应该在两个明显的地方考虑使用模式匹配：
 任何您正在使用 `is` 或 `as` 运算符，并根据更具体的类型有条件地执行代码的地方。
 任何您拥有使用相同值进行所有条件的 `if/else-if/else-if/else` 序列，并且可以用 `switch` 语句替代的地方。

如果您发现自己多次使用 `var ... when` 这种模式（换句话说，唯一的条件出现在 `when` 保护子句中），您可能需要问问自己这是否真的是模式匹配。我当然遇到过这样的场景，到目前为止，我倾向于仍然使用模式匹配。即使感觉略有滥用，但在我看来，它比 `if/else` 序列更清晰地传达了匹配单个条件并执行单一操作的意图。

这两者都是对现有代码结构的转换，只改变了实现细节。它们并没有改变您思考和组织逻辑的方式。那种更宏大的风格转变——可能仍然是在单个类型的可见 API 内，或者通过更改内部细节在程序集的公共 API 内进行重构——更难被发现。有时它可能意味着不再使用继承；计算逻辑在一个单一的位置、考虑所有不同情况来表达，可能比将其作为代表每种情况的类型的一部分来表达更清晰。12.3 节中形状周长的例子就是其中之一，但您可以轻松地将同样的想法应用到许多业务案例中。这是不相交联合类型在 C# 中可能变得更加广泛应用的领域。

正如我所说，这些都是初步的思考。一如既往，我鼓励您进行有意识的实验和内省：在编码时考虑机会，如果您尝试了新东西，完成后反思它的优缺点。

> 本章节的核心在于强调技术特性的“适用性”与“演进性”。作者指出，解构与模式匹配虽强大，但并非“银弹”。解构的合理性取决于类型是否存在“天然、无歧义”的组成部分顺序，这使其更适用于坐标、颜色、时间等具有明确维度或通用约定的领域，而非大多数业务实体。模式匹配则显著提升了类型测试与条件分支的可读性与简洁性，特别是在替代 `is`/`as` 操作和复杂的 `if`/`else` 链时。作者预见，随着 C# 的发展（如表达式主体方法的增强），这些特性的使用场景将动态演变。更深层的启示在于，这些特性不仅仅是语法糖，它们能推动我们反思代码的组织逻辑，例如将分散在继承体系中的行为逻辑，通过基于模式的 `switch` 集中表达，这可能导向更函数式、数据驱动的设计风格（如不相交联合类型），从而改变我们构建系统的方式。





