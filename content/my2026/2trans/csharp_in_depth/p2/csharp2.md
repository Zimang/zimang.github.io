---
weight: 1
title: "C#2"
---

**本章涵盖内容**

- 使用**泛型类型**和**泛型方法**编写灵活、安全的代码
- 用**可空值类型**表示信息的缺失
- 以相对简单的方式构建**委托**
- 实现**迭代器**而无需编写样板代码

如果你的C#经验足够久远，本章将提醒我们已走了多远，并让你由衷感激一支专注而智慧的语言设计团队。如果你从未在没有泛型的情况下编写过C#，你最终可能会疑惑：没有这些特性，C#当初是如何流行起来的？无论如何，你仍可能在这里发现未曾了解的特性或从未思考过的细节。

C# 2 发布（随 Visual Studio  2005）已逾十年，因此要为我们后视镜中的特性感到兴奋可能并不容易。但你不应低估它当时发布的重要性。升级过程也是痛苦的：从 C# 1 和  .NET 1.x 升级到 C# 2 和 .NET 2.0 在业界花费了很长时间才完成。随后的演进则快速得多。C# 2  的第一个特性，也是几乎所有开发者认为最重要的特性，便是：**泛型**。

# 泛型

泛型允许你编写通用代码，这些代码在编译时是类型安全的，可以在多个位置使用相同的类型，而无需预先知道该具体类型是什么。泛型最初引入时，主要用于集合，但在现代 C# 代码中，它们无处不在。它们可能最常被用于以下方面：

- **集合**（它们在集合中一如既往地有用）
- **委托**，特别是在 LINQ 中
- **异步代码**，其中 `Task<T>` 表示一个未来类型为 `T` 的值的承诺
- **可空值类型**，我将在 2.2 节中详细讨论

这绝非其用途的全部，但仅这四点就意味着 C# 程序员每天都在使用泛型。**集合**提供了解释泛型益处的最简单方式，因为你可以对比 .NET 1 中的集合和 .NET 2 中的泛型集合。

> **核心价值**：泛型的核心优势在于**编译时类型安全**和**代码复用**。它通过将类型参数化，使得一段逻辑可以服务于多种具体类型，同时编译器能进行严格的类型检查，避免了运行时的类型转换错误，这是与非泛型代码（如使用 `object` 的集合）的本质区别。
>
> **应用场景的扩展**：作者列举的四个场景清晰地展示了泛型如何从最初的“集合工具”演变为C#类型系统的**基石性构造块**：
>
> 1. **集合**：解决了 `ArrayList` 等非泛型集合的装箱拆箱和类型安全问题。
> 2. **委托（如 `Func<T>`，`Action<T>`）**：为高阶函数和LINQ提供了类型安全的函数抽象。
> 3. **异步（`Task<T>`）**：将异步操作与其结果类型优雅地绑定。
> 4. **可空值类型（`Nullable<T>`）**：为值类型体系增加了“值缺失”的语义。

## **通过示例入门：泛型之前的集合**

.NET 1 主要存在三类集合：

- **数组** —— 它们直接得到语言和运行时的支持。大小在初始化时固定。
- **基于对象的集合** —— 值（以及相关的键）在 API 中使用 `System.Object` 来描述。它们没有特定于集合的语言或运行时支持，但可以使用索引器和 `foreach` 语句等语言特性。`ArrayList` 和 `Hashtable` 是最常用的例子。
- **专用集合** —— 值在 API 中用特定类型描述，该集合只能用于该类型。例如，`StringCollection` 是一个字符串集合；其 API 看起来类似 `ArrayList`，但在任何涉及值的地方都使用 `String` 而不是 `Object`。

数组和专用集合是**静态类型**的，我指的是 API 会阻止你将错误类型的值放入集合，并且当你从集合中获取值时，你不需要将结果强制转换回你期望的类型。

**注意**：由于**数组协变性**，引用类型数组在存储值时只是基本安全。我认为数组协变性是一个早期的设计失误，这超出了本书的范围。Eric Lippert 在其关于协变和逆变的系列博客文章中对此进行了阐述，详见 http://mng.bz/gYPv。

让我们具体化：假设你想在一个方法（`GenerateNames`）中创建一个字符串集合，并在另一个方法（`PrintNames`）中打印这些字符串。你将看到三种保存姓名集合的选项——数组、`ArrayList` 和 `StringCollection`——并权衡每种方案的优缺点。每种情况的代码看起来都很相似（特别是 `PrintNames`），但请耐心听我解释。

我们将从数组开始。

> 数组协变性这个“早期的设计失误”，是一个关于**类型系统设计**和**向后兼容性**权衡的绝佳案例。它允许将 `string[]` 赋值给 `object[]` 变量，但这破坏了数组的写入类型安全（因为你可能通过 `object[]` 引用向 `string[]` 存入非字符串对象）。
>
> 这引出了一个更深层的思考：当一门语言或平台在演进过程中发现了早期设计的根本性缺陷，它应该如何应对？**是像Java那样（在泛型中）通过类型擦除来保持完全兼容但牺牲一些表达力，还是采取更激进但破坏性的改进？****C# 在引入泛型时，选择了在运行时保留类型信息的路径，这带来了更好的性能和表达能力，但也增加了运行时的复杂性。**这种**设计哲学上的抉择**，其影响远比一个特性的具体语法更为深远。

```c#
//Listing 2.1 Generating and printing names by using arrays
static string[] GenerateNames()
{
    string[] names = new string[4]; //size of array needs to be known at create time
    names[0] = "Gamma";
    names[1] = "Vlissides";
    names[2] = "Johnson";
    names[3] = "Helm";
    return names;
}
static void PrintNames(string[] names)
{
    foreach (string name in names)
    {
    	Console.WriteLine(name);
    }
}
```

**这里我没有使用数组初始化器，因为我想模拟一种情况：姓名只能逐个获取，例如从文件中读取。**请注意，你需要一开始就分配一个大小合适的数组。如果你确实是从文件中读取，那么你要么需要在开始之前知道有多少个姓名，要么就需要编写更复杂的代码。例如，你可以先分配一个数组，如果第一个数组填满了，就将内容复制到一个更大的数组中，依此类推。然后，如果你最终得到的数组比实际的姓名数量大，可能还需要考虑创建一个大小刚好合适的最终数组。

用来跟踪我们集合当前大小、重新分配数组等的代码是重复的，可以封装到一个类型中。事实上，这正是 `ArrayList` 所做的。

> `ArrayList` 通过封装动态扩容逻辑，简化了开发，但其基于 `object` 的设计引入了新的问题。这引出了一个常见的软件设计困境：**在解决一个问题的同时，是否会引入新的、甚至更复杂的问题？**
>
> 泛型集合 `List<T>` 的出现，可以看作是对这个困境的完美回应。它在保留 `ArrayList` 动态扩容优点的同时，通过编译时类型检查消除了类型安全问题，并避免了值类型的装箱拆箱开销。



```c#
//Listing 2.2  Generating and printing names by using ArrayList
static ArrayList GenerateNames()
{
    ArrayList names = new ArrayList();
    names.Add("Gamma");
    names.Add("Vlissides");
    names.Add("Johnson");
    names.Add("Helm");
    return names;
}
static void PrintNames(ArrayList names)
{
    foreach (string name in names)
    {
        //What happens if the ArrayList contains a nonstring?
    	Console.WriteLine(name);
    }
}
```



就我们的 `GenerateNames` 方法而言，这更简洁：你不需要在开始添加元素之前就知道有多少个姓名。但同样，也无法阻止你向集合中添加非字符串对象；`ArrayList.Add` 方法的参数类型就是 `Object`。

此外，尽管 `PrintNames` 方法在类型方面看起来安全，但实际上并非如此。集合可以包含任何类型的对象引用。如果你向集合中添加一个完全不同的类型（举个奇怪的例子，比如 `WebRequest`），然后尝试打印它，会发生什么？由于 `name` 变量的类型，`foreach` 循环隐藏了一个从 `object` 到 `string` 的隐式转换。这个转换可能会以正常方式失败，并抛出 `InvalidCastException`。因此，你解决了一个问题，却导致了另一个问题。有什么方法可以同时解决这两个问题吗？

`foreach (string name in names)` 这行代码看似自然，实则隐藏了一个危险的**运行时类型转换**。编译器信任开发者的判断，但若集合内包含非字符串对象，程序将在迭代时崩溃。这破坏了代码的健壮性。

```c#
//Listing 2.3 Generating and printing names by using StringCollection
static StringCollection GenerateNames()
{
    StringCollection names = new StringCollection();
    names.Add("Gamma");
    names.Add("Vlissides");
    names.Add("Johnson");
    names.Add("Helm");
    return names;
}
static void PrintNames(StringCollection names)
{
    foreach (string name in names)
    {
    	Console.WriteLine(name);
    }
}
```

代码清单 2.3 与清单 2.2 完全相同，只是将所有出现的 `ArrayList` 替换为 `StringCollection`。这正是 `StringCollection` 的全部意义：它应该像一个好用的通用集合，但专门用于处理字符串。`StringCollection.Add` 的参数类型是 `String`，因此你无法通过代码中的某些奇怪错误将 `WebRequest` 添加进去。这样做的结果是，在打印姓名时，你可以确信 `foreach` 循环不会遇到任何非字符串引用。（当然，你仍可能看到 `null` 引用。）

**如果你只需要处理字符串，那这很好。但如果你需要其他类型的集合，你只能期望框架中已经存在合适的集合类型，或者自己编写一个。这是一个如此常见的任务，以至于框架中提供了一个 `System.Collections.CollectionBase` 抽象类，以使这项工作不那么重复。此外，还可以使用代码生成器来避免全部手动编写。**

这解决了之前方案中的两个问题，但维护所有这些额外类型的成本实在太高了。随着代码生成器的变化，保持它们最新的维护成本很高。在编译时间、程序集大小、JIT 编译时间和内存中代码保持方面，都存在效率成本。最重要的是，跟踪所有可用集合类的人力成本也很高。

即使这些成本不是太高，你也会失去以静态类型方式编写适用于任何集合类型的方法的能力，并且无法在方法的其他参数或返回类型中使用集合的元素类型。例如，假设你想编写一个方法，将集合的前 N  个元素复制到一个新集合并返回。你可以编写一个返回 `ArrayList` 的方法，但这失去了静态类型的好处。如果你传入一个 `StringCollection`，你会希望返回一个 `StringCollection`。字符串这一特性是方法输入的一部分，也需要传播到输出。在使用 C# 1 时，你无法在语言中表达这一点。于是，泛型登场了。

## 泛型拯救世界

让我们直接看看 `GenerateNames`/`PrintNames` 代码的解决方案，使用 `List<T>` 泛型类型。`List<T>` 是一个集合，其中 `T` 是集合的元素类型——在我们的例子中是 `string`。你可以在所有地方用 `List<string>` 替换 `StringCollection`。

```c#
//Listing 2.4 Generating and printing names with List<T>
static List<string> GenerateNames()
{
    List<string> names = new List<string>();
    names.Add("Gamma");
    names.Add("Vlissides");
    names.Add("Johnson");
    names.Add("Helm");
    return names;
}
static void PrintNames(List<string> names)
{
    foreach (string name in names)
    {
    	Console.WriteLine(name);
    }
}
```

`List<T>` 解决了我们之前讨论的所有问题：

- 你不需要像数组那样预先知道集合的大小。
- 公开的 API 在需要引用元素类型的地方都使用 `T`，因此你知道 `List<string>` 将只包含字符串引用。如果你尝试添加其他任何内容，都会得到编译时错误，这与 `ArrayList` 不同。
- 你可以将其用于任何元素类型，而无需担心生成代码和管理结果，这与 `StringCollection` 及类似类型不同。

泛型还解决了将元素类型表达为方法输入的问题。为了更深入地探讨这方面，你需要更多的术语。

> 泛型通过 `List<T>` 解决了集合的问题，但它的野心远不止于此。`List<T>` 的成功揭示了一个更宏大的图景：如果集合可以参数化，那么**任何其行为逻辑与特定类型紧密相关，但又希望该逻辑能复用于多种类型的结构**，都可以从泛型中受益。
>
> 这正是为什么泛型迅速成为了C#类型系统的核心。从 `Nullable<T>` 到 `Task<T>`，从 `Func<T>` 到 `IEnumerable<T>`，泛型使得我们可以创建**类型安全的容器、计算、委托和序列**。它不仅仅是一个特性，更是一种新的思维方式——**类型驱动的编程**，鼓励我们思考如何将算法和数据结构从具体的类型中解耦，从而创造出更通用、更健壮、更可复用的抽象。

> **类型参数parameter 与类型实参ARGUMENTS**
> 术语“参数”和“实参”在 C# 中早于泛型出现，并且已在其他语言中使用了几十年。方法将其输入声明为**参数**，而调用代码以**实参**的形式提供它们。图 2.1 展示了两者之间的关系。

![image-20260113070228822](https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260113070228903.png)

<img src="https://ddd-1313653702.cos.ap-guangzhou.myqcloud.com/now/20260113070240019.png" alt="image-20260113070239965" style="zoom:50%;" />

实参的值被用作方法内参数的初始值。在泛型中，你有**类型参数**和**类型实参**，这是相同的概念，但应用于类型。泛型类型或方法的声明在名称后的尖括号中包含类型参数。在声明体内，代码可以将类型参数用作普通类型（只是对其了解不多）。

使用泛型类型或方法的代码在名称后的尖括号中指定类型实参。图 2.2 在 `List<T>` 的上下文中展示了这种关系。

现在想象一下 `List<T>` 的完整 API：所有的方法签名、属性等等。如果你使用的是图中所示的 `list` 变量，那么 API 中出现的任何 `T` 都会变成 `string`。例如，`List<T>` 中的 `Add` 方法具有以下签名

```c#
public void Add(T item)
```

但如果你在 Visual Studio 中输入 `list.Add(`，IntelliSense 会提示你，就好像 `item` 参数已被声明为 `string` 类型一样。如果你尝试传入另一种类型的实参，将导致编译时错误。

虽然图 2.2  指的是泛型类，但方法也可以是泛型的。方法声明类型参数，这些类型参数可以在方法签名的其他部分使用。方法的类型参数经常被用作签名中其他类型的类型实参。下面的代码清单展示了你之前无法实现的方法的解决方案：以静态类型方式创建一个新集合，包含现有集合的前 N 个元素。

```c#
//Listing 2.5  Copying elements from one collection to another
public static List<T> CopyAtMost<T>(    // 方法声明了一个类型参数 T，
    List<T> input, int maxElements)     // 并将其用于参数和返回类型。
{
    int actualCount = Math.Min(input.Count, maxElements);
    List<T> ret = new List<T>(actualCount);    // 类型参数在方法体中使用
    for (int i = 0; i < actualCount; i++)
    {
        ret.Add(input[i]);
    }
    return ret;
}

static void Main()
{
    List<int> numbers = new List<int>();
    numbers.Add(5);
    numbers.Add(10);
    numbers.Add(20);

    List<int> firstTwo = CopyAtMost<int>(numbers, 2);    // 调用方法，使用 int
    Console.WriteLine(firstTwo.Count);                   // 作为类型参数
}
```

许多泛型方法在签名中只使用一次类型参数，并且没有将其用作任何泛型类型的类型实参。但是，**使用类型参数来表达常规参数类型与返回类型之间关系的能力，正是泛型强大功能的重要组成部分**。

类似地，泛型类型在声明基类或实现的接口时，可以将其类型参数用作类型实参。例如，`List<T>` 类型实现了 `IEnumerable<T>` 接口，因此类声明可以这样写：

```c#
public class List<T> : IEnumerable<T>
```

> **注意**：实际上，`List<T>` 实现了多个接口；这是简化形式。

> **泛型方法的威力**：`CopyAtMost<T>` 方法完美解决了之前“无法以静态类型方式编写通用集合算法”的困境。它的精髓在于**用同一个类型参数 `T` 同时约束了输入集合 `List<T> input` 和返回集合 `List<T>`**，从而在编译时确保了类型的一致性。这实现了算法逻辑与元素类型的完美解耦与安全复用。
>
> **类型参数的关联作用**：正如作者强调的，泛型真正的力量往往不在于使用类型参数的次数，而在于**用它建立类型之间的关联**。在这个例子中，`T` 像一个纽带，将输入、输出以及方法体内部的局部变量（`ret`）紧紧地关联在一起，保证了整个操作过程的类型安全。
>
> **泛型与继承的结合**：泛型类型（如 `List<T>`）可以将其类型参数（`T`）“传递”给它所继承或实现的泛型接口（如 `IEnumerable<T>`）。这表明泛型可以无缝地融入面向对象的类型层次结构中，构建出复杂的、类型安全的抽象网络。

> `List<T> : IEnumerable<T>` 这个简单的声明，揭示了一个更深层次的模式：**泛型类型可以通过其类型参数，定义自己与其他泛型类型之间的“合约”关系**。
>
> 这自然引出一个更高级的话题：如果 `Student` 是 `Person` 的子类，那么 `List<Student>` 是否可以被当作 `IEnumerable<Person>` 来使用？换句话说，泛型类型的继承关系能否随着类型实参的继承关系而传递？这就是 **泛型协变与逆变** 要解决的问题。C# 4.0 通过 `in`/`out` 类型参数修饰符，在接口和委托中引入了这种能力，从而让泛型类型系统在保持安全性的同时，获得了前所未有的灵活性。理解 `List<T>` 如何实现 `IEnumerable<T>`，是通往理解这些更高级类型关系的重要一步。

**泛型类型与方法的元数**

泛型类型或方法可以通过在尖括号内用逗号分隔来声明多个类型参数。例如，.NET 1 中 `Hashtable` 类的泛型等效类的声明如下：

```c#
public class Dictionary<TKey, TValue>
```

一个声明的**泛型元数**是指它所拥有的类型参数数量。坦率地说，这个术语对作者（编写本书时）比日常编写代码时更有用，但我认为仍然值得了解。你可以将一个非泛型声明视为泛型元数为 0 的声明。

一个声明的泛型元数实质上是构成其独特性的一个部分。举例来说，我之前提到了 .NET 2.0 引入的 `IEnumerable<T>` 接口，但它与已经是 .NET 1.0 一部分的非泛型 `IEnumerable` 接口是截然不同的类型。同样，你可以编写名称相同但泛型元数不同的方法，即使它们的签名在其他方面相同：

```c#
public void Method() {}
public void Method<T>() {}
public void Method<T1, T2>() {}
```

当声明具有不同泛型元数的类型时，这些类型不必是同一类别，尽管它们通常是。作为一个极端的例子，考虑以下类型声明，它们可以共存于一个非常令人困惑的程序集中：

```c#
public enum IAmConfusing {}
public class IAmConfusing<T> {}
public struct IAmConfusing<T1, T2> {}
public delegate void IAmConfusing<T1, T2, T3> {}
public interface IAmConfusing<T1, T2, T3, T4> {}
```

尽管我强烈建议不要编写上述代码，但一种比较常见的模式是：一个非泛型的静态类提供辅助方法，这些方法引用同名的其他泛型类型（有关静态类的更多信息，请参见第2.5.2节）。例如，你将在第2.1.4节中看到Tuple类，它用于创建各种泛型Tuple类的实例。

正如多个类型可以具有相同的名称但不同的泛型元数一样，泛型方法也可以如此。这类似于基于参数进行重载，但这里是基于类型参数的数量进行重载。请注意，尽管泛型元数使声明彼此独立，但类型参数名不会。例如，你不能像这样声明两个方法：

```c#
//编译时错误；不能仅通过类型参数名进行重载。
public void Method<TFirst>() {}
public void Method<TSecond>() {}
```

这些被视为具有等效的签名，因此根据方法重载的正常规则是不允许的。你可以编写使用不同类型参数名的方法重载，只要这些方法在其他方面有所不同（例如常规参数的数量），尽管我不记得我曾经想这样做过。

当我们讨论多个类型参数时，你不能在同一声明中给两个类型参数相同的名称，就像你不能声明两个同名的常规参数一样。例如，你不能这样声明一个方法：

```c#
public void Method<T, T>() {} //编译时错误；重复的类型参数T。
```

但是，两个类型实参可以是相同的，而且这通常是你想要的。例如，要创建一个字符串到字符串的映射，你可以使用`Dictionary<string, string>`。

前面IAmConfusing的例子使用了枚举作为非泛型类型。这不是巧合，因为我想用它来证明我的下一个观点。

## 哪些成员可以是泛型的？

并非所有类型或类型成员都可以是泛型的。对于类型来说，这相对简单，部分原因是可以声明的类型种类较少。**枚举不能是泛型的**，但**类、结构体、接口和委托都可以是泛型**。

对于类型成员来说，情况稍微有些令人困惑；有些成员可能看起来像是泛型的，因为它们使用了其他泛型类型。请记住，**只有引入了新的类型参数的声明才是泛型的**。

方法和嵌套类型可以是泛型的，但以下所有成员都必须是**非泛型**的：

- 字段
- 属性
- 索引器
- 构造函数
- 事件
- 析构函数

举个例子，你可能会误以为某个字段是泛型的，但实际上并非如此。考虑以下泛型类：

```c#
public class ValidatingList<TItem>
{
    private readonly List<TItem> items = new List<TItem>();
    // ... 许多其他成员
}
```

我将类型参数命名为 `TItem`，只是为了与 `List<T>` 的 `T` 类型参数区分开。这里，`items` 字段的类型是 `List<TItem>`。它使用了类型参数 `TItem` 作为 `List<T>` 的类型实参，但这是由类声明引入的类型参数，而不是字段声明引入的。

**对于这些成员中的大多数，很难想象它们如何成为泛型。不过，偶尔我也会想编写泛型构造函数或索引器，而答案几乎总是：改为编写一个泛型方法。**

说到泛型方法，我之前在描述泛型方法的调用方式时，只对类型实参给出了简化说明。在某些情况下，编译器可以在你无需在源代码中提供类型实参的情况下，确定调用的类型实参。

> **“使用泛型”与“是泛型”的区分**：一个成员（如字段）的类型可以是泛型类型（如 `List<TItem>`），但这并不意味着该成员本身是泛型的——它没有引入新的类型参数。

## 方法的类型实参类型推断

让我们回顾一下代码清单 2.5 的关键部分。你有一个像这样声明的泛型方法：

```c#
public static List<T> CopyAtMost<T>(List<T> input, int maxElements)
```

然后，在 `Main` 方法中，你声明了一个 `List<int>` 类型的变量，并随后将其用作该方法的参数：

```c#
List<int> numbers = new List<int>();
...
List<int> firstTwo = CopyAtMost<int>(numbers, 2);
```

我在这里高亮了方法调用。你需要为 `CopyAtMost` 调用提供一个类型实参，因为它有一个类型参数。但你不必在源代码中指定这个类型实参。你可以像下面这样重写代码：

```c#
List<int> numbers = new List<int>();
...
List<int> firstTwo = CopyAtMost(numbers, 2);
```

就编译器生成的 IL 而言，这完全是同一个方法调用。但你无需指定 `int` 这个类型实参；编译器为你推断出来了。它是根据你为方法第一个参数传递的实参来推断的。你使用了一个 `List<int>` 类型的实参作为 `List<T>` 类型参数的值，所以 `T` 必须是 `int`。

类型推断只能使用你传递给方法的实参，而不能利用你对结果的处理。它还必须完整；你要么显式指定所有类型实参，要么一个也不指定。

虽然类型推断仅适用于方法，但它可以用来更轻松地构造泛型类型的实例。例如，考虑 .NET 4.0 中引入的 `Tuple` 类型族。它包括一个非泛型的静态 `Tuple` 类和多个泛型类：`Tuple<T1>`、`Tuple<T1, T2>`、`Tuple<T1, T2, T3>`，等等。静态类有一组重载的 `Create` 工厂方法，如下所示：

```c#
public static Tuple<T1> Create<T1>(T1 item1)
{
    return new Tuple<T1>(item1);
}
public static Tuple<T1, T2> Create<T1, T2>(T1 item1, T2 item2)
{
    return new Tuple<T1, T2>(item1, item2);
}
```

这些方法看起来简单得毫无意义，但它们允许在原本创建元组时必须显式指定类型实参的地方使用类型推断。代替：

```c#
new Tuple<int, string, int>(10, "x", 20)
```

你可以这样写：

```c#
Tuple.Create(10, "x", 20)
```

这是一个值得了解的强大技巧；实现起来通常很简单，并且可以使泛型代码的使用体验愉快得多。

我不打算深入探讨泛型类型推断如何工作的细节。随着语言设计者找到使其在更多情况下工作的方法，它已经发生了很大变化。重载解析和类型推断紧密相连，它们与各种其他特性（如 C# 4 中的继承、转换和可选参数）相互交叉。这是我发现规范中最复杂的领域，我无法在此详述。

幸运的是，这个领域的细节理解对日常编码的帮助不大。在任何特定情况下，存在三种可能：

- **类型推断成功并给出了你想要的结果。** 太好了。
- **类型推断成功但给出了你不想要的结果。** 只需显式指定类型实参或对部分实参进行类型转换。例如，如果你想要从前面的 `Tuple.Create` 调用中得到一个 `Tuple<int, object, int>`，你可以显式指定 `Tuple.Create` 的类型实参，或者直接调用 `new Tuple<int, object, int>(...)`，或者调用 `Tuple.Create(10, (object) "x", 20)`。
- **类型推断在编译时失败。** 有时可以通过转换部分实参来修复。例如，`null` 字面量没有类型，所以 `Tuple.Create(null, 50)` 的类型推断会失败，但 `Tuple.Create((string) null, 50)` 会成功。其他时候你只需要显式指定类型实参。

根据我的经验，对于后两种情况，你选择哪种方案对可读性的影响通常不大。理解类型推断的细节可以让你更容易预测什么会起作用、什么不会，但可能不值得投入时间去研究规范。如果你好奇，我绝不会阻止任何人阅读规范。只是当你发现它时而感觉像是一个由全都相似、迂回曲折的小通道组成的迷宫，时而又感觉像是一个由全都不同、迂回曲折的小通道组成的迷宫时，不要感到惊讶。

不过，这些关于复杂语言细节的危言耸听不应削弱类型推断带来的便利性。C# 因为有它而变得相当易用。

到目前为止，我们讨论的所有类型参数都是**无约束的**。它们可以代表任何类型。但这并不总是你想要的；有时，你只希望某些类型被用作特定类型参数的类型实参。这就是类型约束的用武之地。

> **务实的开发者态度**：作者坦诚地指出类型推断（及其相关的重载解析）规范极其复杂，并给出了务实建议：**无需深究其所有细节**。开发者只需知道三种可能结果（成功、成功但不符预期、失败）及其应对策略（显式指定类型实参、类型转换）即可。这体现了“工具使用者”而非“语言规范研究者”的实用主义思维。

## 类型约束

当泛型类型或方法声明一个类型参数时，它还可以指定**类型约束**，以限制哪些类型可以作为类型实参提供。假设你想编写一个格式化项目列表的方法，并确保以特定的区域性而不是线程的默认区域性进行格式化。`IFormattable` 接口提供了一个合适的 `ToString(string, IFormatProvider)` 方法，但你如何确保你有一个合适的列表呢？你可能会期望这样的签名：

```c#
static void PrintItems(List<IFormattable> items)
```

但这几乎没什么用。例如，你不能将 `List<decimal>` 传递给它，即使 `decimal` 实现了 `IFormattable`；`List<decimal>` 不能转换为 `List<IFormattable>`。

**注意**：我们将在第4章更深入地探讨其原因，届时我们将考虑泛型变体。目前，只需将此视为约束的一个简单示例。

你需要表达的是：参数是一个某种元素类型的列表，其中该元素类型实现了 `IFormattable` 接口。“某种元素类型”这部分表明你可能希望使方法成为泛型的，而“其中该元素类型实现了 `IFormattable` 接口”正是类型约束给我们的能力。你在方法声明的末尾添加一个 `where` 子句，像这样：

```c#
static void PrintItems<T>(List<T> items) where T : IFormattable
```

你在这里约束 `T` 的方式不仅改变了可以传递给方法的参数值；它还改变了你在方法内可以如何使用 `T` 类型的值。编译器知道 `T` 实现了 `IFormattable`，因此允许在任何 `T` 值上调用 `IFormattable.ToString(string, IFormatProvider)` 方法。



```c#
//代码清单 2.6 使用类型约束在固定区域性中打印项
static void PrintItems<T>(List<T> items) where T : IFormattable
{
    CultureInfo culture = CultureInfo.InvariantCulture;
    foreach (T item in items)
    {
        Console.WriteLine(item.ToString(null, culture));
    }
}
```



如果没有类型约束，那个 `ToString` 调用将无法编译；编译器知道的 `T` 的唯一 `ToString` 方法是在 `System.Object` 中声明的那一个。

类型约束不限于接口。可用的类型约束如下：

- **引用类型约束** —— `where T : class`。类型实参必须是引用类型。（不要被 `class` 关键字所迷惑；它可以是任何引用类型，包括接口和委托。）
- **值类型约束** —— `where T : struct`。类型实参必须是非可空值类型（结构体或枚举）。可空值类型（在第2.2节中描述）不满足此约束。
- **构造函数约束** —— `where T : new()`。类型实参必须具有公共的无参数构造函数。这使得在代码体内可以使用 `new T()` 来构造 `T` 的新实例。
- **转换约束** —— `where T : SomeType`。这里，`SomeType` 可以是一个类、一个接口或另一个类型参数，如下所示：
  - `where T : Control`
  - `where T : IFormattable`
  - `where T1 : T2`

有一些中等复杂的规则说明了如何组合约束。通常，当你违反这些规则时，编译器错误信息会很明显地指出问题所在。

一种有趣且相当常见的约束形式是在约束本身中使用类型参数：

```c#
public void Sort<T>(List<T> items) where T : IComparable<T>
```

该约束使用 `T` 作为泛型 `IComparable<T>` 接口的类型实参。这允许我们的排序方法使用 `IComparable<T>` 的 `CompareTo` 方法对 `items` 参数中的元素进行两两比较：

```c#
T first = ...;
T second = ...;
int comparison = first.CompareTo(second);
```

**我使用基于接口的类型约束比其他任何类型都多，尽管我怀疑你使用什么很大程度上取决于你编写的代码类型。**

当泛型声明中存在多个类型参数时，每个类型参数可以有一组完全不同的约束，如下例所示：

```c#
TResult Method<TArg, TResult>(TArg input)
    where TArg : IComparable<TArg>
    where TResult : class, new()
```

我们已经快完成泛型的旋风之旅了，但我还有几个主题要描述。我将从 C# 2 中可用的两个与类型相关的运算符开始。

> **丰富的约束类型**：C# 提供了多样化的约束类型，涵盖了引用/值类型、构造能力、继承/实现关系等。这允许开发者精确地表达对类型参数的要求，从而在灵活性和安全性之间取得平衡。
>
> **递归约束**：像 `where T : IComparable<T>` 这样的约束，其中约束本身引用了类型参数 `T`，是一种强大且常见的模式。它确保了类型能够与自身进行比较，这对于实现排序、查找等算法至关重要，是构建通用算法库的基础。

> 类型约束极大地增强了泛型的能力，但它也引入了**耦合**：泛型代码现在依赖于特定的接口或基类。这引发了一个设计层面的思考：当我们需要一个类型参数满足多种不相关的功能（例如，既要是可比较的，又要是可序列化的）时，是应该定义一个新的、组合了这些需求的接口，还是使用多个约束？
>
> 这触及了接口设计的一个原则：**接口隔离**。在泛型上下文中，有时使用多个独立的约束（`where T : IComparable<T>, ISerializable`）更灵活，因为它允许类型参数来自不同的、无关的继承层次。而定义一个新的组合接口可能会迫使许多本不相关的类型实现它，造成污染。因此，约束的灵活组合能力，使得我们可以构建出高度可复用且不强迫类型改变其设计的泛型组件。

## `default` 和 `typeof` 操作符

C# 1 中就已经有了 `typeof()` 操作符，它接受一个类型名称作为其唯一操作数。C# 2 添加了 `default()` 操作符，并略微扩展了 `typeof` 的用途。

`default` 操作符很容易描述。其操作数是一个类型或类型参数的名称，结果是该类型的默认值——与你声明一个字段但没有立即为其赋值时所得到的值相同。**对于引用类型，那就是一个 `null` 引用；对于不可空值类型，是“全零”值（0, 0.0, 0.0m, `false`，数值为0的UTF-16代码单元等）；对于可空值类型，则是该类型的 `null` 值。**

`default` 操作符可以与类型参数一起使用，也可以与提供了适当类型实参的泛型类型一起使用（这些实参也可以是类型参数）。例如，在一个声明了类型参数 `T` 的泛型方法中，以下所有形式都是有效的：

- `default(T)`
- `default(int)`
- `default(string)`
- `default(List<T>)`
- `default(List<List<string>>)`

`default` 操作符的类型就是其中指定的类型。它最常与泛型类型参数一起使用，因为否则你通常可以用其他方式指定默认值。例如，你可能希望将一个局部变量的初始值设为默认值，而这个变量之后可能会也可能不会被赋予其他值。具体来说，这里有一个你可能熟悉的简单方法实现：

```c#
public T LastOrDefault<T>(IEnumerable<T> source)
{
    T ret = default(T);      // 声明一个局部变量，并将 T 的默认值赋给它
    foreach (T item in source)
    {
        ret = item;          // 用序列中的当前值替换局部变量的值
    }
    return ret;              // 返回最后赋的值
}
```

`typeof` 操作符稍微复杂一些。需要考虑四种主要情况：

- **完全不涉及泛型**；例如 `typeof(string)`
- **涉及泛型但没有类型参数**；例如 `typeof(List<int>)`
- **仅是一个类型参数**；例如 `typeof(T)`
- **在操作数中使用类型参数涉及泛型**；例如，在声明了一个名为 `TItem` 的类型参数的泛型方法中，使用 `typeof(List<TItem>)`
- **涉及泛型但在操作数中未指定类型实参**；例如 `typeof(List<>)`

第一种情况很简单，完全没有改变。其他所有情况都需要稍微注意一下，最后一种情况引入了一种新的语法。`typeof` 操作符仍然定义为返回一个 `Type` 值，那么在每种情况下它应该返回什么呢？`Type` 类得到了增强，以支持泛型。有多种情况需要考虑；以下是几个例子：

- 例如，如果你列出包含 `List<T>` 的程序集中的类型，你期望得到的是 `List<T>`，而不带任何特定的 `T` 类型实参。它是一个**泛型类型定义**。
- 如果你在一个 `List<int>` 对象上调用 `GetType()`，你希望得到一个包含类型实参信息的类型。
- 如果你问一个声明为 `class StringDictionary<T> : Dictionary<string, T>` 的类的泛型类型定义的基类型，你最终会得到一个类型，它有一个“具体”的类型实参（`string`，对应 `Dictionary<TKey, TValue>` 的 `TKey` 类型参数）和一个仍然是类型参数的实参（`T`，对应 `TValue` 类型参数）。

坦率地说，这一切都非常令人困惑，但这是问题域固有的。`Type` 类中有很多方法和属性可以让你从泛型类型定义转到提供了所有类型实参的类型，反之亦然。

让我们回到 `typeof` 操作符。最容易理解的例子是 `typeof(List<int>)`。它返回表示具有 `int` 类型实参的 `List<T>` 的 `Type`，就像你调用了 `new List<int>().GetType()` 一样。

下一个情况，`typeof(T)`，返回代码中该点 `T` 的类型实参是什么。这将始终是一个**封闭的构造类型**，这是规范中用来表示它是一个不涉及任何类型参数的真实类型的方式。尽管在大多数地方我都试图彻底解释术语，但围绕泛型的术语（开放的、封闭的、构造的、绑定的、未绑定的）令人困惑，在现实生活中几乎毫无用处。我们稍后需要讨论封闭的构造类型，但我不会涉及其他术语。

最容易演示 `typeof(T)` 含义的方式，你可以在同一个例子中查看 `typeof(List<T>)`。以下代码清单声明了一个泛型方法，它将 `typeof(T)` 和 `typeof(List<T>)` 的结果打印到控制台，然后用两个不同的类型实参调用该方法。

> **默认值的高效抽象**：`default` 操作符是 C# 中获取类型默认值的**统一且安全**的方式。在泛型编程中至关重要，因为你无法预先知道类型参数 `T` 是引用类型还是值类型。它避免了在泛型代码中需要编写条件逻辑来分别处理 `null` 和零值。
>
> **运行时类型信息的泛化**：`typeof` 操作符的增强反映了 .NET 运行时**对泛型的全面支持**。它不仅能获取非泛型类型或封闭构造类型（如 `List<int>`）的 `Type` 对象，还能获取**泛型类型定义**（如 `List<>`）和**包含未绑定类型参数的构造类型**（如 `List<T>`）的元数据。这为基于反射的泛型代码生成和操作提供了基础。
>
> **术语与复杂性**：作者坦诚地指出围绕泛型类型的术语（开放/封闭/构造）非常混乱且不实用。这提醒我们，学习的关键在于理解**概念本身**（如“运行时需要知道具体类型才能实例化一个对象”），而非死记硬背术语。编译器会确保我们正确使用这些概念。

> `typeof(List<>)` 这种“不指定类型实参”的语法，用于获取泛型类型定义，这是一个相当底层的特性。这自然引出一个问题：**在什么实际场景下，我们会需要直接操作泛型类型定义，而不是具体的封闭类型？**
>
> 这正是**反射和元编程**的核心领域。**例如，如果你想编写一个通用的数据绑定框架，能够自动将数据库结果集（`DataReader`）映射到用户定义的泛型实体类 `Repository<TEntity>` 上，你可能需要获取 `Repository<>` 这个泛型类型定义，然后根据运行时确定的 `TEntity` 类型（比如 `Customer`），动态地构造出封闭类型 `Repository<Customer>` 并创建其实例。**`default` 和增强的 `typeof` 操作符，为这类高级、灵活的运行时编程模式提供了必要的基石。

```c#
static void PrintType<T>()
{
    //Prints both typeof(T) and typeof(List<T>)
    Console.WriteLine("typeof(T) = {0}", typeof(T));
    Console.WriteLine("typeof(List<T>) = {0}", typeof(List<T>));
} 
static void Main() 
{
    // Calls the method with a
    // type argument of string
    PrintType<string>(); 
    
    //Calls the method with
    //a type argument of int
    PrintType<int>(); 
} 
//结果
//typeof(T) = System.String
//typeof(List<T>) = System.Collections.Generic.List`1[System.String]
//typeof(T) = System.Int32
//typeof(List<T>) = System.Collections.Generic.List`1[System.Int32]
```

关键点在于：当你在 `T` 的类型实参为 `string` 的上下文中执行时（第一次调用期间），`typeof(T)` 的结果与 `typeof(string)` 相同。同样，`typeof(List<T>)` 的结果与 `typeof(List<string>)` 相同。当你再次使用 `int` 作为类型实参调用该方法时，得到的结果与 `typeof(int)` 和 `typeof(List<int>)` 相同。

**每当代码在泛型类型或方法内部执行时，类型参数总是指向一个封闭的构造类型。**

从输出中得到的另一个要点是使用反射时泛型类型的名称格式。`List`\`1 表明这是一个名为 `List`、泛型元数为 1（一个类型参数）的泛型类型，类型实参随后在方括号中显示。

我们之前列表中的最后一项是 `typeof(List<>)`。这看起来完全省略了类型实参。这种语法仅在 `typeof` 操作符中有效，表示引用泛型类型定义。对于泛型元数为 1 的类型，语法是 `TypeName<>`；对于每个额外的类型参数，你在尖括号内添加一个逗号。要获取 `Dictionary<TKey, TValue>` 的泛型类型定义，你可以使用 `typeof(Dictionary<,>)`。要获取 `Tuple<T1, T2, T3>` 的定义，你可以使用 `typeof(Tuple<,,>)`。

理解泛型类型定义和封闭的构造类型之间的区别对于我们最后一个主题至关重要：**类型如何初始化以及类型范围（静态）状态如何处理**。

> **运行时类型的具体化**：在泛型方法或类型内部，类型参数 `T` 在运行时**总是一个具体的封闭构造类型**（如 `string` 或 `int`）。因此，`typeof(T)` 返回的是该具体类型的 `Type` 对象。这澄清了泛型代码执行时的一个关键事实：**不存在“泛型运行时对象”**，只有具体类型的实例。
>
> **反射名称格式**：`List`1[System.String] 这样的名称格式是 .NET 反射 API 中用于表示泛型类型的标准方式。理解这个格式有助于在调试或编写反射代码时识别泛型类型。``1` 称为**泛型类型元数标记**，`[System.String]` 是类型实参列表。
>
> **未绑定泛型类型**：`typeof(List<>)` 语法用于获取**泛型类型定义**（未绑定泛型类型）。这是反射 API 中的一个重要概念，用于动态构造泛型类型（例如，在运行时根据数据库表名创建 `Repository<>` 的特定实例）。它与封闭构造类型有本质区别。

> 既然每个封闭构造类型（如 `List<string>` 和 `List<int>`）都有自己的静态字段，那么这是否意味着泛型也可以用于实现**类型安全的单例模式**，其中每个 `T` 对应一个独立的单例实例？
>
> 是的，这确实是可能的，并且是一个有用的模式。例如，你可以有一个泛型类 `Cache<T>`，它为每种类型 `T` 维护一个独立的静态缓存字典。这种**按类型参数隔离的静态状态**是泛型的一个强大特性，使得我们可以为不同的类型创建独立但结构相同的静态上下文。这为构建类型安全的工厂、缓存、注册表等模式提供了新的可能性。理解泛型类型初始化和静态状态的处理方式，是掌握这些高级用法的基础。

##  泛型类型的初始化与状态

正如你在使用 `typeof` 操作符时看到的，`List<int>` 和 `List<string>` 实际上是由同一个泛型类型定义构造出来的不同类型。这不仅体现在你如何使用这些类型，也体现在类型如何初始化以及静态字段如何处理上。**每个封闭的构造类型都是独立初始化的，并且拥有自己独立的一组静态字段**。下面的代码清单通过一个简单（非线程安全）的泛型计数器来演示这一点。

```c#
//2.8 探索泛型类型中的静态字段
class GenericCounter<T>
{
    private static int value; // 每个封闭构造类型都有一个独立的字段

    static GenericCounter()
    {
        Console.WriteLine("Initializing counter for {0}", typeof(T));
    }

    public static void Increment()
    {
        value++;
    }

    public static void Display()
    {
        Console.WriteLine("Counter for {0}: {1}", typeof(T), value);
    }
}

class GenericCounterDemo
{
    static void Main()
    {
        GenericCounter<string>.Increment(); // 触发 GenericCounter<string> 的初始化
        GenericCounter<string>.Increment();
        GenericCounter<string>.Display();

        GenericCounter<int>.Display();      // 触发 GenericCounter<int> 的初始化
        GenericCounter<int>.Increment();
        GenericCounter<int>.Display();
    }
}
```

代码清单 2.8 的输出如下：

```c#
Initializing counter for System.String
Counter for System.String: 2
Initializing counter for System.Int32
Counter for System.Int32: 0
Counter for System.Int32: 1
```

在这个输出中，有两个结果值得关注。首先，`GenericCounter<string>` 的值独立于 `GenericCounter<int>`。其次，静态构造函数运行了两次：每个封闭的构造类型一次。如果你没有静态构造函数，每个类型初始化的确切时机会有更少的保证，但本质上你可以将 `GenericCounter<string>` 和 `GenericCounter<int>` 视为独立的类型。

更复杂的是，泛型类型可以嵌套在其他泛型类型中。当这种情况发生时，**每个类型参数的组合都会产生一个独立的类型**。例如，考虑如下类：

```c#
class Outer<TOuter>
{
    class Inner<TInner>
    {
        static int value;
    }
}
```

使用 `int` 和 `string` 作为类型参数，以下类型是独立的，每个都有其自己的 `value` 字段：

- `Outer<string>.Inner<string>`
- `Outer<string>.Inner<int>`
- `Outer<int>.Inner<string>`
- `Outer<int>.Inner<int>`

在大多数代码中，这种情况相对少见，但只要你意识到重要的是完全指定的类型（包括叶子类型和任何封闭类型的所有类型参数），处理起来就足够简单了。

以上就是关于泛型的内容，它是 C# 2 中迄今为止最大的单个功能，也是相对于 C# 1 的巨大改进。我们的下一个主题是**可空值类型**，它完全基于泛型构建。

> **独立的静态上下文**：每个封闭的构造泛型类型（如 `GenericCounter<string>` 和 `GenericCounter<int>`）都拥有自己**完全独立的静态字段和静态构造函数**。它们之间互不影响，就像两个完全不相关的类一样。这为按类型参数隔离状态提供了可能。
>
> **初始化的独立性**：每个封闭构造类型的静态构造函数在其类型首次被使用（如调用静态方法）时独立触发。这确保了静态字段的初始化时机是确定的（在有静态构造函数时），且彼此隔离。
>
> **嵌套泛型的组合爆炸**：当泛型类型嵌套时，最终的类型是**所有外层和嵌套类型参数的组合**。每个独特的组合都会产生一个独立的类型，拥有独立的静态字段。理解这一点对于复杂泛型结构的推理至关重要。



> 泛型类型的静态字段独立性带来了一种强大的设计模式：**为每个类型 `T` 创建一个唯一的、类型安全的上下文或服务实例**。这可以用来实现诸如按类型注册的依赖注入容器、类型特定的缓存或单例工厂等。
>
> 这自然引出一个问题：如果我们需要一个跨所有 `T` 共享的全局静态状态，该怎么办？答案是将静态字段放在一个**非泛型的基础类或单独的静态类**中。这揭示了泛型设计中的一个关键决策点：状态是应该与类型参数 `T` 绑定，还是应该全局共享？理解这种独立性的机制，使我们能够做出更明智的设计选择。





