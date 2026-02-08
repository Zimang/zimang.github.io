---
weight: 5
title: " C# 8及更高版本"
---

**本章涵盖内容**
- 为引用类型表达空值和非空值的预期
- 使用带有模式匹配的switch表达式
- 针对属性递归匹配模式
- 使用索引和范围语法实现简洁一致的代码
- 使用using、foreach和yield语句的异步版本

在撰写本文时，C# 8仍在设计中。GitHub仓库显示了许多潜在功能，但只有少数功能达到了编译器公开预览版的阶段。本章内容是基于经验的推测；这里没有任何内容是确定不变的。几乎不可想象所有正在考虑的特性都会被纳入C# 8，我仅限于讨论我认为相当有可能被纳入的特性。我详细介绍了撰写本文时可用的预览特性，但即便如此，这并不意味着不会发生进一步的变化。

**注意**：在撰写本文时，只有少数C# 8功能在预览版中可用，并且不同版本具有不同的功能。可空引用类型的预览版仅支持完整.NET项目（而非.NET Core SDK风格的项目），如果您的所有项目都使用新的项目格式，这使得在实际代码上实验它们变得更加困难。我期望这些限制会在后续版本中得到解决，可能在您阅读本文时已经解决。

我们将从可空引用类型开始。

# **可空引用类型**

啊，空引用。Tony Hoare在1960年代引入它们后，于2009年道歉的所谓"十亿美元的错误"。很难找到一个有经验的C#开发人员没有至少几次被NullReferenceException咬过。C#团队有一个计划来驯服空引用，使我们更清楚地知道应该在哪里期望找到它们。

## **可空引用类型解决了什么问题？**

作为我在本节中扩展的示例，让我们考虑下面的类。如果您在下载的源代码中跟随，您会看到我将它们声明为每个示例中的单独嵌套类，因为代码会随时间变化。

**清单15.1 C# 8之前的初始模型**
```csharp
public class Customer
{
    public string Name { get; set; }
    public Address Address { get; set; }
}

public class Address
{
    public string Country { get; set; }
}
```

地址通常包含比国家多得多的信息，但单个属性足以用于本章的示例。有了这些类，这段代码有多安全？

```csharp
Customer customer = ...;
Console.WriteLine(customer.Address.Country);
```

如果您（以某种方式）知道customer非空且客户总是有一个关联的地址，那可能没问题。但您如何知道呢？如果您知道仅仅是因为您查看了文档，那么需要改变什么才能使代码更安全？

自C# 2以来，我们有了可空值类型、不可空值类型和隐式可空的引用类型。一个可空/不可空与值/引用类型的网格已经填充了四个格子中的三个，但第四个一直难以捉摸，如表15.1所示。

**表15.1 C# 7中引用类型和值类型的可空性和不可空性支持**

| 类型     | 可空               | 不可空 |
| -------- | ------------------ | ------ |
| 引用类型 | 隐式               | 不支持 |
| 值类型   | Nullable<T>或?后缀 | 默认   |

顶行只有一个支持的格子，这意味着我们无法表达某些引用值可能为空而其他引用值应该永远不为空的意图。当您遇到意外的空值问题时，很难确定故障所在，除非代码经过仔细记录并一致地实现了空值检查。

考虑到现在存在大量.NET代码，没有机器可读的区别来区分可以合理为空的引用和必须始终非空的引用，任何尝试纠正这种情况都只能是谨慎的。我们能做什么？

## **使用引用类型时改变含义**

空安全功能的广泛理念是假设当开发人员有意区分不可空和可空引用类型时，默认是不可空的。为可空引用类型引入了新语法：string是不可空引用类型，而string?是可空引用类型。网格然后演变为表15.2所示。

**表15.2 C# 8中引用类型和值类型的可空性和不可空性支持**

| 类型     | 可空                           | 不可空                         |
| -------- | ------------------------------ | ------------------------------ |
| 引用类型 | 无CLR类型表示，但?后缀作为注释 | 启用可空引用类型支持时的默认值 |
| 值类型   | Nullable<T>或?后缀             | 默认值                         |

这听起来与谨慎相反；它改变了所有处理引用类型的C#代码的含义！启用该功能会将默认值从可空更改为不可空。预期是空引用意图有效的地方远少于它应该永远不会出现的地方。

让我们回到客户和地址的示例。在不更改任何代码的情况下，编译器警告我们Customer和Address类允许不可空属性未初始化。这可以通过添加具有不可空参数的构造函数来修复，如以下清单所示。

**清单15.2 到处是不可空属性的模型**
```csharp
public class Customer
{
    public string Name { get; set; }
    public Address Address { get; set; }
    
    public Customer(string name, Address address) =>
        (Name, Address) = (name, address);
}

public class Address
{
    public string Country { get; set; }
    
    public Address(string country) =>
        Country = country;
}
```

此时，您"不能"在不提供非空姓名和地址的情况下构造Customer，也"不能"在不提供非空国家的情况下构造Address。我故意将"不能"放在引号中，原因您将在15.1.4节看到。

但现在再次考虑我们的控制台输出代码：
```csharp
Customer customer = ...;
Console.WriteLine(customer.Address.Country);
```

假设每个人都正确遵守契约，这是安全的。它不仅不会抛出异常，而且您不会将空值传递给Console.WriteLine，因为地址中的国家不会为空。

好的，所以编译器可以检查事物是否为空。但是当您想允许空值时呢？是时候探索我之前提到的新语法了。

## **可空引用类型的引入**

用于指示可以为空的引用类型的语法设计为立即熟悉。它与可空值类型的语法相同：在类型名称后添加问号。这可以在引用类型出现的大多数地方使用。例如，考虑这个方法：

```csharp
string FirstOrSecond(string? first, string second) =>
    first ?? second;
```

该方法的签名显示：
- first的类型是可空字符串。
- second的类型是不可空字符串。
- 返回类型是不可空字符串。

然后编译器使用该信息来警告您是否尝试误用可能为空的值。例如，如果您执行以下操作，它可以警告您：
- 将可能为空的值赋值给不可空变量或属性。
- 将可能为空的值作为不可空参数的参数传递。
- 解引用可能为空的值。

让我们将其构建到客户模型中。假设客户地址可能为空。您需要按如下方式修改Customer类：
- 更改属性类型。
- 要么删除地址的构造函数参数，使其可为空，要么重载它。

Address类型本身没有改变，仅改变其使用方式。以下清单显示了新的Customer类。我选择删除地址的构造函数参数。

**清单15.3 使客户Address属性可空**
```csharp
public class Customer
{
    public string Name { get; set; }
    public Address? Address { get; set; } // 地址现在是可选信息
    
    public Customer(string name) =>
        Name = name; // 从构造函数中移除地址参数
}
```

很好，您现在已明确表达了您的意图：Name属性不会为空，但Address属性可能为空。当您尝试显示用户地址的国家时，编译器现在给您一个不同的警告：
```
CS8602 可能解引用空引用。
```

太好了！它现在识别了您最初面临的问题，该问题导致了NullReferenceException。如何修复问题？是时候看看可空引用类型的行为，而不仅仅是语法了。

## **编译时和运行时的可空引用类型**

新功能的一个黄金法则是没有行为被隐式更改。即使您的代码的含义已更改为假定不可空类型的意图，但行为没有改变。唯一的区别是编译时生成的警告。没有涉及新的真实类型；CLR没有可空与不可空引用类型的概念。使用属性来传播可空性信息，但仅此而已。这类似于有关元组元素名称的额外信息，这些信息在运行时不是类型的一部分。这有两个重要后果：

- 防御性编程仍然是最佳实践。使用您到目前为止编写的代码，Name可能为空，因为用户可能忽略警告或使用仅使用C# 7的代码。参数验证仍然很重要。
- 要充分理解该功能，您需要理解编译器警告。您绝对不应该忽略它们；它们的存在是为了提供价值。

让我们看看您当前面临的警告，并考虑所有可以避免它的方法。您目前有：
```csharp
Console.WriteLine(customer.Address.Country);
```

编译器正确地告诉您这是危险的，因为customer.Address可能为空。您将看到三种可以使代码更安全的方法。首先，您可以一起使用空条件运算符和空合并运算符，如以下清单所示。

**清单15.4 使用空条件运算符安全解引用**

```csharp
Console.WriteLine(customer.Address?.Country ?? "(Address unknown)");
```

如果customer.Address为空，表达式customer.Address?.Country将不会尝试评估Country属性，表达式的结果将为空。然后空合并运算符提供一个默认值进行打印。编译器理解您不再尝试解引用任何可能为空的东西，警告消失。

您此刻可能对此感到有点不安。如果不小心，很容易迷失在问号的海洋中。我相信C#开发人员会随着时间的推移对此变得更加舒适，但这不是唯一可用的解决方案。您可以采取更冗长但易于遵循的方法，如以下清单所示。

**清单15.5 使用局部变量检查引用**
```csharp
Address? address = customer.Address; // 将地址提取到新的局部变量
if (address != null) // 检查是否为空，并且仅在非空时解引用
{
    Console.WriteLine(address.Country);
}
else
{
    Console.WriteLine("(Address unknown)");
}
```

这里有一个有趣的点需要注意：编译器需要跟踪的不仅仅是变量的类型。如果规则像"解引用可空引用类型的值会导致警告"那么简单，那么这段代码仍然会产生警告，尽管是安全的。相反，编译器跟踪变量的值在代码中的每个位置是否可能为空，类似于它跟踪明确赋值的方式。当您到达if语句的主体时，编译器知道address的值不能为空，因此在解引用它时不会警告。我们的第三种方法，如下一个清单所示，与第二种类似，但没有局部变量。

**清单15.6 通过重复属性访问检查引用**
```csharp
if (customer.Address != null)
{
    Console.WriteLine(customer.Address.Country);
}
else
{
    Console.WriteLine("(Address unknown)");
}
```

即使您理解第二个示例如何在没有警告的情况下编译，清单15.6可能有点令人惊讶。编译器不仅跟踪变量值是否可能为空；它对属性也是如此。它假设如果您在相同的值上访问相同的属性两次，结果两次都是相同的。

这可能会让您担心。这意味着该功能不能保证阻止您的代码解引用空值。另一个线程可能在您看到的两次调用之间修改Address属性，或者Address属性本身可能被编写为有时随机返回空值。还有其他方法可以欺骗编译器相信您的代码没问题，而实际上并不绝对安全。这是C#设计团队知道并接受的，因为这是在安全和尴尬之间的务实平衡。使用C# 8功能的代码将比之前编写的代码更安全，但使其100%安全几乎肯定需要更具侵入性的更改，这会使许多开发人员望而却步。只要您理解它试图实现的目标的局限性，您就会没事。

您已经看到编译器努力理解什么可能为空或可能不为空。当它没有您那么多上下文时，您能做什么？

## **该死的或感叹号运算符**

还有一个您尚未看到的额外语法：该死的（dammit）或感叹号（bang）运算符。这是一个表达式末尾的感叹号，它是一种告诉编译器忽略它认为知道的关于该表达式的任何信息，只将其视为非空的方式。

这在两种相反的情况下很有用：
- 有时您拥有比编译器更多的信息，所以您知道一个值不会为空，即使编译器认为它可能为空。
- 有时您想故意传入空值以检查您的参数验证。

第一种情况的简短示例有些做作，因为您通常会尝试重新组织代码以避免陷入那种情况。在小例子中，这几乎总是可行的，但在实际应用中更难。以下清单显示了一个方法，用于打印可能为空的字符串的长度。

**清单15.7 使用感叹号运算符满足编译器**
```csharp
static void PrintLength(string? text) // 输入可以为空
{
    if (!string.IsNullOrEmpty(text)) // 如果IsNullOrEmpty返回false，则不为空
    {
        Console.WriteLine($"{text}: {text!.Length}"); // 使用感叹号运算符说服编译器
    }
    else
    {
        Console.WriteLine("Empty or null");
    }
}
```

在这个例子中，您知道编译器不知道的一些信息，关于string.IsNullOrEmpty的输入和返回值之间的关系。如果string.IsNullOrEmpty返回false，输入不能为空，因此解引用该值以获取字符串长度是可以的。如果您只是尝试使用text.Length，编译器会发出警告。使用text!.Length，您是在告诉编译器您更清楚，有效地承担了推理值的责任。

现在如果编译器确实理解string.IsNullOrEmpty方法的输入/结果关系，那就太好了。我们将在15.1.7节回到这个想法。

感叹号运算符的第二种用法用一个现实的例子更容易演示。我之前提到，您仍然应该验证参数是否为空，因为您仍然完全有可能收到空值。然后您可能想为该验证添加单元测试，但编译器会警告您，因为您提供了一个空值，而您说过它不应该为空。以下清单显示了感叹号运算符如何修复此问题。

**清单15.8 在单元测试中使用感叹号运算符**
```csharp
public class Customer
{
    public string Name { get; }
    public Address? Address { get; }
    
    public Customer(string name, Address? address)
    {
        Name = name ?? throw new ArgumentNullException(nameof(name));
        Address = address;
    }
}

public class Address
{
    public string Country { get; }
    
    public Address(string country)
    {
        Country = country ?? throw new ArgumentNullException(nameof(country));
    }
}

[Test]
public void Customer_NameValidation()
{
    Address address = new Address("UK");
    Assert.Throws<ArgumentNullException>(
        () => new Customer(null!, address)); // 故意为不可空参数传入空值
}
```

在清单15.8中，为了简单起见，我将Customer和Address类型设为不可变的。有趣的是，编译器在验证本身上没有引发任何类型的警告。即使它知道该值不应该为空，它也不会抱怨代码检查它是否为空。但它确实会尝试强制在测试中调用构造函数时，第一个参数是非空的。在早期版本的C#中，测试中的lambda表达式如下所示：
```csharp
() => new Customer(null, address)
```

该代码会生成警告，正如您在几乎所有情况下所希望的那样。将参数更改为null!可以满足编译器，并且测试执行您想要的操作。这就提出了在实践中使用可空引用类型是什么样的问题，特别是如何迁移现有代码以使用该功能。

## **可空引用类型迁移经验**

没有比尝试更好的方法来感受一个功能是如何工作的。我使用C# 8预览版与Noda Time一起查看需要多少工作才能使其无警告，以及它是否发现了任何错误。本节描述了这一经验以及我发现自己遵循的一些指导原则。您的代码可能面临不同的挑战，但我怀疑会有很多共性。

**在C# 8之前使用属性表达可空意图**
很长时间以来，Noda Time一直使用属性（至少对于所有公共方法）来指示引用类型参数是否可以为空，同样指示返回值是否可以返回空值。例如，这是IDateTimeZoneProvider中的一个方法签名：
```csharp
[CanBeNull] DateTimeZone GetZoneOrNull([NotNull] string id);
```

这表明id参数的参数必须不为空，但该方法可能返回空引用。我已经表达了关于空值的意图，只是不是以C#编译器理解的方式。这意味着我的第一遍只是去代码中所有我说过空值允许的地方，并将它们更改为使用可空引用类型。

我恰好使用了ReSharper提供的JetBrains注解。这允许ReSharper执行与C# 8在语言中相同的检查。除了注意到它们可用之外，我不会深入这些注解的细节。但是，您完全不必使用第三方注解集。您可以轻松创建自己的属性并立即应用它们。即使没有任何工具支持，这也可以使您的代码更易于维护，并且您将处于更好的位置以在将来迁移到C# 8可空引用类型。

**迭代是自然的**
在第一遍之后，我大约有100个警告。我检查并修复了大部分，然后重新构建。第二遍之后，我大约有110个警告——比之前更多！我检查并修复了大部分，然后重新构建。第三遍之后，我仍然有大约100个警告。我检查并修复了大部分，然后重新构建。

我不记得这花了多少次迭代，但这并不表示有任何问题。使代码库符合可空引用类型的过程就像打地鼠：您决定更改一个地方的可空性，然后这可能导致使用该值的任何地方都出现警告。您更改那些地方，问题又转移了。关于可空性的决策通过代码传播，需要仔细检查。这很好，是您应该期望发生的事情。

但是当代码的一部分需要一个值可空而另一部分需要它不可空时，您发现了一个问题。这不是C# 8引入的问题；而是该功能揭示的问题。您如何处理它将取决于具体上下文。

**使用感叹号运算符的最佳实践**
如果您不得不在生产代码中使用感叹号运算符，请添加注释解释为什么这样做。如果您使用易于搜索的格式（例如，在注释中包含NULLABLEREF），您将能够在以后找到它们。您可能能够通过进一步的工具改进在以后删除该运算符。使用该运算符并不是错误的，但它断言您比编译器更清楚，我宁愿不那么信任自己。

我更常在测试代码中使用该运算符，主要用于执行您在前一节看到的那种验证测试。除此之外，如果我期望一个值是非空的，因为我已经设置了测试，我通常很乐意强制编译器接受它，特别是如果我知道它会被我之后调用的代码验证。如果我错了，结果应该是测试失败，要么是ArgumentNullException，要么是NullReferenceException，这没关系，因为我会知道我的假设无效。可以说，测试代码通常应该比生产代码更少防御性；与其以优雅的方式处理意外情况，不如让它们失败。

**空不一致的泛型**
我发现在Noda Time中为引用类型实现IEqualityComparer<T>很奇怪，因为它在可空引用类型被考虑之前很久就定义了。Equals和GetHashCode都是根据类型T的参数定义的，但它们在空处理方面不一致：Equals旨在处理空值，但GetHashCode旨在抛出ArgumentNullException。

不清楚这应如何在实现中表达。如果我有一个Period类的相等比较器，我应该实现IEqualityComparer<Period?>以允许空参数，还是实现IEqualityComparer<Period>以禁止它们？无论哪种方式，调用者可能在编译时或运行时感到惊讶。

除了实现问题之外，我不清楚这如何在接口本身中更清楚地表达。可能需要更多的语言设计工作来表达应如何处理泛型类型参数。仅仅在接口中使用T?会感觉不对，因为当T是值类型时，您不想接受Nullable<T>。

尽管我恰好通过IEqualityComparer<T>遇到了这个问题，但我预计同样的问题会出现在其他接口甚至泛型类中。我在这里主要提到它，是为了让您在遇到它时不要认为自己做错了什么。

**最终结果**
Noda Time代码库不大，但也不小。整个过程花了我大约五个小时，包括诊断Roslyn预览版中的一个错误的时间。最后，我在Noda Time中发现了一个错误（现已修复），涉及不一致地处理一个奇怪的情况，即TimeZoneInfo.Local在Mono上的某些环境中返回空值。我还发现了一些缺失的注解，并不得不澄清一些内部成员的意图。

我对结果感到满意；知道编译器正在检查代码的一致性提高了我的信心。此外，在我发布了使用C# 8构建的Noda Time版本后，任何从C# 8使用该库的人都将受益于额外信息。这将有助于将更多错误从运行时转移到编译时，让用户更有信心地使用Noda Time。这是一个双赢的局面。

所有这些经验都是基于2018年上半年的预览版。然而，这并不是语言设计或实现的最终状态。让我们推测一下未来。

## **未来改进**

2018年6月，我与C#语言设计团队负责人Mads Torgersen在会议和用户组中度过了一段时间。我带着一长串基于我在Noda Time的经验提出的功能请求和问题清单旅行，他的回应让我对这些功能的未来放心。

C#团队知道已经可用的预览版尚未完全准备好供主流采用。有一些事情需要更多的工作，但预览版允许团队收集早期反馈。这里列出的更改不会是唯一的，但它们是我最感兴趣的。

**为编译器提供更多语义信息**
当我在15.1.5节介绍感叹号运算符时，我展示了编译器不理解string.IsNullOrEmpty的语义。（编译器不推断如果方法返回false，输入不可能是空。）这不是唯一一个输入和输出之间的关系应该能够帮助编译器的情况。以下是三个感觉应该在没有警告的情况下编译的示例（包括string.IsNullOrEmpty以保持完整）：
```csharp
string? a = ...;
if (!string.IsNullOrEmpty(a))
{
    Console.WriteLine(a.Length);
}

object b = ...;
if (!ReferenceEquals(b, null))
{
    Console.WriteLine(b.GetHashCode());
}

XElement c = ...;
string d = (string) c;
```

在每种情况下，您调用的代码的语义很重要。对于这些示例，编译器需要知道：
- 如果string.IsNullOrEmpty的结果是false，输入不能为空。
- 如果ReferenceEquals的结果是false，并且已知其中一个输入是空引用，则另一个输入不能为空。
- 如果XElement到字符串转换运算符的输入非空，则输出也非空。

这些都是输入和输出之间关系的示例，目前无法表达这些关系。我怀疑如果编译器理解这些关系，预览版中的大多数感叹号运算符的使用都可以避免。编译器如何获得这些额外信息？

一种适用于这些特定示例的方法是为编译器硬编码这些信息。这对C#设计团队来说很容易，但在其他方面不能令人满意。这会将框架库放在与第三方库不同的基础上，这很烦人。例如，我可能想在Noda Time中表达这样的关系，这将使其使用起来更愉快。

C#团队可能会设计一种全新的迷你语言，可以用属性表达，以给编译器提供额外的语义信息，使其更智能地确定特定值是否应被视为"绝对非空"。这将需要大量的设计和实现工作，但将提供一个更完整的解决方案。

**对泛型的深入思考**
泛型给可空性设计带来了有趣的挑战。我在实现IEqualityComparer<T>时提到了一个例子，但问题远不止于此。考虑以下在C# 7中已经有效的简单类：
```csharp
public class Wrapper<T>
{
    public T Value { get; set; }
}
```

那应该有效吗？它意味着什么？特别是，构造它的实例而不设置Value属性的结果是什么？
- 对于Wrapper<int>，Value的值默认将为0。
- 对于Wrapper<int?>，Value的值默认将为int?的空值。
- 对于Wrapper<string>，Value的值默认将为空引用。这很糟糕，因为它违反了Value的类型是不可空字符串类型。
- 对于Wrapper<string?>，Value的值默认将为空引用。这没关系，因为Value的类型是可空字符串类型。

当您考虑到在运行时，Wrapper<int>和Wrapper<int?>将是不同的CLR类型，但Wrapper<string>和Wrapper<string?>将是相同的CLR类型时，这变得更加混乱。

我不知道C# 8将如何解决这种混淆，但团队意识到了这一点。我很高兴这是他们的工作而不是我的工作，因为仅仅思考它就让我头疼。

该示例仅使用在C# 7中有效的语法，根本没有显式引用可空类型。如果您尝试在泛型类型或方法中使用T?会怎样？

在C# 7中，如果您有类型参数T，类型T?只能在T被约束为不可空值类型时使用，此时它意味着Nullable<T>。这相当简单，但是对于可空引用类型，您能做什么？似乎您可能需要一个新的泛型约束，即不可空引用类型，此时当T被约束为不可空值类型或被约束为不可空引用类型时，T?可以使用。我不期望单个约束表示"某个不可空类型"，因为相应可空类型的表示在值类型和引用类型之间非常不同。

**选择加入的参数验证**
到目前为止实现的更改仅在编译时。编译器生成的IL没有改变，您仍然需要执行参数验证以防止代码忽略编译器警告、使用感叹号运算符或针对早期版本的C#编译。

这有道理，但验证感觉像是样板代码。空合并运算符、nameof运算符和throw表达式都是在某些情况下帮助改进验证所需代码的功能，但它仍然很烦人且容易忘记。

正在讨论的一个功能是允许在参数名称后使用感叹号，以指示编译器应在方法开头生成空验证。考虑一个当前可能这样写的方法：
```csharp
static void PrintLength(string text)
{
    string validated = text ?? throw new ArgumentNullException(nameof(text));
    Console.WriteLine(validated.Length);
}
```

您可以改为这样写：
```csharp
static void PrintLength(string text!)
{
    Console.WriteLine(text.Length);
}
```

自动空验证。属性可能以相同的方式具有自动验证。

**启用可空性检查**
在我使用的预览版中，可空性检查默认是打开的。虽然您可以以正常方式抑制警告，但C# 8编译器在发布前可能会有更细致的设置。有许多不同的场景需要考虑。

当开发人员升级到C# 8编译器时，他们可能希望在不看到任何新警告的情况下进行。如果项目设置将警告视为错误，这一点尤其重要。我怀疑这意味着可空性检查将默认关闭，至少对于现有项目是如此。

并非所有类库都会同时采用C# 8。对于使用C# 8并启用可空性检查的代码来说，能够使用尚未迁移的库是很重要的。这可能倾向于报告尽可能少的错误。例如，编译器可以将库的所有输入视为可空，但库的所有输出视为不可空。此外，需要一种方式让库指示它何时已迁移。

当开发人员决定迁移一个项目以使用可空引用类型时，他们可能希望在几次更改的过程中进行。他们的项目可能包含生成的代码，这些代码不容易修改以表达可空性。这表明能够按类型表达"此代码表达了可空性"的概念将是有用的。

这些考虑对C#来说是新的。我们从未有过对兼容性产生如此广泛影响的语言功能。我期望团队在C# 8最终发布之前在这一方面进行多次迭代。

可空引用类型可能是C# 8中最大的功能，但其他功能在预览版中已经可用。我最喜欢的一个是switch表达式。



> 本章节深入探讨了C# 8中最具变革性的特性之一：可空引用类型。这一特性通过引入编译时的空值安全性分析，旨在显著减少困扰C#开发多年的空引用异常问题。与可空值类型类似，C# 8允许使用`?`后缀明确声明可空的引用类型（如`string?`），而默认的引用类型则被视为不可空。
>
> 这一改变不仅是语法上的扩展，更是思维方式的转变。它要求开发人员明确表达对引用类型空值的预期，使代码意图更加清晰。编译器会基于这些声明进行静态分析，在可能发生空引用解引用时发出警告，从而将运行时的错误提前到编译时发现。
>
> 然而，这一特性也面临诸多挑战。由于与现有代码的兼容性问题，它被设计为可选的、渐进式的功能。迁移现有代码库到可空引用类型是一个迭代过程，可能像"打地鼠"一样，修复一处警告可能在其他地方引发新的警告。泛型中的可空性处理尤为复杂，因为同一泛型类型对值类型和引用类型的处理方式不同。
>
> 尽管存在这些挑战，可空引用类型代表了C#语言向更安全、更健壮方向迈出的重要一步。它不仅仅是语法糖，而是一种通过编译器辅助的契约式编程，有望大幅提高C#代码的质量和可靠性。随着工具链的完善和开发人员经验的积累，这一特性有望成为C#开发中不可或缺的一部分。



# Switch 表达式

switch 语句从 C# 一开始就已存在，在这段时间里，它唯一的改变是在 C# 7 中允许模式匹配。它仍然是一个命令式的控制结构：如果这个 case 匹配，执行这个；如果那个 case 匹配，执行那个。然而，switch 语句的许多用法更具函数式风格，每个 case 计算一个结果：如果这个 case 匹配，结果是 X；如果那个 case 匹配，结果是 Y。这是函数式编程语言中的常见结构，其中许多函数纯粹通过模式匹配来表达。

表达式体成员的引入使这一点显得尤为突出。许多方法可以用单个表达式实现，但如果你想使用 switch/case 结构，就必须使用块体。这通常只是一个不便之处，但仍然是一个摩擦点。

C# 8 引入了 switch 表达式作为 switch 语句的替代方案。这使用了与 switch 语句有些不同的语法，因此值得对两者进行比较。在第12章中，当我介绍模式匹配时，您看到了一个使用 switch 语句计算不同形状周长的例子。这是第12章中使用的代码：

```csharp
static double Perimeter(Shape shape)
{
    switch (shape)
    {
        case null:
            throw new ArgumentNullException(nameof(shape));
        case Rectangle rect:
            return 2 * (rect.Height + rect.Width);
        case Circle circle:
            return 2 * PI * circle.Radius;
        case Triangle triangle:
            return triangle.SideA + triangle.SideB + triangle.SideC;
        default:
            throw new ArgumentException(
                $"Shape type {shape.GetType()} perimeter unknown",
                nameof(shape));
    }
}
```

下面的清单展示了使用 switch 表达式的等效代码，但仍然使用常规的块体方法。

**清单15.9 将 switch 语句转换为 switch 表达式**
```csharp
static double Perimeter(Shape shape)
{
    return shape switch
    {
        null => throw new ArgumentNullException(nameof(shape)),
        Rectangle rect => 2 * (rect.Height + rect.Width),
        Circle circle => 2 * PI * circle.Radius,
        Triangle triangle => triangle.SideA + triangle.SideB + triangle.SideC,
        _ => throw new ArgumentException(
            $"Shape type {shape.GetType()} perimeter unknown",
            nameof(shape))
    };
}
```

这里有很多需要指出的地方，所以我并没有尝试将它们全部作为注释塞进代码中。以下是 switch 语句和 switch 表达式之间的所有区别：

- switch 表达式的引入语法是 `value switch`，而不是 `switch (value)`。
- 在模式和要返回的结果（如果该模式匹配）之间使用粗箭头 `=>`。（在 switch 语句中，使用冒号代替。）
- switch 表达式中根本不使用 `case` 关键字。`=>` 的左侧只是一个模式，可选地带有使用 `when` 关键字的保护子句。
- `=>` 的右侧只是一个表达式。不使用 `return` 关键字，因为每个模式要么产生一个值，要么抛出异常。同样，也从不使用 `break` 语句。
- 模式之间用逗号分隔。如果您将 switch 语句转换为 switch 表达式，这通常意味着将分号改为逗号。
- 没有 default 情况。相反，使用丢弃符 `_`（下划线）来匹配任何尚未匹配的内容。

我的经验主要是编写直接返回 switch 表达式结果的方法，但您也可以像使用任何其他表达式一样使用它。例如，您可以这样写：
```csharp
double circumference = shape switch
{
    // switch 表达式的主体如前所述
};
```

这很好，但正如我之前提到的，switch 表达式最美好的一个方面是将其用于表达式体方法。下面的清单展示了清单15.9如何演变为表达式体方法。

**清单15.10 使用 switch 表达式实现表达式体方法**
```csharp
static double Perimeter(Shape shape) =>
    shape switch
    {
        null => throw new ArgumentNullException(nameof(shape)),
        Rectangle rect => 2 * (rect.Height + rect.Width),
        Circle circle => 2 * PI * circle.Radius,
        Triangle triangle => triangle.SideA + triangle.SideB + triangle.SideC,
        _ => throw new ArgumentException(
            $"Shape type {shape.GetType()} perimeter unknown",
            nameof(shape))
    };
```

您可以按照您喜欢的任何方式格式化它，也许将 `shape switch` 移到第一行，或者将大括号缩进到与方法声明相同的级别。

switch 语句和 switch 表达式之间的一个重要区别是，switch 表达式必须总是产生某个结果（可能是一个异常）。不允许 switch 表达式什么都不做且不产生任何值。您可以使用 `_` 丢弃符来确保这一点，但有可能编写一个非穷举的 switch 表达式——换句话说，一个可能并非总是匹配的表达式。在我使用的预览版中，这会产生一个编译器警告，然后编译器会发出无效的 IL。这可能会变成编译时错误，或者编译器可能会注入代码来抛出异常（可能是 `InvalidOperationException`）以指示代码遇到了未预期的情况。

目前我对 switch 表达式的一个问题是，无法表达多个应计算为相同结果的模式。在 switch 语句中，您可以指定多个 case 标签，但 switch 表达式中尚无等效功能。C# 团队已经意识到了对此的需求，因此希望它在 C# 8 发布之前能够被包含进来。

C# 8 中模式的使用不仅仅通过 switch 表达式得到改进。模式本身也在范围上有所扩展。

# 递归模式匹配

作为提醒，C# 7 中引入的模式如下：
- 类型模式 (`expression is Type t`)
- 常量模式 (`expression is 10`、`expression is null` 等)
- var 模式 (`expression is var v`)

C# 8 将引入递归模式（模式可以嵌套在更大的模式中）以及解构模式。解释递归模式的最简单方法是展示它们的实际应用。我们稍后再讨论解构模式。

## **匹配模式中的属性**

为了在整体模式中使用额外的模式匹配属性，您需要使用大括号，其中包含针对属性的逗号分隔的模式列表。属性模式使用任何正常的模式类型将属性值与嵌套模式进行匹配。作为一个例子，让我们再看一下清单15.10中用于计算矩形、圆形和三角形周长的三种模式：

```csharp
Rectangle rect => 2 * (rect.Height + rect.Width),
Circle circle => 2 * PI * circle.Radius,
Triangle triangle => triangle.SideA + triangle.SideB + triangle.SideC,
```

在每种情况下，您不需要形状本身；只需要它的属性。您可以使用嵌套的 var 模式将这些属性与任何值进行匹配，并为每个您需要的属性提取模式变量。下面的清单展示了带有嵌套模式的完整方法。

**清单15.11 匹配嵌套模式**
```csharp
static double Perimeter(Shape shape) => shape switch
{
    null => throw new ArgumentNullException(nameof(shape)),
    Rectangle { Height: var h, Width: var w } => 2 * (h + w),
    Circle { Radius: var r } => 2 * PI * r,
    Triangle { SideA: var a, SideB: var b, SideC: var c } => a + b + c,
    _ => throw new ArgumentException(
        $"Shape type {shape.GetType()} perimeter unknown", nameof(shape))
};
```

这比之前的代码更清晰吗？我不确定。我将其用作一个从上一个例子自然延续的示例，但我可能很容易坚持使用清单15.10中的代码。稍后您将看到一个更复杂的例子，其中该特性变得更具吸引力，但也更难立即理解。

请注意，虽然这里您已经停止在它们自己的模式变量（之前的 `rect`、`circle` 和 `triangle`）中捕获 `Rectangle`、`Circle` 或 `Triangle`，但这只是因为您不需要对它们进行任何操作。以这种方式引入模式变量仍然是有效的。例如，如果您正在描述形状，您可能有一个模式来描述高度为零的扁平矩形：

```csharp
Rectangle { Height: 0 } rect => $"Flat rectangle of width {rect.Width}"
```

当您有许多属性但只针对其中少数几个属性测试模式时，这很有用。接下来，我们来看看解构模式。

## **解构模式**

您在第12.1节中看到了元组的解构，在第12.2节中看到了通过 `Deconstruct` 方法进行的解构。C# 8 中的模式将扩展为允许在内部使用嵌套模式进行解构。作为一个有些刻意设计的例子，您可能认为将 `Triangle` 解构为其所有三个边是很自然的：

```csharp
public void Deconstruct(out double sideA, out double sideB, out double sideC) =>
    (sideA, sideB, sideC) = (SideA, SideB, SideC);
```

然后，您可以将周长计算简化为解构为三个变量，而不是指定每个属性名称。因此，在我们的 switch 表达式中，您可以将这个 case：
```csharp
Triangle { SideA: var a, SideB: var b, SideC: var c } => a + b + c
```
改为这样：
```csharp
Triangle (var a, var b, var c) => a + b + c
```

同样，这比仅针对类型进行匹配更易读吗？也许吧。随着时间的推移，我怀疑每个开发人员都会围绕模式匹配找出自己的偏好，并且理想情况下，在他们工作的代码库中也会形成约定。

## **从模式中省略类型**

即使在不测试值类型的情况下，能够查看对象内部也使模式变得有用。在这一点上，在模式中指定类型就显得多余了。对于这个例子，让我们回到用于可空引用类型的客户和地址示例。您将回到第一个数据模型：全部可变，全部可为空：

```csharp
public class Customer
{
    public string Name { get; set; }
    public Address Address { get; set; }
}

public class Address
{
    public string Country { get; set; }
}
```

现在假设您希望根据客户地址中的国家/地区以不同的方式问候客户。您的输入可能是 `Customer` 类型，因此您不希望必须在模式中重复这一点。当您在模式中匹配客户的 `Address` 时，该属性将始终是 `Address` 类型，因此您也不需要指定该类型。

下面的清单展示了匹配不同类型客户的多种模式。它还演示了 `{ }` 模式，这是属性模式的一个特例，它没有任何要匹配的属性。该模式匹配任何非空值。

**清单15.12 简洁地匹配客户与多种模式**
```csharp
static void Greet(Customer customer)
{
    string greeting = customer switch
    {
        { Address: { Country: "UK" } } => "Welcome, customer from the United Kingdom!", // 匹配国家为 UK
        { Address: { Country: "USA" } } => "Welcome, customer from the USA!", // 匹配国家为 USA
        { Address: { Country: string country } } => $"Welcome, customer from {country}!", // 匹配任何国家，但它必须存在
        { Address: { } } => "Welcome, customer whose address has no country!", // 匹配任何地址
        { } => "Welcome, customer of an unknown address!", // 匹配任何客户，即使地址为空
        _ => "Welcome, nullness my old friend!" // 匹配任何内容，甚至是空客户引用
    };
    Console.WriteLine(greeting);
}
```

这里的顺序很重要。例如，一个地址国家为 USA 的客户可以匹配除第一个模式之外的所有模式。您可以改为使用更具选择性的模式（例如，使用常量空模式来匹配 `Address` 属性值为空的客户），但依赖顺序更简单。

C# 8 中对模式匹配的增强将允许它们在更多当前需要使用 if 语句的情况下使用。Switch 表达式也增加了这种灵活性。我预计越来越多的代码将使用模式编写。与往常一样，重要的是避免过度；并非所有代码使用模式编写都会比使用我们之前拥有的控制结构更简单。尽管如此，C# 演变的这个领域确实有很大的潜力。我们的下一个特性实际上是由两个新的框架类型启用的一对特性。



> Switch表达式代表了C#语言向更声明式、函数式编程风格的重大迈进。它将传统的命令式switch语句转化为表达式，强制每个分支必须产生一个值或抛出异常，这带来了编译时的穷举性检查，增强了代码的健壮性。
>
> 递归模式匹配和解构模式的引入，使得代码能够更自然地表达复杂的数据形状检查与提取逻辑。属性模式允许深入检查对象内部状态，而无需显式类型转换和空值检查，显著提升了代码的简洁性和可读性。
>
> 这些特性共同推动C#成为一种更强大的数据查询和转换语言，特别适合于处理具有复杂层次结构的数据（如JSON、XML或领域模型）。然而，开发者需要谨慎使用，避免因过度嵌套的模式而导致代码可读性下降。对于简单的类型判断，传统的`is`操作符或if语句可能仍然是更清晰的选择。新模式语法需要团队内部建立一致的编码规范。





# 索引和范围

与可空引用类型和改进的模式处理相比，索引和范围即使结合起来也感觉像是一个小特性。但我怀疑随着时间的推移，我们会想知道为什么花了这么长时间才拥有它们。在深入研究细节之前，下面的清单提供了一个小小的体验。

**清单15.13 使用范围从字符串中修剪第一个和最后一个字符**
```csharp
string quotedText = "'This text was in quotes'";
Console.WriteLine(quotedText);
Console.WriteLine(quotedText.Substring(1..^1)); // 使用范围字面量获取字符串的子字符串
```

输出如下：
```
'This text was in quotes'
This text was in quotes
```

高亮的表达式 `1..^1` 是有趣的部分。要理解这段代码，您需要了解两种新类型。

**Index 和 Range 类型及字面量**
这个想法很简单。`Index` 和 `Range` 是框架中将提供的两个结构体，但目前需要在您自己的代码中定义：

- `Index` 是一个整数，表示可索引对象的起始或结束位置。索引值永远不会是负数。
- `Range` 是一对索引：一个用于范围的起始位置，一个用于范围的结束位置。

然后有三种重要的语法：
- 从 `int` 到"从起始开始"的 `Index` 的常规隐式转换。
- 一个新的单目运算符（`^`），可以与 `int` 一起使用以创建"从结束开始"的 `Index`。这里的值 `0` 表示结束后的那个元素，值 `1` 表示最后一个元素。
- 一个新的二元类似运算符（`..`），带有可选的起始和结束操作数，用于创建 `Range`。

`..` 运算符是"二元类似"的，因为可以有零个、一个或两个操作数。下面的清单展示了所有这些的示例。您没有将索引或范围应用于任何东西；只是创建这些值。

**清单15.14 索引和范围字面量**
```csharp
Index start = 2;
Index end = ^2;
Range all = ..;
Range startOnly = start..;
Range endOnly = ..end;
Range startAndEnd = start..end;
Range implicitIndexes = 1..5;
```

需要注意的一点是，范围的起始点和结束点可以是任何索引。例如，您可以有一个 `^5..10` 的范围，表示从末尾开始的第五个元素到从起始开始的第十个元素。这通常不常见，但是有效的。

这是对索引和范围的直接语言支持的总和。当它们也有框架支持时，它们才会变得有用。

## 应用索引和范围

本节中的所有示例都需要 C# 8 预览版支持的扩展方法和扩展运算符。确切的 API 可能会改变，预览版中提供的扩展仅适用于有限的类型集；这足以演示其好处。在清单15.13中，我展示了 `Substring` 方法如何与 `Range` 一起使用。索引和范围都将被应用，并且最常用于表示某种形式序列的类型，例如：

- 数组
- Span
- 字符串（作为 UTF-16 代码单元的序列）

这些都支持两种操作：
- 检索单个元素
- 创建一个切片来表示序列的一部分

单元素检索操作已经有一个使用接受 `int` 参数的索引器的通用表示，但这使得以统一的方式检索最后一个元素变得困难。`Index` 类型通过其"从起始开始"或"从结束开始"的特性解决了这个问题。切片操作以前根据所涉及的类型采用不同的形式。例如，`Span<T>` 有一个 `Slice` 方法，而 `String` 有一个 `Substring` 方法。

通过添加接受 `Index` 和 `Range` 值的索引器重载，您可以使用一致且方便的语法在所有相关类型上执行这两种操作。下面的清单展示了类似的调用在字符串和 `Span<int>` 上的工作。

**清单15.15 在字符串和 span 中使用索引和范围的索引器重载**
```csharp
string text = "hello world";
Console.WriteLine(text[2]);        // 通过从起始开始的索引访问单个字符
Console.WriteLine(text[^3]);       // 通过从结束开始的索引访问单个字符
Console.WriteLine(text[2..7]);     // 使用范围获取子字符串

Span<int> span = stackalloc int[] { 5, 2, 7, 8, 2, 4, 3 };
Console.WriteLine(span[2]);        // 通过从起始开始的索引访问单个元素
Console.WriteLine(span[^3]);       // 通过从结束开始的索引访问单个元素
Span<int> slice = span[2..7];      // 使用范围创建切片
Console.WriteLine(string.Join(", ", slice.ToArray()));
```

输出如下：
```
l
r
llo w
7
2
7, 8, 2, 4, 3
```

接受 `Range` 的字符串和 span 索引器都将范围的上限视为独占的：范围 `[2..7]` 返回索引为 2、3、4、5 和 6 的元素。

在清单15.15中，范围包括起始和结束索引，并且两个索引值都是从起始计算的。只要索引对其应用的序列有效，您可以将任何范围与索引器一起使用。例如，在清单15.15的代码中使用 `text[^5..]` 将返回 `world` 作为 `text` 的最后五个字符。

同样，您可以编写 `text[^10..5]`，它将返回 `ello`。在长度为 11（`hello world`）的字符串上下文中，索引 `^10` 等价于索引 `1`，所以 `text[^10..5]`（在这种情况下，它确实取决于 `text` 的长度）等价于 `text[1..5]`，返回第一个字符之后的四个字符。接下来，我们来看看对异步性的更多语言支持。

# 更多异步集成

当 `async/await` 在 C# 5 中被引入时，它彻底改变了许多 C# 开发人员的异步编程。但到目前为止，一些语言特性仍然是同步的，使得很难完全投入异步编程。在本节中，我们将探讨以下内容：

- 异步处置（async disposal）
- 异步迭代（`foreach`）
- 异步迭代器（`yield return`）

这些特性需要框架支持以及语言支持。例如，编译器通过在不同的线程上执行同步代码来近似异步是不合适的。让我们从异步处置开始，这是三个特性中最简单的一个。

**使用 using await 进行异步资源处置**
具有单个 `Dispose` 方法的 `IDisposable` 接口自然是同步的。如果该方法需要执行 I/O 操作，例如刷新流，那么它可能会阻塞，并带来所有正常的问题。

将引入一个新接口用于支持异步处置的类：
```csharp
public interface IAsyncDisposable
{
    Task DisposeAsync();
}
```

没有要求实现 `IAsyncDisposable` 的类型也实现 `IDisposable`，尽管我怀疑许多类型会这样做。

然后有相应语言支持，形式为 `using await` 语句，它的工作方式符合您的预期，自动调用 `DisposeAsync` 并等待产生的任务。下面的清单展示了实现接口然后使用它的示例。

**清单15.16 实现 IAsyncDisposal 并使用 using await 调用它**
```csharp
class AsyncResource : IAsyncDisposable
{
    public async Task DisposeAsync()
    {
        Console.WriteLine("Disposing asynchronously...");
        await Task.Delay(2000);
        Console.WriteLine("... done");
    }
    
    public async Task PerformWorkAsync()
    {
        Console.WriteLine("Performing work asynchronously...");
        await Task.Delay(2000);
        Console.WriteLine("... done");
    }
}

async static Task Main()
{
    using await (var resource = new AsyncResource())
    {
        await resource.PerformWorkAsync();
    }
    Console.WriteLine("After the using await statement");
}
```

输出显示资源处置过程：
```
Performing work asynchronously...
... done
Disposing asynchronously...
... done
After the using await statement
```

这很简单，但它隐藏了两个需要解决的复杂性方面：

- 库通常使用 `ConfigureAwait(false)` 来等待任务。应用程序通常不使用这个来等待任务。如果编译器自动执行等待，用户如何配置这个？
- 处置过程中有可用的取消会很自然。那如何适应接口和调用站点？

C# 团队意识到了这两点，我期望它们在发布前以某种形式得到解决。同样的问题也出现在 C# 8 的其他异步特性中，我希望它们都能以类似的方式解决。现在让我们看看下一个特性：使用 `foreach` 的异步迭代。

**使用 foreach await 进行异步迭代**
剧透警告：在本节中，我们到达语言特性之前有相当多的文字。这是为了正确解释它，但结果是像这样的代码将是有效的，其中 `asyncSequence` 需要异步工作来检索项目：
```csharp
foreach await (var item in asyncSequence)
{
    // 使用 item
}
```

为异步迭代引入的接口并不像处置的接口那样直接。有两个接口，在某种程度上反映了 `IEnumerable<T>` 和 `IEnumerator<T>`，但并不那么明显：
```csharp
public interface IAsyncEnumerable<out T>
{
    IAsyncEnumerator<T> GetAsyncEnumerator();
}

public interface IAsyncEnumerator<out T>
{
    Task<bool> WaitForNextAsync();
    T TryGetNext(out bool success);
}
```

`IAsyncEnumerable<T>` 可能比您预期的更接近 `IEnumerable<T>`；它内部没有任何异步的内容。它没有 `GetEnumerator()`，而是有 `GetAsyncEnumerator()`，并且它返回一个 `IAsyncEnumerator<T>`，但它是同步执行的。对于某些实现来说，这可能会有问题，但我期望这是大多数异步序列的自然方法。任何希望将异步操作作为设置的一部分执行的实现可能需要将该工作推迟到调用者开始迭代结果时。

`IAsyncEnumerator<T>` 接口与 `IEnumerator<T>` 相差更远，并且反映了现实世界实现中的一个常见模式。异步性通常在有 I/O 参与时使用，例如通过网络检索结果。这通常自然导致以块的形式检索序列；您可能执行一个查询并一起检索前 10 个结果，然后接下来的 7 个，然后被告知这是完整的结果集。

当您在已缓冲的一组结果中进行迭代时，不需要异步性。尽管异步性相当高效，但它并非完全免费，因此如果可以避免，还是值得的。相反，您可以同步迭代，只要有一种方法可以确定何时到达当前结果集的末尾。此时，您可以异步获取下一个结果集，并再次同步遍历它。

`IAsyncEnumerator<T>` 接口通过其两个方法暴露了这个模式：
- `WaitForNextAsync` 是异步的，返回一个任务，指示是否检索到更多结果，或者是否已到达序列的末尾。
- `TryGetNext` 是同步的，返回下一个项目。`out` 参数用于指示是否有下一个项目要返回。当它为 `false` 时，并不意味着您一定到达了序列的末尾；只是意味着您需要再次调用 `WaitForNextAsync`。

这一切听起来可能很复杂，但好消息是您可能不需要自己执行任何这些操作；新的 `foreach await` 语句为您处理所有这一切。

让我们看一个例子，它大量借鉴了我在 Google Cloud Platform API 方面的经验。许多 API 都有列表操作，例如列出地址簿中的联系人或集群中的虚拟机。可能有太多结果无法在单个 RPC 响应中返回，因此我们有一个基于页面的模式：每个响应包含一个"下一页令牌"，客户端在后续请求中提供该令牌以检索更多数据。对于第一个请求，客户端不提供页面令牌，而最终的响应不包含页面令牌。API 的简化视图可能如下所示。

**清单15.17 用于列出城市的简化基于 RPC 的服务**
```csharp
public interface IGeoService
{
    Task<ListCitiesResponse> ListCitiesAsync(ListCitiesRequest request);
}

public class ListCitiesRequest
{
    public string PageToken { get; }
    public ListCitiesRequest(string pageToken) => PageToken = pageToken;
}

public class ListCitiesResponse
{
    public string NextPageToken { get; }
    public List<string> Cities { get; }
    public ListCitiesResponse(string nextPageToken, List<string> cities) =>
        (NextPageToken, Cities) = (nextPageToken, cities);
}
```

这直接使用起来很麻烦，但可以很容易地包装在一个暴露此 API 的客户端中，如下一个清单所示。

**清单15.18 围绕 RPC 服务的包装器，提供更简单的 API**
```csharp
public class GeoClient
{
    public GeoClient(IGeoService service) { ... } // 使用 RPC 服务构造 GeoClient
    public IAsyncEnumerable<string> ListCitiesAsync() { ... } // 提供城市的简单异步序列
}
```

有了 `GeoClient`，您最终可以使用 `foreach await`，如以下清单所示。

**清单15.19 对 GeoClient 使用 foreach await**
```csharp
var client = new GeoClient(service);
foreach await (var city in client.ListCitiesAsync())
{
    Console.WriteLine(city);
}
```

这里的最终代码比为了设置示例而必须展示的所有代码简单得多，而且甚至还没有查看 `GeoClient` 的实现。但这是一件好事；它显示了该特性的好处。您已经以简单高效的方式使用了 `foreach await` 来消费 `IGeoService` 和 `IAsyncEnumerable<T>` 中相对复杂的定义。

**注意**：可下载的源代码包含一个完整示例，其中有一个内存中的假服务实现。

您可能感到惊讶的一点是 `IAsyncEnumerator<T>` 没有实现 `IAsyncDisposable`。这可能会在发布前改变，但即使没有，我期望编译器在运行时发现枚举器实现了 `IAsyncDisposable` 时会处置它。

就像同步的 `foreach` 语句一样，`foreach await` 不要求实现 `IAsyncEnumerable<T>` 和 `IAsyncEnumerator<T>` 接口。它将基于模式，因此任何提供 `GetAsyncEnumerator()` 方法的类型，该方法返回一个类型，而该类型又提供适当的 `WaitForNextAsync` 和 `TryGetNext` 方法，都将得到支持。这可能允许一些优化，但我期望接口在大多数情况下被使用。

到目前为止，您已经了解了如何消费异步序列。那么如何生成它们呢？

## 异步迭代器

C# 2 引入了带有 `yield return` 和 `yield break` 语句的迭代器，以便于编写返回 `IEnumerable<T>` 或 `IEnumerator<T>` 的方法。C# 8 将为异步序列提供相同的特性。该特性在预览版中尚不可用，但以下清单展示了我期望它的工作方式。

**清单15.20 使用迭代器实现 ListCitiesAsync**
```csharp
public async IAsyncEnumerable<string> ListCitiesAsync()
{
    string pageToken = null;
    do
    {
        var request = new ListCitiesRequest(pageToken);
        var response = await service.ListCitiesAsync(request);
        foreach (var city in response.Cities)
        {
            yield return city;
        }
        pageToken = response.NextPageToken;
    } while (pageToken != null);
}
```

异步迭代器方法与 `IAsyncEnumerator<T>` 接口之间的映射，及其异步和同步部分的混合，实现起来会很复杂。每当您继续执行异步方法中的代码时，它可以通过几种方式完成该特定调用：

- 它可以等待一个未完成的异步操作。
- 它可以到达一个 `yield return` 语句。
- 它可以到达一个 `yield break` 语句。
- 它可以到达方法的末尾。
- 它可以抛出异常。

这些情况将如何被处理，取决于调用者是在执行 `WaitForNextAsync()` 还是 `TryGetNext()`。为了使这高效，生成的代码应该有效地在同步模式（如果您正在产生值而没有中间的等待）和异步模式（如果您正在等待一个异步操作）之间切换。我可以大致设想这如何实现，但我很高兴我不必去实现它。

C# 8 预览版中还有其他尚不可用的特性。我们将更简要地看看这些。

# 尚未在预览版中的特性

如果 C# 8 最终只有我目前为止列出的特性，那仍然会是一个大事件。在某些方面，我希望我们可以有一个只包含可空引用类型的版本，等待一年左右让大多数代码库更新到它，然后再继续增加更多特性。但 C# 8 很可能包含比我目前展示的更多特性。

本节讨论我认为最有可能被纳入 C# 8 的特性。甚至有更多的特性已经被 C# 团队的成员或外部开发人员提出。C# 团队使用 GitHub 来跟踪语言提案，这使得了解进展并贡献自己的力量变得容易；请参见 https://github.com/dotnet/csharplang。我们将从一个受 Java 启发的特性开始。

## 默认接口方法

尽管 C# 为 LINQ 引入了扩展方法，但 Java 采取了不同的方法来支持其流（streams），这在许多方面与 LINQ 的使用场景相似。在 Java 8 中，Oracle 引入了 Java 接口中的默认方法：接口可以声明一个方法及其默认实现，然后可以在具体实现中重写。默认实现不能声明任何状态（例如字段）；它必须根据接口的其他成员来表达。

这两个特性在某些方面类似：它们都允许表达逻辑，以便接口的使用者可以调用一个方法，而无需每个接口实现直接了解或实现它。每种方法都有其优缺点：

- 扩展方法可以由任何人引入，而不仅仅是接口的作者。您不能向您无法控制的接口添加默认方法。（当然，扩展方法也可以应用于类和结构体。）
- 默认方法可以被实现类重写，通常是为了优化。扩展方法不能被重写；它们只是带有语法糖的静态方法，使调用它们看起来更像是常规的实例方法。

使用 LINQ 的 `Enumerable.Count()` 方法作为例子，可以很容易地理解第二点。默认情况下，它通过调用 `GetEnumerator()` 然后计算在该枚举器上调用 `MoveNext()` 返回 `true` 的次数来计算序列中的元素数量。

许多 `IEnumerable<T>` 的实现有更高效的方式来确定元素的数量。`Enumerable.Count()` 专门针对其中一些进行了优化，例如 `ICollection` 和 `ICollection<T>` 的实现。但是，一个集合如果不希望实现这些接口中的任何一个，但希望廉价地提供 `Count` 呢？它陷入了困境；它无法与 `Enumerable.Count()` 通信，告知它可以自己更高效地实现 LINQ 的那部分。然而，如果 `Count()` 是 `IEnumerable<T>` 中的一个带有默认实现的方法，我们的新集合就可以重写该方法。

以下是一个使用 C# 8 默认接口方法声明 `IEnumerable<T>` 的示例：
```csharp
public interface IEnumerable<T>
{
    IEnumerator<T> GetEnumerator();
    
    int Count()
    {
        using (var iterator = GetEnumerator())
        {
            int count = 0;
            while (iterator.MoveNext())
            {
                count++;
            }
            return count;
        }
    }
}
```

默认接口方法还允许接口以更加版本友好的方式随着时间的推移进行扩展。可以添加带有默认实现的新方法，要么使用现有成员实现新功能，要么可能抛出 `NotSupportedException`。这样，旧的实现仍然可以构建，即使新方法不能被可靠地调用。版本控制是一个棘手的主题，但工具箱中有另一个选择是受欢迎的。在我维护的代码中，这在许多情况下会使事情变得更简单。

默认接口方法被证明是一个有争议的特性。它们需要 CLR 支持，这使得该特性在完全投入之前更难进行实验。如果该特性被包含，观察其采用率将会很有趣。在支持它的运行时版本被广泛采用之前，它可能仍然很少被使用。接下来，我们将看一个已经讨论甚至原型制作了很长时间的特性。

## 记录类型

记录类型的前身是一个名为主构造函数（primary constructors）的特性，最初打算出现在 C# 6 中。语言团队对原始设计中的一些粗糙之处不满意，所以他们决定推迟引入，直到可以改进为止。

记录类型旨在使创建具有给定属性集的不可变类或结构体变得容易。我倾向于从匿名类型开始思考它们，但添加了各种特性。它们可以极其简单地声明。例如，这里是一个完整的类声明：
```csharp
public class Point(int X, int Y, int Z);
```

这为您生成了一堆成员，尽管您仍然可以引入自己的行为。生成的成员包括构造函数、属性、相等性方法、用于解构的 `Deconstruct` 方法，以及一个像这样的 `With` 方法：
```csharp
public Point With(int X = this.X, int Y = this.Y, int Z = this.Z) =>
    new Point(X, Y, Z);
```

目前，对于可选参数的默认值来说，这不是有效的语法，并且不清楚是否允许显式编写该代码，但它至少展示了该方法行为的意图。

`With` 方法旨在与 `with` 表达式形式的新语法互操作。其想法是，方法和语法都使创建不可变类型的新实例变得容易，该实例与现有实例相同，但一个或多个属性发生了变化。`WithFoo` 方法在不可变类型中已经很常见（其中 `Foo` 是类型中的属性名称），但它们通常一次只处理一个属性。例如，对于一个具有 `X`、`Y` 和 `Z` 属性的不可变 `Point` 类，您可能使用以下代码创建一个新点，该点具有与前一个点相同的 `Z` 值，但有新的 `X` 和 `Y` 值：
```csharp
var newPoint = oldPoint.WithX(10).WithY(20);
```

每个 `WithFoo` 方法调用一个构造函数，传入除方法中命名的属性之外的所有现有属性，其中使用参数中指定的新值。这些方法变得繁琐，并且也有性能影响：要"更改" N 个属性，您需要进行 N 次方法调用，每次都会创建一个新对象。

记录类型的 `With` 方法不同：对于类型的每个属性，它都有一个参数，如果该参数未被指定，则具有指示应从当前对象获取值的新语法作为默认参数值。例如，考虑我们的 `Point` 类型中的 `With` 方法。您可以直接调用它：
```csharp
var newPoint = oldPoint.With(X: 10, Y: 20);
```

或者使用新的 `with` 表达式语法，它看起来更像对象初始化器：
```csharp
var newPoint = oldPoint with { X = 10, Y = 20 };
```

两者将编译为相同的 IL。这样，只构造了一个新对象。

这只是一个简单的例子。当您有一个复杂的类型，并且只想修改一个叶子节点时，它会变得更加棘手。例如，如果您有一个带有 `Address` 属性的 `Contact` 类型，您可能希望创建一个与旧联系人相同但 `Address` 属性的一部分不同的新联系人。这在 C# 8 中可能仍然很棘手，但 `with` 表达式语法可能会随着时间的推移而增强，使其更简单，就像模式匹配的语法已经扩展了一样。

我对这里的可能性感到兴奋。长期以来，在 C# 中创建和使用不可变类型一直是一件痛苦的事情。虽然 C# 7 元组填补了匿名类型留下的一个空白，但记录类型填补了另一个空白。我一直很喜欢匿名类型，因为编译器为您生成了相等性、构造函数和属性代码。遗憾的是，我们无法为它们命名或稍后添加更多功能。记录类型修复了所有这些问题，甚至更多。最后，我想强调一些需要更多创新思维的特性。

## 简要介绍更多特性

尽管一些次要特性更有可能进入 C# 8，但它们不如我在这里讨论的那些有趣。请记住，您可以随时查看 GitHub 以了解更多可能被包含的特性及其最新状态。

**类型类（又称概念、形状或结构泛型约束）**
尽管泛型在许多情况下都很出色，但它们也有局限性。有些数据类型的"形状"无法用泛型表达，例如运算符和构造函数。虽然您可以要求类型参数具有无参数构造函数，但不能要求它具有特定参数列表的构造函数。

此外，有时类型可以在某种有用的方式上具有相同的形状，但不实现任何公共接口或除了 `System.Object` 之外有任何公共基类。类型类将是一种新的类型来解决这些问题。它们有点像接口，但实现类不需要知道它们。您将能够通过类型类来约束泛型类型参数。

这有可能很强大，但也有些令人困惑；我自己对此持两种看法。为了高效执行，它可能需要运行时更改。C# 开发人员（或至少是我）可能需要一段时间才能弄清楚它在什么情况下有用，在什么情况下只是令人困惑。在语言演变的这个阶段添加一种全新的类型感觉像是迈出了一大步。尽管有所有这些注意事项，这个特性肯定填补了一个空白：在您需要此功能的地方，当前的工具没有提供任何干净的解决方案。

**扩展一切**
在撰写本文时，这在 GitHub 上的里程碑是 X.0，但如果它被提升优先级，我也不会过于惊讶。这个名字很好地解释了这个特性：扩展方法的概念将应用于其他成员类型，例如属性、构造函数和运算符。它还可能允许引入静态扩展成员——那些看起来像是扩展类型上的静态方法的成员。（例如，您可以在 `StringExtensions` 中编写一个方法，该方法可以被称为 `string.IsNullOrTabs`，作为 `string.IsNullOrWhiteSpace` 的更具体版本。）

用于扩展方法的语法并不适合其他成员类型，因此可能会使用一种全新的语法。这可能是一个扩展类型，仅用于在特定扩展类型上创建多个扩展成员。

扩展类型仍然不能引入新状态。任何扩展属性都可能呈现现有属性的不同视图。例如，您可以在 `DateTime` 上有一个扩展属性，称为 `FinancialQuarter`，它知道您公司的财务报告日期，并使用现有的 `Year`/`Month`/`Day` 属性来计算适当的季度。

**目标类型化的 new**
当涉及长类型名时，使用 `var` 进行隐式类型化对于减少混乱很有用。但它对字段没有帮助，因为字段不能被隐式类型化。我们最终还是会得到这样的代码：
```csharp
Dictionary<string, List<DateTime>> entryTimesByName =
    new Dictionary<string, List<DateTime>>();
```

目标类型化的 new 特性不会影响您可以使用 `var` 的地方。相反，它将缩短声明的右侧：
```csharp
Dictionary<string, List<DateTime>> entryTimesByName = new();
```

每当编译器能够判断出您在调用构造函数时可能指的是哪种类型时，您就可以完全省略类型名称。这引入了与成员调用相关的有趣复杂性。例如，`Method(new())` 将从方法参数中获取目标类型，这很好，直到 `Method` 是泛型或重载的。

我对这个特性提案既爱又恨，大致程度相当。如果过度使用，它肯定会使代码难以阅读，但几乎任何特性都可能被误用。另一方面，我渴望消除长字段初始化的重复。

我期望这比默认接口方法更具争议性。我们将拭目以待，您也可以参与讨论。

# 参与其中

C# 设计过程比以往任何时候都更加开放。尽管在 Microsoft 办公室的语言设计会议（LDMs）中有很多幕后工作，但也有足够的社区参与空间。GitHub 仓库 https://github.com/dotnet/csharplang 是一个起点。它包含 LDMs 的笔记、提案、讨论和规范。欢迎您参与以下任何级别的活动：

- 尝试预览版，看看新特性与您现有代码的契合程度如何
- 讨论当前提出的特性
- 提出新特性
- 在 Roslyn 中为新特性制作原型
- 帮助为新特性起草规范语言
- 发现现有规范中的错误（确实会发生！）

您可能觉得等待具有完整文档和精良实现的正式发布是更好的时间利用方式。这也完全可以。您可以随时涉足，哪怕只是为了查看给定里程碑的拟议特性集。

这种开放的设计过程相对较新，我期望随着时间的推移它会得到改进。如果团队回到更封闭的过程，我会感到惊讶。尽管这样的社区参与在时间上是昂贵的，但在确保新特性确实是开发人员真正需要的方面，它有巨大的好处。

### 结论

本章中有比代码多得多的文字，主要是因为我不想呈现太多在 C# 8 发布时可能错误的代码。我怀疑我所描述的所有特性都会出现在 C# 8 中，但我认为至少其中一些可能会出现。如果可空引用类型或与模式相关的特性没有进入 C# 8，我会感到惊讶。

那之后呢？嗯，可能是 C# 8 线路上的小版本，然后继续到 C# 9。C# 9 的一些特性可能已经在 GitHub 上作为提案出现，但我怀疑会有一些尚未被讨论过的特性。我期望 C# 继续发展，以满足开发人员在计算环境变化中的需求

> C# 8 引入了一系列重要特性，这些特性共同推动了语言的现代化进程：
>
> **索引和范围** 提供了简洁直观的序列操作语法，使开发者能够以更自然的方式表达切片操作。这不仅减少了样板代码，还提高了代码的可读性，特别是在处理字符串和数组时。该特性反映了现代编程语言对序列处理的重视。
>
> **异步增强** 标志着 C# 异步编程的进一步完善。异步处置、异步迭代和异步迭代器的引入，使开发者能够构建完全异步的数据处理管道，避免在混合同步和异步代码时出现的各种问题。这对于构建高性能、可伸缩的应用程序至关重要。
>
> **默认接口方法** 是 C# 对接口演化的重大改进，为版本控制和库设计提供了新的可能性。尽管这一特性与扩展方法在某些方面重叠，但它在允许接口提供默认实现方面具有独特优势，特别是在性能优化和向后兼容性方面。
>
> **记录类型** 旨在简化不可变数据类型的创建，这是函数式编程和现代并发编程中的重要概念。通过自动生成相等性比较、解构和克隆方法，记录类型大大减少了创建值类型所需的样板代码，有望推动 C# 中不可变数据模型的更广泛采用。
>
> **类型类** 和 **扩展一切** 等前瞻性特性展示了 C# 继续演化的方向。虽然这些特性可能不会全部进入 C# 8，但它们反映了语言设计者正在探索如何使 C# 更灵活、更具表达力，同时保持其强类型和面向对象的本质。
>
> 总体而言，C# 8 延续了近年来语言的快速演进趋势，在保持向后兼容性的同时，引入了许多受现代编程范式启发的特性。这些改进使 C# 能够更好地适应云计算、大数据和分布式系统等新兴领域的开发需求。

