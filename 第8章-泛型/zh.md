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

## 泛型函数：算法的抽象化

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

可在 [Go Playground](https://go.dev/play/p/MYYW3e7cpkX) 或 [第 8 章代码仓库](https://github.com/learning-go-book-2e/ch08) 的 _sample_code/map_filter_reduce_ 目录中自行体验。

## 泛型与接口

不仅可以使用 any 和 comparable 作为类型约束（type constraint），任何接口都可以用作类型约束。例如，假设你想创建一个类型，用于存储任意两个相同类型的值，前提是该类型实现了 fmt.Stringer 接口。泛型在编译时就能确保这种类型约束：

```go
type Pair[T fmt.Stringer] struct {
    Val1 T
    Val2 T
}
```

你还可以创建包含类型参数的接口。例如，下面的一个接口，其中包含一个方法，用于与指定类型的值进行比较并返回一个 float64 值，并且它还内嵌了 fmt.Stringer 接口：

```go
type Differ[T any] interface {
    fmt.Stringer
    Diff(T) float64
}
```

接下来，利用这两个类型来创建一个比较函数。该函数接收两个 Pair 实例，它们的字段类型均为 Differ，并返回差异较小的那个 Pair 实例

```go
func FindCloser[T Differ[T]](pair1, pair2 Pair[T]) Pair[T] {
    d1 := pair1.Val1.Diff(pair1.Val2)
    d2 := pair2.Val1.Diff(pair2.Val2)
    if d1 < d2 {
        return pair1
    }
    return pair2
}
```

注意，FindCloser 接收的 Pair 实例，其字段需满足 Differ 接口。Pair 要求其字段都是相同的类型，且此类型实现 fmt.Stringer 接口；而这个函数则更加严格。如果 Pair 实例的字段不满足 Differ 接口要求，编译器会阻止你将其用在 FindCloser 函数上。

现在定义几个满足 Differ 接口的类型：

```go
type Point2D struct {
    X, Y int
}

func (p2 Point2D) String() string {
    return fmt.Sprintf("{%d,%d}", p2.X, p2.Y)
}

func (p2 Point2D) Diff(from Point2D) float64 {
    x := p2.X - from.X
    y := p2.Y - from.Y
    return math.Sqrt(float64(x*x) + float64(y*y))
}

type Point3D struct {
    X, Y, Z int
}

func (p3 Point3D) String() string {
    return fmt.Sprintf("{%d,%d,%d}", p3.X, p3.Y, p3.Z)
}

func (p3 Point3D) Diff(from Point3D) float64 {
    x := p3.X - from.X
    y := p3.Y - from.Y
    z := p3.Z - from.Z
    return math.Sqrt(float64(x*x) + float64(y*y) + float64(z*z))
}
```

下面是使用这段代码的样子：

```go
func main() {
    pair2Da := Pair[Point2D]{Point2D{1, 1}, Point2D{5, 5}}
    pair2Db := Pair[Point2D]{Point2D{10, 10}, Point2D{15, 5}}
    closer := FindCloser(pair2Da, pair2Db)
    fmt.Println(closer)

    pair3Da := Pair[Point3D]{Point3D{1, 1, 10}, Point3D{5, 5, 0}}
    pair3Db := Pair[Point3D]{Point3D{10, 10, 10}, Point3D{11, 5, 0}}
    closer2 := FindCloser(pair3Da, pair3Db)
    fmt.Println(closer2)
}
```

你可以在 [Go Playground](https://go.dev/play/p/1_tlI22De7r) 上运行这段代码，或者在[第八章代码仓库](https://github.com/learning-go-book-2e/ch08/tree/main)的 _sample_code/generic_interface_ 目录中执行相关代码。

## 使用类型项来指定运算符

还有一件事需要用泛型表示：运算符。divAndRemainder 函数在处理 int 类型时表现得很好，但是如果你想使用其他整数类型，就需要进行类型转换，而且 uint 能够表示比 int 更大的数值。如果你想写一个 divAndRemainder 的泛型版本，你需要一种方式来指明你可以使用 / 和 % 运算符。Go 泛型通过接口中的类型元素（type element）来实现这一点，它由一个或多个类型项（type terms）组成：

```go
type Integer interface {
    int | int8 | int16 | int32 | int64 | uint | uint8 | uint16 | uint32 | uint64 | uintptr
}
```

在第 162 页的“嵌入和接口”中，你学习了接口嵌入，用来表明一个包含接口的方法集包含了被嵌入接口的全部方法。类型元素指定了哪些类型能赋值给类型参数，以及支持哪些运算符。里面列出了具体类型，并用 | 进行分隔。对所有列出的类型都有效的运算符，才是允许使用的。取模（%）运算符只对整数有效，所以我们列出了所有整数类型。（你可以省略 byte 和 rune，因为它们分别是 uint8 和 int32 的类型别名。）

请注意，带有类型元素的接口只能作为类型约束时才有效。将其用作变量、字段、返回值或参数的类型，会在编译时引发错误。

现在你可以写 divAndRemainder 的泛型版本了，并使用内置的 uint 类型（或 Integer 中列出的其他类型）：

```go
func divAndRemainder[T Integer](num, denom T) (T, T, error) {
    if denom == 0 {
        return 0, 0, errors.New("cannot divide by zero")
    }
    return num / denom, num % denom, nil
}

func main() {
    var a uint = 18_446_744_073_709_551_615
    var b uint = 9_223_372_036_854_775_808
    fmt.Println(divAndRemainder(a, b))
}
```

默认情况下，必须严格匹配类型项。如果你尝试给 divAndRemainder 函数传入一个自定义的类型，该类型的底层类型是 Integer 列表中的一种，也会引发错误。比如这段代码：

```go
type MyInt int
var myA MyInt = 10
var myB MyInt = 20
fmt.Println(divAndRemainder(myA, myB))
```

会产生以下错误：

```bash
MyInt does not satisfy Integer (possibly missing ~ for int in Integer)
```

错误信息给出了如何解决这个问题的提示。如果你希望某个类型项对以该类型项作为底层类型的任意类型都有效，可在其前加上 ~。这样 Integer 的定义将改为：

```go
type Integer interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 | ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr
}
```

你可以在 [Go Playground](https://go.dev/play/p/5rD41rZbPw0) 或者[第 8 章代码仓库](https://github.com/learning-go-book-2e/ch08)的 _sample_code/type_terms_ 目录中查看 divAndRemainder 函数的泛型版本。

通过添加类型项，你就能定义一种类型来实现泛型比较函数

```go
type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 | ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr | ~float32 | ~float64 | ~string
}
```

Ordered 接口列出了所有支持 ==、!=、<、>、<= 和 >= 运算符的类型。指定变量为可排序类型，会非常实用，因此 Go 1.21 版本新增了 cmp 包，其定义这个 Ordered 接口。该包还定义了两个比较函数：Compare 函数根据第一个参数是小于、等于还是大于第二个参数，分别返回 -1、0 或 1；而 Less 函数在第一个参数小于第二个参数时返回 true。

在用作类型参数的接口中同时包含类型元素和方法元素是合法的。例如，你可以指定一个类型必须有 int 的底层类型和 String() string 方法：

```go
type PrintableInt interface {
    ~int
    String() string
}
```

请注意，Go 允许你声明一个实际上无法实例化的类型参数接口。如果 PrintableInt 用的是 int 而不是 ~int，就没有类型能够匹配它，因为 int 没有方法。这似乎不妥，但编译器会及时纠正。如果你声明一个带有不合理类型参数的类型或函数，任何尝试使用它的操作都会导致编译错误。假设你声明了这些类型：

```go
type ImpossiblePrintableInt interface {
    int
    String() string
}

type ImpossibleStruct[T ImpossiblePrintableInt] struct {
    val T
}

type MyInt int

func (mi MyInt) String() string {
    return fmt.Sprint(mi)
}
```

尽管你无法实例化 ImpossibleStruct，但编译器对这些声明并不会报错。然而，一旦你尝试使用 ImpossibleStruct，编译器就会报错。如下代码：

```go
s := ImpossibleStruct[int]{10}
s2 := ImpossibleStruct[MyInt]{10}
```

会产生如下编译时错误：

```bash
int does not implement ImpossiblePrintableInt (missing String method)
MyInt does not implement ImpossiblePrintableInt (possibly missing ~ for int in constraint ImpossiblePrintableInt)
```

可以在 [Go Playground](https://go.dev/play/p/MRSprnfhyeT) 或 [第 8 章代码仓库](https://github.com/learning-go-book-2e/ch08)中的 _sample_code/impossible_ 目录下试一试。

除了内置的基本类型外，类型项还可以是 slice、map、数组、channel、结构体，甚至是函数。当你想确保类型参数具有特定的底层类型以及一个或多个方法时，这些类型项尤其有用。

## 类型推断与泛型

正如 Go 在使用 := 操作符时支持类型推断（type inference）一样，Go 在调用泛型函数时也支持类型推断，从而简化调用过程。你可以在前面对 Map、Filter 和 Reduce 的调用中看到这一点。在某些情况下，类型推断会不起作用（例如，当类型参数仅作为返回值使用时）。在这种情况下，就必须指定所有的类型参数。这里有一段有点蠢的代码，示范了类型推断不起作用的情况：

```go
type Integer interface {
    int | int8 | int16 | int32 | int64 | uint | uint8 | uint16 | uint32 | uint64
}

func Convert[T1, T2 Integer](in T1) T2 {
    return T2(in)
}

func main() {
    var a int = 10
    b := Convert[int, int64](a) // can't infer the return type
    fmt.Println(b)
}
```

可以在 [Go Playground](https://go.dev/play/p/bDffkpsewcj) 上试一下，或在[第 8 章的仓库](https://github.com/learning-go-book-2e/ch08)中的 _sample_code/type_inference_ 目录中找到相应示例。

## 类型元素对常量的限制

类型元素还指定了哪些常量可以赋值给泛型类型的变量。跟运算符一样，这些常量需要对类型元素中的所有类型项都有效。在 Ordered 列出的每种类型中，均不存在可赋值的常量，所以你不能将一个常量赋值给该泛型类型的变量。如果你使用 Integer 接口，下面的代码将无法编译，因为你不能将值 1,000 赋值给一个 8 位整数：

```go
// INVALID!
func PlusOneThousand[T Integer](in T) T {
    return in + 1_000
}
```

然而，以下代码是有效的：

```go
// VALID
func PlusOneHundred[T Integer](in T) T {
    return in + 100
}
```

## 结合泛型函数与泛型数据结构

让我们回到二叉树的例子，看看如何结合所学的一切，来构建一个适用于各种具体类型的通用树（a single tree）。

关键是要意识到，你的树需要一个能够比较两个值并确定它们顺序的泛型函数：

```go
type OrderableFunc [T any] func(t1, t2 T) int
```

现在有了 OrderableFunc，可以稍微修改一下树的实现。首先，将其拆分为两个类型：Tree 和 Node：

```go
type Tree[T any] struct {
    f    OrderableFunc[T]
    root *Node[T]
}

type Node[T any] struct {
    val         T
    left, right *Node[T]
}
```

你可以用一个构造函数来构造一个新的 Tree：

```go
func NewTree[T any](f OrderableFunc[T]) *Tree[T] {
    return &Tree[T]{
        f: f,
    }
}
```

Tree 的方法非常简单，因为它们只是调用 Node 来完成所有实际的工作。

```go
func (t *Tree[T]) Add(v T) {
    t.root = t.root.Add(t.f, v)
}

func (t *Tree[T]) Contains(v T) bool {
    return t.root.Contains(t.f, v)
}
```

Node 上的 Add 和 Contains 方法与你之前看到的非常相似。唯一的区别是用来排序元素的函数是作为参数传入的：

```go
func (n *Node[T]) Add(f OrderableFunc[T], v T) *Node[T] {
    if n == nil {
        return &Node[T]{val: v}
    }
    switch r := f(v, n.val); {
    case r <= -1:
        n.left = n.left.Add(f, v)
    case r >= 1:
        n.right = n.right.Add(f, v)
    }
    return n
}

func (n *Node[T]) Contains(f OrderableFunc[T], v T) bool {
    if n == nil {
        return false
    }
    switch r := f(v, n.val); {
    case r <= -1:
        return n.left.Contains(f, v)
    case r >= 1:
        return n.right.Contains(f, v)
    }
    return true
}
```

现在你需要一个符合 OrderedFunc 定义的函数。幸运的是，你已见到过一个了：cmp 包中的 Compare。将它与 Tree 一起使用时，它看起来像这样：

```go
t1 := NewTree(cmp.Compare[int])
t1.Add(10)
t1.Add(30)
t1.Add(15)
fmt.Println(t1.Contains(15))
fmt.Println(t1.Contains(40))
```

对于结构体，你有两个选择。你可以写一个函数：

```go
type Person struct {
    Name string
    Age int
}

func OrderPeople(p1, p2 Person) int {
    out := cmp.Compare(p1.Name, p2.Name)
    if out == 0 {
        out = cmp.Compare(p1.Age, p2.Age)
    }
    return out
}
```

然后在创建树的时候传入该函数：

```go
t2 := NewTree(OrderPeople)
t2.Add(Person{"Bob", 30})
t2.Add(Person{"Maria", 35})
t2.Add(Person{"Bob", 50})
fmt.Println(t2.Contains(Person{"Bob", 30}))
fmt.Println(t2.Contains(Person{"Fred", 25}))
```

你也可以不使用函数，而是向 NewTree 提供一个方法。正如我在第 149 页的“方法也是函数”中谈到的，你可以使用方法表达式将方法当作函数来使用。让我们在这里实现一下。首先，写个方法：

```go
func (p Person) Order(other Person) int {
    out := cmp.Compare(p.Name, other.Name)
    if out == 0 {
        out = cmp.Compare(p.Age, other.Age)
    }
    return out
}
```

然后使用它：

```go
t3 := NewTree(Person.Order)
t3.Add(Person{"Bob", 30})
t3.Add(Person{"Maria", 35})
t3.Add(Person{"Bob", 50})
fmt.Println(t3.Contains(Person{"Bob", 30}))
fmt.Println(t3.Contains(Person{"Fred", 25}))
```

你可以在 [Go Playground](https://go.dev/play/p/g0ECtbJBEBr) 或者[第 8 章代码仓库](https://github.com/learning-go-book-2e/ch08)的 _sample_code/generic_tree_ 目录中找到这个树的代码。

## 可比较类型续讲

正如你在第 165 页的“接口是可比较的”中所看到的，接口是 Go 中的可比较类型之一。这意味着在使用接口类型的变量时，需要谨慎使用 == 和 != 。如果接口的底层类型不可比较，你的代码在运行时会触发 panic 。

在泛型中使用可比较接口时，这个陷阱依旧存在。假设你定义了一个接口及几个实现：

```go
type Thinger interface {
    Thing()
}

type ThingerInt int

func (t ThingerInt) Thing() {
    fmt.Println("ThingInt:", t)
}

type ThingerSlice []int

func (t ThingerSlice) Thing() {
    fmt.Println("ThingSlice:", t)
}
```

你还定义了一个泛型函数，该函数只接受可比较类型的值：

```go
func Comparer[T comparable](t1, t2 T) {
    if t1 == t2 {
        fmt.Println("equal!")
    }
}
```

调用此函数时，使用 int 或 ThingerInt 类型的变量是合法的：

```go
var a int = 10
var b int = 10
Comparer(a, b) // prints true

var a2 ThingerInt = 20
var b2 ThingerInt = 20
Comparer(a2, b2) // prints true
```

但编译器不允许你使用 ThingerSlice（或[]int）类型的变量来调用此函数：

```go
var a3 ThingerSlice = []int{1, 2, 3}
var b3 ThingerSlice = []int{1, 2, 3}
Comparer(a3, b3) // compile fails: "ThingerSlice does not satisfy comparable"
```

然而，用 Thinger 类型的变量来调用是完全合法的。如果使用 ThingerInt，代码可以正常编译并按预期运行：

```go
var a4 Thinger = a2
var b4 Thinger = b2
Comparer(a4, b4) // prints true
```

你也可以将 ThingerSlice 赋值给 Thinger 类型的变量，但这时问题就出现了：

```go
a4 = a3
b4 = b3
Comparer(a4, b4) // compiles, panics at runtime
```

编译器不会阻止你构建这段代码，但如果运行它，程序将 panic 并输出错误信息 "panic: runtime error: comparing uncomparable type main.ThingerSlice (更多相关信息，请参考第 218 页的“panic 和 recover”)。"

你可以在 [Go Playground](https://go.dev/play/p/eEWZ2IuF47i) 或 [第八章代码仓库](https://github.com/learning-go-book-2e/ch08)中的 _sample_code/more_comparable_ 目录下尝试这些代码。

若想了解更多关于可比较类型与泛型如何交互的技术细节，以及为何做出这样的设计决策，请阅读 Go 团队的 Robert Griesemer 的博客文章 [“All Your Comparable Types”](https://go.dev/blog/comparable)。

## 未包含的特性

Go 坚持其小巧、专注的特性，因而在其泛型实现中未包含其他语言泛型所具备的多项特性。本节将概述 Go 泛型初始实现中未包含的一些特性。

尽管你可以构建一个同时适用于自定义类型和内置类型的通用树，但像 Python、Ruby 和 C++ 等语言则以不同的方式解决这个问题。它们包含了运算符重载，允许用户自定义类型为运算符指定实现。Go 不打算加入此特性。因此，这意味着你无法使用 range 遍历自定义的容器类型，或是使用 [] 对它们进行索引操作。

不包含运算符重载是有充分理由的。一方面，Go 的运算符数量相当可观。另一方面，Go 也没有函数或方法重载，因此你需要一种方式为不同类型指定不同的运算符功能。此外，运算符重载可能导致代码难以理解，因为开发人员为了巧妙利用符号，可能会为其赋予不同的含义（例如在 C++ 中，对于某些类型，<< 表示“位左移”，而对于其他类型，则表示“将右侧的值写入左侧的值”）。这些都是 Go 试图避免的可读性问题。

Go 泛型初始实现中，另一个未包含的实用特性是方法上的额外类型参数。回顾 Map/Reduce/Filter 函数，你或许认为将其作为方法会很有用，例如这样：

```go
type functionalSlice[T any] []T

// THIS DOES NOT WORK
func (fs functionalSlice[T]) Map[E any](f func(T) E) functionalSlice[E] {
    out := make(functionalSlice[E], len(fs))
    for i, v := range fs {
        out[i] = f(v)
    }
    return out
}

// THIS DOES NOT WORK
func (fs functionalSlice[T]) Reduce[E any](start E, f func(E, T) E) E {
    out := start
    for _, v := range fs {
        out = f(out, v)
    }
    return out
}
```

你可以像这样使用：

```go
var numStrings = functionalSlice[string]{"1", "2", "3"}
sum := numStrings.Map(func(s string) int {
    v, _ := strconv.Atoi(s)
    return v
}).Reduce(0, func(acc int, cur int) int {
    return acc + cur
})
```

不幸的是，对于函数式编程爱好者来说，这种方式并不生效。你不能进行方法的链式调用，而是需要嵌套函数调用，或者使用更易读的方式，一次调用一个函数，并将中间值赋值给变量。类型参数提案详细解释了未包含参数化方法（parameterized method）的原因。

Go 也没有可变类型参数。在第 95 页的“可变参数和切片”中讨论过，要实现一个接受可变数量参数的函数，你需要指定最后一个参数的类型以 ... 开头。例如，无法为这些可变参数指定某种类型模式，比如交替的字符串和整数。所有可变参数的变量都必须匹配单个已声明的类型，无论该类型是否为泛型。

Go 泛型中未包含的其他特性则更加晦涩难懂，包括以下几点：

-   特化（Specialization）

    除了泛型版本之外，一个函数或方法可以有一个或多个针对特定类型的重载版本。由于 Go 不支持重载，此特性不在考虑范围内。

-   柯里化（Currying）

    允许你基于另一个泛型函数或类型，通过指定部分类型参数来部分实例化函数或类型。

-   元编程（Metaprogramming）

    允许你指定在编译时运行的代码，以生成在运行时执行的代码。

## Go 语言习惯用法与泛型

引入泛型显然改变了一些关于如何地道使用 Go 的惯例。使用 float64 来表示任何数值类型的做法将成为过去。你应该使用 any 而不是 interface{} 来表示数据结构或函数参数中的未指定类型。你可以用一个函数处理不同的切片类型。但不要急于立即将你所有的代码改用类型参数。随着新的设计模式不断创造和改进，你的旧代码仍然可以正常工作。

现在判断泛型对性能的长期影响还为时过早。截至撰写本文时，编译时间没有受到影响。Go 1.18 编译器比之前的版本要慢，但 Go 1.20 的编译器解决了这个问题。

泛型对运行时的影响，也已有不少探讨。Vicent Marti 写了一篇[详细的博客文章](https://planetscale.com/blog/generics-can-make-your-go-code-slower)，探讨了泛型导致代码变慢的情况，并解释了其中的原因及实现细节。相反，Eli Bendersky 写了一篇[博客文章](https://eli.thegreenplace.net/2022/faster-sorting-with-go-generics/)，表明泛型能让排序算法更快。

尤其是，不要为了提高性能，而将一个接口参数的函数改为泛型类型参数的函数。例如，将这个简单函数：

```go
type Ager interface {
  age() int
}

func doubleAge(a Ager) int {
  return a.age() * 2
}
```

改为：

```go
func doubleAgeGeneric[T Ager](a T) int {
  return a.age() * 2
}
```

在 Go 1.20 中，这会使函数调用大约变慢了 30%。（对于一个复杂函数，性能差异并不显著。）你可以使用[第 8 章代码仓库](https://github.com/learning-go-book-2e/ch08)中 _sample_code/perf_ 目录下的代码运行基准测试。

这可能会让有其他语言泛型经验的开发者感到惊讶。例如，在 C++ 中，编译器使用泛型处理抽象数据类型，将通常在运行时进行的操作（确定使用的是哪种具体类型）转变为编译时操作，为每种具体类型生成一个唯一的函数。虽然这使生成的二进制文件更大，但也更快。正如 Vicent 在他的博客文章中解释的那样，当前的 Go 编译器仅为不同的底层类型生成唯一的函数。此外，所有指针类型共享一个生成的函数。为了区分传递给共享生成函数的不同类型，编译器添加了额外的运行时查找操作。这会导致性能下降。

同样，随着未来 Go 版本中泛型实现的成熟，预期运行时性能将有所提升。始终如一，我们的目标是写出既可维护又能充分满足需求的高效程序。使用第 393 页“使用基准测试”中讨论的基准测试和性能分析工具来评估和改进你的代码。

## 向标准库添加泛型

Go 1.18 版本中首次引入泛型时非常谨慎。它在全局块（universe block）中添加了新的接口 any 和 comparable，但标准库中没有进行任何 API 更改来支持泛型。风格上进行了调整：标准库中几乎所有使用 interface{} 的地方都被替换为 any。

现在，随着 Go 社区对泛型更加熟悉，我们开始看到更多变化。从 Go 1.21 开始，标准库有了使用泛型实现常见算法的函数，这些算法适用于切片、map 和并发。在第 3 章中，我介绍了 slices 和 maps 包中的 Equal 和 EqualFunc 函数。这些包中的其他函数简化了切片和 map 的操作。slices 包内的 Insert、Delete 和 DeleteFunc 函数，让开发者避开了编写那些意想不到地复杂的切片操作代码。maps.Clone 函数利用 Go 运行时提供了一种更快的方式来创建 map 的浅拷贝。在第 308 页的“代码只运行一次”章节中，你将学到 sync.OnceValue 和 sync.OnceValues，它们使用泛型来构建只被调用一次并返回一个或两个值的函数。建议优先使用这些包中的函数，而非自行实现。未来版本的标准库很可能会有更多利用泛型的新函数和类型。
