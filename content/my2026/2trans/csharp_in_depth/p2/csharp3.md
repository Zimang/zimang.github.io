---
weight: 10
title: "C# 3: LINQ and everything that comes with it"
---

**本章内容**
- 以简洁方式实现简单的属性
- 更简洁地初始化对象和集合
- 为本地数据创建匿名类型
- 使用 lambda 表达式构建委托和表达式树
- 用查询表达式简化复杂查询的表示

C# 2 的新特性大多相互独立。可空值类型依赖于泛型，但它们仍是独立的功能，并未共同构建某个总体目标。

C# 3 则不同。它包含许多新特性，每个特性本身都有其用途，但几乎所有这些特性都共同服务于 LINQ 这个更大的目标。本章将逐一介绍每个特性，然后演示它们如何协同工作。我们要看的第一个特性是唯一与 LINQ 没有直接关联的一个。

# 自动实现的属性
在 C# 3 之前，每个属性都必须手动实现，包含 `get` 和/或 `set` 访问器的主体。编译器乐意为类似字段的事件提供实现，但对属性却不提供。这意味着存在大量如下所示的属性：

```csharp
private string name;
public string Name
{
    get { return name; }
    set { name = value; }
}
```

代码风格不同，格式也会不同，但无论属性是写成一行、11 行，还是介于两者之间的五行（如前例所示），它始终只是噪音。这是一种非常冗长的方式，来表达想要一个字段并通过属性将其值暴露给调用者的意图。

C# 3 通过使用**自动实现的属性**（通常称为自动属性或 autoprops）大大简化了这一点。这些属性没有访问器主体；编译器提供实现。前面所有代码都可以替换为一行：

```csharp
public string Name { get; set; }
```

注意，现在源代码中没有字段声明。仍然存在一个字段，但它是由编译器自动为你创建的，并且其名称无法在 C# 代码中的任何地方被引用。

在 C# 3 中，不能声明只读的自动实现的属性，也不能在声明时提供初始值。这两个特性最终在 C# 6 中引入，并在 8.2 节中描述。在 C# 6 之前，一种相当常见的做法是通过提供私有 `set` 访问器来模拟只读属性，如下所示：

```csharp
public string Name { get; private set; }
```

C# 3 中自动实现的属性的引入，在减少样板代码方面产生了巨大影响。它们仅在属性只是简单地获取和设置字段值时有用，但根据我的经验，这占了属性的很大一部分。

如前所述，自动实现的属性并不直接贡献于 LINQ。让我们继续看第一个与 LINQ 相关的特性：数组和局部变量的隐式类型。

# 隐式类型
为了尽可能清晰地介绍 C# 3 中引入的特性，我首先需要定义一些术语。

## 类型术语
许多术语用于描述编程语言与其类型系统的交互方式。有些人使用“弱类型”和“强类型”这些术语，但我尽量避免使用它们，因为它们定义不明确，对不同开发者意味着不同的事情。另外两个方面则有更多共识：静态/动态类型和显式/隐式类型。让我们依次看看这些方面。

**静态与动态类型**
静态类型语言通常是编译型语言；编译器能够确定每个表达式的类型并检查其使用是否正确。例如，如果你在一个对象上调用方法，编译器可以使用类型信息，基于方法调用表达式的类型、方法名称以及参数的数量和类型，检查是否存在合适的方法可以调用。确定方法调用或字段访问等内容的含义称为**绑定**。动态类型语言则将全部或大部分绑定过程推迟到执行时。

注意：正如你在不同地方看到的，C# 中的某些表达式在源代码中没有类型，例如 `null` 字面量。但编译器总是根据使用表达式的上下文推断出一个类型，此时该类型可用于检查表达式的使用方式。

除了 C# 4 中引入的动态绑定（在第 4 章描述）外，C# 是一种静态类型语言。即使虚方法的哪个实现应该执行取决于调用它的对象的执行时类型，确定方法签名的绑定过程也全部发生在编译时。

**显式与隐式类型**
在显式类型语言中，源代码指定所涉及的所有类型。例如，这可以用于局部变量、字段、方法参数或方法返回类型。隐式类型语言允许开发者从源代码中省略类型，以便某些其他机制（无论是编译器还是执行时的某个东西）可以根据其他上下文推断出所指的是哪种类型。

C# 主要是显式类型的。即使在 C# 3 之前，也存在一些隐式类型，例如你在 2.1.4 节中看到的泛型类型参数的类型推断。可以说，隐式转换（例如从 `int` 到 `long`）的存在也使语言的显式类型程度降低。

区分了这些不同类型方面之后，你可以看看 C# 3 中关于隐式类型的特性。我们将从隐式类型局部变量开始。

## 隐式类型局部变量 (var)
隐式类型局部变量是使用上下文关键字 `var` 而不是类型名称声明的变量，例如：

```csharp
var language = "C#";
```

使用 `var` 声明局部变量与使用类型名称声明的结果仍然是具有已知类型的局部变量；唯一的区别是类型由编译器根据分配给它的值的编译时类型推断出来。前面的代码将生成与此完全相同的代码：

```csharp
string language = "C#";
```

提示：当 C# 3 首次推出时，许多开发者避免使用 `var`，因为他们认为这会移除很多编译时检查或导致执行时性能问题。它完全没有这样做；它只是推断局部变量的类型。声明之后，该变量的行为完全就像用显式类型名称声明的一样。

类型推断的方式导致了隐式类型局部变量的两个重要规则：

1. 变量必须在声明点初始化。
2. 用于初始化变量的表达式必须具有类型。

下面是一些无效的代码来演示这些规则：

```csharp
var x;       // 未提供初始值
x = 10;

var y = null; // 初始值没有类型
```

在某些情况下，通过分析对变量执行的所有赋值并从这些赋值中推断类型，本可以避免这些规则。有些语言这样做，但 C# 语言设计者更倾向于保持规则尽可能简单。

另一个限制是 `var` 只能用于局部变量。很多时候我渴望隐式类型字段，但它们仍然不可用（至少在 C# 7.3 中）。在前面的例子中，使用 `var` 几乎没有什么好处。显式声明是可行的，可读性也一样。通常使用 `var` 有三个原因：

1. 当变量的类型无法命名时，因为它是匿名的。你将在 3.4 节中看到匿名类型。这是该特性中与 LINQ 相关的部分。
2. 当变量的类型名称很长，并且人类读者可以很容易地根据用于初始化它的表达式推断出来时。
3. 当变量的确切类型不是特别重要，并且用于初始化它的表达式为任何阅读代码的人提供了足够的信息时。

我将把第一点的例子留到 3.4 节，但很容易展示第二点。假设你想创建一个字典，将名称映射到十进制值列表。你可以使用显式类型变量来实现：

```csharp
Dictionary<string, List<decimal>> mapping =
    new Dictionary<string, List<decimal>>();
```

这真的很丑陋。我不得不用两行来包装它以适应页面，并且有很多重复。使用 `var` 可以完全避免这种重复：

```csharp
var mapping = new Dictionary<string, List<decimal>>();
```

这用更少的文本表达了相同的信息量，因此分散你对其他代码注意力的内容更少。当然，这仅在希望变量的类型与初始化表达式的类型完全相同时才有效。如果你希望 `mapping` 变量的类型是 `IDictionary<string, List<decimal>>`（接口而不是类），那么 `var` 就无济于事了。但对于局部变量来说，接口和实现之间的这种分离通常不那么重要。

当我撰写《C# in Depth》第一版时，我对隐式类型局部变量持谨慎态度。除了 LINQ 之外，我很少使用它们，除非直接调用构造函数，如前例所示。我担心在阅读代码时无法轻松推断出变量的类型。

十年过去了，这种谨慎大多已消失。我在测试代码中几乎所有局部变量都使用 `var`，在生产代码中也大量使用。我的担忧并未成为现实；几乎在每种情况下，我都能很容易地通过检查推断出类型应该是什么。在无法做到的情况下，我会乐意改用显式声明。

我声称自己在这点上并不完全一致，当然也不教条。因为显式类型变量生成的代码与隐式类型变量完全相同，所以以后改变主意（无论向哪个方向）都没问题。我建议你与将最常使用你代码的其他人（无论是同事还是开源协作者）讨论，了解每个人的舒适度，并尝试遵守这一点。

C# 3 中隐式类型的另一个方面则有所不同。它与 `var` 没有直接关系，但具有相同的特性：移除类型名称让编译器推断它。



## 隐式类型数组  

有时你需要创建一个数组而不填充它，并保持所有元素为默认值。自 C# 1 以来，其语法未变；通常如下所示：  
`int[] array = new int[10];`  
但你经常需要创建具有特定初始内容的数组。在 C# 3 之前，有两种方式：  
`int[] array1 = { 1, 2, 3, 4, 5};`  
`int[] array2 = new int[] { 1, 2, 3, 4, 5};`  
第一种形式仅在作为指定数组类型的变量声明的一部分时才有效。例如，这是无效的：  
`int[] array;`  
`array = { 1, 2, 3, 4, 5 }; // 无效`  
第二种形式始终有效，因此上述示例的第二行可以写作：  
`array = new int[] { 1, 2, 3, 4, 5 };`  
C# 3 引入了第三种形式，其中数组类型根据内容隐式推断：  
`array = new[] { 1, 2, 3, 4, 5 };`  
只要编译器能够从指定的数组元素推断出数组元素类型，此形式可用于任何地方。它也适用于多维数组，例如：  
`var array = new[,] { { 1, 2, 3 }, { 4, 5, 6 } };`  
接下来的明显问题是编译器如何推断类型。与常见情况一样，为处理各种边界情况，精确细节较为复杂，但简化的步骤序列如下：  

1. 通过考虑每个具有类型的数组元素的类型，找到一组候选类型。  
2. 对于每个候选类型，检查每个数组元素是否具有到该类型的隐式转换。移除任何不满足此条件的候选类型。  
3. 如果恰好剩余一种类型，那就是推断出的元素类型，编译器会创建相应的数组。否则（如果没有类型或剩余多种类型），则发生编译时错误。  
数组元素类型必须是数组初始化器中某个表达式的类型。不会尝试寻找公共基类或共同实现的接口。表3.1提供了一些示例来说明这些规则。  

表3.1 隐式类型数组的类型推断示例  
| 表达式                          | 结果       | 说明                                                         |
| ------------------------------- | ---------- | ------------------------------------------------------------ |
| `new[] { 10, 20 }`              | `int[]`    | 所有元素均为 int 类型。                                      |
| `new[] { null, null }`          | 错误       | 没有元素具有类型。                                           |
| `new[] { "xyz", null }`         | `string[]` | 唯一的候选类型是 string，且 null 字面量可转换为 string。     |
| `new[] { "abc", new object() }` | `object[]` | 候选类型为 string 和 object；存在从 string 到 object 的隐式转换，但反之则无。 |
| `new[] { 10, new DateTime() }`  | 错误       | 候选类型为 int 和 DateTime，但两者之间无转换。               |
| `new[] { 10, null }`            | 错误       | 唯一的候选类型是 int，但无法从 null 转换为 int。             |

隐式类型数组主要是一种便利，以减少所需的源代码，除了匿名类型——即使你想显式声明数组类型，也无法做到。即便如此，如果现在必须没有它们，我肯定会想念这种便利。  
下一个特性延续了更简单创建和初始化对象的主题，但以不同的方式。

> 1. 深入 C# 类型推断机制 → 研究编译器如何分析表达式与上下文。  
> 2. 对比其他语言的类型推断（如 Java 的 `var`、TypeScript 的类型推导） → 理解不同设计哲学。  
> 3. 探索匿名类型与集合初始值设定项的配合 → 分析 C# 对灵活数据结构的支持。  
> 4. 考察类型推断在现代编程中的权衡 → 讨论可读性与简洁性的平衡。

# **对象和集合初始化器**

对象初始化器和集合初始化器使得创建带有初始值的新对象或集合变得非常容易，就像你可以在一个表达式中创建并填充一个数组一样。这一功能对于 LINQ 来说非常重要，因为查询在翻译时依赖这种机制；但事实证明，它在其他场景中同样非常有用。

不过，它确实要求类型是**可变的（mutable）**，如果你试图以函数式风格来编写代码，这一点可能会让人感到烦恼。但在可以应用的地方，它的效果非常好。在深入细节之前，我们先来看一个简单的示例。



## 对象和集合初始化器简介

以一个极度简化的示例为例，让我们考虑电子商务系统中的订单可能是什么样子。以下代码清单展示了建模订单、客户以及订单中单个条目的三个类。

清单 3.1 电子商务系统中的订单建模
```csharp
public class Order
{
    private readonly List<OrderItem> items = new List<OrderItem>();
    public string OrderId { get; set; }
    public Customer Customer { get; set; }
    public List<OrderItem> Items { get { return items; } }
}

public class Customer
{
    public string Name { get; set; }
    public string Address { get; set; }
}

public class OrderItem
{
    public string ItemId { get; set; }
    public int Quantity { get; set; }
}
```

如何创建一个订单？你需要创建一个 Order 实例并为其 OrderId 和 Customer 属性赋值。你无法对 Items 属性赋值，因为它是只读的。相反，你可以向它返回的列表中添加条目。如果你没有对象和集合初始化器，并且无法修改类以简化操作，下面的清单展示了可能如何实现。

清单 3.2 不使用对象和集合初始化器创建并填充订单
```csharp
var customer = new Customer();
customer.Name = "Jon";
customer.Address = "UK"; // 创建 Customer

var item1 = new OrderItem();
item1.ItemId = "abcd123";
item1.Quantity = 1; // 创建第一个 OrderItem

var item2 = new OrderItem();
item2.ItemId = "fghi456";
item2.Quantity = 2; // 创建第二个 OrderItem

var order = new Order();
order.OrderId = "xyz";
order.Customer = customer;
order.Items.Add(item1);
order.Items.Add(item2); // 创建订单
```

这段代码可以通过向各个类添加构造函数来根据参数初始化属性进行简化。即使有了对象和集合初始化器，我也会这样做。但为了简洁起见，请相信我，由于各种原因，这并不总是可行的。除此之外，你并不总是控制着你所使用类的代码。对象和集合初始化器使得创建和填充我们的订单变得简单得多，如下面的清单所示。

清单 3.3 使用对象和集合初始化器创建并填充订单
```csharp
var order = new Order
{
    OrderId = "xyz",
    Customer = new Customer { Name = "Jon", Address = "UK" },
    Items =
    {
        new OrderItem { ItemId = "abcd123", Quantity = 1 },
        new OrderItem { ItemId = "fghi456", Quantity = 2 }
    }
};
```

我不能代表所有人，但我发现清单 3.3 比清单 3.2 更易读。对象的结构在缩进中变得清晰，重复也更少。让我们更仔细地看看代码的每个部分。

> 1. 分析初始化器的语法糖本质 → 研究编译器如何将初始化器转换为标准赋值与集合操作。
> 2. 对比构造函数与初始化器的适用场景 → 理解在类设计时如何选择初始化策略。
> 3. 探究初始化器与不可变性的关系 → 分析只读属性或集合在初始化模式下的处理方式。
> 4. 扩展到更复杂的嵌套结构与LINQ查询 → 观察初始化器如何与其他C#特性协同提升表达力。



## 对象初始化器

在语法上，对象初始化器是大括号内的一系列成员初始化器。每个成员初始化器的形式为 `属性 = 初始化值`，其中 `属性` 是要初始化的字段或属性的名称，而 `初始化值` 可以是一个表达式、一个集合初始化器，或另一个对象初始化器。

**注意：** 对象初始化器最常用于属性，本章中我也是这样描述的。字段没有访问器，但显然也有对应的操作：读取字段而不是调用 get 访问器，写入字段而不是调用 set 访问器。

**对象初始化器只能用作构造函数调用或另一个对象初始化器的一部分。构造函数调用可以像往常一样指定参数，但如果你不想指定任何参数，你根本不需要参数列表，因此可以省略 `()`。没有参数列表的构造函数调用等效于提供空参数列表。**例如，以下两行代码是等效的：

```csharp
Order order = new Order() { OrderId = "xyz" };
Order order = new Order { OrderId = "xyz" };
```

仅当提供对象或集合初始化器时，才能省略构造函数参数列表。以下写法是无效的：

```csharp
Order order = new Order; // 无效
```

对象初始化器简单地说明了如何初始化其成员初始化器中提到的每个属性。如果 `初始化值` 部分（`=` 号的右侧）是一个普通表达式，则计算该表达式，并将该值传递给属性的 set 访问器。清单 3.3 中的大多数对象初始化器都是这样工作的。Items 属性使用了集合初始化器，我们稍后会看到。

如果 `初始化值` 是另一个对象初始化器，则永远不会调用 set 访问器。相反，会调用 get 访问器，然后将嵌套的对象初始化器应用于该属性返回的值。例如，清单 3.4 创建了一个 `HttpClient` 并修改了随每个请求发送的默认标头集。代码设置了 From 和 Date 标头，我选择它们只是因为它们是最容易设置的。

清单 3.4 使用嵌套对象初始化器修改新 HttpClient 的默认标头

```csharp
HttpClient client = new HttpClient
{
    DefaultRequestHeaders = // 调用 DefaultRequestHeaders 的属性 get 访问器
    {
        From = "user@example.com", // 调用 From 的属性 set 访问器
        Date = DateTimeOffset.UtcNow // 调用 Date 的属性 set 访问器
    }
};
```

清单 3.4 中的代码等效于以下代码：

```csharp
HttpClient client = new HttpClient();
var headers = client.DefaultRequestHeaders;
headers.From = "user@example.com";
headers.Date = DateTimeOffset.UtcNow;
```

单个对象初始化器可以包含嵌套对象初始化器、集合初始化器以及普通表达式的混合序列。说到集合初始化器，现在让我们来看看它们。



## 集合初始化器

在语法上，集合初始化器是大括号内以逗号分隔的元素初始化器列表。每个元素初始化器可以是一个单独的表达式，也可以是大括号内以逗号分隔的表达式列表。集合初始化器只能用作构造函数调用或对象初始化器的一部分。它们可以使用的类型还有进一步的限制，我们稍后会提到。在清单 3.3 中，你看到了一个作为对象初始化器一部分使用的集合初始化器。以下是该清单的再次展示，其中集合初始化器已用粗体标出：

```csharp
var order = new Order
{
    OrderId = "xyz",
    Customer = new Customer { Name = "Jon", Address = "UK" },
    Items =
    {
        new OrderItem { ItemId = "abcd123", Quantity = 1 },
        new OrderItem { ItemId = "fghi456", Quantity = 2 }
    }
};
```

不过，集合初始化器在创建新集合时可能更常用。例如，下面这行代码声明了一个新的字符串列表变量并填充该列表：

```csharp
var beatles = new List<string> { "John", "Paul", "Ringo", "George" };
```

编译器会将其编译为构造函数调用，然后是一系列对 `Add` 方法的调用：

```csharp
var beatles = new List<string>();
beatles.Add("John");
beatles.Add("Paul");
beatles.Add("Ringo");
beatles.Add("George");
```

但如果你使用的集合类型没有单参数的 `Add` 方法怎么办？这就是带大括号的元素初始化器的用武之地。在 `List<T>` 之后，第二常见的泛型集合可能是 `Dictionary<TKey, TValue>`，它有一个 `Add(key, value)` 方法。可以使用如下集合初始化器填充字典：

```csharp
var releaseYears = new Dictionary<string, int>
{
    { "Please please me", 1963 },
    { "Revolver", 1966 },
    { "Sgt. Pepper’s Lonely Hearts Club Band", 1967 },
    { "Abbey Road", 1970 }
};
```

编译器将每个元素初始化器视为单独的 `Add` 调用。如果元素初始化器是简单的、没有大括号的形式，则该值作为单个参数传递给 `Add`。我们的 `List<string>` 集合初始化器中的元素就是这种情况。如果元素初始化器使用大括号，它仍然被视为对 `Add` 的单个调用，但大括号内的每个表达式对应一个参数。前面的字典示例实际上等效于：

```csharp
var releaseYears = new Dictionary<string, int>();
releaseYears.Add("Please please me", 1963);
releaseYears.Add("Revolver", 1966);
releaseYears.Add("Sgt. Pepper’s Lonely Hearts Club Band", 1967);
releaseYears.Add("Abbey Road", 1970);
```

然后，重载解析像往常一样进行，以找到最合适的 `Add` 方法，如果存在任何泛型 `Add` 方法，还会执行类型推断。

**集合初始化器仅适用于实现了 `IEnumerable` 的类型，尽管它们不必实现 `IEnumerable<T>`。语言设计者查看了框架中具有 `Add` 方法的类型，并确定将它们区分为集合和非集合的最佳方法是查看它们是否实现了 `IEnumerable`。**举个例子来说明为什么这一点很重要，考虑 `DateTime.Add(TimeSpan)` 方法。`DateTime` 类型显然不是集合，因此能够这样写会很奇怪：

```csharp
DateTime invalid = new DateTime(2020, 1, 1) { TimeSpan.FromDays(10) }; // 无效
```

编译器在编译集合初始化器时从不使用 `IEnumerable` 的实现。我有时发现，在测试项目中创建具有 `Add` 方法和仅抛出 `NotImplementedException` 的 `IEnumerable` 实现的类型很方便。这对于构建测试数据可能很有用，但我不建议在生产代码中这样做。我希望有一个属性可以让我表达这种类型应该可用于集合初始化器而不需要实现 `IEnumerable`，但我怀疑这永远不会发生。



## **初始化为单个表达式的好处**

你可能想知道这一切与 LINQ 有什么关系。我说过 C# 3 中几乎所有的特性都是为 LINQ 做铺垫，那么对象和集合初始化器如何融入其中呢？答案是，其他 LINQ 特性要求代码能够以单个表达式表示。（例如，在查询表达式中，你不能编写一个 select 子句，该子句需要多个语句来为给定输入产生输出。）然而，在单个表达式中初始化新对象的能力不仅对 LINQ 有用。它对于简化字段初始化器、方法参数，甚至条件 `?:` 运算符中的操作数也很重要。例如，我发现它对于构建有用的查找表的静态字段初始化器特别有用。当然，初始化表达式变得越大，你可能越需要考虑将其分离出来。甚至对于特性本身来说，这也是递归重要的。例如，如果我们不能使用对象初始化器来创建 `OrderItem` 对象，那么集合初始化器填充 `Order.Items` 属性就不会那么方便了。

在本书的其余部分，每当我提到一个新增或改进的特性对单个表达式有特殊情况时（例如第 3.5 节中的 lambda 表达式或第 8.3 节中的表达式主体成员），都值得记住，对象和集合初始化器会立即使该特性比原本更有用。

对象和集合初始化器允许使用更简洁的代码来创建类型的实例并填充它，但它们确实要求你已经有一个合适的类型来构造。我们下一个特性，匿名类型，允许你创建对象，甚至无需预先声明对象的类型。这并不像听起来那么奇怪。

# **匿名类型**

匿名类型允许你构建可以以静态类型方式引用的对象，而无需预先声明类型。这听起来像是类型可能在运行时动态创建，但现实比这更微妙一些。我们将看看匿名类型在源代码中的样子，编译器如何处理它们，以及它们的一些限制。

## **语法和基本行为**

解释匿名类型最简单的方式是从一个示例开始。以下清单展示了一段简单的代码，用于创建一个具有 Name 和 Score 属性的对象。

清单 3.5 具有 Name 和 Score 属性的匿名类型
```csharp
var player = new
{
    Name = "Rajesh",
    Score = 3500
}; // 创建一个具有 Name 和 Score 属性的匿名类型的对象

Console.WriteLine("Player name: {0}", player.Name);
Console.WriteLine("Player score: {0}", player.Score); // 显示属性值
```

这个简短的例子展示了关于匿名类型的重要几点：
- 语法有点类似于对象初始化器，但没有指定类型名称；它只是 `new`、左大括号、属性、右大括号。这称为匿名对象创建表达式。属性值可以是嵌套的匿名对象创建表达式。
- 你在 `player` 变量的声明中使用了 `var`，因为该类型没有名称供你用来代替 `var`。（如果你使用 `object` 代替，声明也会有效，但几乎没什么用。）
- 这段代码仍然是静态类型的。Visual Studio 可以自动完成 `player` 变量的 Name 和 Score 属性。如果你忽略这一点并尝试访问不存在的属性（例如，如果你尝试使用 `player.Points`），编译器将引发错误。属性类型从分配给它们的值推断出来；`player.Name` 是一个字符串属性，`player.Score` 是一个整数属性。

这就是匿名类型的样子，但它们有什么用呢？这就是 LINQ 的用武之地。在执行查询时，无论是使用 SQL 数据库作为底层数据存储，还是使用对象集合，通常都需要一种特定的数据形状，它不是原始类型，并且在查询之外可能没有太多意义。

例如，假设你正在使用一组人构建查询，每个人都表达了自己喜欢的颜色。你可能希望结果是一个直方图：结果集合中的每个条目都是颜色和选择该颜色作为最爱的人数。那种表示喜欢颜色和类型的类型可能在其他任何地方都没有用，但在这种特定上下文中是有用的。匿名类型允许我们简洁地表达这些一次性情况，而不会丢失静态类型的好处。

> **与 Java 匿名类的比较**
> 如果你熟悉 Java，你可能想知道 C# 的匿名类型和 Java 的匿名类之间的关系。它们听起来很相似，但在语法和目的上都有很大不同。
>
> 历史上，Java 中匿名类的主要用途是实现接口或扩展抽象类以仅覆盖一两个方法。C# 的匿名类型不允许你实现接口或从 `System.Object` 以外的任何类派生；它们的目的更多是关于数据而不是可执行代码。
>

C# 在匿名对象创建表达式中提供了一个额外的简写形式，当你有效地从其他地方复制属性或字段，并且你愿意使用相同的名称时。这种语法称为投影初始化器。举个例子，让我们回到我们简化的电子商务数据模型。你有三个类：
- Order：OrderId、Customer、Items
- Customer：Name、Address
- OrderItem：ItemId、Quantity

在代码的某个时刻，你可能希望有一个对象包含特定订单项的所有这些信息。如果你有名为 `order`、`customer` 和 `item` 的相关类型的变量，你可以轻松使用匿名类型来表示扁平化的信息：

```csharp
var flattenedItem = new
{
    order.OrderId,
    CustomerName = customer.Name,
    customer.Address,
    item.ItemId,
    item.Quantity
};
```

在这个例子中，除了 CustomerName 之外的每个属性都使用了投影初始化器。结果与以下代码相同，该代码在匿名类型中显式指定了属性名称：

```csharp
var flattenedItem = new
{
    OrderId = order.OrderId,
    CustomerName = customer.Name,
    Address = customer.Address,
    ItemId = item.ItemId,
    Quantity = item.Quantity
};
```

当你正在执行查询并希望仅选择属性的子集，或者将多个对象的属性合并为一个时，投影初始化器最有用。如果你想在匿名类型中给出的属性名称与你复制的字段或属性的名称相同，编译器可以为你推断该名称。所以，与其这样写：

```csharp
SomeProperty = variable.SomeProperty
```

你可以只写：

```csharp
variable.SomeProperty
```

如果你要复制多个属性，投影初始化器可以显著减少源代码中的重复量。它很容易使得表达式短到足以保持在一行，或者长到值得每个属性单独一行。

> **重构和投影初始化器**
> 尽管说前面两个清单的结果是相同的，但这并不意味着它们在其他方面的行为也相同。考虑将 Address 属性重命名为 CustomerAddress。
>
> 在使用投影初始化器的版本中，匿名类型中的属性名称也会改变。在显式指定属性名称的版本中，它不会改变。根据我的经验，这很少成为问题，但值得注意这一差异。
>

我已经描述了匿名类型的语法，并且你知道结果对象具有可以像普通类型一样使用的属性。但是幕后发生了什么？





## 编译器生成的类型

尽管该类型从未在源代码中出现，但编译器确实会生成一个类型。运行时无需处理任何魔法；它只是看到一个类型，其名称恰好是在 C# 中无效的。该类型有一些有趣的方面。其中一些由规范保证；其他则不是。使用 Microsoft C# 编译器时，匿名类型具有以下特征：

- 它是一个类（保证）。
- 它的基类是 object（保证）。
- 它是密封的（不保证，尽管很难看出使其不密封会有什么用）。
- 属性都是只读的（保证）。
- 构造函数参数与属性同名（不保证；有时对反射有用）。
- 它在程序集内部（不保证；在使用动态类型时可能会令人烦恼）。
- 它重写了 `GetHashCode()` 和 `Equals()`，以便仅当所有属性都相等时两个实例才相等。（它处理属性为 null 的情况。）重写这些方法是保证的，但计算哈希码的具体方式不是。
- 它以有用的方式重写了 `ToString()`，并列出属性名称及其值。这不保证，但在诊断问题时非常有用。
- 该类型是泛型的，每个属性都有一个类型参数。具有相同属性名称但不同属性类型的多个匿名类型将对同一泛型类型使用不同的类型参数。这不保证，并且可能因编译器而异。
- 如果两个匿名对象创建表达式在同一程序集中使用相同的属性名称、相同的顺序和相同的属性类型，则保证结果是两个相同类型的对象。

最后一点对于变量重新分配和使用匿名类型的隐式类型数组很重要。根据我的经验，你很少会想要重新分配使用匿名类型初始化的变量，但它是可行的，这很好。例如，这是完全有效的：

```csharp
var player = new { Name = "Pam", Score = 4000 };
player = new { Name = "James", Score = 5000 };
```

同样，使用第 3.2.3 节中描述的隐式类型数组语法，通过匿名类型创建数组也是可以的：

```csharp
var players = new[]
{
    new { Name = "Priti", Score = 6000 },
    new { Name = "Chris", Score = 7000 },
    new { Name = "Amanda", Score = 8000 },
};
```

**请注意，要使两个匿名对象创建表达式使用相同的类型，属性必须具有相同的名称和类型，并且顺序相同。**例如，由于第二个数组元素中的属性顺序与其他元素不同，以下代码将无效：

```csharp
var players = new[]
{
    new { Name = "Priti", Score = 6000 },
    new { Score = 7000, Name = "Chris" },
    new { Name = "Amanda", Score = 8000 },
};
```

尽管每个数组元素单独都是有效的，但第二个元素的类型阻止了编译器推断数组类型。如果你添加了额外的属性或更改了其中一个属性的类型，情况也是如此。

尽管匿名类型在 LINQ 中很有用，但这并不使该特性成为解决所有问题的正确工具。让我们简要看一下你可能不想使用它们的地方。

 

> 1. 深入探究编译器生成匿名类型的实现细节 → 分析反射如何查看生成类的结构。
> 2. 研究匿名类型在LINQ查询中的实际编译过程 → 理解其如何优化查询性能。
> 3. 比较匿名类型与C# 9记录类型（record）的异同 → 分析两者在不可变数据模型上的演进关系。
> 4. 探索匿名类型在跨程序集或动态绑定时的限制 → 了解其使用边界与替代方案。

## 限制

当你想要一个仅数据的局部表示时，匿名类型非常有用。所谓局部，我指的是你感兴趣的数据形状仅在该特定方法内相关。一旦你想在多个地方表示相同的形状，就需要寻找不同的解决方案。虽然可以从方法返回匿名类型的实例或将其作为参数接受，但只能使用泛型或 `object` 类型。类型的匿名性使你无法在方法签名中表达它们。

在 C# 7 之前，如果你想在多个方法中使用通用的数据结构，通常需要为其声明自己的类或结构体。C# 7 引入了元组，正如你将在第 11 章看到的，它可以作为一个替代解决方案，具体取决于你希望封装的程度。**说到封装，匿名类型基本上不提供任何封装。你无法在类型中放置任何验证或为其添加额外行为。如果你发现自己想这样做，这可能是一个很好的迹象，表明你应该创建自己的类型。**

最后，我之前提到，由于匿名类型是内部的，通过 C# 4 的动态类型在程序集之间使用匿名类型变得更加困难。我通常在 MVC Web 应用程序中看到这种情况，其中页面的模型可能是使用匿名类型构建的，然后在视图中使用动态类型（你将在第 4 章看到）访问。如果两段代码在同一程序集中，或者包含模型代码的程序集使用 `[InternalsVisibleTo]` 使内部成员对包含视图代码的程序集可见，那么这可以工作。根据你使用的框架，安排其中任何一种都可能很尴尬。考虑到静态类型的好处，我通常建议将模型声明为常规类型。这比使用匿名类型需要更多的前期工作，但从长远来看可能会节省你的时间。

**注意**：Visual Basic 也有匿名类型，但它们的行为并不完全相同。在 C# 中，所有属性都用于确定相等性和哈希码，并且它们都是只读的。在 VB 中，只有使用 `Key` 修饰符声明的属性才表现为这样。非关键属性是读/写的，并且不影响相等性或哈希码。

我们已经完成了大约一半的 C# 3 特性，到目前为止，它们都与数据有关。接下来的特性更多地关注可执行代码，首先是 lambda 表达式，然后是扩展方法。



> 1. 分析匿名类型在方法间传递的限制 → 研究泛型或object类型作为替代方案的实现。
> 2. 比较匿名类型与C#元组（Tuple）在数据传递中的适用场景 → 理解两者在封装性和可用性上的权衡。
> 3. 探究匿名类型在动态跨程序集场景中的解决方案 → 分析InternalsVisibleTo属性的应用与限制。
> 4. 研究现代C#中记录类型（record）对匿名类型使用场景的替代 → 了解不可变数据类型的演进方向。





# **Lambda 表达式**

在第 2 章中，你看到了匿名方法如何通过内联包含代码来更容易地创建委托实例，就像这样：

```csharp
Action<string> action = delegate(string message)
{
	// 使用匿名方法创建委托
    Console.WriteLine("In delegate: {0}", message);
}; 

action("Message"); // 调用委托
```

C# 3 中引入了 Lambda 表达式，使这更加简洁。术语 **匿名函数** 用于指代匿名方法和 Lambda 表达式。我将在本书的其余部分多处使用它，并且它在 C# 规范中被广泛使用。

**注意**：Lambda 表达式这个名称来自 lambda 演算，这是一个由阿隆佐·邱奇在 20 世纪 30 年代开创的数学和计算机科学领域。邱奇在他的函数表示法中使用了希腊字母 lambda (λ)，这个名字就此沿用。

对于语言设计者来说，投入如此多的精力来简化委托实例的创建有多种原因，但 LINQ 是最重要的一个。当你在 3.7 节中查看查询表达式时，你会看到它们实际上被转换为使用 Lambda 表达式的代码。不过，你也可以在不使用查询表达式的情况下使用 LINQ，而这几乎总是涉及到直接在源代码中使用 Lambda 表达式。

首先，我们将介绍 Lambda 表达式的语法，然后讨论它们行为的一些细节。最后，我们将讨论将代码表示为数据的表达式树。

## **Lambda 表达式语法**

Lambda 表达式的基本语法始终是以下形式：
`parameter-list => body`

然而，参数列表和函数体都有多种表示形式。在其最明确的形式中，Lambda 表达式的参数列表看起来像普通方法或匿名方法的参数列表。同样，Lambda 表达式的函数体可以是一个代码块：一系列语句都包含在一对大括号中。在这种形式下，Lambda 表达式看起来类似于匿名方法：

```csharp
Action<string> action = (string message) =>
{
    Console.WriteLine("In delegate: {0}", message);
};
action("Message");
```

到目前为止，这看起来并没有好多少；你只是用 `=>` 替换了 `delegate` 关键字。但特殊规则允许 Lambda 表达式变得更短。

首先，让函数体更简洁。如果函数体仅由一个 `return` 语句或单个表达式组成，则可以简化为该单个表达式。如果原来有 `return` 关键字，则将其移除。在前面的例子中，我们的 Lambda 表达式的函数体只是一个方法调用，因此你可以简化它：

```csharp
Action<string> action = (string message) => Console.WriteLine("In delegate: {0}", message);
```

你很快就会看到一个返回值的示例。像这样缩短的 Lambda 表达式被称为具有 **表达式体**，而使用大括号的 Lambda 表达式被称为具有 **语句体**。

接下来，如果编译器能够根据你尝试将 Lambda 表达式转换到的类型来推断参数类型，你可以使参数列表更短。Lambda 表达式本身没有类型，但可以转换为兼容的委托类型，编译器通常可以推断出参数类型作为该转换的一部分。例如，在前面的代码中，编译器知道 `Action<string>` 有一个类型为 `string` 的参数，因此它能够推断参数的类型。在实践中，所有这些特殊规则在很多情况下都适用，特别是在 LINQ 中。

现在你了解了语法，可以开始查看委托实例的行为，特别是在它捕获的任何变量方面。



## **捕获变量**

在第 2.3.2 节中，当我描述匿名方法中捕获的变量时，我承诺我们会在 Lambda 表达式的上下文中回到这个话题。这可能是 Lambda 表达式最令人困惑的部分。它无疑是 Stack Overflow 上许多问题的根源。

要从 Lambda 表达式创建委托实例，编译器会将 Lambda 表达式中的代码转换为某个地方的方法。然后，就像你拥有一个方法组一样，可以在运行时创建委托。本节展示了编译器执行的那种转换。我这样写，就好像编译器将源代码转换为不包含 Lambda 表达式的更多源代码，但编译器当然不需要那个转换后的源代码。它可以直接发出适当的 IL。

首先，回顾一下什么算是捕获的变量。在 Lambda 表达式中，你可以使用在该点常规代码中能够使用的任何变量。这可以是静态字段、实例字段（如果你在实例方法中编写 Lambda 表达式）、`this` 变量、方法参数或局部变量。所有这些都是捕获的变量，因为它们是在 Lambda 表达式直接上下文之外声明的变量。将其与 Lambda 表达式的参数或在 Lambda 表达式内部声明的局部变量进行比较；那些不是捕获的变量。以下清单显示了一个捕获各种变量的 Lambda 表达式。然后你将查看编译器如何处理该代码。

清单 3.6 在 Lambda 表达式中捕获变量
```csharp
class CapturedVariablesDemo
{
    private string instanceField = "instance field";

    public Action<string> CreateAction(string methodParameter)
    {
        string methodLocal = "method local";
        string uncaptured = "uncaptured local";

        Action<string> action = lambdaParameter =>
        {
            string lambdaLocal = "lambda local";
            Console.WriteLine("Instance field: {0}", instanceField);
            Console.WriteLine("Method parameter: {0}", methodParameter);
            Console.WriteLine("Method local: {0}", methodLocal);
            Console.WriteLine("Lambda parameter: {0}", lambdaParameter);
            Console.WriteLine("Lambda local: {0}", lambdaLocal);
        };
        methodLocal = "modified method local";
        return action;
    }
}

// 在其他代码中
var demo = new CapturedVariablesDemo();
Action<string> action = demo.CreateAction("method argument");
action("lambda argument");
```

这里涉及很多变量：
- `instanceField` 是 `CapturedVariablesDemo` 类中的一个实例字段，并被 Lambda 表达式捕获。
- `methodParameter` 是 `CreateAction` 方法中的一个参数，并被 Lambda 表达式捕获。
- `methodLocal` 是 `CreateAction` 方法中的一个局部变量，并被 Lambda 表达式捕获。
- `uncaptured` 是 `CreateAction` 方法中的一个局部变量，但它从未被 Lambda 表达式使用，因此未被其捕获。
- `lambdaParameter` 是 Lambda 表达式本身的一个参数，因此它不是捕获的变量。
- `lambdaLocal` 是 Lambda 表达式中的一个局部变量，因此它不是捕获的变量。

重要的是要理解，Lambda 表达式捕获的是变量本身，而不是创建委托时变量的值。如果你在创建委托和调用委托之间的时间修改了任何捕获的变量，输出将反映这些更改。同样，Lambda 表达式可以更改捕获变量的值。编译器如何使这一切工作？它如何确保在调用委托时所有这些变量仍然可用？

**使用生成的类实现捕获的变量**
需要考虑三种主要情况：
- 如果根本没有捕获任何变量，编译器可以创建一个静态方法。不需要额外的上下文。
- 如果捕获的唯一变量是实例字段，编译器可以创建一个实例方法。捕获一个实例字段相当于捕获 100 个实例字段，因为你只需要访问 `this`。
- 如果捕获了局部变量或参数，编译器会创建一个私有的嵌套类来包含该上下文，然后在该类中创建一个包含 Lambda 表达式代码的实例方法。包含 Lambda 表达式的方法被更改为使用该嵌套类来访问所有捕获的变量。

**实现细节可能有所不同**
你可能会看到我所描述的一些变化。例如，对于没有捕获变量的 Lambda 表达式，编译器可能会创建一个具有单个实例的嵌套类，而不是静态方法。根据委托创建的确切方式，执行委托的效率可能存在细微差别。在本节中，我描述了编译器为使捕获的变量可用而必须做的最小工作。如果需要，它可以引入更多的复杂性。

最后一种情况显然是最复杂的，因此我们将重点讨论它。让我们从清单 3.6 开始。提醒一下，以下是创建 Lambda 表达式的方法；为了简洁起见，我省略了类声明：
```csharp
public Action<string> CreateAction(string methodParameter)
{
    string methodLocal = "method local";
    string uncaptured = "uncaptured local";

    Action<string> action = lambdaParameter =>
    {
        string lambdaLocal = "lambda local";
        Console.WriteLine("Instance field: {0}", instanceField);
        Console.WriteLine("Method parameter: {0}", methodParameter);
        Console.WriteLine("Method local: {0}", methodLocal);
        Console.WriteLine("Lambda parameter: {0}", lambdaParameter);
        Console.WriteLine("Lambda local: {0}", lambdaLocal);
    };
    methodLocal = "modified method local";
    return action;
}
```

正如我之前描述的，编译器为它需要的额外上下文创建一个私有的嵌套类，然后在该类中为 Lambda 表达式中的代码创建一个实例方法。上下文存储在嵌套类的实例变量中。在我们的例子中，这意味着：
- 对 `CapturedVariablesDemo` 原始实例的引用，以便稍后可以访问 `instanceField`
- 用于捕获的方法参数的字符串变量
- 用于捕获的局部变量的字符串变量

以下清单显示了嵌套类以及 `CreateAction` 方法如何使用它。

清单 3.7 带有捕获变量的 Lambda 表达式的转换
```csharp
private class LambdaContext
{
    public CapturedVariablesDemoImpl originalThis; // 生成的类来保存捕获的变量
    public string methodParameter; // 捕获的变量
    public string methodLocal;
}

public void Method(string lambdaParameter) // Lambda 表达式的主体变为实例方法。
{
    string lambdaLocal = "lambda local";
    Console.WriteLine("Instance field: {0}", originalThis.instanceField);
    Console.WriteLine("Method parameter: {0}", methodParameter);
    Console.WriteLine("Method local: {0}", methodLocal);
    Console.WriteLine("Lambda parameter: {0}", lambdaParameter);
    Console.WriteLine("Lambda local: {0}", lambdaLocal);
}

Action<string> CreateAction(string methodParameter)
{
    LambdaContext context = new LambdaContext();
    context.originalThis = this;
    context.methodParameter = methodParameter;
    context.methodLocal = "method local"; // 生成的类用于所有捕获的变量。
    string uncaptured = "uncaptured local";

    Action<string> action = context.Method;
    context.methodLocal = "modified method local";
    return action;
}
```

请注意在 `CreateAction` 方法末尾附近如何修改 `context.methodLocal`。当最终调用委托时，它将“看到”该修改。同样，如果委托修改了任何捕获的变量，每次调用都会看到之前调用的结果。这只是强调编译器确保捕获的是变量，而不是其值的快照。

在清单 3.6 和 3.7 中，你只需为捕获的变量创建一个单一的上下文。用规范的术语来说，每个局部变量只被实例化一次。让我们把事情弄得更复杂一些。

**局部变量的多次实例化**
为了更简单一些，这次你将捕获一个局部变量，不捕获参数或实例字段。以下清单显示了一个创建操作列表然后逐个执行它们的方法。每个操作捕获一个 `text` 变量。

清单 3.8 多次实例化局部变量
```csharp
static List<Action> CreateActions()
{
    List<Action> actions = new List<Action>();
    for (int i = 0; i < 5; i++)
    {
        string text = string.Format("message {0}", i); // 在循环内声明一个局部变量
        actions.Add(() => Console.WriteLine(text)); // 在 Lambda 表达式中捕获变量
    }
    return actions;
}

// 在其他代码中
List<Action> actions = CreateActions();
foreach (Action action in actions)
{
    action();
}
```

`text` 在循环内部声明这一事实确实非常重要。每次你到达该声明时，变量都会被实例化。每个 Lambda 表达式捕获该变量的不同实例。实际上有五个不同的 `text` 变量，每个都被单独捕获。它们是完全独立的变量。尽管这段代码在初始赋值后碰巧没有修改它们，但它当然可以在 Lambda 表达式内部或循环内的其他地方进行修改。修改一个变量不会影响其他变量。

编译器通过为每个实例化创建生成类型的不同实例来模拟这种行为。因此，清单 3.8 的 `CreateAction` 方法可以转换为以下清单。 

>
> **探索路线**：
>
> 1. 分析闭包的内存管理 → 研究捕获变量的生命周期与垃圾回收的关系。
> 2. 探究多次实例化场景下的性能影响 → 比较循环内捕获变量与外部声明变量的差异。
> 3. 研究C#中闭包与函数式编程的关系 → 理解Lambda表达式如何支持函数式范式。
> 4. 探索现代C#中局部函数（local function）对变量捕获的改进 → 分析其如何避免某些闭包陷阱。



清单 3.9 创建多个上下文实例，每个实例化一个
```csharp
private class LambdaContext
{
    public string text;
}

public void Method()
{
    Console.WriteLine(text);
}

static List<Action> CreateActions()
{
    List<Action> actions = new List<Action>();
    for (int i = 0; i < 5; i++)
    {
        LambdaContext context = new LambdaContext(); // 为每次循环迭代创建新上下文
        context.text = string.Format("message {0}", i);
        actions.Add(context.Method); // 使用上下文创建操作
    }
    return actions;
}
```

希望这仍然讲得通。你从为 Lambda 表达式使用单一上下文，转变为为循环的每次迭代使用一个上下文。我将用一个更复杂的例子来结束关于捕获变量的讨论，这个例子混合了两种情况。

**从多个作用域捕获变量**
`text` 变量的作用域意味着它为循环的每次迭代实例化一次。但是，单个方法内可能存在多个作用域，每个作用域可以包含局部变量声明，而单个 Lambda 表达式可以从多个作用域捕获变量。清单 3.10 给出了一个示例。你创建两个委托实例，每个实例捕获两个变量。它们都捕获相同的 `outerCounter` 变量，但每个捕获一个独立的 `innerCounter` 变量。委托只是打印计数器的当前值并递增它们。你执行每个委托两次，这使得捕获变量之间的区别变得清晰。

清单 3.10 从多个作用域捕获变量
```csharp
static List<Action> CreateCountingActions()
{
    List<Action> actions = new List<Action>();
    int outerCounter = 0; // 被两个委托捕获的一个变量

    for (int i = 0; i < 2; i++)
    {
        int innerCounter = 0; // 每次循环迭代的新变量
        Action action = () =>
        {
            Console.WriteLine(
                "Outer: {0}; Inner: {1}",
                outerCounter, innerCounter); // 显示并递增计数器
            outerCounter++;
            innerCounter++;
        };
        actions.Add(action);
    }
    return actions;
}

// 在其他代码中
List<Action> actions = CreateCountingActions();
actions[0]();
actions[0]();
actions[1]();
actions[1](); // 每次委托调用两次
```

清单 3.10 的输出如下：
```
Outer: 0; Inner: 0
Outer: 1; Inner: 1
Outer: 2; Inner: 0
Outer: 3; Inner: 1
```

前两行由第一个委托打印。后两行由第二个委托打印。正如我在清单前所述，两个委托使用相同的外部计数器，但它们有独立的内部计数器。

编译器如何处理这个？每个委托需要自己的上下文，但该上下文还需要引用一个共享的上下文。编译器创建两个私有嵌套类而不是一个。以下清单展示了编译器如何处理清单 3.10 的示例。

清单 3.11 从多个作用域捕获变量导致多个类
```csharp
private class OuterContext
{
    public int outerCounter; // 外部作用域的上下文
}

private class InnerContext
{
    public OuterContext outerContext; // 内部作用域的上下文，引用外部上下文
    public int innerCounter;
}

public void Method() // 用于创建委托的方法
{
    Console.WriteLine(
        "Outer: {0}; Inner: {1}",
        outerContext.outerCounter, innerCounter);
    outerContext.outerCounter++;
    innerCounter++;
}

static List<Action> CreateCountingActions()
{
    List<Action> actions = new List<Action>();
    OuterContext outerContext = new OuterContext(); // 创建单一外部上下文
    outerContext.outerCounter = 0;

    for (int i = 0; i < 2; i++)
    {
        InnerContext innerContext = new InnerContext(); // 每次循环迭代创建内部上下文
        innerContext.outerContext = outerContext;
        innerContext.innerCounter = 0;
        Action action = innerContext.Method;
        actions.Add(action);
    }
    return actions;
}
```

你很少需要像这样查看生成的代码，但这在性能方面可能会产生影响。如果你在性能关键的代码中使用 Lambda 表达式，你应该注意将创建多少个对象来支持它捕获的变量。

我还可以给出更多示例，比如在同一作用域中使用多个 Lambda 表达式捕获不同的变量集，或者在值类型的方法中使用 Lambda 表达式。我发现探索编译器生成的代码很有趣，但你大概不会想要一整本这样的书。如果你想知道编译器如何处理特定的 Lambda 表达式，很容易对结果运行反编译器或 ildasm。

到目前为止，你只看到了将 Lambda 表达式转换为委托，这你已经可以用匿名方法做到。然而，Lambda 表达式还有另一个超能力：它们可以转换为表达式树。

>
> **探索路线**：
>
> 1. 分析嵌套类结构的内存布局 → 研究闭包在堆上的分配与生命周期。
> 2. 探究捕获变量对多线程编程的影响 → 理解闭包在并发环境下的线程安全问题。
> 3. 研究C#编译器的优化策略 → 了解不同场景下编译器如何减少闭包开销。
> 4. 探索表达式树（Expression Tree）与委托的转换机制 → 分析Lambda表达式如何同时支持两种编译模式。



## **表达式树**

表达式树是将代码表示为数据的一种形式。这是 LINQ 能够与 SQL 数据库等数据提供程序高效协作的核心。你用 C# 编写的代码可以在运行时被分析并转换为 SQL。

委托提供的是可以运行的代码，而表达式树提供的是可以检查的代码，有点像反射。虽然你可以直接在代码中构建表达式树，但更常见的是通过将 lambda 表达式转换为表达式树来让编译器为你完成这项工作。以下清单通过创建一个简单的表达式树（仅用于将两个数字相加）来给出一个简单的示例。

清单 3.12 用于将两个整数相加的简单表达式树
```csharp
Expression<Func<int, int, int>> adder = (x, y) => x + y;
Console.WriteLine(adder);
```

考虑到这只有两行代码，其中发生了很多事情。让我们从输出开始。如果你尝试打印一个普通的委托，结果将只是类型，没有行为的指示。然而，清单 3.12 的输出准确地显示了表达式树所做的事情：
```
(x, y) => x + y
```

编译器并不是通过在某个地方硬编码字符串来作弊的。该字符串表示是从表达式树构造出来的。这表明代码在运行时可以被检查，这正是表达式树的全部意义所在。

让我们看看 `adder` 的类型：`Expression<Func<int, int, int>>`。最简单的方法是将其分为两部分：`Expression<TDelegate>` 和 `Func<int, int, int>`。第二部分用作第一部分的类型参数。第二部分是一个具有两个整数参数和一个整数返回类型的委托类型。（返回类型由最后一个类型参数表示，因此 `Func<string, double, int>` 将接受一个字符串和一个双精度浮点数作为输入，并返回一个整数。）

`Expression<TDelegate>` 是与 `TDelegate` 关联的表达式树类型，`TDelegate` 必须是一个委托类型。（这不是以类型约束的形式表达的，但在运行时强制执行。）这只是表达式树涉及的众多类型之一。它们都位于 `System.Linq.Expressions` 命名空间中。非泛型的 `Expression` 类是所有其他表达式类型的抽象基类，它也用作创建具体子类实例的工厂方法的便捷容器。

我们的 `adder` 变量类型是一个接受两个整数并返回整数的函数的表达式树表示。然后你使用 lambda 表达式为那个变量赋值。编译器生成代码以在运行时构建适当的表达式树。在这种情况下，它相当简单。你也可以自己编写相同的代码，如下面的清单所示。

清单 3.13 手工编写代码创建用于将两个整数相加的表达式树
```csharp
ParameterExpression xParameter = Expression.Parameter(typeof(int), "x");
ParameterExpression yParameter = Expression.Parameter(typeof(int), "y");
Expression body = Expression.Add(xParameter, yParameter);
ParameterExpression[] parameters = new[] { xParameter, yParameter };
Expression<Func<int, int, int>> adder =
    Expression.Lambda<Func<int, int, int>>(body, parameters);
Console.WriteLine(adder);
```

这是一个小例子，但它仍然比 lambda 表达式冗长得多。当你添加方法调用、属性访问、对象初始化器等时，它会变得复杂且容易出错。这就是为什么编译器能够通过将 lambda 表达式转换为表达式树来为你完成这项工作如此重要。不过，围绕这一点有一些规则。

**转换为表达式树的限制**
最重要的限制是，只有表达式体的 lambda 表达式才能转换为表达式树。尽管我们之前的 `(x, y) => x + y` lambda 表达式没问题，但以下代码会导致编译错误：
```csharp
Expression<Func<int, int, int>> adder = (x, y) => { return x + y; };
```

自 .NET 3.5 以来，表达式树 API 已经扩展，包含了代码块和其他构造，但 C# 编译器仍然有这个限制，并且这与为 LINQ 使用表达式树是一致的。这也是对象和集合初始化器如此重要的一个原因：它们允许在单个表达式中捕获初始化，这意味着它可以在表达式树中使用。

此外，lambda 表达式不能使用赋值运算符，也不能使用 C# 4 的动态类型或 C# 5 的异步功能。（尽管对象和集合初始化器确实使用了 `=` 符号，但在那种上下文中它不是赋值运算符。）



**将表达式树编译为委托**
正如我之前提到的，针对远程数据源执行查询的能力并非表达式树的唯一用途。它们可以是在运行时动态构造高效委托的强大方式，尽管这通常是一个至少部分表达式树是通过手写代码而非从 lambda 表达式转换而来的领域。

`Expression<TDelegate>` 有一个 `Compile()` 方法，它返回委托。然后你可以像处理任何其他委托一样处理这个委托。作为一个简单的例子，下面的清单使用我们之前的 adder 表达式树，将其编译为委托，然后调用它，产生输出 5。

清单 3.14 将表达式树编译为委托并调用结果
```csharp
Expression<Func<int, int, int>> adder = (x, y) => x + y;
Func<int, int, int> executableAdder = adder.Compile(); // 将表达式树编译为委托
Console.WriteLine(executableAdder(2, 3)); // 正常调用委托
```

这种方法可以与反射结合使用，用于属性访问和方法调用，以生成委托并缓存它们。结果就像你手工编写了等效代码一样高效。对于单个方法调用或属性访问，已经有直接创建委托的方法，但有时你需要额外的转换或操作步骤，这些步骤可以轻松地用表达式树表示。

当我们把所有内容结合起来时，我们会回到为什么表达式树在 LINQ 中如此重要。你只需要再看两个语言特性。接下来是扩展方法。

> **探索路线**：
>
> 1. 探究表达式树编译的性能优化 → 分析编译后委托与直接编码的性能差异。
> 2. 研究表达式树与反射的结合应用 → 了解其在动态调用和框架开发中的高级用法。
> 3. 分析扩展方法的编译器实现原理 → 理解`this`参数如何被处理。
> 4. 结合表达式树和扩展方法 → 探索如何构建动态LINQ查询提供程序。



# **扩展方法**

扩展方法在初次描述时听起来似乎毫无意义。它们是静态方法，但可以基于其第一个参数，像实例方法一样被调用。假设你有这样一个静态方法调用：

```csharp
ExampleClass.Method(x, y);
```
如果你将 `ExampleClass.Method` 转换为扩展方法，则可以这样调用：
```csharp
x.Method(y);
```
这就是扩展方法的全部功能。这是 C# 编译器执行的最简单的转换之一。然而，在将方法调用链式连接时，它对代码的可读性产生了巨大影响。稍后我们将通过 LINQ 的实际示例来了解这一点，但首先让我们看看语法。

## **声明扩展方法**

通过在第一个参数前添加 `this` 关键字来声明扩展方法。该方法必须在非嵌套、非泛型的静态类中声明，并且在 C# 7.2 之前，第一个参数不能是 `ref` 参数。（你将在第 13.5 节中了解更多相关信息。）虽然包含该方法的类不能是泛型的，但扩展方法本身可以是泛型方法。

第一个参数的类型有时被称为扩展方法的 **目标** 或 **扩展类型**。（遗憾的是，规范没有给这个概念命名。）

以 Noda Time 中的一个例子为例，我们有一个将 `DateTimeOffset` 转换为 `Instant` 的扩展方法。`Instant` 结构体中已经有一个静态方法可以完成此转换，但将其作为扩展方法也很实用。清单 3.15 展示了该方法的代码。这一次，我包含了命名空间声明，因为这在了解 C# 编译器如何查找扩展方法时非常重要。

清单 3.15 Noda Time 中针对 DateTimeOffset 的 ToInstant 扩展方法
```csharp
using System;

namespace NodaTime.Extensions
{
    public static class DateTimeOffsetExtensions
    {
        public static Instant ToInstant(this DateTimeOffset dateTimeOffset)
        {
            return Instant.FromDateTimeOffset(dateTimeOffset);
        }
    }
}
```
编译器会将 `[Extension]` 属性添加到方法及其声明类上，仅此而已。此属性位于 `System.Runtime.CompilerServices` 命名空间中。它是一个标记，表明开发人员应能够像调用 `DateTimeOffset` 中声明的实例方法一样调用 `ToInstant()`。

## **调用扩展方法**

你已经看到了调用扩展方法的语法：你可以像在第一个参数的类型上调用实例方法一样调用它。但你需要确保编译器能够找到该方法。

首先，有一个优先级问题：如果存在适用于方法调用的常规实例方法，编译器总是会优先选择实例方法而不是扩展方法。扩展方法是否具有“更好”的参数并不重要；如果编译器能够使用实例方法，它甚至不会查找扩展方法。

在穷尽了对实例方法的搜索之后，编译器将根据调用代码所在的命名空间和任何存在的 `using` 指令来查找扩展方法。假设你正在从 `CSharpInDepth.Chapter03` 命名空间中的 `ExtensionMethodInvocation` 类进行调用。以下清单展示了如何做到这一点，为编译器提供查找扩展方法所需的所有信息。

清单 3.16 在 Noda Time 外部调用 ToInstant() 扩展方法
```csharp
using NodaTime.Extensions; // 导入 NodaTime.Extensions 命名空间
using System;

namespace CSharpInDepth.Chapter03
{
    class ExtensionMethodInvocation
    {
        static void Main()
        {
            var currentInstant = DateTimeOffset.UtcNow.ToInstant(); // 调用扩展方法
            Console.WriteLine(currentInstant);
        }
    }
}
```
编译器将在以下位置查找扩展方法：
- `CSharpInDepth.Chapter03` 命名空间中的静态类。
- `CSharpInDepth` 命名空间中的静态类。
- 全局命名空间中的静态类。
- 通过 `using` 命名空间指令指定的命名空间中的静态类。（这些是仅指定命名空间的 `using` 指令，例如 `using System;`。）
- 仅在 C# 6 中，通过 `using static` 指令指定的静态类。我们将在第 10.1 节中讨论这一点。

编译器有效地从最深的命名空间向外工作到全局命名空间，并在每个步骤中查找该命名空间中的静态类，或者由命名空间声明中的 `using` 指令提供的类。排序的细节几乎从不重要。如果你发现移动 `using` 指令会改变所使用的扩展方法，那么最好重命名其中一个。但重要的是要理解，在每个步骤中，可能会找到多个适用于调用的扩展方法。在这种情况下，编译器在该步骤中找到的所有扩展方法之间执行正常的重载解析。在编译器找到要调用的正确方法后，它为调用生成的 IL 与你编写常规静态方法调用而不是使用其扩展方法功能时完全相同。

> **扩展方法可以在 null 值上调用**
> 扩展方法在处理 null 值方面与实例方法不同。回顾我们最初的例子：
> ```csharp
> x.Method(y);
> ```
> 如果 `Method` 是实例方法，并且 `x` 是 null 引用，则会抛出 `NullReferenceException`。相反，如果 `Method` 是扩展方法，即使 `x` 为 null，也会以 `x` 作为第一个参数调用该方法。有时，方法会指定第一个参数不能为 null，在这种情况下，它应该验证并抛出 `ArgumentNullException`。在其他情况下，扩展方法可能被明确设计为优雅地处理第一个参数为 null 的情况。
>

让我们回到为什么扩展方法对 LINQ 很重要。现在是我们的第一个查询的时候了。

## **链式方法调用**

清单 3.17 展示了一个简单的查询。它接收一个单词序列，按长度过滤，以自然方式排序，然后将其转换为大写。它使用了 Lambda 表达式和扩展方法，但没有使用其他 C# 3 特性。我们将在本章末尾将所有内容整合在一起。现在，我想重点关注这段简单代码的可读性。

清单 3.17 字符串上的简单查询
```csharp
string[] words = { "keys", "coat", "laptop", "bottle" }; // 一个简单的数据源
IEnumerable<string> query = words
    .Where(word => word.Length > 4) // 过滤、排序、转换
    .OrderBy(word => word)
    .Select(word => word.ToUpper());

foreach (string word in query)
{
    Console.WriteLine(word); // 显示结果
}
```

请注意代码中 `Where`、`OrderBy` 和 `Select` 调用的顺序。这就是操作发生的顺序。LINQ 的惰性求值和尽可能流式处理的特性使得准确描述何时发生什么变得复杂，但查询的阅读顺序与其执行顺序相同。以下清单是相同的查询，但没有利用这些方法是扩展方法这一事实。

清单 3.18 未使用扩展方法的简单查询
```csharp
string[] words = { "keys", "coat", "laptop", "bottle" };
IEnumerable<string> query =
    Enumerable.Select(
        Enumerable.OrderBy(
            Enumerable.Where(words, word => word.Length > 4),
            word => word),
        word => word.ToUpper());
```

我已尽可能将清单 3.18 格式化为可读的形式，但它仍然很糟糕。调用在源代码中的布局顺序与它们执行的顺序相反：`Where` 是第一个要执行的操作，但却是清单中的最后一个方法调用。其次，不清楚哪个 Lambda 表达式对应哪个调用：`word => word.ToUpper()` 是 `Select` 调用的一部分，但这两段文本之间有大量代码。

你可以用另一种方式处理这个问题，将每个方法调用的结果赋值给一个局部变量，然后通过该变量进行方法调用。清单 3.19 展示了这样做的一种选择。（在这种情况下，你本可以一开始就声明 `query` 变量并在每一行重新赋值，但情况并非总是如此。）这次，为了简洁，我也使用了 `var`。

清单 3.19 分多步执行的简单查询
```csharp
string[] words = { "keys", "coat", "laptop", "bottle" };
var tmp1 = Enumerable.Where(words, word => word.Length > 4);
var tmp2 = Enumerable.OrderBy(tmp1, word => word);
var query = Enumerable.Select(tmp2, word => word.ToUpper());
```

这比清单 3.18 要好；操作回到了正确的顺序，并且很清楚哪个 Lambda 表达式用于哪个操作。但额外的局部变量声明分散了注意力，而且很容易用错变量。

当然，方法链的好处不仅限于 LINQ。将一个调用的结果作为另一个调用的起点是很常见的。但是扩展方法允许你以可读的方式对任何类型进行这种操作，而不是由类型本身声明支持链式调用的方法。`IEnumerable<T>` 对 LINQ 一无所知；它的唯一职责是表示一个通用序列。是 `System.Linq.Enumerable` 类添加了所有用于过滤、分组、连接等操作。

C# 3 本可以在此止步。到目前为止描述的特性已经为语言增添了许多能力，并使许多 LINQ 查询能够以完全可读的形式编写。但是当查询变得更加复杂，特别是当它们包含连接和分组时，直接使用扩展方法可能会变得复杂。于是，查询表达式登场了。

> 
>
> **探索路线**：
>
> 1. 探究方法链（Fluent API）的设计模式 → 分析扩展方法如何支持流畅接口。
> 2. 研究LINQ标准查询运算符的实现原理 → 理解`IEnumerable<T>`扩展方法的工作机制。
> 3. 比较链式调用与查询表达式的转换关系 → 分析编译器如何将查询表达式翻译为扩展方法调用。
> 4. 探索现代C#中`IAsyncEnumerable<T>`的异步流式处理 → 了解链式调用在异步场景下的演进。



# 查询表达式

尽管 C# 3 中几乎所有的特性都为 LINQ 做出了贡献，但只有查询表达式是 LINQ 特有的。查询表达式允许你使用查询特定的子句（`select`、`where`、`let`、`group by` 等）编写简洁的代码。然后，编译器将查询转换为非查询形式并进行常规编译。让我们从一个简短的例子开始，使其更清晰。提醒一下，在清单 3.17 中你有这样的查询：

```csharp
IEnumerable<string> query = words
    .Where(word => word.Length > 4)
    .OrderBy(word => word)
    .Select(word => word.ToUpper());
```
以下清单显示了写为查询表达式的相同查询。

清单 3.20 包含过滤、排序和投影的入门查询表达式
```csharp
IEnumerable<string> query = from word in words
                            where word.Length > 4
                            orderby word
                            select word.ToUpper();
```
清单 3.20 中加粗的部分是查询表达式，它确实非常简洁。`word` 作为参数在 lambda 表达式中的重复使用，被替换为在 `from` 子句中指定一次范围变量的名称，然后在其他每个子句中使用它。清单 3.20 中的查询表达式发生了什么？

## **查询表达式从 C# 转换到 C#**

在本书中，我已经用更多的 C# 源代码表达了许多语言特性。例如，在查看第 3.5.2 节中的捕获变量时，我展示了你可以编写的 C# 代码，以达到与使用 lambda 表达式相同的结果。这只是为了解释编译器生成的代码。我并不期望编译器生成任何 C#。规范描述了捕获变量的效果，而不是源代码转换。

查询表达式的工作方式不同。规范将查询表达式描述为在发生任何重载解析或绑定之前进行的语法转换。清单 3.20 中的代码不仅与清单 3.17 具有相同的最终效果；它在进一步处理之前确实被转换为清单 3.17 中的代码。语言对该进一步处理的结果没有特定的期望。在许多情况下，转换的结果将是对扩展方法的调用，但这并非语言规范所要求。它们也可以是实例方法调用，或是对名为 `Select`、`Where` 等属性返回的委托的调用。

查询表达式的规范规定了对某些方法存在的期望，但并没有要求所有这些方法都必须存在。例如，如果你编写了一个具有合适 `Select`、`OrderBy` 和 `Where` 方法的 API，即使你不能使用包含 `join` 子句的查询表达式，你也可以使用清单 3.20 中所示的那种查询。

虽然我们不会详细研究查询表达式中可用的每个子句，但我需要提请你注意两个相关概念。这些部分地证明了语言设计者将查询表达式引入该语言的合理性。

## **范围变量和透明标识符**

查询表达式引入了范围变量，它们与其他常规变量不同。它们充当查询每个子句中的每项输入。你已经看到了查询表达式开头的 `from` 子句如何引入一个范围变量。这里是清单 3.20 中的查询表达式再次展示，其中范围变量被高亮显示：

```
from word in words          // 在 from 子句中引入范围变量
where word.Length > 4       // 在后续子句中使用范围变量
orderby word
select word.ToUpper()
```
当只有一个范围变量时，这很容易理解，但初始的 `from` 子句并不是引入范围变量的唯一方式。引入新范围变量的子句最简单的例子可能就是 `let`。假设你想在查询中多次引用单词的长度，而不必每次都调用 `Length` 属性。例如，你可以根据它排序并将其包含在输出中。`let` 子句允许你编写如以下清单所示的查询。

清单 3.21 引入新范围变量的 let 子句
```csharp
from word in words
let length = word.Length
where length > 4
orderby length
select string.Format("{0}: {1}", length, word.ToUpper());
```
现在，你同时有两个范围变量在作用域内，从 `select` 子句中同时使用 `length` 和 `word` 可以看出。这就提出了一个问题：如何才能在查询转换中表示这一点。你需要一种方法来获取原始的单词序列，并有效地创建一个单词/长度对的序列。然后，在可以使用这些范围变量的子句中，你需要访问该对中的相关项。以下清单展示了编译器如何使用匿名类型来表示这对值，从而翻译清单 3.21。

清单 3.22 使用透明标识符的查询翻译
```csharp
words.Select(word => new { word, length = word.Length })
     .Where(tmp => tmp.length > 4)
     .OrderBy(tmp => tmp.length)
     .Select(tmp => string.Format("{0}: {1}", tmp.length, tmp.word.ToUpper()));
```
这里的名称 `tmp` 不是查询转换的一部分。规范使用 * 代替，并且没有指明在构建查询的表达式树表示时应给参数什么名称。名称无关紧要，因为在编写查询时你不会看到它。这被称为**透明标识符**。

我不打算详细介绍查询翻译的所有细节。这本身可能就是完整的一章。但我想提出透明标识符，原因有两点。首先，如果你了解如何引入额外的范围变量，当你反编译查询表达式看到它们时就不会感到惊讶。其次，根据我的经验，它们提供了使用查询表达式的最大动机。

>
> **探索路线**：
>
> 1. 深入探究查询表达式的完整转换规则 → 研究编译器如何处理`join`、`group by`等复杂子句。
> 2. 分析透明标识符的实现机制 → 理解编译器如何管理多个范围变量的生命周期。
> 3. 比较查询表达式与扩展方法链的适用场景 → 权衡可读性与灵活性的选择。
> 4. 探索现代C#中查询表达式的异步版本（如`IAsyncEnumerable`与异步LINQ） → 了解声明式查询在异步场景下的演进。





## 决定何时对 LINQ 使用哪种语法

查询表达式可能很吸引人，但它们并非总是表示查询的最简单方式。查询表达式总是以一个 `from` 子句开始，并以一个 `select` 或 `group by` 子句结束。这听起来很合理，但意味着如果你想要执行单个过滤操作的查询，例如，你最终会包含相当多的冗余代码。例如，如果我们只取基于单词查询的过滤部分，查询表达式会是这样：

```csharp
from word in words
where word.Length > 4
select word
```
与方法语法的查询版本进行比较：
```csharp
words.Where(word => word.Length > 4)
```
两者都编译为相同的代码，但对于如此简单的查询，我会使用第二种语法。

**注意**：对于不使用查询表达式语法的情况，并没有一个普遍通用的术语。我见过它被称为方法语法、点语法、流畅语法和 lambda 语法等。我将始终称之为方法语法，但如果你听到其他术语，不必试图寻找其含义上的细微差别。

即使查询变得稍微复杂一些，方法语法也可能更灵活。LINQ 中有许多方法没有对应的查询表达式语法，包括那些除了项目本身外还提供项目在序列中索引的 `Select` 和 `Where` 重载。此外，如果你希望在查询末尾有一个方法调用（例如，使用 `ToList()` 将结果具体化为 `List<T>`），你必须将整个查询表达式放在括号内，而使用方法语法时，你只需在末尾添加调用。

我并非像听起来那样贬低查询表达式。在许多情况下，两种语法选项之间没有明显的赢家，我可能会将我们之前过滤、排序、投影的例子归为此类。当编译器通过处理所有那些透明标识符为你做更多工作时，查询表达式才能真正大放异彩。当然，你完全可以手动完成所有操作，但我发现构建匿名类型作为结果并在后续步骤中解构它们很快就会变得烦人。查询表达式使这一切变得容易得多。

这一切的结果是，我强烈建议你熟悉这两种查询风格。如果你固执地总是使用或从不使用查询表达式，你将错失使代码更具可读性的机会。我们已经介绍了 C# 3 的所有特性，但我想花点时间退一步，展示它们如何组合在一起形成 LINQ。

# **最终成果：LINQ**

我不打算尝试涵盖当今可用的各种 LINQ 提供程序。我使用最多（目前为止）的 LINQ 技术是 LINQ to Objects，使用 `Enumerable` 静态类和委托。但为了展示所有部分如何发挥作用，让我们假设你有一个类似 Entity Framework 的查询。这不是你可以测试的真实代码，但如果你有一个合适的数据库结构，这将是可行的：

```csharp
var products = from product in dbContext.Products
               where product.StockCount > 0
               orderby product.Price descending
               select new { product.Name, product.Price };
```
在这个仅仅四行的单一示例中，使用了所有这些特性：
- 匿名类型，包括投影初始化器（用于仅选择产品的名称和价格）
- 使用 `var` 的隐式类型，因为否则你无法以有用的方式声明 `products` 变量的类型
- 查询表达式（在这种情况下可以不用，但对于更复杂的查询，它使事情简单得多）
- Lambda 表达式（这是查询表达式转换的结果）
- 扩展方法（由于 `dbContext.Products` 实现了 `IQueryable<Product>`，允许转换后的查询通过 `Queryable` 类表达）
- 表达式树（允许将查询中的逻辑作为数据传递给 LINQ 提供程序，以便可以将其转换为 SQL 并在数据库中高效执行）

去掉这些特性中的任何一个，LINQ 的实用性都会大打折扣。当然，没有表达式树你也可以进行内存中的集合处理。没有查询表达式你也可以编写可读的简单查询。不使用扩展方法，你也可以拥有包含所有相关方法的专用类。但所有这些特性完美地结合在一起。

**总结**
- C# 3 中的所有特性都以某种形式与处理数据相关，大多数是 LINQ 的关键部分。
- 自动实现的属性提供了一种简洁的方式来公开不需要任何额外行为的状态。
- 使用 `var` 关键字（以及数组）的隐式类型对于处理匿名类型是必需的，同时也可以避免冗长的重复。
- 对象和集合初始化器使初始化更简单、更可读。它们还允许初始化作为单个表达式发生，这对于 LINQ 的其他方面至关重要。
- 匿名类型允许你以轻量级的方式有效地创建一个仅用于单一局部目的的类型。
- Lambda 表达式提供了一种比匿名方法更简单的委托构造方式。它们还允许通过表达式树将代码表示为数据，LINQ 提供程序可以使用表达式树将 C# 查询转换为其他形式（如 SQL）。
- 扩展方法是静态方法，但可以在其他地方像实例方法一样调用。这允许为原本不是这样设计的类型编写流畅的接口。
- 查询表达式被翻译成更多使用 Lambda 表达式来表达查询的 C# 代码。尽管它们对于复杂查询很棒，但更简单的查询通常使用方法语法编写更容易。

>
> **探索路线**：
>
> 1. 深入研究查询表达式的完整转换规则 → 理解编译器如何将各种子句（如`join`、`group by`）翻译为扩展方法调用。
> 2. 对比分析`IEnumerable<T>`与`IQueryable<T>`的LINQ实现 → 探究表达式树在远程查询中的核心作用。
> 3. 实践复杂查询的构建 → 结合匿名类型、透明标识符和投影初始化器处理多源数据关联与聚合。
> 4. 探索现代C#（如C# 8+）对LINQ的增强 → 了解异步流（`IAsyncEnumerable`）和范围/索引对查询模式的影响。

