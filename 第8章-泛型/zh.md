“不要重复自己”是软件工程中的一条常见建议。重用数据结构或函数比重新创建它们更好，因为很难在重复的代码之间保持代码更改的同步。像 Go 这样的强类型语言中，每个函数参数和结构体字段的类型必须在编译时确定。这种严格性使得编译器能够帮忙验证代码是否正确，但有时你会想用不同的类型来重用函数的逻辑或结构体的字段。Go 通过类型参数（type parameters）提供了这种功能，俗称为泛型。在这一章中，你将了解到为什么大家想要泛型，Go 的泛型实现能做什么，不能做什么，及如何正确使用它。

## 泛型减少重复代码并增强类型安全性

Go 是一种静态类型语言，这意味着在代码编译时会检查变量和参数的类型。内置类型（如 maps、slices、channels）和函数（如 len、cap 或 make）能够接受和返回不同具体类型的值，但在 Go 1.18 之前，用户自定义的 Go 类型和函数则无法做到这一点。

如果你习惯了动态类型的语言（其类型直到代码运行时才会被检查），你或许不明白泛型为何备受瞩目，也可能对它们的确切意义感到模糊。把它们看作“类型参数”会有所帮助。目前为止，在本书中，你已经看到了那些在函数调用时指定参数值的函数。在第 96 页的“多返回值”中，divAndRemainder 函数有两个 int 类型的参数，并返回两个 int 类型的值：

```go
func divAndRemainder(num, denom int) (int, int, error) {
    if denom == 0 {
      return 0, 0, errors.New("cannot divide by zero")
    }
    return num / denom, num % denom, nil
}
```

同样地，当声明结构体时，你需要为字段指定类型。例如，在这里，Node 结构体有一个 `int` 类型的字段和另一个 `*Node` 类型的字段：

```go
type Node struct {
    val int
    next *Node
}
```

在某些情况下，编写函数或结构体时，将参数或字段的具体类型保留为未指定状态，直到使用时才确定，这样做会更有用。

通过具体案例就很容易理解泛型类型，在第 148 页的“为 nil 实例编写方法”中，你看到了一个用于`int`类型的二叉树。如果你想要一个用于`string`或`float64`类型的二叉树并且希望是类型安全的，你有几个选择。第一种可能是为每种类型都设计一种特定的树，但是这样做会有大量重复的代码，既冗长又容易出错。

没有泛型的情况下，避免重复代码的唯一方法就是修改你的树实现，便可使用接口来确定如何对值进行排序，这个接口看起来会像这样：

```go
type Orderable interface {
    // Order returns:
    // a value < 0 when the Orderable is less than the supplied value,
    // a value > 0 when the Orderable is greater than the supplied value,
    // and 0 when the two values are equal.
    Order(any) int
}
```

现在你有了`Orderable`接口，就可以修改你的`Tree`实现来支持它了：

```go
type Tree struct {
    val         Orderable
    left, right *Tree
}

func (t *Tree) Insert(val Orderable) *Tree {
    if t == nil {
        return &Tree{val: val}
    }

    switch comp := val.Order(t.val); {
    case comp < 0:
        t.left = t.left.Insert(val)
    case comp > 0:
        t.right = t.right.Insert(val)
    }
    return t
}
```

有了 `OrderableInt`类型，然后你就可以插入`int`类型的值：

```go
type OrderableInt int

func (oi OrderableInt) Order(val any) int {
    return int(oi - val.(OrderableInt))
}
func main() {
    var it *Tree
    it = it.Insert(OrderableInt(5))
    it = it.Insert(OrderableInt(3))
    // etc...
}
```

尽管这段代码能够正常运行，但编译器无法确认插入到数据结构中的值是否都属于同一类型。如果你还定义了 OrderableString 类型：

```go
type OrderableString string

func (os OrderableString) Order(val any) int {
    return strings.Compare(string(os), val.(string))
}
```

下面的代码也能通过编译：

```go
var it *Tree
it = it.Insert(OrderableInt(5))
it = it.Insert(OrderableString("nope"))
```

`Order` 函数接受 `any` 类型的参数值，这样做实质上绕过了 Go 语言的一大优势：编译时类型安全检查。当你尝试编译将一个 OrderableString 对象插入到已包含 OrderableInt 对象的 Tree 中的代码时，这段代码能通过编译。但运行时，程序会发生 panic：

```bash
panic: interface conversion: interface {} is main.OrderableInt, not string
```

你可以使用[第八章的代码仓库]()中的 _sample_code/non_generic_tree_ 目录下的这段代码进行尝试。

有了泛型，就可以为多种类型一次性实现一个通用的数据结构。并在编译时检测到不兼容的数据。稍后你将看到如何正确地使用它们。

虽然没有泛型的的数据结构使用起来会较不便，但真正的限制在于函数的编写上。Go 标准库中的许多实现决策都是因为语言一开始没有泛型而做出的。例如，Go 并不是编写多个函数来处理不同的数值类型，而是使用 float64 参数实现了诸如 math.Max、math.Min 和 math.Mod 这样的函数，因为 float64 的取值范围足够大，几乎可以精确表示其他的数值类型。（不过有例外情况，当 int、int64 或 uint 的值大于 2^53−1 或小于 −2^53−1 时，则无法用 float64 精确表示）

没有泛型，一些事情是无法做到的。你不能根据接口类型创建一个新的变量实例，你也不能要求具有相同接口类型的两个参数必须属于相同的具体类型。没有泛型，你无法写一个能处理任意类型切片的函数，除非使用反射，而这会牺牲一部分性能并放弃编译时的类型安全检查（这正是 sort.Slice 的工作原理）。这意味着在泛型引入 Go 之前，对切片进行操作的函数（如 map、reduce 和 filter）将会针对每种切片类型重复实现。对于软件工程师来说，简单算法虽然可以直接复制粘贴，但由于编译器无法自动处理此类代码重复，还是让许多（甚至大多数）工程师感到不爽。

## 在 Go 中引入泛型

自从 Go 首次发布以来，人们就一直呼吁将泛型添加到该语言中。Go 的开发负责人 Russ Cox 在 2009 年写了一篇博客文章，解释了为何最初没有包含泛型。Go 强调快速编译、代码可读性及优异的执行效率，但他们当时了解到的任何泛型实现都无法同时满足这三个要求。经过十年的研究，Go 团队找到了一个可行的方法，并在[类型参数提案](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md)中进行了阐述。

你可以通过栈来了解 Go 中的泛型如何工作的。栈是一种数据类型，其中的元素按照“后进先出”（LIFO）的顺序添加和移除。如果你没有计算机科学的背景，可以理解栈就好像一堆等待清洗的盘子，先放入的在底部，你只能处理后来添加的那些盘子才能拿到它们。让我们看看如何使用泛型来创建一个栈：

```go
type Stack[T any] struct {
    vals []T
}

func (s *Stack[T]) Push(val T) {
    s.vals = append(s.vals, val)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.vals) == 0 {
        var zero T
        return zero, false
    }
    top := s.vals[len(s.vals)-1]
    s.vals = s.vals[:len(s.vals)-1]
    return top, true
}
```

这里有几个需要注意的地方。首先，类型声明后面跟着`[T any]`。类型参数信息放在方括号内，由两部分组成：第一部分是类型参数的名称，你可以选择任何名称，但习惯上使用大写字母；第二部分是类型约束（type constraint），它使用一个 Go 接口来指定哪些类型是有效的。如果任何类型都适用，就用`any`标识符指定，你在[第 166 页的“空接口毫无意义”]()一节中已经见过了。在`Stack`的声明中，你声明`vals`类型为`[]T`。

接下来，看一下方法的声明。正如你在 `vals` 声明中使用了 `T` 一样，这里也是同样做法。并且，在接收器部分，你用 `Stack[T]` 而不是 `Stack` 来表示类型。

最后，泛型在处理零值上，稍许有趣。在`Pop`方法中，你不能直接返回`nil`，因为对于像 `int` 这样的值类型来说，`nil` 并不是一个有效的值。要获取泛型的零值，最简单的方法就是简单地声明一个使用 `var` 的变量，并返回它。因为按照定义，如果没有进行赋值，`var` 总是会将其变量初始化为零值。

泛型类型的使用与非泛型类型大同小异：

```go
func main() {
    var intStack Stack[int]
    intStack.Push(10)
    intStack.Push(20)
    intStack.Push(30)
    v, ok := intStack.Pop()
    fmt.Println(v, ok)
}
```

唯一的区别在于，在声明变量时，你需要指定要与 `Stack` 一起使用的类型，——这里是以 `int` 类型为例。如果你尝试将一个字符串推入栈中，编译器会捕捉到这个错误。假如添加如下代码行：

```go
intStack.Push("nope")
```

编译器会产生以下错误信息：

```bash
cannot use "nope" (untyped string constant) as int value in argument to intStack.Push
```

你可以在 [Go Playground]() 上或者在[第 8 章的代码仓库]()的 _sample_code/stack_ 目录中尝试使用下泛型栈。

接下来，向栈添加一个方法来判断栈中是否包含某个值：

```go
func (s Stack[T]) Contains(val T) bool {
    for _, v := range s.vals {
        if v == val {
            return true
        }
    }
    return false
}
```

遗憾的是，这段代码无法通过编译。它会报错：

```bash
invalid operation: v == val (type parameter T is not comparable with ==)
```

如同 `interface{}` 不限定任何具体类型，any 同样如此，它仅能用来存储和检索任意类型的值。然而要使用 ==，你需要声明不同的类型。考虑到几乎所有 Go 类型都能使用 == 和 != 进行比较，Go 在"宇宙块"中引入了一个新的内置接口 `comparable`。如果你将栈的声明改为使用 comparable：

```go
type Stack[T comparable] struct {
    vals []T
}
```

现在你可以使用新方法了：

```go
func main() {
    var s Stack[int]
    s.Push(10)
    s.Push(20)
    s.Push(30)
    fmt.Println(s.Contains(10))
    fmt.Println(s.Contains(5))
}
```

这将输出以下内容：

```bash
true
false
```

你可以在 [The Go Playground]() 上或者在[第 8 章的代码仓库]()的 _sample_code/comparable_stack_ 目录中尝试使用这个更新后的栈。

稍后，你将了解如何创建一个泛型二叉树。在此之前，我将先介绍一些额外的概念：泛型函数（generic functions）、泛型如何与接口配合使用，以及类型术语（terms）。

## 泛型函数：抽象算法

如我先前所暗示的，你也可以编写泛型函数。我之前提到过，没有泛型，使得很难写出适用于所有类型的 map、reduce 和 filter。而泛型会让这一切变得简单。以下是来自类型参数提案的实现示例：

```go
// Map turns a []T1 to a []T2 using a mapping function.
// This function has two type parameters, T1 and T2.
// This works with slices of any type.
func Map[T1, T2 any](s []T1, f func(T1) T2) []T2 {
    r := make([]T2, len(s))
    for i, v := range s {
        r[i] = f(v)
    }
    return r
}

// Reduce reduces a []T1 to a single value using a reduction function.
func Reduce[T1, T2 any](s []T1, initializer T2, f func(T2, T1) T2) T2 {
    r := initializer
    for _, v := range s {
        r = f(r, v)
    }
    return r
}

// Filter filters values from a slice using a filter function.
// It returns a new slice with only the elements of s
// for which f returned true.
func Filter[T any](s []T, f func(T) bool) []T {
    var r []T
    for _, v := range s {
        if f(v) {
            r = append(r, v)
        }
    }
    return r
}
```

在函数中，类型参数放在函数名之后、变量参数之前。其中 Map 和 Reduce 有两个任意类型的类型参数，而 Filter 有一个。当你运行以下代码时：

```go
words := []string{"One", "Potato", "Two", "Potato"}
filtered := Filter(words, func(s string) bool {
    return s != "Potato"
})
fmt.Println(filtered)
lengths := Map(filtered, func(s string) int {
    return len(s)
})
fmt.Println(lengths)
sum := Reduce(lengths, 0, func(acc int, val int) int {
    return acc + val
})
fmt.Println(sum)

```

你将得到如下输出：

```bash
[One Two]
[3 3]
6
```

可在 [Go Playground](https://go.dev/play/p/MYYW3e7cpkX) 或 [第 8 章代码仓库](https://github.com/learning-go-book-2e/ch08) 的 _sample_code/map_filter_reduce_ 目录中自行尝试。
