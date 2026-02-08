---
weight: 9
title: "字符串属性"
---



本章涵盖：
- 使用内插字符串字面量实现更易读的格式化
- 使用FormattableString进行本地化和自定义格式化
- 使用nameof进行易于重构的引用

每个人都知道如何使用字符串。即使字符串不是你学习的第一个.NET数据类型，也很可能是第二个。在.NET的发展历程中，string类本身并没有太大变化，自C#1以来，C#语言也没有引入太多面向字符串的特性。然而，C#6通过另一种字符串字面量和一个新运算符改变了这一点。你将在本章详细研究这两者，但值得记住的是，字符串本身根本没有改变。这两个特性只是提供了获取字符串的新方法，仅此而已。

就像你在第8章看到的特性一样，字符串内插并不能让你做以前做不到的事情；它只是让你更清晰、更简洁地完成。这并不是贬低这个特性的重要性。任何能让你更快地编写更清晰的代码，并且之后能更快地阅读的东西，都会提高你的生产力。

nameof运算符是C#6中真正的新功能，但它是一个相当小的特性。它所做的只是允许你获取代码中已经出现的标识符，但在执行时作为字符串。它不会像LINQ或async/await那样改变你的世界，但它有助于避免拼写错误，并允许重构工具为你做更多工作。在我向你展示任何新内容之前，让我们回顾一下你已经知道的内容。

# .NET中字符串格式化回顾

你几乎肯定知道本节中的所有内容。你可能已经使用字符串很多年了，而且几乎可以肯定，从你使用C#开始就一直在用。尽管如此，为了理解C#6中的内插字符串字面量特性是如何工作的，最好把这些知识牢记在心。请耐心听我讲解.NET处理字符串格式化的基础知识。我保证我们很快就会讲到新东西。

## 简单字符串格式化

如果你像我一样，你喜欢通过编写微不足道的控制台应用程序来试验新语言，这些应用程序没有任何用处，但能给你信心和坚实的基础，以便进行更令人印象深刻的壮举。因此，我不记得我已经用多少种语言实现了以下功能——询问用户的名字，然后向该用户问好：

```csharp
Console.Write("What's your name? ");
string name = Console.ReadLine();
Console.WriteLine("Hello, {0}!", name);
```

最后一行是本章最相关的一行。它使用了Console.WriteLine的一个重载，该重载接受一个包含格式项的组合格式字符串，然后是要替换这些格式项的参数。前面的示例有一个格式项{0}，它被name变量的值替换。格式项中的数字指定要填充空缺的参数索引（其中0表示第一个值，1表示第二个，依此类推）。

这种模式用于各种API中。最明显的例子是string类中的静态Format方法，它只是适当地格式化字符串。到目前为止，一切都很好。让我们做一些更复杂的事情。

## 使用格式字符串进行自定义格式化

澄清一下，我包含这个小节的动机既是为了我未来的自己，也是为了你，亲爱的读者。如果MSDN显示我访问任何给定页面的次数，那么关于组合格式字符串的页面的访问次数将会是惊人的。我总是忘记具体内容和使用的术语，我想如果我把这些信息放在这里，我可能会开始更好地记住它。我希望你同样觉得它有用。

组合格式字符串中的每个格式项指定要格式化的参数的索引，但它还可以指定以下选项来格式化值：
- 对齐方式：指定最小宽度以及值应左对齐还是右对齐。右对齐用正值表示；左对齐用负值表示。
- 值的格式字符串。这可能最常用于日期和时间值或数字。例如，要根据ISO-8601格式化日期，可以使用格式字符串yyyy-MM-dd。要将数字格式化为货币值，可以使用格式字符串C。格式字符串的含义取决于要格式化的值的类型，因此您需要查阅相关文档以选择正确的格式字符串。

图9.1显示了可用于显示价格的组合格式字符串的所有部分。

![image-20260126214203964](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260126214203989.png)



对齐方式和格式字符串是独立可选的；您可以指定其中一个、两个或不指定。格式项中的逗号表示对齐方式，冒号表示格式字符串。如果您需要在格式字符串中使用逗号，那没问题；没有第二个对齐值的概念。

作为一个稍后扩展的具体示例，让我们在图9.1的代码中使用更广泛的上下文，显示不同长度的结果以演示对齐的要点。清单9.1显示价格（$95.25）、小费（$19.05）和总计（$114.30），将标签左对齐，数值右对齐。

在默认使用美国英语文化设置的计算机上，输出如下：
```
价格：    $95.25
小费：    $19.05
总计：   $114.30
```

为了使数值右对齐（或者从另一个角度看，左边用空格填充），代码使用了对齐值9。如果你有一个巨大的账单（例如，一百万美元），对齐将不起作用；它只指定最小宽度。如果你想编写右对齐所有可能值的代码，你必须先计算出最大值会有多宽。这是相当令人不快的代码，恐怕C#6中没有任何东西能使其变得更容易。

**清单9.1：以对齐方式显示价格、小费和总计**
```csharp
decimal price = 95.25m;
decimal tip = price * 0.2m;  // 20% 小费
Console.WriteLine("价格: {0,9:C}", price);
Console.WriteLine("小费:    {0,9:C}", tip);
Console.WriteLine("总计: {0,9:C}", price + tip);
```

当我在美国英语文化的计算机上显示清单9.1的输出时，关于文化的部分很重要。在使用英国英语文化的计算机上，代码将使用£符号。在法国文化的计算机上，十进制分隔符将变为逗号，货币符号将变为欧元符号，并且该符号将位于字符串的末尾而不是开头！这就是本地化的乐趣，接下来你将看到。

## 本地化

广义上说，本地化是确保您的代码为所有用户做正确的事情，无论他们身处世界何处。任何声称本地化简单的人要么比我有经验得多，要么做得不够多，不知道它有多痛苦。考虑到地球基本上是圆的，它似乎确实有很多棘手的角落情况需要处理。本地化在所有编程语言中都是痛苦的，但每种语言都有稍微不同的解决问题的方式。

**注意**：虽然我在本节中使用术语"本地化"，但其他人可能更喜欢术语"全球化"。微软使用这两个术语的方式与其他行业机构略有不同，而且差异有些微妙。专家们，请原谅我这里的粗略说明；大局比术语的细节更重要，仅此一次。

在.NET中，为了本地化目的，最重要的类型是CultureInfo。它负责一种语言（如英语）的文化偏好，或特定位置的语言（如加拿大的法语），或特定位置的语言的特定变体（如台湾使用的简体中文）。这些文化偏好包括各种翻译（例如，用于星期几的单词），并指示文本如何排序以及数字如何格式化（是使用句点还是逗号作为十进制分隔符）等等。

通常，您不会在方法签名中看到CultureInfo，而是看到IFormatProvider接口，CultureInfo实现了该接口。大多数格式化方法都有重载，其中IFormatProvider作为第一个参数，位于格式字符串本身之前。例如，考虑string.Format中的这两个签名：

```csharp
static string Format(IFormatProvider provider, string format, params object[] args)
static string Format(string format, params object[] args)
```

通常，如果您提供仅相差一个参数的重载，那么该参数是最后一个，因此您可能希望provider参数在args之后。但是，这行不通，因为args是一个参数数组（它使用params修饰符）。如果一个方法有参数数组，那么它必须是最后一个参数。

即使参数的类型是IFormatProvider，您作为参数传入的值几乎总是CultureInfo。例如，如果你想用美国英语格式化我的出生日期——1976年6月19日——你可以使用以下代码：

```csharp
var usEnglish = CultureInfo.GetCultureInfo("en-US");
var birthDate = new DateTime(1976, 6, 19);
string formatted = string.Format(usEnglish, "Jon was born on {0:d}", birthDate);
```

这里，d是短日期的标准日期/时间格式说明符，在美国英语中对应于月/日/年。例如，我的出生日期将被格式化为6/19/1976。在英国英语中，短日期格式是日/月/年，因此同一日期将被格式化为19/06/1976。请注意，不仅顺序不同：在英国格式化中，月份也被0填充到两位数字。

其他文化可以使用完全不同的格式化。了解同一值在不同文化之间的格式化结果有多大差异是很有启发的。例如，你可以在.NET知道的每种文化中格式化同一个日期，如下一个清单所示。

**清单9.2：在每个文化中格式化单个日期**
```csharp
var cultures = CultureInfo.GetCultures(CultureTypes.AllCultures);
var birthDate = new DateTime(1976, 6, 19);
foreach (var culture in cultures)
{
    string text = string.Format(
        culture, "{0,-15} {1,12:d}", culture.Name, birthDate);
    Console.WriteLine(text);
}
```

泰国的输出显示我出生于泰国佛历2519年，阿富汗的输出显示我出生于伊斯兰历1355年：
```
...
tg-Cyrl         19.06.1976
tg-Cyrl-TJ      19.06.1976
th              19/6/2519
th-TH           19/6/2519
ti              19/06/1976
ti-ER           19/06/1976
...
ur-PK           19/06.1976
uz              19/06.1976
uz-Arab         29/03 1355
uz-Arab-AF      29/03 1355
uz-Cyrl         19/06/1976
uz-Cyrl-UZ      19/06/1976
...
```

此示例还显示了对齐值为负，使用{0,-15}格式项左对齐文化名称，同时使用{1,12:d}格式项右对齐日期。

#### 使用默认文化进行格式化

如果您未指定格式提供程序，或者将null作为对应于IFormatProvider参数的参数传递，则将使用CultureInfo.CurrentCulture作为默认值。确切含义将取决于您的上下文；它可以按线程设置，并且某些Web框架将在特定线程上处理请求之前设置它。

关于使用默认值，我所能建议的是要小心：确保您知道特定线程中的值将是合适的。（例如，如果您开始跨多个线程并行化操作，检查确切行为特别值得。）如果您不想依赖默认文化，您需要知道需要为其格式化文本的最终用户的文化，并明确地这样做。

#### 为机器格式化

到目前为止，我们假设您正在尝试为最终用户格式化文本。但通常情况并非如此。对于机器对机器通信（例如，在将由Web服务解析的URL查询参数中），您应该使用固定文化，该文化通过静态属性CultureInfo.InvariantCulture获得。

例如，假设您正在使用Web服务从出版商获取畅销书列表。Web服务可能使用URL `https://manning.com/webservices/bestsellers`，但允许一个名为date的查询参数，以便您查找特定日期的畅销书。我希望该查询参数使用ISO-8601格式（年份在前，年、月、日之间使用破折号）表示日期。例如，如果您想检索截至2017年3月20日的畅销书，您将使用URL `https://manning.com/webservices/bestsellers?date=2017-03-20`。要在允许用户选择特定日期的应用程序中编写代码构建该URL，您可能会编写如下内容：

```csharp
string url = string.Format(
    CultureInfo.InvariantCulture,
    "{0}?date={1:yyyy-MM-dd}",
    webServiceBaseUrl,
    searchDate);
```

请注意，大多数情况下，您不应该自己直接为机器对机器通信格式化数据。我建议您尽可能避免字符串转换；它们通常是一种代码异味，表明您要么没有正确使用库或框架，要么存在数据设计问题（例如，将日期以文本形式存储在数据库中，而不是本机日期/时间类型）。话虽如此，您可能会发现自己经常手动构建这样的字符串；只需注意应该使用哪种文化。

好了，这是一个很长的介绍。但是，所有这些格式化信息在您脑海中嗡嗡作响，以及一些丑陋的示例让您感到困扰，您正好可以迎接C#6中的内插字符串字面量。所有对string.Format的调用看起来都不必要地冗长，而且必须查看格式字符串和参数列表之间以了解什么将放在哪里是令人讨厌的。当然，我们可以使代码更清晰。



# 引入内插字符串字面量

C# 6中的内插字符串字面量允许您以更简单的方式执行所有这些格式化操作。格式字符串和参数的概念仍然适用，但使用内插字符串字面量时，您可以内联指定值及其格式化信息，从而使代码更易于阅读。如果您查看代码并发现大量使用硬编码格式字符串调用string.Format的情况，您会喜欢内插字符串字面量。

字符串内插并不是一个新概念。它已经在许多编程语言中存在很长时间，但我从未感到它能像在C#中那样巧妙地集成。考虑到在语言已经成熟时添加特性比在第一个版本中构建它更困难，这一点尤其值得注意。

在本节中，您将先看一些简单示例，然后探索内插逐字字符串字面量。您将学习如何使用FormattableString应用本地化，然后更仔细地了解编译器如何处理内插字符串字面量。我们将在本节结束时讨论此特性最有用之处及其限制。

## 简单内插

在C# 6中演示内插字符串字面量的最简单方法是向您展示早期示例的等效代码，其中我们询问了用户的姓名。代码看起来并没有太大不同；特别是，只有最后一行发生了变化。

**C# 5 - 旧式格式化**
```csharp
Console.Write("What's your name? ");
string name = Console.ReadLine();
Console.WriteLine("Hello, {0}!", name);
```

**C# 6 - 内插字符串字面量**
```csharp
Console.Write("What's your name? ");
string name = Console.ReadLine();
Console.WriteLine($"Hello, {name}!");
```

内插字符串字面量以粗体显示。它在开头双引号前以$开头；就编译器而言，这就是使其成为内插字符串字面量而不是常规字符串的原因。它包含{name}而不是{0}作为格式项。大括号中的文本是一个表达式，该表达式被求值然后在字符串中格式化。由于您已经提供了所需的所有信息，因此不再需要WriteLine的第二个参数。

**注意**：为了简单起见，我在这里撒了一点谎。这段代码的工作方式与原始代码不完全相同。原始代码将所有参数传递给适当的Console.WriteLine重载，后者为您执行格式化。现在，所有格式化都是通过string.Format调用执行的，然后Console.WriteLine调用使用只有一个字符串参数的重载。不过，结果将是相同的。

就像表达式主体成员一样，这看起来并不是一个巨大的改进。对于单个格式项，原始代码没有太多令人困惑的地方。您前几次看到这种情况时，阅读内插字符串字面量甚至可能比阅读字符串格式化调用花费更长的时间。我曾经怀疑我到底会有多喜欢它们，但现在我经常发现自己几乎自动地将旧代码片段转换为使用它们，而且我发现可读性的提高通常是显著的。

现在您已经看到了最简单的示例，让我们做一些更复杂的事情。您将遵循与之前相同的顺序，首先更仔细地查看控制值的格式化，然后考虑本地化。

## 内插字符串字面量中的格式字符串

好消息！这里没有新东西要学。如果要在内插字符串字面量中提供对齐或格式字符串，您可以像在普通组合格式字符串中那样做：在对齐前添加逗号，在格式字符串前添加冒号。我们之前的组合格式化示例以明显的方式发生了变化，如下一个清单所示。

**清单9.3 使用内插字符串字面量对齐值**
```csharp
decimal price = 95.25m;
decimal tip = price * 0.2m;  // 20%小费
Console.WriteLine($"价格: {price,9:C}");  // 使用九位对齐右对齐价格
Console.WriteLine($"小费:    {tip,9:C}");
Console.WriteLine($"总计: {price + tip,9:C}");
```

请注意，在最后一行中，内插字符串不仅包含参数的简单变量；它还执行了小费与价格的加法。表达式可以是计算值的任何表达式。（例如，您不能只调用返回类型为void的方法。）如果该值实现了IFormattable接口，则将调用其ToString(string, IFormatProvider)方法；否则，将使用System.Object.ToString()。

## 内插逐字字符串字面量

您以前肯定见过逐字字符串字面量；它们在双引号前以@开头。在逐字字符串字面量中，反斜杠和换行符包含在字符串中。例如，在逐字字符串字面量@"c:\Windows"中，反斜杠确实是反斜杠；它不是转义序列的开头。逐字字符串字面量中唯一的转义序列是当您有两个双引号字符在一起时，这会在结果字符串中产生一个双引号字符。逐字字符串字面量通常用于以下情况：

* 跨多行的字符串
* 正则表达式（使用反斜杠进行转义，与C#编译器在常规字符串字面量中使用的转义完全不同）
* 硬编码的Windows文件名

**注意**：对于多行字符串，您应该小心字符串中到底包含哪些字符。尽管“回车”和“回车换行分隔符”之间的差异在大多数代码中无关紧要，但在逐字字符串字面量中却很重要。

以下快速展示了每种情况的示例：
```csharp
string sql = @"
SELECT City, ZipCode  -- SQL在多行拆分时更容易阅读
FROM Address
WHERE Country = 'US'";

Regex lettersDotDigits = new Regex(@"[a-z]+\.\d+");  -- 正则表达式中常见反斜杠

string file = @"c:\users\skeet\Test\Test.cs"  -- Windows文件名
```

逐字字符串字面量也可以被内插；您在@前面放一个$，就像您内插常规字符串字面量一样。我们早期的多行输出可以使用单个内插逐字字符串字面量编写，如下一个清单所示。

**清单9.4 使用单个内插逐字字符串字面量对齐值**
```csharp
decimal price = 95.25m;
decimal tip = price * 0.2m;  // 20%小费
Console.WriteLine($@"价格: {price,9:C}
小费:    {tip,9:C}
总计: {price + tip,9:C}");
```

我可能不会这样做；它不如使用三个单独的语句清晰。我仅使用前面的代码作为可能性的简单示例。考虑在您已经合理使用逐字字符串字面量的地方使用它。

**提示**：符号的顺序很重要。$@"Text"是有效的内插逐字字符串字面量，但@$"Text"不是。我承认我还没有找到好的助记符来记住这一点。只需尝试您认为正确的方式，如果编译器报错就更改它！

这一切都非常方便，但我只展示了表面现象。我假设您购买这本书是因为您想深入了解这些特性。

## 编译器对内插字符串字面量的处理（第1部分）

这里的编译器转换很简单。它将内插字符串字面量转换为对string.Format的调用，并从格式项中提取表达式，并将它们作为组合格式字符串之后的参数传递。表达式被替换为适当的索引，因此第一个格式项变为{0}，第二个变为{1}，依此类推。

为了更清楚，让我们考虑一个简单的示例，这次将格式化与输出分开以便清晰：
```csharp
int x = 10;
int y = 20;
string text = $"x={x}, y={y}";
Console.WriteLine(text);
```

编译器处理此代码的方式就好像您写了以下代码：
```csharp
int x = 10;
int y = 20;
string text = string.Format("x={0}, y={1}", x, y);
Console.WriteLine(text);
```

转换就是这么简单。如果您想深入了解并亲自验证，可以使用诸如ildasm之类的工具来查看编译器生成的IL。

这种转换的一个副作用是，与常规或逐字字符串字面量不同，内插字符串字面量不算作常量表达式。尽管在某些情况下编译器可以合理地认为它们是常量（如果它们没有任何格式项，或者所有格式项都只是没有任何对齐或格式字符串的字符串常量），但这些将是角落情况，会使语言复杂化而收益甚微。

到目前为止，我们所有的内插字符串都导致了对string.Format的调用。但情况并非总是如此，而且有很好的理由，正如您将在下一节中看到的那样。

# 使用FormattableString进行本地化

在9.1.3节中，我演示了字符串格式化如何利用不同的格式提供程序（通常使用CultureInfo）来执行本地化。到目前为止，您看到的所有内插字符串字面量都将使用执行线程的默认文化进行求值，因此我们在9.1.2和9.2.2节中的价格示例在您的计算机上可能产生与我展示的结果不同的输出。

要在特定文化中执行格式化，您需要三条信息：
* 组合格式字符串，其中包括硬编码文本和作为真实值占位符的格式项
* 值本身
* 您要格式化字符串的文化

您可以稍微重写我们在文化中格式化的第一个示例，将每条信息存储在单独的变量中，然后最后调用string.Format：
```csharp
var compositeFormatString = "Jon was born on {0:d}";
var value = new DateTime(1976, 6, 19);
var culture = CultureInfo.GetCultureInfo("en-US");
var result = string.Format(culture, compositeFormatString, value);
```

如何使用内插字符串字面量做到这一点？内插字符串字面量包含前两条信息（组合格式字符串和要格式化的值），但没有地方放置文化。如果您以后可以访问各个信息，那也没关系，但到目前为止，您看到的每个内插字符串字面量的使用都执行了字符串格式化，只留下一个字符串作为结果。

这就是FormattableString的用武之地。这是.NET 4.6（以及.NET Core世界中的.NET Standard 1.3）中引入的System命名空间中的一个类。它保存组合格式字符串和值，以便以后可以在您想要的任何文化中格式化。编译器知道FormattableString，并可以在必要时将内插字符串字面量转换为FormattableString而不是字符串。这允许您将我们简单的出生日期示例重写如下：
```csharp
var dateOfBirth = new DateTime(1976, 6, 19);
FormattableString formattableString = 
    $"Jon was born on {dateOfBirth:d}";  // 将组合格式字符串和值保存在FormattableString中
var culture = CultureInfo.GetCultureInfo("en-US");
var result = formattableString.ToString(culture);  // 在指定文化中格式化
```

现在您知道了FormattableString存在的基本原因，您可以看看编译器如何使用它，然后更详细地检查本地化。尽管本地化肯定是FormattableString的主要动机，但它也可以用于其他情况，您将在9.3.3节中看到。然后，如果您的代码针对的是早期版本的.NET，本节将以您的选项结束。

## 编译器对内插字符串字面量的处理（第2部分）

与我之前的方法相反，这次在详细检查其用途之前先讨论编译器如何考虑FormattableString是有意义的。内插字符串字面量的编译时类型是string。没有从string到FormattableString或IFormattable（FormattableString实现的接口）的转换，但是从内插字符串字面量表达式到FormattableString和IFormattable都有转换。

从表达式到类型的转换与从类型到另一个类型的转换之间的差异有些微妙，但这并不是新事物。例如，考虑整数字面量5。它的类型是int，所以如果您声明var x = 5，x的类型将是int，但您也可以使用它来初始化类型为byte的变量。例如，byte y = 5;是完全可以的。这是因为语言规定，对于byte范围内的常量整数表达式（包括整数字面量），存在从表达式到byte的隐式转换。如果您能理解这一点，您可以将完全相同的想法应用于逐字字符串字面量。

当编译器需要将内插字符串字面量转换为FormattableString时，它执行与转换为string时大部分相同的步骤。但它不是调用string.Format，而是调用System.Runtime.CompilerServices.FormattableStringFactory类上的静态Create方法。这是与FormattableString同时引入的另一种类型。回到前面的示例，假设您有以下源代码：
```csharp
int x = 10;
int y = 20;
FormattableString formattable = $"x={x}, y={y}";
```

编译器处理此代码的方式就好像您写了以下代码（当然，使用适当的命名空间）：
```csharp
int x = 10;
int y = 20;
FormattableString formattable = FormattableStringFactory.Create(
    "x={0}, y={1}", x, y);
```

FormattableString是一个抽象类，其成员如下一个清单所示。

**清单9.5 FormattableString声明的成员**
```csharp
public abstract class FormattableString : IFormattable
{
    protected FormattableString();
    public abstract object GetArgument(int index);
    public abstract object[] GetArguments();
    public static string Invariant(FormattableString formattable);
    string IFormattable.ToString(string ignored, IFormatProvider formatProvider);
    public override string ToString();
    public abstract string ToString(IFormatProvider formatProvider);
    public abstract int ArgumentCount { get; }
    public abstract string Format { get; }
}
```

## 在特定文化中格式化FormattableString

到目前为止，FormattableString最常见的用途是在显式指定的文化中执行格式化，而不是在线程的默认文化中。我期望大多数用途是针对单一文化：固定文化。这非常常见，以至于它有自己的静态方法：Invariant。调用此方法等效于将CultureInfo.InvariantCulture传递到ToString(IFormatProvider)方法中，其行为完全符合您的期望。但使Invariant成为静态方法意味着作为语言细节的微妙推论，调用起来更简单，您刚刚在9.3.1节中查看了这些细节。它接受FormattableString作为参数这一事实意味着您可以将内插字符串字面量作为参数使用，编译器知道它必须应用相关的转换；不需要强制转换或单独的变量。

让我们考虑一个具体示例以使其更清晰。假设您有一个DateTime值，并且您希望仅将其日期部分以ISO-8601格式格式化为机器对机器通信的URL查询参数的一部分。您希望使用固定文化以避免使用默认文化产生任何意外结果。

**注意**：即使您为日期和时间指定了自定义格式字符串，并且即使该自定义格式仅使用数字，文化仍然会产生影响。最大的影响是，该值以文化的默认日历系统表示。如果您在文化ar-SA（沙特阿拉伯的阿拉伯语）中格式化2016年10月21日（公历），您将得到年份为1438的结果。

您可以通过四种方式执行此格式化，所有这些方式都一起显示在以下清单中。所有四种方法都给出完全相同的结果，但我展示了所有方法，以演示多种语言特性如何协同工作以提供干净的最终选项。

**清单9.6 在固定文化中格式化日期**
```csharp
DateTime date = DateTime.UtcNow;
// 使用string.Format的旧式格式化
string parameter1 = string.Format(CultureInfo.InvariantCulture, "x={0:yyyy-MM-dd}", date);
// 强制转换为FormattableString并调用ToString(IFormatProvider)
string parameter2 = ((FormattableString)$"x={date:yyyy-MM-dd}").ToString(CultureInfo.InvariantCulture);
// 常规调用FormattableString.Invariant
string parameter3 = FormattableString.Invariant($"x={date:yyyy-MM-dd}");
// 缩短调用FormattableString.Invariant
string parameter4 = Invariant($"x={date:yyyy-MM-dd}");
```

主要的有趣区别在于parameter2和parameter3的初始化器之间。为了确保您有一个FormattableString而不是一个字符串，您必须将内插字符串字面量强制转换为该类型。另一种方法是声明一个单独的类型为FormattableString的局部变量，但这将同样冗长。与parameter3的初始化方式进行比较，后者使用接受FormattableString类型参数的Invariant方法。这允许编译器推断您希望使用从内插字符串字面量到FormattableString的隐式转换，因为这是使调用有效的唯一方式。

对于parameter4，我作弊了。我使用了一个您尚未见过的特性，通过using static指令使类型的静态方法可用。您可以稍后翻阅详细信息（第10.1.1节）或现在相信我它是有效的。您只需要在using指令列表中使用using static System.FormattableString。

#### 在非固定文化中格式化

如果您希望在除固定文化之外的任何文化中格式化FormattableString，您需要使用其中一个ToString方法。在大多数情况下，您会希望直接调用ToString(IFormatProvider)重载。作为比您之前看到的稍短的示例，以下是使用“带短时间的一般日期/时间”标准格式字符串（"g"）以美国英语格式化当前日期和时间的代码：
```csharp
FormattableString fs = $"The current date and time is: {DateTime.Now:g}";
string formatted = fs.ToString(CultureInfo.GetCultureInfo("en-US"));
```

有时，您可能希望将FormattableString传递给另一段代码以执行最终格式化步骤。在这种情况下，值得记住的是FormattableString实现了IFormattable接口，因此任何接受IFormattable的方法都将接受FormattableString。FormattableString的IFormattable.ToString(string, IFormatProvider)实现忽略string参数，因为它已经拥有所需的一切：它使用IFormatProvider参数调用ToString(IFormatProvider)方法。

现在您知道了如何在本地化中使用内插字符串字面量，您可能想知道为什么FormattableString的其他成员存在。在下一节中，您将看到一个示例。



## **FormattableString 的其他用途**

除了在 9.3.2 节中展示的文化场景，我并不指望 `FormattableString` 会被广泛使用，但了解它能做什么是值得的。我选择这个例子是因为它本身清晰易懂且优雅，但我不至于推荐使用它。除了这里展示的代码缺乏验证和一些特性外，它可能给不经意的读者（以及静态代码分析工具）留下错误印象。当然，可以将其作为一个思路来探索，但需谨慎对待。

大多数开发者都知道 SQL 注入攻击是一种安全漏洞，许多人知道参数化 SQL 这一常见解决方案。清单 9.7 展示了您**不**应该做的事情。如果用户输入了包含单引号的值，他们就对数据库拥有了巨大控制权。假设您有一个数据库，其中包含用户可以按用户标识符分区添加标签的条目。您试图列出用户指定标签的所有描述，但仅限于该用户。

**清单 9.7 警告！警告！不要使用此代码！**
```csharp
var tag = Console.ReadLine();
using (var conn = new SqlConnection(connectionString))
{
    conn.Open();
    string sql = $@"SELECT Description FROM Entries
                    WHERE Tag='{tag}' AND UserId={userId}";
    using (var command = new SqlCommand(sql, conn))
    {
        using (var reader = command.ExecuteReader())
        {
            // 使用数据
        }
    }
}
```

我在 C# 中看到的大多数 SQL 注入漏洞使用字符串连接而非字符串格式化，但实质相同。它以令人担忧的方式混合了代码（SQL）和数据（用户输入的值）。

我假设您知道过去如何通过参数化 SQL 和适当调用 `command.Parameters.Add(...)` 来修复此问题。代码和数据被适当分离，一切恢复如常。遗憾的是，这段安全代码看起来不如清单 9.7 中的代码吸引人。如果能两全其美呢？如果能编写既清晰表达意图又安全参数化的 SQL 呢？使用 `FormattableString`，您完全可以做到。

我们将逆向工作，从期望的用户代码开始，通过实现使其成为可能。以下清单展示了清单 9.7 的安全等价版本。

**清单 9.8 使用 FormattableString 的安全 SQL 参数化**
```csharp
var tag = Console.ReadLine();
using (var conn = new SqlConnection(connectionString))
{
    conn.Open();
    using (var command = conn.NewSqlCommand(
        $@"SELECT Description FROM Entries
            WHERE Tag={tag:NVarChar}
            AND UserId={userId:Int}"))
    {
        using (var reader = command.ExecuteReader())
        {
            // 使用数据
        }
    }
}
```

此清单大部分与清单 9.7 相同。唯一区别在于如何构建 `SqlCommand`。您不是使用内插字符串字面量将值格式化为 SQL，然后将该字符串传给 `SqlCommand` 构造函数，而是使用一个名为 `NewSqlCommand` 的新方法（您将编写的扩展方法）。不出所料，该方法的第二个参数不是 `string` 而是 `FormattableString`。内插字符串字面量不再在 `{tag}` 周围使用单引号，并且您将每个参数的数据库类型指定为格式字符串。这确实不寻常。它在做什么？

首先，考虑编译器在为您做什么。它将内插字符串字面量分成两部分：组合格式字符串和格式项参数。编译器创建的组合格式字符串如下：

```
SELECT Description FROM Entries
WHERE Tag={0:NVarChar} AND UserId={1:Int}
```

而您希望的 SQL 最终应该是：

```
SELECT Description FROM Entries
WHERE Tag=@p0 AND UserId=@p1
```

这很容易实现：只需格式化组合格式字符串，传入将计算为 `"@p0"` 和 `"@p1"` 的参数。如果这些参数的类型实现 `IFormattable`，调用 `string.Format` 也会传递 `NVarChar` 和 `Int` 格式字符串，因此可以适当设置 `SqlParameter` 对象的类型。您可以自动生成参数名称，值直接来自 `FormattableString`。

让 `IFormattable.ToString` 实现产生副作用极不寻常，但您仅为此单次调用使用此格式捕获类型，并且可以将其安全隐藏在其他代码之外。以下清单是一个完整实现。

**清单 9.9 实现安全的 SQL 格式化**
```csharp
public static class SqlFormattableString
{
    public static SqlCommand NewSqlCommand(
        this SqlConnection conn, FormattableString formattableString)
    {
        SqlParameter[] sqlParameters = formattableString.GetArguments()
            .Select((value, position) =>
                new SqlParameter(Invariant($"@p{position}"), value))
            .ToArray();
        object[] formatArguments = sqlParameters
            .Select(p => new FormatCapturingParameter(p))
            .ToArray();
        string sql = string.Format(formattableString.Format, formatArguments);
        var command = new SqlCommand(sql, conn);
        command.Parameters.AddRange(sqlParameters);
        return command;
    }

    private class FormatCapturingParameter : IFormattable
    {
        private readonly SqlParameter parameter;
        internal FormatCapturingParameter(SqlParameter parameter)
        {
            this.parameter = parameter;
        }

        public string ToString(string format, IFormatProvider formatProvider)
        {
            if (!string.IsNullOrEmpty(format))
            {
                parameter.SqlDbType = (SqlDbType) Enum.Parse(
                    typeof(SqlDbType), format, true);
            }
            return parameter.ParameterName;
        }
    }
}
```

此代码唯一公开部分是 `SqlFormattableString` 静态类及其 `NewSqlCommand` 方法。其他都是隐藏的实现细节。对于格式字符串中的每个占位符，您创建 `SqlParameter` 和对应的 `FormatCapturingParameter`。后者用于在 SQL 中将参数名格式化为 `@p0`、`@p1` 等，并提供给 `ToString` 方法的值被设置到 `SqlParameter` 中。如果用户在格式字符串中指定了类型，也会设置参数类型。

此时，您需要自行决定是否希望在生产代码库中看到这样的代码。我会想实现额外特性（例如在格式字符串中包含大小；无法使用格式项的对齐部分，因为 `string.Format` 自身处理对齐），但它当然可以被适当产品化。但是否太聪明了？您是否需要向项目的每个新开发者解释："我知道这看起来像有巨大 SQL 注入漏洞，但真的没问题"？

无论此具体示例如何，您很可能能找到类似场景，可以利用编译器提取数据并将其与内插字符串字面量文本分离的功能。始终仔细考虑此类解决方案是否真正提供益处，或者只是让您有机会感觉聪明。

如果您以 .NET 4.6 为目标，这一切都有用，但如果您困在旧框架版本上怎么办？仅使用 C# 6 编译器并不意味着您必须针对现代框架版本。幸运的是，C# 编译器不将此与特定框架版本绑定；它只需要以某种方式提供正确类型。

## **在旧版 .NET 中使用 FormattableString**

就像扩展方法的特性和调用者信息特性一样，C# 编译器不固定认为应由哪个程序集包含其依赖的 `FormattableString` 和 `FormattableStringFactory` 类型。编译器关心命名空间并期望 `FormattableStringFactory` 上存在适当的静态 `Create` 方法，仅此而已。如果您想利用 `FormattableString` 的好处但困在早期框架版本，可以自己实现这两种类型。

展示代码前，我应指出这应视为最后手段。当您最终升级环境以 .NET 4.6 为目标时，应立即删除这些类型以避免编译器警告。尽管即使在 .NET 4.6 中执行也能使用自己的实现，我会尽量避免这种情况；根据我的经验，不同程序集中有相同类型可能导致难以诊断的问题。

抛开所有警告，实现很简单。清单 9.10 展示了两种类型。为简洁起见，我没有包含验证，将 `FormattableString` 设为具体类型，并使两个类都是内部的，但编译器不介意这些更改。将类型设为内部的原因是为了避免其他程序集依赖您的实现；这是否适合您的具体情况难以预测，但请在公开类型前仔细考虑。

**清单 9.10 从头实现 FormattableString**
```csharp
using System.Globalization;

namespace System.Runtime.CompilerServices
{
    internal static class FormattableStringFactory
    {
        internal static FormattableString Create(
            string format, params object[] arguments) =>
            new FormattableString(format, arguments);
    }
}

namespace System
{
    internal class FormattableString : IFormattable
    {
        public string Format { get; }
        private readonly object[] arguments;

        internal FormattableString(string format, object[] arguments)
        {
            Format = format;
            this.arguments = arguments;
        }

        public object GetArgument(int index) => arguments[index];
        public object[] GetArguments() => arguments;
        public int ArgumentCount => arguments.Length;

        public static string Invariant(FormattableString formattable) =>
            formattable?.ToString(CultureInfo.InvariantCulture);

        public string ToString(IFormatProvider formatProvider) =>
            string.Format(formatProvider, Format, arguments);

        public string ToString(
            string ignored, IFormatProvider formatProvider) =>
            ToString(formatProvider);
    }
}
```

我不会解释代码细节，因为每个成员都很简单。唯一可能需要稍作解释的是 `Invariant` 方法调用 `formattable?.ToString(CultureInfo.InvariantCulture)`。此表达式的 `?.` 部分是空条件运算符，您将在 10.3 节更详细地了解。

现在您了解了内插字符串字面量能做的一切，但应如何使用它们呢？





# **使用指南和限制**

与表达式主体成员类似，内插字符串字面量是一个可以安全尝试的特性。您可以调整代码以满足自己（或团队）的标准。如果您稍后改变主意想回到旧代码，这样做很简单。除非您开始在 API 中使用 `FormattableString`，否则内插字符串字面量的使用是隐藏的实现细节。当然，这并不意味着应该到处使用它们。在本节中，我们将讨论在哪些情况下使用内插字符串字面量有意义，哪些情况下没有意义，以及您可能发现即使想用也无法使用的情况。

## **开发者和机器，但不一定是最终用户**

首先，好消息：几乎任何您已经在使用硬编码组合格式字符串或普通字符串连接的地方，都可以使用内插字符串。大多数情况下，代码会变得更易读。

这里的硬编码部分很重要。内插字符串字面量不是动态的。组合格式字符串在您的源代码中；编译器只是稍作修改以使用常规格式项。当您事先知道所需文本和格式时，这很好，但不够灵活。

对字符串分类的一种方法是考虑谁或什么将使用它们。在本节中，我将考虑三种使用者：
* 供其他代码解析的字符串
* 给其他开发者的消息
* 给最终用户的消息

让我们依次看看每种字符串，思考内插字符串字面量是否有用。

### **机器可读字符串**

许多代码是为了读取其他字符串而构建的。有机器可读的日志格式、URL查询参数和基于文本的数据格式，如 XML、JSON 或 YAML。所有这些都有固定格式，任何值都应使用固定文化进行格式化。正如您所见，如果您需要自己执行格式化，这是使用 `FormattableString` 的好地方。提醒一下，无论如何，您通常应该利用适当的 API 来格式化机器可读字符串。

请记住，这些字符串中的每一个也可能包含针对人类的嵌套字符串；日志文件的每一行可能以特定方式格式化，以便将其视为单个记录，但其消息部分可能是针对其他开发者的。您需要跟踪代码的每个部分在哪个嵌套级别上工作。

### **给其他开发者的消息**

如果您查看大型代码库，可能会发现许多字符串字面量是针对其他开发者的，无论是同一公司的同事还是使用您发布的 API 的开发者。这些主要包括：
* 工具字符串，如控制台应用程序中的帮助消息
* 写入日志或控制台的诊断或进度消息
* 异常消息

根据我的经验，这些通常是英文的。尽管包括微软在内的一些公司会费力地本地化他们的错误消息，但大多数公司不会。本地化在数据翻译和正确使用代码方面都有显著成本。如果您知道您的受众至少能比较舒服地阅读英文，特别是如果他们可能想在 Stack Overflow 等英文网站上分享消息，那么通常不值得费力本地化这些字符串。

但是，是否确保文本中的值都以固定文化格式化则是另一回事。这肯定有助于提高一致性，但我怀疑我不是唯一一个没有尽可能注意这一点的开发者。然而，我鼓励您对日期使用非歧义格式。ISO 格式 `yyyy-MM-dd` 易于理解，并且没有 `dd/MM/yyyy` 或 `MM/dd/yyyy` 的"月份优先还是日期优先？"问题。如前所述，文化可能会影响产生的数字，因为世界不同地区使用不同的日历系统。请仔细考虑是否要使用固定文化来强制使用公历。例如，为无效参数抛出异常的代码可能如下所示：

```csharp
throw new ArgumentException(Invariant(
    $"Start date {start:yyyy-MM-dd} should not be earlier than year 2000."));
```

如果您知道所有阅读这些字符串的开发者都将处于相同的非英语文化中，那么完全可以用该文化编写所有消息。

### **给最终用户的消息**

最后，几乎所有的应用程序都至少有一些显示给最终用户的文本。与开发者一样，您需要了解每个用户的期望，以便做出正确的文本呈现决策。在某些情况下，您可以确信所有最终用户都乐于使用单一文化。这通常适用于您为企业或基于一个位置的其他组织内部构建的应用程序。这里您更可能使用当地文化而不是英语，但您不需要担心两个用户希望以不同方式查看相同信息。

到目前为止，所有这些情况都适合使用内插字符串字面量。我特别喜欢将它们用于异常消息。它们让我可以编写简洁的代码，同时为不幸的开发者提供有用的上下文，他们正在仔细查看日志并试图找出这次出了什么问题。

但是，当您有多种文化的最终用户时，内插字符串字面量很少有帮助，如果您不进行本地化，它们可能会损害您的产品。在这里，格式字符串很可能在资源文件中，而不是在您的代码中，因此您甚至不太可能看到使用内插字符串字面量的可能性。偶尔会有例外，例如当您只格式化一个信息片段以放入特定的 HTML 标签或类似内容时。在这些特殊情况下，内插字符串字面量应该没问题，但不要期望经常使用它们。

您已经看到不能将内插字符串字面量用于资源文件。接下来，您将看到该特性根本设计不用于帮助您的其他情况。

## **内插字符串字面量的硬性限制**

每个功能都有其限制，内插字符串字面量也不例外。这些限制有时有变通方法，但在建议您首先不要尝试它们之前，我会向您展示。

#### **无动态格式化**

您已经看到，不能更改构成内插字符串字面量的大部分组合格式字符串。但有一个部分感觉应该是可以动态表达的，但事实并非如此：单个格式字符串。让我们从前面的例子中取一部分：

```csharp
Console.WriteLine($"Price: {price,9:C}");
```

这里，我选择了 9 作为对齐，因为我知道我要格式化的值将恰好适合 9 个字符。但是，如果您知道有时您需要格式化的所有值都很小，而有时它们可能很大呢？能够使这个 9 部分动态化会很好，但没有简单的方法可以做到这一点。最接近的方法是使用内插字符串字面量作为 `string.Format` 或等效 `Console.WriteLine` 重载的输入，如下例所示：

```csharp
int alignment = GetAlignmentFromValues(allTheValues);
Console.WriteLine($"Price: {{0,{alignment}:C}}", price);
```

第一组和最后一组大括号在字符串格式中作为转义机制加倍，因为您希望内插字符串字面量的结果是一个像 `"Price: {0,9}"` 这样的字符串，然后使用 `price` 变量来填充格式项。这不是我想编写或阅读的代码。

#### **无表达式重新求值**

编译器总是将内插字符串字面量转换为代码，该代码立即求值格式项中的表达式，并使用它们构建字符串或 `FormattableString`。求值不能延迟或重复。考虑以下清单中的简短示例。它打印相同的值两次，即使开发者可能期望它使用延迟执行。

**清单 9.11 即使 FormattableString 也会急切求值表达式**
```csharp
string value = "Before";
FormattableString formattable = $"Current value: {value}";
Console.WriteLine(formattable); // 打印"Current value: Before"
value = "After";
Console.WriteLine(formattable); // 仍然打印"Current value: Before"
```

如果您非常需要，可以解决这个问题。如果您将表达式更改为包含捕获值的 lambda 表达式，您可以滥用这一点，在每次格式化时对其求值。尽管 lambda 表达式本身会立即转换为委托，但生成的委托将捕获 `value` 变量，而不是其当前值，并且您可以在每次格式化 `FormattableString` 时强制求值委托。这是一个足够糟糕的想法，因此我不会在这里展示它。

#### **无裸冒号**

尽管您可以在内插字符串字面量中使用几乎任何计算值的表达式，但条件运算符 `?:` 有一个问题：它会混淆编译器，实际上是 C# 语言的语法。除非您小心，否则冒号最终会被处理为表达式和格式字符串之间的分隔符，从而导致编译时错误。例如，这是无效的：

```csharp
Console.WriteLine($"Adult? {age >= 18 ? "Yes" : "No"}");
```

通过使用括号括起条件表达式，可以很容易地修复：

```csharp
Console.WriteLine($"Adult? {(age >= 18 ? "Yes" : "No")}");
```

我很少发现这是一个问题，部分原因是我通常尽量保持表达式比这更短。我可能首先将是/否值提取到一个单独的字符串变量中。

## **可以使用但真的不应该使用的情况**

编译器不会介意您是否滥用内插字符串字面量，但您的同事可能会。即使可以使用，也有两个主要理由不使用它们。

#### **为可能不使用的字符串延迟格式化**

有时，您想将格式字符串和将要格式化的参数传递给一个可能使用它们也可能不使用方法。例如，如果您有一个前置条件验证方法，您可能希望传入要检查的条件以及仅在条件失败时创建的异常消息的格式和参数。很容易写出这样的代码：

```csharp
Preconditions.CheckArgument(
    start.Year < 2000,
    Invariant($"Start date {start:yyyy-MM-dd} should not be earlier than year 2000."));
```

或者，您可能有一个日志框架，仅在执行时配置了适当级别时才记录。例如，您可能希望记录服务器接收到的请求的大小：

```csharp
Logger.Debug("Received request with {0} bytes", request.Length);
```

您可能想通过将代码更改为以下内容来使用内插字符串字面量：

```csharp
Logger.Debug($"Received request with {request.Length} bytes");
```

这将是一个坏主意；即使字符串将被丢弃，它也会强制进行格式化，因为格式化将在调用方法之前无条件执行，而不是仅在需要时在方法内部执行。尽管字符串格式化在性能方面并不非常昂贵，但您不想不必要地这样做。

您可能想知道 `FormattableString` 是否会在这里帮助。如果验证或日志记录库接受 `FormattableString` 作为输入参数，您可以延迟格式化并在单个位置控制用于格式化的文化。尽管这是真的，但它仍然涉及每次创建对象，这仍然是不必要的成本。

#### **为可读性而格式化**

不使用内插字符串字面量的第二个原因是它们可能使代码更难阅读。短表达式绝对没问题，有助于可读性。但是，当表达式变长时，弄清楚字面量的哪些部分是代码，哪些部分是文本需要更多时间。我发现括号是杀手；如果表达式中有多个方法或构造函数调用，它们最终会变得混乱。当文本中也包含括号时，情况更是如此。

这是一个来自 Noda Time 的真实示例。它在一个测试中而不是生产代码中，但我仍然希望测试是可读的：

```csharp
private static string FormatMemberDebugName(MemberInfo m) =>
    string.Format("{0}.{1}({2})",
        m.DeclaringType.Name,
        m.Name,
        string.Join(", ", GetParameters(m).Select(p => p.ParameterType)));
```

这还不错，但想象一下把这三个参数放在字符串里面。我做过，而且不好看；您最终会得到一个超过 100 个字符的字面量。您不能像上面那样用垂直格式拆分它，让每个参数独立，所以它最终会成为噪音。

最后，举一个讽刺的例子来说明这可能是一个多么糟糕的想法，请记住本章开头的代码：

```csharp
Console.Write("What's your name? ");
string name = Console.ReadLine();
Console.WriteLine("Hello, {0}!", name);
```

您可以使用内插字符串字面量将所有这些放在一个语句中。您可能持怀疑态度；毕竟，这段代码由三个单独的语句组成，而内插字符串字面量只能包含表达式。这是真的，但语句体 lambda 表达式仍然是表达式。您需要将 lambda 表达式转换为特定的委托类型，然后需要调用它以获得结果，但这是可行的。只是不愉快。这是一个选项，至少通过内插逐字字符串字面量在每个语句中使用单独的行，但这是它唯一的好处：

```csharp
Console.WriteLine($@"Hello {((Func<string>)(() =>
{
    Console.Write("What's your name? ");
    return Console.ReadLine();
}))()}!");
```

我强烈建议：运行代码以证明它有效，然后尽快远离它。在您从震惊中恢复时，让我们看看 C# 6 中另一个面向字符串的特性。



# **使用 nameof 访问标识符**

`nameof` 运算符描述起来很简单：它接受一个引用成员或局部变量的表达式，结果是该成员或变量的简单名称的编译时常量字符串。就这么简单。任何时候您硬编码类、属性或方法的名称时，使用 `nameof` 运算符会更好。您的代码现在和面对更改时都会更加健壮。

## **nameof 的第一个示例**

在语法上，`nameof` 运算符类似于 `typeof` 运算符，只是括号内的标识符不必是类型。以下清单展示了一个包含几种成员的简短示例。

**清单 9.12 打印类、方法、字段和参数的名称**
```csharp
using System;

class SimpleNameof
{
    private string field;

    static void Main(string[] args)
    {
        Console.WriteLine(nameof(SimpleNameof)); // 类名
        Console.WriteLine(nameof(Main));         // 方法名
        Console.WriteLine(nameof(args));         // 参数名
        Console.WriteLine(nameof(field));        // 字段名
    }
}
```

结果正是您可能期望的：
```
SimpleNameof
Main
args
field
```

到目前为止，一切顺利。但是，显然，您可以使用字符串字面量达到相同的结果。代码也会更短。那么为什么使用 `nameof` 更好呢？用一个词来说就是：健壮性。如果您在字符串字面量中打错字，没有什么会告诉您，而如果您在 `nameof` 操作数中打错字，您将得到编译时错误。

**注意**：如果您引用了具有相似名称的不同成员，编译器仍然无法发现问题。如果您有两个仅大小写不同的成员，例如 `filename` 和 `fileName`，您可以轻松引用错误的成员而编译器不会注意到。这是避免使用如此相似名称的一个好理由，但命名如此相似一直是个坏主意；即使您不会混淆编译器，也很容易混淆人类读者。

编译器不仅会在您出错时告诉您，而且它知道您的 `nameof` 代码与您命名的成员或变量相关联。如果您以支持重构的方式重命名它，您的 `nameof` 操作数也会更改。

例如，考虑以下清单。其目的无关紧要，但请注意 `oldName` 出现了三次：参数声明、使用 `nameof` 获取其名称以及作为简单表达式获取其值。

**清单 9.13 一个在其主体中两次使用其参数的简单方法**
```csharp
static void RenameDemo(string oldName)
{
    Console.WriteLine($"{nameof(oldName)} = {oldName}");
}
```

在 Visual Studio 中，如果您将光标放在 `oldName` 的任何一次出现中，并按 F2 进行重命名操作，所有三个将被一起重命名，如图 9.2 所示。

相同的方法适用于其他名称（方法、类型等）。基本上，`nameof` 以一种硬编码字符串字面量无法做到的方式支持重构。但是您应该在什么时候使用它呢？

![image-20260127083715211](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260127083715288.png)





## nameof的常见用法

我并不打算声称此处的示例是nameof唯一合理的用法。它们仅仅是我最常遇到的一些场景。这些地方大多在C# 6之前，你会看到要么是硬编码的名称，要么可能使用了表达式树作为解决方案——这种方法虽然对重构友好，但比较复杂。

**参数验证**
在第8章中，当我展示Noda Time中`Preconditions.CheckNotNull`的用法时，那并不是库中实际的代码。真实的代码包含了值为空的参数名称，这使得它有用得多。我在那里展示的`InZone`方法如下所示：

```csharp
public ZonedDateTime InZone(
    DateTimeZone zone,
    ZoneLocalMappingResolver resolver)
{
    Preconditions.CheckNotNull(zone, nameof(zone));
    Preconditions.CheckNotNull(resolver, nameof(resolver));
    return zone.ResolveLocal(this, resolver);
}
```

其他的前置条件方法也以类似的方式使用。这是我迄今为止发现的`nameof`最常见的用法。如果你还没有开始验证公有方法的参数，我强烈建议你开始这样做；`nameof`使得执行健壮的验证并提供信息性消息变得比以往任何时候都更容易。

**计算属性的属性更改通知**
正如你在7.2节中看到的，`CallerMemberNameAttribute`使得在`INotifyPropertyChanged`实现中，当属性本身改变时，轻松地引发事件。但是，如果更改一个属性的值会影响到另一个属性呢？例如，假设你有一个`Rectangle`类，具有读/写的`Height`和`Width`属性以及一个只读的`Area`属性。能够为`Area`属性引发事件并以一种安全的方式指定属性名称是很有用的，如下面的列表所示。

**列表9.14 使用nameof引发属性更改通知**
```csharp
public class Rectangle : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler PropertyChanged;

    private double width;
    private double height;

    public double Width
    {
        get { return width; }
        set
        {
            // 当值未改变时避免引发事件
            if (width == value)
            {
                return;
            }
            width = value;
            // 为Width属性引发事件
            RaisePropertyChanged();
            // 为Area属性引发事件
            RaisePropertyChanged(nameof(Area));
        }
    }

    public double Height { ... } // 实现方式与Width类似
    // 计算属性
    public double Area => Width * Height;
}

// 根据7.2节的更改通知实现
private void RaisePropertyChanged(
    [CallerMemberName] string propertyName = null) { ... }
```

这个列表的大部分内容与你用C# 5编写的代码完全相同，但**加粗**的那一行原本必须是`RaisePropertyChanged("Area")` 或 `RaisePropertyChanged(() => Area)`。后一种方法在`RaisePropertyChanged`代码方面会很复杂，并且效率低下，因为它构建一个表达式树仅仅是为了检查名称。`nameof`解决方案要简洁得多。

**特性**
有时特性会引用其他成员以指示成员之间如何相互关联。当你想要引用一个类型时，你已经可以使用`typeof`来建立这种关系，但这不适用于任何其他类型的成员。作为一个具体例子，NUnit允许测试使用从字段、属性或方法中提取的值进行参数化，这是通过`TestCaseSource`特性实现的。`nameof`运算符允许你以安全的方式引用该成员。下面的列表展示了Noda Time中的另一个示例，测试从时区数据库（TZDB，现由IANA托管）加载的所有时区在时间的起点和终点是否表现正常。

**列表9.15 使用nameof指定测试用例源**
```csharp
static readonly IEnumerable<DateTimeZone> AllZones =
    DateTimeZoneProviders.Tzdb.GetAllZones(); // 用于检索所有TZDB时区的字段

[Test]
[TestCaseSource(nameof(AllZones))] // 使用nameof引用该字段
public void AllZonesStartAndEnd(DateTimeZone zone)
{
    // ... 测试方法体省略
}
// 测试方法依次传入每个时区进行调用
```

这里的实用性并不局限于测试。它适用于任何指示关系的特性。你可以想象前面章节中一个更复杂的`RaisePropertyChanged`方法，其中属性之间的关系可以用特性而不是在代码内部指定：

```csharp
[DerivedProperty(nameof(Area))]
public double Width { ... }
```

事件引发方法可以维护一个缓存的数据结构，表明无论何时它被通知`Width`属性已更改，它也应该为`Area`引发一个更改通知。

类似地，在对象关系映射技术中，例如Entity Framework，在一个类中拥有两个属性是相当常见的：一个用于外键，另一个用于表示该键所代表的实体。下面的例子展示了这一点：

```csharp
public class Employee
{
    [ForeignKey(nameof(Employer))]
    public Guid EmployerId { get; set; }
    public Company Employer { get; set; }
}
```

毫无疑问，还有许多其他特性可以利用这种方法。

既然你意识到了这一点，你可能会在你现有的代码库中找到受益于`nameof`的地方。特别是，你应该寻找那些你需要对名称使用反射的代码，这些名称在编译时你是知道的，但以前无法以简洁的方式指定。然而，为了完整性，还有一些小细节需要介绍。

---





## **使用nameof的技巧与陷阱**



你可能永远不需要了解本节中的任何细节。这部分内容主要是为了以防你对`nameof`的行为感到惊讶。总的来说，这是一个相当简单的特性，但有几个方面可能会让你感到意外。

**引用其他类型的成员**
通常，能够从另一个类型的代码中引用一个类型的成员是很有用的。回到`TestCaseSource`特性，例如，除了名称，你还可以指定一个类型，NUnit将在该类型中查找该名称。如果你有一个将被多个测试使用的信息源，把它放在一个公共位置是有意义的。要用`nameof`做到这一点，你也需要用类型来限定它。结果将是简单名称：

```csharp
[TestCaseSource(typeof(Cultures), nameof(Cultures.AllCultures))]
```

这等同于以下代码，除了`nameof`的所有常规好处：

```csharp
[TestCaseSource(typeof(Cultures), "AllCultures")]
```

你也可以使用相关类型的变量来访问成员名称，尽管只适用于实例成员。反过来，你可以同时为静态成员和实例成员使用类型的名称。下面的列表展示了所有有效的排列组合。

**列表9.16 访问其他类型成员名称的所有有效方式**
```csharp
class OtherClass
{
    public static int StaticMember => 3;
    public int InstanceMember => 3;
}

class QualifiedNameof
{
    static void Main()
    {
        OtherClass instance = null;
        Console.WriteLine(nameof(instance.InstanceMember));
        Console.WriteLine(nameof(OtherClass.StaticMember));
        Console.WriteLine(nameof(OtherClass.InstanceMember));
    }
}
```

我倾向于尽可能总是使用类型名称；如果你使用变量代替，看起来变量的值可能很重要，但实际上它仅在编译时用于确定类型。如果你使用的是匿名类型，就没有类型名可用，所以你必须使用变量。

成员必须仍然可访问，你才能使用`nameof`引用它；如果列表9.16中的`StaticMember`或`InstanceMember`是私有的，尝试访问它们名称的代码将无法编译。

**泛型**
你可能想知道如果你尝试获取泛型类型或方法的名称会发生什么，以及它必须如何指定。特别是，`typeof`允许使用已绑定和未绑定的类型名称；`typeof(List<string>)` 和 `typeof(List<>)` 都是有效的，并给出不同的结果。

对于`nameof`，必须指定类型参数，但结果中不包含它。此外，结果中没有任何关于类型参数数量的指示：`nameof(Action<string>)` 和 `nameof(Action<string, string>)` 的值都只是 `"Action"`。这可能会令人恼火，但它消除了关于结果名称应如何表示数组、匿名类型、进一步的泛型类型等任何问题。

在我看来，将来可能会取消指定类型参数的要求，既是为了与`typeof`保持一致，也是为了避免必须指定一个对结果没有影响的类型。但是，将结果更改为包含类型参数数量或类型参数本身将是一个破坏性更改，我不认为这会发生。在大多数这种情况下，无论如何，使用`typeof`获取`Type`对象会是更好的选择。

你可以将类型参数与`nameof`运算符一起使用，但与`typeof(T)`不同，它将始终返回类型参数的名称，而不是在执行时用于该类型参数的类型实参的名称。下面是一个最小示例：

```csharp
static string Method<T>() => nameof(T); // 总是返回 "T"
```

无论你如何调用该方法：`Method<Guid>()` 或 `Method<Button>()` 都将返回 `"T"`。

**使用别名**
通常，提供类型或命名空间别名的`using`指令在运行时没有效果。它们只是引用相同类型或命名空间的不同方式。`nameof`运算符是这条规则的一个例外。下面列表的输出是`GuidAlias`，而不是`Guid`。

**列表9.17 在nameof运算符中使用别名**
```csharp
using System;
using GuidAlias = System.Guid;

class Test
{
    static void Main()
    {
        Console.WriteLine(nameof(GuidAlias));
    }
}
```

**预定义别名、数组和可空值类型**
`nameof`运算符不能与任何预定义别名（`int`、`char`、`long`等）或表示可空值类型的`?`后缀或数组类型一起使用。因此，以下所有内容都是无效的：

```csharp
nameof(float)      // System.Single的预定义别名
nameof(Guid?)      // Nullable<Guid>的简写
nameof(String[])   // 数组
```

这些有点烦人，但你必须为预定义别名使用CLR类型名称，并为可空值类型使用`Nullable<T>`语法：

```csharp
nameof(Single)
nameof(Nullable<Guid>)
```

正如前面关于泛型的小节所指出的，`Nullable<T>`的名称无论如何都总是`Nullable`。

**名称、简单名称，且仅为名称**
`nameof`运算符在某种程度上是神话般的`infoof`运算符的表亲，后者从未在C#语言设计会议之外被见过。（有关`infoof`的更多信息，请参见 http://mng.bz/6GVe。）如果团队真的设法捕获并驯服了`infoof`，它可能会返回对`MethodInfo`、`EventInfo`、`PropertyInfo`对象及其友元对象的引用。唉，`infoof`迄今为止已被证明是难以捉摸的，但许多它用来逃避捕获的技巧对更简单的`nameof`运算符来说是不可用的。尝试获取重载方法的名称？没问题；反正它们都有相同的名称。无法轻易分辨你指的是属性还是类型？同样，如果它们都有相同的名称，你使用哪个并不重要。尽管`infoof`如果能被合理设计出来，肯定能提供超越`nameof`的好处，但`nameof`运算符要简单得多，并且仍然解决了许多相同的用例。

关于返回结果需要注意的一点——简单名称或用不那么规范的术语说的“末尾的那部分”：无论你使用`nameof(Guid)`还是在一个导入了`System`命名空间的类中使用`nameof(System.Guid)`，结果仍然只是`"Guid"`。

**命名空间**
我没有详细说明`nameof`可以用于哪些所有成员，因为这是你所期望的集合：基本上，除了终结器和构造函数之外的所有成员。但是因为我们通常根据类型和类型内的成员来思考成员，你可能会惊讶地发现你可以获取命名空间的名称。是的，命名空间也是其他命名空间的成员。

但是考虑到前面关于只返回简单名称的规则，这并不特别有用。如果你使用`nameof(System.Collections.Generic)`，我猜你希望结果是`System.Collections.Generic`，但实际上，它只是`Generic`。我从未遇到过这种情况有用的时候，但话说回来，将命名空间作为编译时常量来了解也很少重要。

---

**总结**
* 内插字符串字面量允许你编写更简单的字符串格式化代码。
* 你仍然可以在内插字符串字面量中使用格式字符串来提供更多格式化细节，但格式字符串必须在编译时已知。
* 内插逐字字符串字面量提供了内插字符串字面量和逐字字符串字面量特性的混合。
* `FormattableString`类型提供了在格式化发生之前访问格式化字符串所需的所有信息。
* `FormattableString`在.NET 4.6和.NET Standard 1.3中开箱即用，但如果你在早期版本中提供自己的实现，编译器将使用它。
* `nameof`运算符提供了对重构友好且能防止拼写错误的方式来访问C#代码中的名称。