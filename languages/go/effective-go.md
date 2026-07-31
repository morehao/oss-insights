# Effective Go

> **原文**：[go.dev/doc/effective_go](https://go.dev/doc/effective_go)
>
> **翻译说明**：本文翻译自 Go 官方文档 Effective Go。正文使用中文，技术术语保留英文原文（如 `defer`、`goroutine`、`channel`、`slice`、`map`、`interface` 等），代码块原样保留不做修改。
>
> **时效性声明**：本文档撰写于 2009 年 Go 发布时，**不再活跃更新**。虽然对核心语言仍然是很好的指南，但不涵盖后续的重大变更，如泛型（generics）、模块系统（modules）以及新增的标准库。详见 [issue 28782](https://go.dev/issue/28782)。完整的变更列表请参阅 [release notes](https://go.dev/doc/devel/release)。

---

## 引言

Go 是一门开源的编程语言，专注于简洁、可靠和高效，专为大规模软件开发而设计。虽然它借鉴了现有语言的思想，但它有一些独特的特性，使得有效的 Go 程序在风格上与其他语言编写的程序截然不同。将 C++ 或 Java 程序直接逐行翻译成 Go，不太可能产生令人满意的结果——Java 程序应当用 Java 的思维编写，而非 Go 的思维。反过来，从 Go 的视角思考问题，可能会产生一个成功但完全不同的程序。换句话说，要写出好的 Go 代码，理解它的特性和惯用法（idioms）至关重要。同样重要的是了解 Go 编程的既定约定，例如命名、格式化、程序结构等，这样你写的程序才能被其他 Go 程序员轻松理解。

本文档提供了编写清晰、地道的 Go 代码的技巧。它是对[语言规范](https://go.dev/ref/spec)、[Go 语言之旅](https://go.dev/tour/)和[如何编写 Go 代码](https://go.dev/doc/code.html)的补充，建议你先阅读这些内容。

### 示例

Go 的[包源码](https://go.dev/src/)不仅作为核心标准库，也作为如何使用语言的示例。此外，许多包包含可独立运行的可执行示例，你可以直接在 [go.dev](https://go.dev/) 网站上运行，比如[这个](https://go.dev/pkg/strings/#example-Map)（如有必要，点击单词"Example"来展开它）。如果你对如何解决某个问题或如何实现某个功能有疑问，标准库中的文档、代码和示例可以为你提供答案、思路和背景。

## 格式化

格式化问题是最容易引发争论但也最无关紧要的问题。人们可以适应不同的格式化风格，但不必如此会更好，如果每个人都遵循相同的风格，花在这个话题上的时间就会更少。问题在于如何在没有冗长的风格指南的情况下实现这个目标。

在 Go 中，我们采用了一种不寻常的方法：让机器来处理大部分的格式化问题。`gofmt` 程序（也可以使用 `go fmt`，后者作用于包级别而非单文件级别）读取 Go 程序并以标准的缩进和垂直对齐方式输出源代码，同时保留并在必要时重新格式化注释。如果你想知道如何处理某些新的布局情况，运行 `gofmt`；如果结果看起来不对，重新组织你的程序（或提交一个关于 `gofmt` 的 bug），而不是绕过它。

例如，不必花时间对齐结构体字段上的注释。`gofmt` 会为你处理。给定以下声明：

```go
type T struct {
    name string // name of the object
    value int // its value
}
```

`gofmt` 会对齐列：

```go
type T struct {
    name    string // name of the object
    value   int    // its value
}
```

标准包中的所有 Go 代码都已使用 `gofmt` 格式化。

还有一些格式化的细节。简要说明如下：

- **缩进**：我们使用 tab 进行缩进，`gofmt` 默认输出 tab。只在必要时使用空格。
- **行长度**：Go 没有行长度限制。不必担心行会超长。如果某行感觉太长，折行并用额外的 tab 缩进。
- **括号**：Go 需要的括号比 C 和 Java 少：控制结构（`if`、`for`、`switch`）在语法上没有括号。此外，运算符优先级层次更短更清晰，所以：

```go
x<<8 + y<<16
```

它的含义与间距所暗示的一致，而不像其他语言那样令人困惑。

## 注释

Go 提供 C 风格的 `/* */` 块注释和 C++ 风格的 `//` 行注释。行注释是常规用法；块注释主要出现在包注释中，但在表达式中或禁用大段代码时也很实用。

出现在顶层声明之前（中间没有空行）的注释被视为对该声明的文档化注释。这些"doc comments"是给定 Go 包或命令的主要文档。关于 doc comments 的更多信息，参见 [Go Doc Comments](https://go.dev/doc/comment)。

## 命名

命名在 Go 中和在其他语言中一样重要。它们甚至有语义效果：一个名字在包外的可见性由其首字母是否大写决定。因此，花点时间讨论 Go 程序中的命名约定是值得的。

### 包名

当一个包被导入时，包名成为访问其内容的入口。例如：

```go
import "bytes"
```

之后导入的包可以引用 `bytes.Buffer`。如果所有使用该包的人都能用同一个名字来引用其内容，那会很有帮助，这意味着包名应该好：简短、简洁、富有表现力。按照惯例，包使用小写的单个单词命名；不需要下划线或混合大小写。倾向于简短，因为每个使用你包的人都会输入那个名字。不要预先担心冲突。包名只是导入时的默认名称；不需要在所有源代码中唯一，并且在罕见的冲突情况下，导入的包可以选择使用不同的本地名称。无论如何，混淆的情况很少，因为 import 中的文件名决定了正在使用的包。

另一个约定是包名是其源目录的基础名称；`src/encoding/base64` 中的包被导入为 `"encoding/base64"`，但其名称为 `base64`，而不是 `encoding_base64` 或 `encodingBase64`。

导入包的一方将使用包名来引用其内容，因此包中导出的名字可以利用这一点来避免重复。（不要使用 `import .` 语法，它可以简化必须在被测包外部运行的测试，但应避免在其他场景中使用。）例如，`bufio` 包中的缓冲 reader 类型叫做 `Reader`，而不是 `BufReader`，因为用户看到的是 `bufio.Reader`，这是一个清晰、简洁的名字。此外，由于导入的实体总是通过其包名来访问，`bufio.Reader` 不会与 `io.Reader` 冲突。类似地，用于创建 `ring.Ring` 新实例的函数——在 Go 中本质上是构造函数——通常会被命名为 `NewRing`，但由于 `Ring` 是该包唯一导出的类型，并且包名叫 `ring`，它被简单地命名为 `New`，包的调用方看到的是 `ring.New`。利用包的结构来帮助你选择好的名称。

另一个简短命名的例子是 `once.Do`；`once.Do(setup)` 读起来很顺畅，写成 `once.DoOrWaitUntilDone(setup)` 并不会更好。长名称不会自动让代码更可读。一条有用的 doc comment 往往比一个超长的名字更有价值。

### Getter

Go 不自动支持 getter 和 setter。自己提供 getter 和 setter 完全没有问题，而且通常也是合适的，但把 `Get` 放入 getter 的名称中既不符合惯用法，也不是必需的。如果你有一个名为 `owner` 的字段（小写，未导出），其 getter 方法应该叫 `Owner`（大写，导出），而不是 `GetOwner`。使用大写名称进行导出，提供了区分字段和方法的机制。如果需要 setter 函数，它很可能会被命名为 `SetOwner`。这两种命名在实践中读起来都很好：

```go
owner := obj.Owner()
if owner != user {
    obj.SetOwner(user)
}
```

### 接口名

按照约定，只有一个方法的接口以方法名加上 `-er` 后缀或类似的修饰来命名，构成代理名词：`Reader`、`Writer`、`Formatter`、`CloseNotifier` 等。

这些名称以及它们所捕获的函数名称有很多，遵守它们是很有益的。`Read`、`Write`、`Close`、`Flush`、`String` 等具有规范的签名和含义。为避免混淆，除非你的方法具有相同的签名和含义，否则不要给它取这些名字。反之，如果你的类型实现了一个与某个知名类型上的方法含义相同的方法，给它取相同的名字和签名；将你的字符串转换方法命名为 `String`，而不是 `ToString`。

### MixedCaps

最后，Go 中的约定是使用 `MixedCaps` 或 `mixedCaps` 而不是下划线来书写多词名称。

## 分号

和 C 一样，Go 的形式语法使用分号来终止语句，但与 C 不同的是，这些分号不会出现在源代码中。相反，词法分析器在扫描时使用一条简单的规则自动插入分号，因此输入文本中基本上不存在分号。

规则是这样的：如果换行前的最后一个 token 是一个标识符（包括 `int` 和 `float64` 之类的单词）、一个基本字面量（如数字或字符串常量）或是以下 token 之一：

```
break continue fallthrough return ++ -- ) }
```

词法分析器总是在该 token 之后插入一个分号。这可以总结为："如果换行出现在一个可能结束语句的 token 之后，就插入分号"。

分号也可以在闭合大括号之前立即省略，所以像这样的语句：

```go
    go func() { for { dst <- <-src } }()
```

不需要分号。地道的 Go 程序只在诸如 `for` 循环子句等地方使用分号，用来分隔初始化、条件和递增元素。如果在同一行写多个语句，分号也是必需的——如果你这样写代码的话。

分号插入规则的一个后果是你不能把控制结构（`if`、`for`、`switch` 或 `select`）的左大括号放在下一行。如果这样做，分号会被插入到括号之前，这可能会导致意外的效果。这样写：

```go
if i < f() {
    g()
}
```

而不是这样写：

```go
if i < f()  // wrong!
{           // wrong!
    g()
}
```

## 控制结构

Go 的控制结构与 C 语言相关，但有重要的区别。没有 `do` 或 `while` 循环，只有一个稍微泛化的 `for`；`switch` 更加灵活；`if` 和 `switch` 接受一个可选的初始化语句，类似于 `for` 的用法；`break` 和 `continue` 语句接受一个可选的标签来标识要 break 或 continue 的目标；还有新的控制结构，包括 type switch 和多路通信复用器 `select`。语法也有细微的不同：没有括号，并且函数体必须始终用大括号括起来。

### If

在 Go 中，一个简单的 `if` 看起来像这样：

```go
if x > 0 {
    return y
}
```

强制使用大括号鼓励将简单的 `if` 语句写成多行。无论如何这都是好的风格，尤其是当函数体包含诸如 `return` 或 `break` 等控制语句时。

由于 `if` 和 `switch` 接受初始化语句，使用它们来设置局部变量是很常见的。

```go
if err := file.Chmod(0664); err != nil {
    log.Print(err)
    return err
}
```

在 Go 标准库中，你会发现当 `if` 语句不流向下一语句时——即函数体以 `break`、`continue`、`goto` 或 `return` 结束时——不必要的 `else` 会被省略。

```go
f, err := os.Open(name)
if err != nil {
    return err
}
codeUsing(f)
```

这是一个常见情况的示例：代码必须防范一系列错误条件。如果成功的控制流沿着页面向下运行，在错误出现时逐项消除，代码阅读起来会很顺畅。由于错误情况倾向于以 `return` 语句结束，因此生成的代码不需要 `else` 语句。

```go
f, err := os.Open(name)
if err != nil {
    return err
}
d, err := f.Stat()
if err != nil {
    f.Close()
    return err
}
codeUsing(f, d)
```

### 重新声明与重新赋值

插一句：上一节的最后一个示例展示了 `:=` 简短声明形式的一个细节。调用 `os.Open` 的声明如下：

```go
f, err := os.Open(name)
```

这个语句声明了两个变量 `f` 和 `err`。几行之后，调用 `f.Stat` 的语句如下：

```go
d, err := f.Stat()
```

这看起来像是声明了 `d` 和 `err`。但请注意，`err` 出现在两个语句中。这种重复是合法的：`err` 在第一个语句中被声明，但在第二个语句中只是**重新赋值**。这意味着对 `f.Stat` 的调用使用了上面声明的现有 `err` 变量，只是给它了一个新值。

在 `:=` 声明中，变量 `v` 即使已经被声明过也可以出现，前提是：

- 此声明与 `v` 的现有声明在同一个作用域内（如果 `v` 已经在外部作用域中声明，声明将创建一个新变量 §）；
- 初始化中对应的值可赋值给 `v`；并且
- 声明至少创建了一个其他新变量。

这个不寻常的特性纯粹是实用性的，使得在较长的 `if-else` 链中，很容易使用单一的 `err` 值。你会经常看到它被使用。

§ 这里值得注意的是，在 Go 中，函数参数和返回值的作用域与函数体相同，尽管它们在词法上出现在包围函数体的大括号之外。

### For

Go 的 `for` 循环与 C 的类似但并不完全相同。它统一了 `for` 和 `while`，并且没有 `do-while`。有三种形式，只有一种包含分号。

```go
// Like a C for
for init; condition; post { }

// Like a C while
for condition { }

// Like a C for(;;)
for { }
```

简短声明可以方便地在循环中声明索引变量。

```go
sum := 0
for i := 0; i < 10; i++ {
    sum += i
}
```

如果你在遍历 array、slice、string 或 map，或者从 channel 读取，可以使用 `range` 子句来管理循环。

```go
for key, value := range oldMap {
    newMap[key] = value
}
```

如果你只需要 range 中的第一个元素（键或索引），丢掉第二个：

```go
for key := range m {
    if key.expired() {
        delete(m, key)
    }
}
```

如果你只需要 range 中的第二个元素（值），使用**空白标识符**（下划线）来丢弃第一个：

```go
sum := 0
for _, value := range array {
    sum += value
}
```

空白标识符有很多用途，如[后面的章节](#空白标识符)所述。

对于 string，`range` 会做更多工作，通过解析 UTF-8 来分解出单个 Unicode 码点。错误的编码消耗一个字节并产生替换 rune U+FFFD。（名称 `rune`（及其关联的内置类型）是 Go 术语，表示单个 Unicode 码点。详见[语言规范](https://go.dev/ref/spec#Rune_literals)。）这个循环：

```go
for pos, char := range "日本\x80語" { // \x80 is an illegal UTF-8 encoding
    fmt.Printf("character %#U starts at byte position %d\n", char, pos)
}
```

输出：

```
character U+65E5 '日' starts at byte position 0
character U+672C '本' starts at byte position 3
character U+FFFD '�' starts at byte position 6
character U+8A9E '語' starts at byte position 7
```

最后，Go 没有逗号运算符，且 `++` 和 `--` 是语句而非表达式。因此，如果你想在 `for` 中操作多个变量，应该使用并行赋值（尽管这会排除 `++` 和 `--` 的使用）。

```go
// Reverse a
for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
    a[i], a[j] = a[j], a[i]
}
```

### Switch

Go 的 `switch` 比 C 的更通用。表达式不必是常量甚至不必是整数，case 从上到下评估直到找到匹配，如果 `switch` 没有表达式，它默认 switch 在 `true` 上。因此，将 `if`-`else`-`if`-`else` 链写成 `switch` 是可能且地道的写法。

```go
func unhex(c byte) byte {
    switch {
    case '0' <= c && c <= '9':
        return c - '0'
    case 'a' <= c && c <= 'f':
        return c - 'a' + 10
    case 'A' <= c && c <= 'F':
        return c - 'A' + 10
    }
    return 0
}
```

没有自动的 fall through，但 case 可以用逗号分隔的列表来表示。

```go
func shouldEscape(c byte) bool {
    switch c {
    case ' ', '?', '&', '=', '#', '+', '%':
        return true
    }
    return false
}
```

尽管在 Go 中不像其他一些类 C 语言中那么常见，`break` 语句可以用来提前终止一个 `switch`。但有时需要跳出外层的循环而不是 switch，在 Go 中这可以通过给循环加标签并 `break` 到该标签来实现。以下示例展示了这两种用法。

```go
Loop:
    for n := 0; n < len(src); n += size {
        switch {
        case src[n] < sizeOne:
            if validateOnly {
                break
            }
            size = 1
            update(src[n])

        case src[n] < sizeTwo:
            if n+1 >= len(src) {
                err = errShortInput
                break Loop
            }
            if validateOnly {
                break
            }
            size = 2
            update(src[n] + src[n+1]<<shift)
        }
    }
```

当然，`continue` 语句也接受一个可选标签，但它仅适用于循环。

作为本节的收尾，这里是一个使用两个 `switch` 语句的字节 slice 比较函数：

```go
// Compare returns an integer comparing the two byte slices,
// lexicographically.
// The result will be 0 if a == b, -1 if a < b, and +1 if a > b
func Compare(a, b []byte) int {
    for i := 0; i < len(a) && i < len(b); i++ {
        switch {
        case a[i] > b[i]:
            return 1
        case a[i] < b[i]:
            return -1
        }
    }
    switch {
    case len(a) > len(b):
        return 1
    case len(a) < len(b):
        return -1
    }
    return 0
}
```

### Type switch

switch 也可以用来发现 interface 变量的动态类型。这种 **type switch** 使用类型断言的语法，关键字 `type` 在括号内。如果 switch 在表达式中声明了一个变量，该变量在每个分支中将具有相应的类型。在这种情况下，重用变量名也是地道的写法，实际上在每个 case 中声明了一个同名但不同类型的新变量。

```go
var t interface{}
t = functionOfSomeType()
switch t := t.(type) {
default:
    fmt.Printf("unexpected type %T\n", t)     // %T 打印 t 的类型
case bool:
    fmt.Printf("boolean %t\n", t)             // t 的类型为 bool
case int:
    fmt.Printf("integer %d\n", t)             // t 的类型为 int
case *bool:
    fmt.Printf("pointer to boolean %t\n", *t) // t 的类型为 *bool
case *int:
    fmt.Printf("pointer to integer %d\n", *t) // t 的类型为 *int
}
```

## 函数

### 多返回值

Go 的一个不寻常的特性是函数和方法可以返回多个值。这种形式可以用来改进 C 程序中的几个笨拙的惯用法：带内错误返回（如用 `-1` 表示 `EOF`）以及通过地址修改传入的参数。

在 C 中，写错误通过负数计数来标识，而错误代码被隐藏在某个易变的位置。在 Go 中，`Write` 可以返回一个计数**和**一个 error："对，你写入了一些字节，但不是全部，因为设备已满"。来自 `os` 包的文件类型上的 `Write` 方法的签名为：

```go
func (file *File) Write(b []byte) (n int, err error)
```

正如文档所述，它返回写入的字节数，当 `n != len(b)` 时返回非 nil 的 `error`。这是一个常见的风格；更多示例请参见错误处理章节。

类似的方法消除了传递返回值指针来模拟引用参数的需要。这里有一个简单的函数，从 byte slice 的某个位置获取一个数字，返回数字和下一个位置。

```go
func nextInt(b []byte, i int) (int, int) {
    for ; i < len(b) && !isDigit(b[i]); i++ {
    }
    x := 0
    for ; i < len(b) && isDigit(b[i]); i++ {
        x = x*10 + int(b[i]) - '0'
    }
    return x, i
}
```

你可以像这样用它来扫描输入 slice `b` 中的数字：

```go
    for i := 0; i < len(b); {
        x, i = nextInt(b, i)
        fmt.Println(x)
    }
```

### 命名的返回值参数

Go 函数的返回或结果"参数"可以被命名，并像传入参数一样作为常规变量使用。当被命名时，它们在函数开始时被初始化为各自类型的零值；如果函数执行了没有参数的 `return` 语句，则结果参数的当前值将被用作返回值。

这些名称不是强制性的，但它们可以使代码更短更清晰：它们是文档。如果我们命名 `nextInt` 的结果，哪个返回的 `int` 对应什么就一目了然了。

```go
func nextInt(b []byte, pos int) (value, nextPos int) {
```

由于命名的结果被初始化并与无参 return 绑定，它们可以同时简化和澄清代码。这是一个 `io.ReadFull` 的版本，很好地使用了它们：

```go
func ReadFull(r Reader, buf []byte) (n int, err error) {
    for len(buf) > 0 && err == nil {
        var nr int
        nr, err = r.Read(buf)
        n += nr
        buf = buf[nr:]
    }
    return
}
```

### Defer

Go 的 `defer` 语句安排了一个函数调用（被延迟的函数）在包含 `defer` 的函数返回之前立即运行。这是一种不寻常但有效的方法，用于处理诸如无论函数从哪条路径返回都必须释放的资源等情况。典型的示例是解锁一个 mutex 或关闭一个文件。

```go
// Contents returns the file's contents as a string.
func Contents(filename string) (string, error) {
    f, err := os.Open(filename)
    if err != nil {
        return "", err
    }
    defer f.Close()  // f.Close 会在我们完成时运行。

    var result []byte
    buf := make([]byte, 100)
    for {
        n, err := f.Read(buf[0:])
        result = append(result, buf[0:n]...) // append 稍后讨论。
        if err != nil {
            if err == io.EOF {
                break
            }
            return "", err  // 如果在这里返回，f 也会被关闭。
        }
    }
    return string(result), nil // 如果在这里返回，f 也会被关闭。
}
```

延迟调用一个函数（如 `Close`）有两个优点。首先，它保证你永远不会忘记关闭文件，这是一个当你后来编辑函数添加新的返回路径时容易犯的错误。其次，它意味着关闭操作紧邻打开操作，这比将其放在函数末尾要清晰得多。

延迟函数的参数（包括 receiver，如果该函数是方法的话）在 `defer` 执行时评估，而不是在调用执行时评估。除了避免担心变量在函数执行过程中改变值之外，这意味着一个单一的 defer 调用点可以延迟多次函数执行。这里有一个简单的示例：

```go
for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i)
}
```

延迟函数按 LIFO 顺序执行，因此这段代码会在函数返回时打印 `4 3 2 1 0`。一个更合理的示例是追踪程序中的函数执行。我们可以编写几个简单的追踪例程序：

```go
func trace(s string)   { fmt.Println("entering:", s) }
func untrace(s string) { fmt.Println("leaving:", s) }

// 像这样使用它们：
func a() {
    trace("a")
    defer untrace("a")
    // do something....
}
```

我们可以利用延迟函数的参数在 `defer` 执行时评估这一事实做得更好。追踪例程可以设置 untrace 例程的参数。这个示例：

```go
func trace(s string) string {
    fmt.Println("entering:", s)
    return s
}

func un(s string) {
    fmt.Println("leaving:", s)
}

func a() {
    defer un(trace("a"))
    fmt.Println("in a")
}

func b() {
    defer un(trace("b"))
    fmt.Println("in b")
    a()
}

func main() {
    b()
}
```

输出：

```
entering: b
in b
entering: a
in a
leaving: a
leaving: b
```

对于习惯于来自其他语言的块级资源管理的程序员来说，`defer` 可能看起来有些奇特，但它最有趣和最强大的应用恰恰来自它不是基于块而是基于函数的这一事实。在 `panic` 和 `recover` 章节中，我们将看到它的另一个应用。


## 数据

### 使用 `new` 分配

Go 有两个内存分配原语，内置函数 `new` 和 `make`。它们做不同的事情，适用于不同的类型，这可能会令人困惑，但规则很简单。先讨论 `new`。它是一个分配内存的内置函数，但与一些其他语言中的同名函数不同，它不**初始化**内存，只将其**归零**。也就是说，`new(T)` 为类型 `T` 的新变量分配零值存储空间，并返回其地址，即一个类型为 `*T` 的值。用 Go 的术语来说，它返回一个指向类型 `T` 的新分配的零值的指针。

从 Go 1.26 开始，`new` 也接受一个（值）表达式作为参数，用于指定变量的初始值。例如，`new(int64(300))` 分配一个类型为 `int64` 的新变量，初始化为 300，并返回其地址。

由于 `new` 返回的内存已被归零，因此当你设计数据结构时，如果每种类型的零值无需进一步初始化即可使用，会很有帮助。这意味着数据结构的使用者可以用 `new` 创建并直接使用之。例如，`bytes.Buffer` 的文档声明"Buffer 的零值是一个可以立即使用的空 buffer"。类似地，`sync.Mutex` 没有显式的构造函数或 `Init` 方法。相反，`sync.Mutex` 的零值被定义为未锁定的 mutex。

零值即可用的属性是可传递的。考虑这个类型声明：

```go
type SyncedBuffer struct {
    lock    sync.Mutex
    buffer  bytes.Buffer
}
```

`SyncedBuffer` 类型的值在分配或声明后也可以立即使用。在以下代码片段中，`p` 和 `v` 无需进一步设置即可正确工作。

```go
p := new(SyncedBuffer)  // type *SyncedBuffer
var v SyncedBuffer      // type  SyncedBuffer
```

### 构造函数和复合字面量

有时零值不够好，初始化构造函数是必需的，如下面从 `os` 包派生的示例所示。

```go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := new(File)
    f.fd = fd
    f.name = name
    f.dirinfo = nil
    f.nepipe = 0
    return f
}
```

这里面有很多样板代码。我们可以使用**复合字面量（composite literal）**来简化它，这是一种每次评估时都创建新实例的表达式。

```go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := File{fd, name, nil, 0}
    return &f
}
```

注意，与 C 不同，返回局部变量的地址是完全 OK 的；与该变量相关联的存储空间在函数返回后仍然存在。实际上，获取复合字面量的地址会在每次评估时分配一个新的实例，因此我们可以合并最后两行。

```go
    return &File{fd, name, nil, 0}
```

复合字面量的字段按顺序排列，必须全部存在。但是，通过将元素显式标记为 `field:value` 对，初始化器可以以任何顺序出现，缺失的字段保留各自的零值。因此我们可以这样写：

```go
    return &File{fd: fd, name: name}
```

作为极限情况，如果一个复合字面量完全没有任何字段，它创建该类型的零值。表达式 `new(File)` 和 `&File{}` 是等价的。

复合字面量也可以为 array、slice 和 map 创建，字段标签可以是索引或 map key。在以下示例中，无论 `Enone`、`Eio` 和 `Einval` 的值如何（只要它们是不同的），初始化都能正确工作。

```go
a := [...]string   {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
s := []string      {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
m := map[int]string{Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
```

### 使用 `make` 分配

回到分配的话题。内置函数 `make(T, args)` 的目的与 `new(T)` 不同。它仅创建 slice、map 和 channel，并返回一个**已初始化**（而非**已归零**）的 `T` 类型值（而非 `*T`）。这种区别的原因是这三种类型在底层表示的是对数据结构的引用，这些数据结构在使用前必须初始化。例如，slice 是一个三项描述符，包含指向数据（在底层 array 中）的指针、长度和容量，在这些项被初始化之前，slice 是 `nil`。对于 slice、map 和 channel，`make` 初始化内部数据结构并准备好值以便使用。例如：

```go
make([]int, 10, 100)
```

分配一个包含 100 个 int 的 array，然后创建一个长度为 10、容量为 100 的 slice 结构，指向该 array 的前 10 个元素。（创建 slice 时，容量可以省略；更多信息参见 slice 章节。）相比之下，`new([]int)` 返回一个指向新分配的、已归零的 slice 结构的指针——即指向 `nil` slice 值的指针。

以下示例说明了 `new` 和 `make` 之间的区别：

```go
var p *[]int = new([]int)       // 分配 slice 结构；*p == nil；几乎不常用
var v  []int = make([]int, 100) // slice v 现在指向一个新的包含 100 个 int 的 array

// 不必要的复杂写法：
var p *[]int = new([]int)
*p = make([]int, 100, 100)

// 地道的写法：
v := make([]int, 100)
```

记住，`make` 仅适用于 map、slice 和 channel，并且不返回指针。要获得显式指针，请使用 `new` 分配或显式获取变量的地址。

### Array

Array 在规划内存的详细布局时很有用，有时可以帮助避免分配，但主要是作为 slice 的构建块，slice 是下一节的主题。为了为该主题奠定基础，这里简要介绍一下 array。

Go 中 array 的工作方式与 C 中有重大区别。在 Go 中：

- Array 是值。将一个 array 赋值给另一个会复制所有元素。
- 特别是，如果你将一个 array 传递给函数，函数将收到该 array 的一个**副本**，而非指向它的指针。
- Array 的大小是其类型的一部分。类型 `[10]int` 和 `[20]int` 是不同的。

值属性可能有用但也可能代价高昂；如果你想要类似 C 的行为和效率，可以传递一个指向 array 的指针。

```go
func Sum(a *[3]float64) (sum float64) {
    for _, v := range *a {
        sum += v
    }
    return
}

array := [...]float64{7.0, 8.5, 9.1}
x := Sum(&array)  // 注意显式的取地址运算符
```

但即使这种风格也不是地道的 Go 写法。请使用 slice。

### Slice

Slice 包装了 array，为数据序列提供了更通用、更强大、更方便的接口。除了具有显式维度的项（如变换矩阵），Go 中的大多数 array 编程都使用 slice 而非简单 array。

Slice 持有对底层 array 的引用，如果你将一个 slice 赋值给另一个，两者引用同一个 array。如果函数接受一个 slice 参数，它对 slice 元素所做的更改将对调用方可见，类似于传递指向底层 array 的指针。因此，`Read` 函数可以接受一个 slice 参数而不是指针和计数；slice 的长度设定了读取数据量的上限。以下是 `os` 包中 `File` 类型的 `Read` 方法的签名：

```go
func (f *File) Read(buf []byte) (n int, err error)
```

该方法返回读取的字节数和一个错误值（如果有）。要读取到较大 buffer `buf` 的前 32 个字节，对 buffer 进行切片（slice，这里用作动词）。

```go
    n, err := f.Read(buf[0:32])
```

这样的切片操作很常见且高效。事实上，暂且不考虑效率的话，以下代码片段也会读取 buffer 的前 32 个字节：

```go
    var n int
    var err error
    for i := 0; i < 32; i++ {
        nbytes, e := f.Read(buf[i:i+1])  // Read one byte.
        n += nbytes
        if nbytes == 0 || e != nil {
            err = e
            break
        }
    }
```

slice 的长度可以改变，只要它仍然适合底层 array 的限制；只需将其赋值给其自身的一个子 slice 即可。slice 的**容量**，可通过内置函数 `cap` 访问，报告 slice 可能达到的最大长度。下面是一个向 slice 追加数据的函数。如果数据超过容量，slice 会被重新分配。结果 slice 被返回。该函数利用了这样一个事实：`len` 和 `cap` 在应用于 `nil` slice 时是合法的，并返回 0。

```go
func Append(slice, data []byte) []byte {
    l := len(slice)
    if l + len(data) > cap(slice) {  // reallocate
        // 分配所需容量的两倍，以供未来增长。
        newSlice := make([]byte, (l+len(data))*2)
        // copy 函数是预先声明的，适用于任何 slice 类型。
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:l+len(data)]
    copy(slice[l:], data)
    return slice
}
```

我们必须返回 slice，因为虽然 `Append` 可以修改 `slice` 的元素，但 slice 本身（持有指针、长度和容量的运行时数据结构）是按值传递的。

向 slice 追加数据这个想法非常有用，以至于它被内置函数 `append` 所涵盖。但要理解该函数的设计，我们需要更多信息，稍后再回来讨论。

### 二维 slice

Go 的 array 和 slice 是一维的。要创建等效的二维 array 或 slice，需要定义 array 的 array 或 slice 的 slice，如下所示：

```go
type Transform [3][3]float64  // 3x3 的 array，实际上是 array 的 array。
type LinesOfText [][]byte     // byte slice 的 slice。
```

因为 slice 是可变长度的，所以每个内部 slice 可以有不同的长度。这可能是一种常见的情况，就像在我们的 `LinesOfText` 示例中：每行有独立的长度。

```go
text := LinesOfText{
    []byte("Now is the time"),
    []byte("for all good gophers"),
    []byte("to bring some fun to the party."),
}
```

有时需要分配二维 slice，例如在处理像素扫描线时。有两种方法可以实现。一种是独立分配每个 slice；另一种是分配一个单独的 array，并将各个 slice 指向其中。选择哪种方法取决于你的应用。如果 slice 可能会增长或缩小，应该独立分配以避免覆盖下一行；如果不是，用单次分配来构建对象可能更高效。以下是两种方法的概要。首先，逐行分配：

```go
// 分配顶层 slice。
picture := make([][]uint8, YSize) // 每 y 单位一行。
// 遍历行，为每行分配 slice。
for i := range picture {
    picture[i] = make([]uint8, XSize)
}
```

然后是单次分配，切分成行：

```go
// 分配顶层 slice，与之前相同。
picture := make([][]uint8, YSize) // 每 y 单位一行。
// 分配一个大的 slice 来保存所有像素。
pixels := make([]uint8, XSize*YSize) // 类型为 []uint8，尽管 picture 是 [][]uint8。
// 遍历行，从剩余 pixels slice 的前端切割每一行。
for i := range picture {
    picture[i], pixels = pixels[:XSize], pixels[XSize:]
}
```


### Map

Map 是一种方便而强大的内置数据结构，将一种类型的值（**key**）与另一种类型的值（**元素**或**值**）关联起来。key 可以是任何定义了相等运算符的类型，例如整数、浮点数和复数、string、指针、interface（只要动态类型支持相等）、struct 和 array。Slice 不能用作 map key，因为对它们没有定义相等。与 slice 一样，map 持有对底层数据结构的引用。如果你将 map 传递给一个函数并修改了 map 的内容，这些更改对调用方是可见的。

Map 可以使用通常的复合字面量语法来构建，带有冒号分隔的 key-value 对，因此在初始化时很容易构建它们。

```go
var timeZone = map[string]int{
    "UTC":  0*60*60,
    "EST": -5*60*60,
    "CST": -6*60*60,
    "MST": -7*60*60,
    "PST": -8*60*60,
}
```

赋值和获取 map 值在语法上与对 array 和 slice 的操作一样，只是索引不需要是整数。

```go
offset := timeZone["EST"]
```

尝试用不存在于 map 中的 key 获取 map 值将返回该 map 中条目类型的零值。例如，如果 map 包含整数，查找一个不存在的 key 将返回 `0`。Set 可以用值类型为 `bool` 的 map 来实现。将 map 条目设置为 `true` 以将值放入 set，然后通过简单的索引来测试。

```go
attended := map[string]bool{
    "Ann": true,
    "Joe": true,
    ...
}

if attended[person] { // 如果 person 不在 map 中，将为 false
    fmt.Println(person, "was at the meeting")
}
```

有时你需要区分缺失条目和零值。"UTC"有一个条目，还是因为根本不在 map 中所以是 0？你可以通过一种多赋值形式来区分。

```go
var seconds int
var ok bool
seconds, ok = timeZone[tz]
```

出于显而易见的原因，这被称为"comma ok"惯用法。在此示例中，如果 `tz` 存在，`seconds` 将被适当地设置，`ok` 将为 true；如果不存在，`seconds` 将被设置为零，`ok` 将为 false。这是一个将它与友好的错误报告结合在一起的函数：

```go
func offset(tz string) int {
    if seconds, ok := timeZone[tz]; ok {
        return seconds
    }
    log.Println("unknown time zone:", tz)
    return 0
}
```

要测试 map 中是否存在某个 key 而不关心实际值，你可以使用[空白标识符](#空白标识符)（`_`）代替通常的值变量。

```go
_, present := timeZone[tz]
```

要删除 map 条目，使用内置函数 `delete`，其参数为 map 和要删除的 key。即使 key 已经不存在于 map 中，这么做也是安全的。

```go
delete(timeZone, "PDT")  // 现在是标准时间了
```

### Printing

Go 中的格式化打印使用与 C 的 `printf` 系列类似的风格，但更丰富、更通用。这些函数位于 `fmt` 包中，名称大写：`fmt.Printf`、`fmt.Fprintf`、`fmt.Sprintf` 等。string 函数（`Sprintf` 等）返回一个 string，而不是填充一个提供的 buffer。

你不必提供格式字符串。对于 `Printf`、`Fprintf` 和 `Sprintf` 中的每一个，都有另一对函数，例如 `Print` 和 `Println`。这些函数不接受格式字符串，而是为每个参数生成默认格式。`Println` 版本还会在参数之间插入空格并在输出后追加一个换行，而 `Print` 版本仅在两侧操作数都不是 string 时添加空格。在这个例子中，每一行产生相同的输出。

```go
fmt.Printf("Hello %d\n", 23)
fmt.Fprint(os.Stdout, "Hello ", 23, "\n")
fmt.Println("Hello", 23)
fmt.Println(fmt.Sprint("Hello ", 23))
```

格式化打印函数 `fmt.Fprint` 及其同类以任何实现了 `io.Writer` 接口的对象作为第一个参数；变量 `os.Stdout` 和 `os.Stderr` 是常见的实例。

这里开始与 C 有所不同。首先，数字格式（如 `%d`）不接受有符号或无符号标志；相反，打印例程使用参数的类型来决定这些属性。

```go
var x uint64 = 1<<64 - 1
fmt.Printf("%d %x; %d %x\n", x, x, int64(x), int64(x))
```

输出：

```
18446744073709551615 ffffffffffffffff; -1 -1
```

如果你只想要默认的转换，比如整数的十进制，可以使用万能格式 `%v`（表示"value"）；结果正是 `Print` 和 `Println` 会产生的。此外，这种格式可以打印**任何**值，甚至 array、slice、struct 和 map。这是打印上一节中定义的时区 map 的一个打印语句。

```go
fmt.Printf("%v\n", timeZone)  // 或者直接用 fmt.Println(timeZone)
```

输出为：

```
map[CST:-21600 EST:-18000 MST:-25200 PST:-28800 UTC:0]
```

对于 map，`Printf` 及其同类会按 key 的字典顺序排序输出。

当打印一个 struct 时，修改格式 `%+v` 会用它们的名称注释 struct 的字段，对于任何值，替代格式 `%#v` 会以完整的 Go 语法打印该值。

```go
type T struct {
    a int
    b float64
    c string
}
t := &T{ 7, -2.35, "abc\tdef" }
fmt.Printf("%v\n", t)
fmt.Printf("%+v\n", t)
fmt.Printf("%#v\n", t)
fmt.Printf("%#v\n", timeZone)
```

输出：

```
&{7 -2.35 abc   def}
&{a:7 b:-2.35 c:abc     def}
&main.T{a:7, b:-2.35, c:"abc\tdef"}
map[string]int{"CST":-21600, "EST":-18000, "MST":-25200, "PST":-28800, "UTC":0}
```

（注意取地址符 &。）引号字符串格式也可以通过 `%q` 应用于类型为 `string` 或 `[]byte` 的值。替代格式 `%#q` 在可能时将使用反引号。（`%q` 格式也适用于整数和 rune，产生一个单引号 rune 常量。）另外，`%x` 适用于 string、byte array 和 byte slice 以及整数，生成一个长的十六进制字符串，并且如果在格式中加入空格（`% x`），会在字节之间放置空格。

另一个方便的格式是 `%T`，它打印值的**类型**。

```go
fmt.Printf("%T\n", timeZone)
```

输出：

```
map[string]int
```

如果你想要为自定义类型控制默认格式，只需要在该类型上定义一个签名为 `String() string` 的方法。对于我们的简单类型 `T`，可能看起来像这样。

```go
func (t *T) String() string {
    return fmt.Sprintf("%d/%g/%q", t.a, t.b, t.c)
}
fmt.Printf("%v\n", t)
```

打印输出为：

```
7/-2.35/"abc\tdef"
```

（如果你需要打印类型 `T` 的**值**以及指向 `T` 的指针，`String` 的 receiver 必须是值类型；这个示例使用了指针，因为对于 struct 类型，这更高效也更地道。更多信息参见下面的[指针 vs 值 receiver](#指针-vs-值)章节。）

我们的 `String` 方法能够调用 `Sprintf`，因为打印例程是完全可重入的，可以这样被包装。然而，关于这种方法有一个重要的细节需要理解：不要通过以某种会无限递归回你的 `String` 方法的方式来调用 `Sprintf` 来构造 `String` 方法。如果 `Sprintf` 调用尝试直接将 receiver 作为 string 打印，就会发生这种情况，这反过来会再次调用该方法。这是一个常见且容易犯的错误，正如下面这个示例所示。

```go
type MyString string

func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", m) // 错误：会无限递归。
}
```

但也很容易修复：将参数转换为不包含该方法的底层 string 类型。

```go
type MyString string
func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", string(m)) // OK：注意类型转换。
}
```

在[初始化章节](#初始化)中，我们将看到另一种避免这种递归的技巧。

另一种打印技巧是将打印例程的参数直接传递给另一个这样的例程。`Printf` 的签名将其最后一个参数的类型声明为 `...interface{}`，以指定任意数量的参数（任意类型）可以出现在格式之后。

```go
func Printf(format string, v ...interface{}) (n int, err error) {
```

在函数 `Printf` 内部，`v` 的行为类似于类型为 `[]interface{}` 的变量，但如果将它传递给另一个可变参数函数，它的行为类似于一个常规的参数列表。以下是我们上面使用的函数 `log.Println` 的实现。它将参数直接传递给 `fmt.Sprintln` 进行实际的格式化。

```go
// Println prints to the standard logger in the manner of fmt.Println.
func Println(v ...interface{}) {
    std.Output(2, fmt.Sprintln(v...))  // Output 接受参数 (int, string)
}
```

我们在嵌套调用 `Sprintln` 时在 `v` 后面写了 `...`，告诉编译器将 `v` 视为参数列表；否则它会将 `v` 作为单个 slice 参数传递。

关于打印，我们要说的就这些，甚至比这还要多。更多信息请参见 `godoc` 中对 `fmt` 的文档。

顺便说一句，`...` 参数可以是一个特定的类型，例如 `...int`，用于选择最小值的 `min` 函数，该函数接受该类型的整数列表。

```go
func Min(a ...int) int {
    min := int(^uint(0) >> 1) // 最大的 int 值
    for _, i := range a {
        if i < min {
            min = i
        }
    }
    return min
}
```


## 初始化

尽管表面上看起来与 C 或 C++ 中的初始化没有太大不同，但 Go 中的初始化更强大。复杂结构可以在初始化期间构建，并且已初始化对象之间（甚至不同包之间）的顺序问题会被正确处理。

### 常量

Go 中的常量就是常量。它们在编译时创建，即使是定义为函数中的局部变量也是如此，并且只能是数字、字符（rune）、string 或 bool。由于编译时的限制，定义它们的表达式必须是常量表达式，可由编译器求值。例如，`1<<3` 是一个常量表达式，而 `math.Sin(math.Pi/4)` 不是，因为对 `math.Sin` 的函数调用需要在运行时进行。

在 Go 中，枚举常量使用 `iota` 枚举器创建。由于 `iota` 可以成为表达式的一部分，并且表达式可以隐式重复，因此很容易构建复杂的值集合。

```go
type ByteSize float64

const (
    _           = iota // 通过赋值给空白标识符忽略第一个值
    KB ByteSize = 1 << (10 * iota)
    MB
    GB
    TB
    PB
    EB
    ZB
    YB
)
```

将诸如 `String` 之类的方法附加到任何用户定义的类型上的能力，使得任意值都可以自动格式化以便打印。尽管你最常见于 struct，但这种技术对于标量类型（如 `ByteSize` 这样的浮点类型）也很有用。

```go
func (b ByteSize) String() string {
    switch {
    case b >= YB:
        return fmt.Sprintf("%.2fYB", b/YB)
    case b >= ZB:
        return fmt.Sprintf("%.2fZB", b/ZB)
    case b >= EB:
        return fmt.Sprintf("%.2fEB", b/EB)
    case b >= PB:
        return fmt.Sprintf("%.2fPB", b/PB)
    case b >= TB:
        return fmt.Sprintf("%.2fTB", b/TB)
    case b >= GB:
        return fmt.Sprintf("%.2fGB", b/GB)
    case b >= MB:
        return fmt.Sprintf("%.2fMB", b/MB)
    case b >= KB:
        return fmt.Sprintf("%.2fKB", b/KB)
    }
    return fmt.Sprintf("%.2fB", b)
}
```

表达式 `YB` 打印为 `1.00YB`，而 `ByteSize(1e13)` 打印为 `9.09TB`。

这里使用 `Sprintf` 来实现 `ByteSize` 的 `String` 方法是安全的（避免了无限递归），不是因为有类型转换，而是因为它用 `%f` 调用 `Sprintf`，这不是一个 string 格式：`Sprintf` 只有在需要 string 时才会调用 `String` 方法，而 `%f` 需要的是浮点值。

### 变量

变量可以像常量一样初始化，但初始化器可以是在运行时计算的通用表达式。

```go
var (
    home   = os.Getenv("HOME")
    user   = os.Getenv("USER")
    gopath = os.Getenv("GOPATH")
)
```

### init 函数

最后，每个源文件可以定义自己的无参 `init` 函数来设置所需的任何状态。（实际上每个文件可以有多个 `init` 函数。）最后意味着最后：`init` 在包中所有的变量声明都已评估其初始化器之后被调用，而这些初始化器仅在所有导入的包都已初始化之后才被评估。

除了无法通过声明来表达的初始化之外，`init` 函数的常见用途是在真正执行开始之前验证或修复程序状态的正确性。

```go
func init() {
    if user == "" {
        log.Fatal("$USER not set")
    }
    if home == "" {
        home = "/home/" + user
    }
    if gopath == "" {
        gopath = home + "/go"
    }
    // gopath 可能被命令行上的 --gopath 标志覆盖。
    flag.StringVar(&gopath, "gopath", gopath, "override default GOPATH")
}
```

## 方法

### 指针 vs 值

正如我们在 `ByteSize` 中看到的，方法可以为任何命名类型定义（除指针或 interface）；receiver 不必是 struct。

在上面关于 slice 的讨论中，我们编写了一个 `Append` 函数。我们可以将其定义为 slice 上的方法。为此，我们首先声明一个可以绑定方法的命名类型，然后让该方法的 receiver 成为该类型的值。

```go
type ByteSlice []byte

func (slice ByteSlice) Append(data []byte) []byte {
    // 函数体与上面定义的 Append 函数完全相同。
}
```

这仍然需要方法返回更新后的 slice。我们可以通过重新定义方法，使其接收一个指向 `ByteSlice` 的**指针**作为 receiver 来消除这种笨拙，这样方法就可以覆盖调用方的 slice。

```go
func (p *ByteSlice) Append(data []byte) {
    slice := *p
    // 函数体如上，但没有 return。
    *p = slice
}
```

事实上，我们可以做得更好。如果我们将函数修改为看起来像标准的 `Write` 方法，像这样：

```go
func (p *ByteSlice) Write(data []byte) (n int, err error) {
    slice := *p
    // 同样，函数体如上。
    *p = slice
    return len(data), nil
}
```

那么类型 `*ByteSlice` 就满足标准 interface `io.Writer`，这很方便。例如，我们可以打印到一个 `ByteSlice` 中。

```go
    var b ByteSlice
    fmt.Fprintf(&b, "This hour has %d days\n", 7)
```

我们传递了 `ByteSlice` 的地址，因为只有 `*ByteSlice` 满足 `io.Writer`。关于指针和值作为 receiver 的规则是：值方法可以在指针和值上调用，但指针方法只能在指针上调用。

这条规则的原因是，指针方法可以修改 receiver；在值上调用它们会导致方法接收到该值的一个副本，因此任何修改都会被丢弃。语言因此禁止这种错误。不过有一个方便的例外。当值是可寻址的（addressable）时，语言通过自动插入取地址运算符来处理在值上调用指针方法的常见情况。在我们的示例中，变量 `b` 是可寻址的，所以我们可以直接用 `b.Write` 调用它的 `Write` 方法。编译器会为我们将其重写为 `(&b).Write`。

顺便说一句，在字节 slice 上使用 `Write` 这一想法是 `bytes.Buffer` 实现的核心。


## 接口和其他类型

### 接口

Go 中的 interface 提供了一种指定对象行为的方式：如果某个东西能完成**这个**，那么它就可以在**这里**使用。我们已经看到了一些简单的示例；自定义打印器可以通过一个 `String` 方法实现，而 `Fprintf` 可以向任何具有 `Write` 方法的对象生成输出。只有一两个方法的 interface 在 Go 代码中很常见，通常被赋予一个从方法名派生的名称，例如 `io.Writer` 用于实现 `Write` 方法的对象。

一个类型可以实现多个 interface。例如，如果集合实现了 `sort.Interface`，即包含 `Len()`、`Less(i, j int) bool` 和 `Swap(i, j int)` 方法，那么它就可以被 `sort` 包中的例程排序，它也可以有一个自定义的格式化器。在这个人为构造的示例中，`Sequence` 同时满足了这两个接口。

```go
type Sequence []int

// sort.Interface 所需的方法。
func (s Sequence) Len() int {
    return len(s)
}
func (s Sequence) Less(i, j int) bool {
    return s[i] < s[j]
}
func (s Sequence) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

// 复制方法返回 Sequence 的一个副本。
func (s Sequence) Copy() Sequence {
    copy := make(Sequence, 0, len(s))
    return append(copy, s...)
}

// 用于打印的方法——在打印之前对元素排序。
func (s Sequence) String() string {
    s = s.Copy() // 复制一份，不要覆盖参数的内容。
    sort.Sort(s)
    str := "["
    for i, elem := range s {
        if i > 0 {
            str += " "
        }
        str += fmt.Sprint(elem)
    }
    return str + "]"
}
```

### 类型转换

`Sequence` 的 `String` 方法重新创建了 `Sprint` 已经为 slice 做的工作。（它也有 O(N²) 的复杂度，这很糟糕。）如果我们在调用 `Sprint` 之前将 `Sequence` 转换为普通的 `[]int`，我们可以共享努力（并且还能加速）。

```go
func (s Sequence) String() string {
    s = s.Copy()
    sort.Sort(s)
    return fmt.Sprint([]int(s))
}
```

这是一个通过类型转换来安全地调用 `Sprintf` 和其他函数的 `String` 方法的另一种技巧的例子。我们仅仅通过转换为基本类型来避免无限递归，该类型没有 `String` 方法。

在 Go 程序中，给定两个类型，如果底层类型相同，或者两者都是未命名的指针类型，指向相同底层类型的命名类型，则可以相互转换。这种转换会改变类型表示，但不会改变值的表示。我们也可以将值的类型限制为更窄的 interface 类型。例如，我们可以将 `Sequence` 的 `String` 方法改写为使用 `sort.IntSlice`，它与 `Sequence` 是相同的类型。

```go
func (s Sequence) String() string {
    s = s.Copy()
    sort.IntSlice(s).Sort()
    return fmt.Sprint([]int(s))
}
```

`Sequence` 的 `String` 方法可以将工作委托给更通用的 `Sprint` 函数，而不会引入递归，这是一个很好的临时转换例子。

### 接口转换与类型断言

Type switch 是一种转换形式：它们接受一个 interface，并且对于每个 case 中的匹配项，在某种意义上将其转换为该 case 的类型。以下简化版本的代码展示了 `fmt.Printf` 如何通过 type switch 将值转换为 string。（如果该值已经是 string，我们需要它持有的实际 interface 值，而如果是具有 `String` 方法的类型，我们需要调用该方法的结果。）

```go
type Stringer interface {
    String() string
}

var value interface{} // 由调用方提供。
switch str := value.(type) {
case string:
    return str
case Stringer:
    return str.String()
}
```

第一个 case 找到一个具体值；第二个则将 interface 转换为另一个 interface。以这种方式混合类型完全没问题。

但如果某个具体类型满足我们想要的 interface 怎么办？如果我们只关心一种类型，只有一个 case 的 type switch 是可行的，不过类型断言（type assertion）也可以做到。类型断言接受一个 interface 值并从中提取指定显式类型的值。语法借鉴了 type switch 的开头部分，但带有一个显式类型而不是 `type` 关键字：

```go
value.(typeName)
```

结果是一个静态类型为 `typeName` 的新值。该类型必须要么是接口持有的具体类型，要么是值可以转换为的第二个 interface 类型。要提取我们知道在该值中的 string，我们可以写：

```go
str := value.(string)
```

但如果该值不包含 string，程序会以运行时错误崩溃。为了避免这种情况，使用"comma, ok"惯用法来安全地测试该值是否是 string：

```go
str, ok := value.(string)
if ok {
    fmt.Printf("string value is: %q\n", str)
} else {
    fmt.Printf("value is not a string\n")
}
```

如果类型断言失败，`str` 将继续存在并具有类型 string，但它将具有零值，即空字符串。

作为这种能力的说明，这里有一个 if-else 语句，它与本节开头部分中的 type switch 是等价的。

```go
if str, ok := value.(string); ok {
    return str
} else if str, ok := value.(Stringer); ok {
    return str.String()
}
```

### 通用性

如果一个类型的存在仅仅是为了实现一个 interface，并且除了该 interface 之外没有任何导出方法，那么该类型本身不需要导出。只导出 interface 可以清楚地表明该值除了 interface 中描述的功能外没有其他有趣的行为。它还避免了在每个公共方法的实例上都需要文档。

在这种情况下，构造函数应该返回一个 interface 值而不是实现的类型。例如，在 hash 库中，`crc32.NewIEEE` 和 `adler32.New` 都返回 interface 类型 `hash.Hash32`。在 Go 程序中，用 `crc32` 替换 `adler32` 只需要改变构造函数调用；其余代码不受算法变化的影响。

类似的方法允许将各种 `crypto` 包中的流密码算法与它们链接起来的块密码分离。`crypto/cipher` 包中的 `Block` interface 指定了块密码的行为，即提供单个数据块的加密。然后，通过类比 `bufio` 包，实现此 interface 的密码包可以用来构造由 `Stream` interface 表示的流密码，而无需知道块加密的细节。

`crypto/cipher` 接口如下所示：

```go
type Block interface {
    BlockSize() int
    Encrypt(dst, src []byte)
    Decrypt(dst, src []byte)
}

type Stream interface {
    XORKeyStream(dst, src []byte)
}
```

以下是计数器模式（CTR）流的定义，它将块密码转换为流密码；请注意，块密码的细节被抽象出去了：

```go
// NewCTR 返回一个 Stream，它使用给定的 Block 在计数器模式下进行加密/解密。
// iv 的长度必须与 Block 的块大小相同。
func NewCTR(block Block, iv []byte) Stream
```

`NewCTR` 不仅适用于一种特定的加密算法和数据源，而是适用于 `Block` interface 的任何实现和任何 `Stream`。因为它们返回 interface 值，用另一种加密模式替换 CTR 加密是一个局部的更改。构造函数调用必须被编辑，但由于周围的代码必须将结果仅视为一个 `Stream`，它不会注意到差异。

### 接口和方法

由于几乎任何东西都可以附加方法，所以几乎任何东西都可以满足一个 interface。以下是一个说明性示例，在 `http` 包中定义了一个 `Handler` interface。任何实现了 `Handler` 的对象都可以提供 HTTP 请求。

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

`ResponseWriter` 本身是一个 interface，提供访问方法以向客户端返回响应。这些方法包括标准的 `Write` 方法，因此 `http.ResponseWriter` 可以在任何可以使用 `io.Writer` 的地方使用。`Request` 是一个包含来自客户端的已解析请求的 struct。

为了简洁起见，我们忽略 POST，假设 HTTP 请求始终是 GET；这种简化不影响处理程序的设置方式。以下是一个简单但完整的用于统计页面访问次数的处理程序实现。

```go
// 简单的计数器服务器。
type Counter struct {
    n int
}

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ctr.n++
    fmt.Fprintf(w, "counter = %d\n", ctr.n)
}
```

（继续我们的主题，注意 `Fprintf` 如何打印到 `http.ResponseWriter`。）在真实服务器中，对 `ctr.n` 的访问需要防止并发访问。参见 `sync` 和 `atomic` 包以获取建议。

作为参考，以下是如何将这样的服务器附加到 URL 树上的节点。

```go
import "net/http"
...
ctr := new(Counter)
http.Handle("/counter", ctr)
```

但为什么要将 `Counter` 设为 struct？只需要一个整数即可。（接收者需要是指针，这样增量才能对调用方可见。）

```go
// 更简单的计数器服务器。
type Counter int

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    *ctr++
    fmt.Fprintf(w, "counter = %d\n", *ctr)
}
```

如果你的程序有一些内部状态需要通知一个页面已被访问怎么办？将 channel 绑定到网页上。

```go
// 每次访问此页面时发送一个通知的 channel。
// （可能希望 channel 是有缓冲的。）
type Chan chan *http.Request

func (ch Chan) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ch <- req
    fmt.Fprint(w, "notification sent")
}
```

最后，假设我们想在 `/args` 上展示调用服务器二进制文件时使用的参数。编写一个打印参数的函数很容易。

```go
func ArgServer() {
    fmt.Println(os.Args)
}
```

将其转换为 HTTP 服务器呢？我们可以将 `ArgServer` 定义为某种类型并忽略其值的方法，但是有一种更简洁的方式。由于我们可以为除指针和 interface 之外的任何类型定义方法，我们可以为函数写一个方法。`http` 包包含以下代码：

```go
// HandlerFunc 类型是一个适配器，允许将普通函数用作 HTTP 处理程序。
// 如果 f 是带有适当签名的函数，则 HandlerFunc(f) 是一个调用 f 的 Handler 对象。
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP 调用 f(w, req)。
func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
    f(w, req)
}
```

`HandlerFunc` 是一个带有方法 `ServeHTTP` 的类型，因此该类型的值可以提供 HTTP 请求。看一下方法的实现：接收者是函数 `f`，而方法调用 `f`。这看起来可能奇怪，但实际上和接收者是 channel 且在方法中向 channel 发送消息没有太大区别。

要使 `ArgServer` 成为一个 HTTP 服务器，我们首先将其修改为具有正确的签名。

```go
// 参数服务器。
func ArgServer(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(w, os.Args)
}
```

`ArgServer` 现在具有与 `HandlerFunc` 相同的签名，因此它可以被转换为该类型以访问其方法，就像我们将 `Sequence` 转换为 `IntSlice` 以访问 `IntSlice.Sort` 一样。设置它的代码很简洁：

```go
http.Handle("/args", http.HandlerFunc(ArgServer))
```

当有人访问页面 `/args` 时，安装在该页面上的处理程序具有值 `ArgServer` 和类型 `HandlerFunc`。HTTP 服务器将以 `ArgServer` 作为接收者调用该类型的 `ServeHTTP` 方法，该方法进而调用 `ArgServer`（通过 `HandlerFunc.ServeHTTP` 中的 `f(w, req)` 调用）。然后参数将被显示。

在本节中，我们用一个 struct、一个整数、一个 channel 和一个函数创建了一个 HTTP 服务器，这些都是因为 interface 就是一堆方法，可以为（几乎）任何类型定义。


## 空白标识符

我们已经在 `for range` 循环和 map 的上下文中多次提到了空白标识符。空白标识符可以被赋值或声明为任何类型的任何值，其值会被无害地丢弃。它有点像写入 Unix 的 `/dev/null` 文件：它表示一个只写值，用作需要变量但实际值无关紧要的占位符。它的用途超出了我们已经看到的范围。

### 空白标识符在多赋值中的使用

在 `for range` 循环中使用空白标识符是一般情况的特殊情况：多赋值。

如果赋值在左侧需要多个值，但程序不会使用其中某个值，那么在赋值左侧使用空白标识符可以避免创建虚拟变量，并明确表示该值将被丢弃。例如，当调用一个返回值和错误但只有错误重要的函数时，使用空白标识符来丢弃无关的值。

```go
if _, err := os.Stat(path); os.IsNotExist(err) {
    fmt.Printf("%s does not exist\n", path)
}
```

有时你会看到代码丢弃错误值以忽略错误；这是一种可怕的做法。始终检查错误返回；它们是有原因的。

```go
// 不好！如果位置路径不存在，这段代码会崩溃。
fi, _ := os.Stat(path)
if fi.IsDir() {
    fmt.Printf("%s is a directory\n", path)
}
```

### 未使用的导入和变量

导入一个包或声明一个变量而不使用它是错误的。未使用的导入会膨胀程序并减慢编译速度，而已初始化但未使用的变量至少是一种浪费的计算，并且可能表明存在更大的 bug。然而，当程序正在积极开发时，未使用的导入和变量经常会出现，而仅仅为了让编译器满意就删除它们可能会很烦人，因为之后可能又需要它们。空白标识符提供了一种变通方法。

以下是一个半编写的程序，包含两个未使用的导入（`fmt` 和 `io`）和一个未使用的变量（`fd`），所以它无法编译，但如果能够看到目前的代码是否正确就更好了。

```go
package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

func main() {
    fd, err := os.Open("test.go")
    if err != nil {
        log.Fatal(err)
    }
    // TODO: use fd.
}
```

要消除关于未使用导入的抱怨，可以使用空白标识符引用导入包中的符号。类似地，将未使用的变量 `fd` 赋值给空白标识符将消除未使用的变量错误。这个版本的程序确实能够编译。

```go
package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

var _ = fmt.Printf // 用于调试；用完删除。
var _ io.Reader    // 用于调试；用完删除。

func main() {
    fd, err := os.Open("test.go")
    if err != nil {
        log.Fatal(err)
    }
    // TODO: use fd.
    _ = fd
}
```

按照惯例，消除导入错误的全局声明应该紧随导入之后，并加以注释，这样既易于找到，也提醒之后进行清理。

### 为了副作用而导入

在前面的例子中未使用的导入（如 `fmt` 或 `io`）最终应该被使用或删除：空白赋值将代码标识为正在进行的工作。但有时导入一个包仅仅是为了其副作用，而不进行任何显式使用。例如，在它的 `init` 函数中，`net/http/pprof` 包注册了提供调试信息的 HTTP 处理程序。它有一个导入的 API，但大多数客户端只需要处理程序的注册，并通过网页访问数据。要仅为副作用导入该包，可以将包重命名为空白标识符：

```go
import _ "net/http/pprof"
```

这种形式的导入明确表示该包正在为其副作用而被导入，因为该包没有其他可能的用途：在这个文件中，它没有名称。（如果它有名称，而我们没有使用该名称，编译器将拒绝该程序。）

### 接口检查

正如我们在上面接口讨论中看到的那样，一个类型不需要显式声明它实现了一个 interface。相反，类型只需实现 interface 的方法就能实现该 interface。在实践中，大多数 interface 转换是静态的，因此在编译时进行检查。例如，当 `*os.File` 没有实现 `io.Reader` interface 时，将一个 `*os.File` 传递给一个期望 `io.Reader` 的函数将不会通过编译。

然而，有些接口检查确实发生在运行时。一个实例在 `encoding/json` 包中，它定义了一个 `Marshaler` interface。当 JSON 编码器接收到一个实现了该 interface 的值时，编码器调用该值的 marshal 方法将其转换为 JSON，而不是进行标准转换。编码器在运行时使用类似下面的类型断言来检查这个属性：

```go
m, ok := val.(json.Marshaler)
```

如果只需要判断某个类型是否实现了某个 interface 而不实际使用该 interface 本身（也许作为错误检查的一部分），可以使用空白标识符来忽略类型断言的值：

```go
if _, ok := val.(json.Marshaler); ok {
    fmt.Printf("value %v of type %T implements json.Marshaler\n", val, val)
}
```

这种情况出现的其中一个地方是当需要保证在实现该类型的包内类型确实满足该 interface 时。如果一个类型—例如 `json.RawMessage`—需要自定义的 JSON 表示，它应该实现 `json.Marshaler`，但没有任何会导致编译器自动验证这一点的静态转换。如果类型无意中未能满足该 interface，JSON 编码器仍然可以工作，但不会使用自定义实现。为确保实现正确，可以在包中使用一个利用空白标识符的全局声明：

```go
var _ json.Marshaler = (*RawMessage)(nil)
```

在此声明中，涉及将 `*RawMessage` 转换为 `Marshaler` 的赋值要求 `*RawMessage` 实现 `Marshaler`，并且该属性将在编译时进行检查。如果 `json.Marshaler` interface 发生变化，这个包将不再编译，我们会注意到它需要更新。

在该构造中出现的空白标识符表明该声明仅为了类型检查而存在，而不是为了创建一个变量。然而，不要对每个满足 interface 的类型都这样做。按照惯例，这样的声明只在代码中不存在其他静态转换时才使用，这种情况很少见。

## 嵌入

Go 不提供典型的、类型驱动的子类化概念，但它确实提供了通过将类型嵌入 struct 或 interface 来"借用"实现片段的能力。

Interface 嵌入非常简单。我们之前提到过 `io.Reader` 和 `io.Writer` interface；以下是它们的定义。

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

`io` 包还导出了几个其他 interface，它们指定了可以实现多个此类方法的对象。例如，有 `io.ReadWriter`，一个包含 `Read` 和 `Write` 的 interface。我们可以通过显式列出这两个方法来指定 `io.ReadWriter`，但更容易且更能唤起记忆的方式是嵌入这两个 interface 来形成新的 interface，像这样：

```go
// ReadWriter 是组合 Reader 和 Writer 接口的接口。
type ReadWriter interface {
    Reader
    Writer
}
```

这正如它看起来的那样：`ReadWriter` 可以做 `Reader` 能做的**和** `Writer` 能做的；它是嵌入接口（必须是不相交的方法集合）的并集。只有 interface 可以嵌入到 interface 中。

同样的基本思想也适用于 struct，但有更深远的影响。`bufio` 包有两个 struct 类型，`bufio.Reader` 和 `bufio.Writer`，每个都实现了 `io` 包中的类似 interface。`bufio` 还实现了一个缓冲的 reader/writer，它通过使用嵌入将 reader 和 writer 组合成一个 struct 来实现：它在 struct 中列出了类型，但没有给它们字段名。

```go
// ReadWriter 存储指向 Reader 和 Writer 的指针。
// 它实现了 io.ReadWriter。
type ReadWriter struct {
    *Reader  // *bufio.Reader
    *Writer  // *bufio.Writer
}
```

嵌入的元素是指向 struct 的指针，当然在使用之前它们必须初始化为指向有效 struct 的指针。`ReadWriter` struct 可以写成：

```go
type ReadWriter struct {
    reader *Reader
    writer *Writer
}
```

但为了提升嵌入类型的方法并满足 `io` interface，我们还需要提供转发方法，像这样：

```go
func (rw *ReadWriter) Read(p []byte) (n int, err error) {
    return rw.reader.Read(p)
}
```

通过直接嵌入 struct，我们避免了这种记账。嵌入类型的方法会免费带来，这意味着 `bufio.ReadWriter` 不仅具有 `bufio.Reader` 和 `bufio.Writer` 的方法，而且还满足所有三个 interface：`io.Reader`、`io.Writer` 和 `io.ReadWriter`。

嵌入与子类化有一个重要的不同之处。当我们嵌入一个类型时，该类型的方法成为外部类型的方法，但当它们被调用时，方法的接收者是内部类型，而不是外部类型。在我们的例子中，当 `bufio.ReadWriter` 的 `Read` 方法被调用时，它与上面写出的转发方法具有完全相同的效果；接收者是 `ReadWriter` 的 `reader` 字段，而不是 `ReadWriter` 本身。

嵌入也可以是一种简单的便利。以下示例展示了一个嵌入字段与一个常规的命名字段并列。

```go
type Job struct {
    Command string
    *log.Logger
}
```

`Job` 类型现在具有 `*log.Logger` 的 `Print`、`Printf`、`Println` 和其他方法。当然，我们可以给 `Logger` 一个字段名，但没有必要这样做。现在，一旦初始化，我们就可以记录到 `Job`：

```go
job.Println("starting now...")
```

`Logger` 是 `Job` struct 的一个常规字段，因此我们可以在 `Job` 的构造函数中以通常的方式初始化它，像这样：

```go
func NewJob(command string, logger *log.Logger) *Job {
    return &Job{command, logger}
}
```

或者用复合字面量：

```go
job := &Job{command, log.New(os.Stderr, "Job: ", log.Ldate)}
```

如果我们需要直接引用一个嵌入字段，该字段的类型名（忽略包限定符）就作为字段名，就像在 `ReadWriter` struct 的 `Read` 方法中那样。在这里，如果我们需要访问 `Job` 类型的变量 `job` 的 `*log.Logger`，我们可以写 `job.Logger`，这对细化 `Logger` 的方法很有用。

```go
func (job *Job) Printf(format string, args ...interface{}) {
    job.Logger.Printf("%q: %s", job.Command, fmt.Sprintf(format, args...))
}
```

嵌入类型会引入命名冲突的问题，但解决它们的规则很简单。首先，字段或方法 `X` 会隐藏该类型更深层嵌套部分中的任何其他项 `X`。如果 `log.Logger` 包含一个名为 `Command` 的字段或方法，那么 `Job` 的 `Command` 字段将覆盖它。

其次，如果同一个名称出现在同一个嵌套层级上，通常是一个错误；如果 `Job` struct 包含另一个名为 `Logger` 的字段或方法，那么嵌入 `log.Logger` 就是错误的。然而，如果重复的名称从未在类型定义外的程序中使用过，那就没问题。这种限定提供了一些保护，防止从外部嵌入的类型发生更改；如果添加的字段与另一个子类型中的另一个字段冲突（如果两个字段都从未被使用过），这没有问题。


## 并发

### 通过通信共享内存

并发编程是一个很大的话题，这里只介绍一些 Go 特有的亮点。

在许多环境中，并发编程由于需要实现对共享变量的正确访问所需的微妙之处而变得困难。Go 鼓励一种不同的方法：通过 channel 传递共享值，这些值永远不会被单独的执行线程主动共享。在任何给定时间只有一个 goroutine 可以访问该值。设计上，数据竞争不会发生。为了鼓励这种思维方式，我们将其简化为一句口号：

> 不要通过共享内存来通信；而是通过通信来共享内存。

这种方法可能会走得太远。例如，引用计数最好通过在整数变量周围放置一个 mutex 来实现。但作为一个高阶方法，使用 channel 来控制访问可以更容易编写清晰、正确的程序。

思考这个模型的一种方式是考虑一个典型的单线程程序，它运行在单个 CPU 上。不需要同步原语。现在运行另一个这样的实例；它也不需要同步。现在让它们通信；如果通信是同步器，仍然不需要其他同步。例如，Unix 管道就完美地符合这个模型。虽然 Go 的并发方法源于 Hoare 的通信顺序进程（CSP），但它也可以被看作是 Unix 管道类型安全泛化。

### Goroutine

它们被称为 goroutine，因为现有的术语——线程、协程、进程等——传达了不准确的含义。goroutine 有一个简单的模型：它是一个在同一地址空间中与其他 goroutine 并发执行的函数。它是轻量级的，只比栈空间的分配多一点成本。而且栈一开始很小，所以它们很廉价，并且会根据需要分配（和释放）堆存储来增长。

Goroutine 被多路复用到多个 OS 线程上，所以如果一个 goroutine 应该阻塞，例如在等待 I/O 时，其他 goroutine 会继续运行。它们的设计隐藏了线程创建和管理的许多复杂性。

在一个函数或方法调用前加上 `go` 关键字，就可以在一个新的 goroutine 中运行该调用。当调用完成时，goroutine 会静默退出。（效果类似于 Unix shell 中用于在后台运行命令的 `&` 符号。）

```go
go list.Sort()  // 并行运行 list.Sort；不等待它完成。
```

函数字面量在 goroutine 调用中很方便。

```go
func Announce(message string, delay time.Duration) {
    go func() {
        time.Sleep(delay)
        fmt.Println(message)
    }()  // 注意括号——必须调用该函数。
}
```

在 Go 中，函数字面量是闭包：实现确保被函数引用的变量在它们活跃期间会保持存在。

这些例子不太实用，因为函数没有办法发出完成的信号。为此，我们需要 channel。

### Channel

与 map 一样，channel 使用 `make` 分配，其结果值充当下层数据结构的引用。如果提供了一个可选的整数参数，它会设置 channel 的缓冲区大小。对于无缓冲或同步 channel，默认值为零。

```go
ci := make(chan int)            // 无缓冲整数 channel
cj := make(chan int, 0)         // 无缓冲整数 channel
cs := make(chan *os.File, 100)  // 缓冲文件指针 channel
```

无缓冲 channel 将通信——值的交换——与同步相结合——保证两个计算（goroutine）处于已知的状态。

有很多不错的 channel 惯用法。这是我们的入门示例。在前一节中，我们在后台启动了一个排序。channel 可以允许启动的 goroutine 等待排序完成。

```go
c := make(chan int)  // 分配一个 channel。
// 在 goroutine 中启动排序；当完成时，在 channel 上发送信号。
go func() {
    list.Sort()
    c <- 1  // 发送一个信号；值无关紧要。
}()
doSomethingForAWhile()
<-c   // 等待排序完成；丢弃发送的值。
```

接收者始终阻塞，直到有数据可接收。如果 channel 是无缓冲的，发送者会阻塞，直到接收者接收到该值。如果 channel 有缓冲区，发送者只在值被复制到缓冲区之前阻塞；如果缓冲区已满，这意味着等到某个接收者检索到一个值。

缓冲的 channel 可以像信号量一样使用，例如用于限制吞吐量。在这个例子中，传入的请求被传递给 `handle`，它向 channel 发送一个值，处理请求，然后从 channel 接收一个值，为下一个消费者准备好"信号量"。

```go
var sem = make(chan int, MaxOutstanding)

func handle(r *Request) {
    sem <- 1    // 等待活跃队列排空。
    process(r)  // 可能需要很长时间。
    <-sem       // 完成；启用下一个要运行的请求。
}

func Serve(queue chan *Request) {
    for {
        req := <-queue
        go handle(req)  // 不等待 handle 完成。
    }
}
```

一旦 `MaxOutstanding` 个处理程序正在执行 `process`，再有尝试发送到已满 channel buffer 的操作都将阻塞，直到某个现有的处理程序完成并从缓冲区接收为止。

然而，这种设计有一个问题：`Serve` 为每个传入的请求创建一个新的 goroutine，即使任何时候最多只有 `MaxOutstanding` 个正在运行。因此，如果请求来得太快，程序可能会消耗无限的资源。我们可以通过改变 `Serve` 来限制 goroutine 的创建来解决这个缺陷。这是一个明显的解决方案，但要注意它有一个 bug，我们稍后会修复：

```go
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func() {
            process(req) // 有 bug；见下文。
            <-sem
        }()
    }
}
```

这个 bug 是：在 Go 的 `for` 循环中，循环变量在每次迭代中被重用，因此 `req` 变量在所有 goroutine 之间共享。这不是我们想要的。我们需要确保 `req` 对每个 goroutine 都是唯一的。这里有一种方法，即将 `req` 的值作为参数传递给 goroutine 中的闭包：

```go
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func(req *Request) {
            process(req)
            <-sem
        }(req)
    }
}
```

将此版本与之前的版本进行比较，以查看声明和运行闭包的差异。另一种解决方案是创建一个同名新变量，如下例所示：

```go
func Serve(queue chan *Request) {
    for req := range queue {
        req := req // 为 goroutine 创建 req 的新实例。
        sem <- 1
        go func() {
            process(req)
            <-sem
        }()
    }
}
```

写起来可能有点奇怪：

```go
req := req
```

但在 Go 中这样做是合法且地道的。你会得到具有相同名称的变量的新版本，在所有 goroutine 中故意隐藏循环变量。

回到服务器的一般问题，另一种很好地管理资源的方法是启动固定数量的 `handle` goroutine，所有 goroutine 都从请求 channel 读取。goroutine 的数量限制了并行调用 `process` 的数量。这个 `Serve` 函数也接受一个 channel，它被告知退出；在启动 goroutine 后，它会阻塞接收来自该 channel 的信号。

```go
func handle(queue chan *Request) {
    for r := range queue {
        process(r)
    }
}

func Serve(clientRequests chan *Request, quit chan bool) {
    // 启动处理程序
    for i := 0; i < MaxOutstanding; i++ {
        go handle(clientRequests)
    }
    <-quit  // 等待被告知退出。
}
```

### Channel 的 channel

Go 最重要的特性之一是 channel 是一个一等值，可以像其他值一样分配和传递。这种特性的常见用法是实现安全的、并行的多路分解。

在上一节的示例中，`handle` 是请求的理想处理程序，但我们没有定义它处理的类型。如果该类型包含一个用于回复的 channel，每个客户端可以提供自己的回复路径。以下是一个 `Request` 类型的示意图定义。

```go
type Request struct {
    args        []int
    f           func([]int) int
    resultChan  chan int
}
```

客户端提供一个函数及其参数，以及请求对象中的一个 channel 来接收答案。

```go
func sum(a []int) (s int) {
    for _, v := range a {
        s += v
    }
    return
}

request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
// 发送请求
clientRequests <- request
// 等待响应。
fmt.Printf("answer: %d\n", <-request.resultChan)
```

在服务器端，处理程序函数是唯一会改变的东西。

```go
func handle(queue chan *Request) {
    for req := range queue {
        req.resultChan <- req.f(req.args)
    }
}
```

显然还有更多可以做的，使其更实用，但这段代码是限速、并行、非阻塞 RPC 系统的框架，并且没有 mutex。

### 并行化

这些思想的另一个应用是跨多个 CPU 核心并行化计算。如果计算可以分解为独立执行的片段，它可以被并行化，通过 channel 在每个片段完成时发出信号。

假设我们对一个向量项进行一项昂贵的操作，并且每个项的操作值是独立的，如以下这个理想化的例子所示。

```go
type Vector []float64

// 将操作应用于 v[i], v[i+1] ... 直到 v[n-1]。
func (v Vector) DoSome(i, n int, u Vector, c chan int) {
    for ; i < n; i++ {
        v[i] += u.Op(v[i])
    }
    c <- 1    // 向 channel 发送信号表示此片段已完成
}
```

我们在循环中独立地启动各片段，每个 CPU 一个。它们可以以任何顺序完成，但无关紧要；我们只是在启动所有 goroutine 后，通过排空 channel 来计数完成信号。

```go
const numCPU = 4 // CPU 核心数

func (v Vector) DoAll(u Vector) {
    c := make(chan int, numCPU)  // 缓冲是可选的，但是合理的。
    for i := 0; i < numCPU; i++ {
        go v.DoSome(i*len(v)/numCPU, (i+1)*len(v)/numCPU, u, c)
    }
    // 排空 channel。
    for i := 0; i < numCPU; i++ {
        <-c    // 等待一个任务完成
    }
    // 全部完成。
}
```

我们可以在运行时询问 CPU 数量的合适值，而不是创建一个常量值。函数 `runtime.NumCPU` 返回机器中的硬件 CPU 核心数，所以我们可以写：

```go
var numCPU = runtime.NumCPU()
```

还有一个函数 `runtime.GOMAXPROCS`，它报告（或设置）Go 程序可以同时运行的用户指定核心数。它默认为 `runtime.NumCPU` 的值，但可以通过设置相同名称的 shell 环境变量或使用正数调用该函数来覆盖。用零调用它仅仅查询该值。因此，如果我们想满足用户的资源请求，我们应该写：

```go
var numCPU = runtime.GOMAXPROCS(0)
```

请务必不要混淆并发和并行的概念：并发将程序构建为独立执行的组件，而并行在多个 CPU 上并行执行计算以提高效率。虽然 Go 的并发特性可以使某些问题易于结构化为并行计算，但 Go 是并发语言，不是并行语言，并且并非所有并行化问题都适合 Go 的模型。关于二者的区别，请参见[此博客文章](https://go.dev/blog/concurrency-is-not-parallelism)中的讨论。

### 一个会泄漏的缓冲区

并发编程的工具甚至可以使非并发的思想更容易表达。以下是一个从 RPC 包中抽象出来的例子。客户端 goroutine 循环从某个源接收数据，可能是网络。为了避免频繁分配和释放缓冲区，它保留了一个空闲列表，并使用一个缓冲的 channel 来表示它。如果 channel 为空，则分配一个新的缓冲区。一旦消息缓冲区就绪，它通过 `serverChan` 发送到服务器。

```go
var freeList = make(chan *Buffer, 100)
var serverChan = make(chan *Buffer)

func client() {
    for {
        var b *Buffer
        // 如果有可用的缓冲区就获取一个；如果没有就分配一个新的。
        select {
        case b = <-freeList:
            // 获取了一个；不做更多事。
        default:
            // 没有空闲的，所以分配一个新的。
            b = new(Buffer)
        }
        load(b)              // 从网络读取下一条消息。
        serverChan <- b      // 发送到服务器。
    }
}
```

服务器循环接收来自客户端的每条消息，处理它，并将缓冲区返回到空闲列表。

```go
func server() {
    for {
        b := <-serverChan    // 等待工作。
        process(b)
        // 如果有空间就重用缓冲区。
        select {
        case freeList <- b:
            // 在空闲列表的缓冲区；不做更多事。
        default:
            // 空闲列表已满，继续。
        }
    }
}
```

客户端尝试从 `freeList` 获取缓冲区；如果没有可用，则分配一个新的。服务器发送到 `freeList` 的 `put` back `b` 操作，除非列表已满，在这种情况下缓冲区被丢弃在地上，由垃圾收集器回收。（`select` 语句中的 `default` 子句在没有其他 case 准备好时执行，这意味着 `select` 永远不会阻塞。）这个实现只用几行代码就构建了一个有漏桶的空闲列表，并依赖于缓冲的 channel 和垃圾收集器来进行记账。

## 错误

库例程通常必须向调用者返回某种错误指示。如前所述，Go 的多值返回使得在返回正常返回值的同时返回详细的错误描述变得容易。通过这个特性提供详细的错误信息是好的风格。例如，正如我们将看到的，`os.Open` 不仅在失败时返回一个 `nil` 指针，它还返回一个描述出错原因的错误值。

按照惯例，错误的类型是 `error`，这是一个简单的内置 interface。

```go
type error interface {
    Error() string
}
```

库编写者可以自由地用一个更丰富的模型来实现这个 interface，这样不仅可以看到错误，还可以提供一些上下文。如前所述，除了通常的 `*os.File` 返回值外，`os.Open` 还返回一个错误值。如果文件成功打开，错误将为 `nil`，但当出现问题时，它会持有一个 `os.PathError`：

```go
// PathError 记录一个错误以及导致它的操作和文件路径。
type PathError struct {
    Op   string  // "open"、"unlink" 等。
    Path string  // 相关的文件。
    Err  error   // 由系统调用返回。
}

func (e *PathError) Error() string {
    return e.Op + " " + e.Path + ": " + e.Err.Error()
}
```

`PathError` 的 `Error` 生成一个像这样的字符串：

```
open /etc/passwx: no such file or directory
```

这样的错误，包含了有问题的文件名、操作以及它触发的操作系统错误，在打印时很有用，即使它离引起错误的调用很远；它比单纯的"no such file or directory"信息量更大。

在可行的情况下，错误字符串应该标识它们的来源，例如通过使用前缀来命名产生错误的操作或包。例如，在 `image` 包中，由于未知格式导致的解码错误的字符串表示是"image: unknown format"。

关心确切错误详情的调用者可以使用 type switch 或类型断言来查找特定错误并提取详细信息。对于 `PathError`，这可能包括检查内部的 `Err` 字段以查找可恢复的故障。

```go
for try := 0; try < 2; try++ {
    file, err = os.Create(filename)
    if err == nil {
        return
    }
    if e, ok := err.(*os.PathError); ok && e.Err == syscall.ENOSPC {
        deleteTempFiles()  // 恢复一些空间。
        continue
    }
    return
}
```

这里的第二个 `if` 语句是另一种类型断言。如果失败，`ok` 将为 false，`e` 将为 `nil`。如果成功，`ok` 将为 true，这意味着错误的类型是 `*os.PathError`，而 `e` 也是如此，我们可以检查它了解更多关于该错误的信息。

### Panic

向调用者报告错误的惯用方式是返回一个 `error` 作为额外的返回值。规范的 `Read` 方法是一个众所周知的实例；它返回一个字节计数和一个 `error`。但如果错误不可恢复怎么办？有时程序根本无法继续。

为此，有一个内置函数 `panic`，实际上它创建一个运行时错误，将停止程序（但请参见下一节）。该函数接受一个任意类型的单一参数——通常是一个 string——在程序终止时打印。这也是一种表明某些事情发生了不可能的事情的方式，例如退出无限循环。

```go
// 用牛顿法计算立方根的玩具实现。
func CubeRoot(x float64) float64 {
    z := x/3   // 任意初始值
    for i := 0; i < 1e6; i++ {
        prevZ := z
        z -= (z*z*z-x) / (3*z*z)
        if veryClose(z, prevZ) {
            return z
        }
    }
    // 经过一百万次迭代后没有收敛；出了点问题。
    panic(fmt.Sprintf("CubeRoot(%g) did not converge", x))
}
```

这只是个例子，但真实的库函数应该避免 panic。如果问题可以被掩盖或绕过，让程序继续运行总是比将其完全终止要好。反例可能在初始化期间：如果库确实无法设置自身，可以合理地 panic。

```go
var user = os.Getenv("USER")

func init() {
    if user == "" {
        panic("no value for $USER")
    }
}
```

### Recover

当调用 `panic` 时，包括对于运行时错误的隐式调用，例如对 slice 的越界索引或类型断言失败，它会立即停止当前函数的执行，并开始展开 goroutine 的调用栈，沿途运行任何延迟的函数。如果展开到达了 goroutine 调用栈的顶部，程序将终止。然而，可以使用内置函数 `recover` 重新获得 goroutine 的控制权并恢复正常执行。

对 `recover` 的调用会停止展开并返回传递给 `panic` 的参数。因为在展开时运行的代码只有延迟函数内部的代码，所以 `recover` 只在延迟函数内部有用。

`recover` 的一个应用是在服务器内部关闭一个失败的 goroutine，而不杀死其他正在执行的 goroutine。

```go
func server(workChan <-chan *Work) {
    for work := range workChan {
        go safelyDo(work)
    }
}

func safelyDo(work *Work) {
    defer func() {
        if err := recover(); err != nil {
            log.Println("work failed:", err)
        }
    }()
    do(work)
}
```

在这个例子中，如果 `do(work)` 发生 panic，结果将被记录，goroutine 将干净地退出，而不会干扰其他 goroutine。在延迟闭包中不需要做任何其他事情；调用 `recover` 就能完全处理该情况。

因为 `recover` 总是返回 `nil`，除非直接从延迟函数调用，所以延迟的代码可以调用本身使用 `panic` 和 `recover` 的库例程而不会失败。例如，`safelyDo` 中的延迟函数可能在调用 `recover` 之前调用日志记录函数，而该日志记录代码将不受 panic 状态的影响地运行。

有了我们恢复的模式，`do` 函数（以及它调用的任何东西）可以通过调用 `panic` 干净地摆脱任何糟糕的情况。我们可以使用这个思想来简化复杂软件中的错误处理。让我们看看 `regexp` 包的一个理想化版本，它通过使用局部错误类型调用 `panic` 来报告解析错误。以下是 `Error` 的定义、`error` 方法和 `Compile` 函数。

```go
// Error 是解析错误的类型；它满足 error interface。
type Error string
func (e Error) Error() string {
    return string(e)
}

// error 是 *Regexp 的一个方法，它通过用 Error 触发 panic 来报告解析错误。
func (regexp *Regexp) error(err string) {
    panic(Error(err))
}

// Compile 返回正则表达式的解析表示。
func Compile(str string) (regexp *Regexp, err error) {
    regexp = new(Regexp)
    // 如果在解析时出现错误，doParse 会 panic。
    defer func() {
        if e := recover(); e != nil {
            regexp = nil    // 清除返回值。
            err = e.(Error) // 如果它不是解析错误，则重新 panic。
        }
    }()
    return regexp.doParse(str), nil
}
```

如果 `doParse` 发生 panic，恢复块将把返回值设置为 `nil`——延迟函数可以修改命名的返回值。然后它将在赋值给 `err` 时检查问题是否是解析错误，断言它有一个局部类型 `Error`。如果不这样做，类型断言将失败，导致一个运行时错误，继续展开调用栈，就好像什么都没有中断一样。这种检查意味着如果发生意外情况，例如索引越界，即使我们使用了 `panic` 和 `recover` 来处理解析错误，代码也会失败。

有了错误处理，`error` 方法（因为它是绑定到类型的方法，所以很自然，即使名称与内置的 `error` 类型相同也很好）使得报告解析错误变得容易，而无需担心手动展开解析调用栈：

```go
if pos == 0 {
    re.error("'*' illegal at start of expression")
}
```

尽管这个模式很有用，但它只应在包内部使用。`Parse` 将其内部的 panic 调用转换为 `error` 值；它不会将 panic 暴露给客户端。这是需要遵守的一条好规则。

顺便说一句，如果发生实际的错误，这种 re-panic 的惯用法会改变 panic 的值。然而，原始的和新的失败都会出现在崩溃报告中，因此问题的根本原因将是可见的。你通常不想显示你已经 panic 并恢复了，所以在恢复的值上调用 `panic` 并设置正确的类型通常是有意义的。


## 一个 Web 服务器

最后以一个完整的 Go 程序收尾：一个 Web 服务器。这其实是一种 Web 重新服务器。Google 在 `chart.apis.google.com` 上提供了一个将数据自动格式化为图表和图形的服务。不过，以交互方式使用这很困难，因为你需要将数据作为 URL 的一部分放入。这里的程序为某种形式的数据提供了一个更好的界面：给定一小段文本，它调用图表服务器来生成一个 QR 码，即一个编码文本的方框矩阵。该图像可以用手机的摄像头抓取并解读，例如作为 URL，省去了在手机的小键盘上输入 URL 的麻烦。

以下是完整的程序。接下来是说明。

```go
package main

import (
    "flag"
    "html/template"
    "log"
    "net/http"
)

var addr = flag.String("addr", ":1718", "http service address") // Q=17, R=18

var templ = template.Must(template.New("qr").Parse(templateStr))

func main() {
    flag.Parse()
    http.Handle("/", http.HandlerFunc(QR))
    err := http.ListenAndServe(*addr, nil)
    if err != nil {
        log.Fatal("ListenAndServe:", err)
    }
}

func QR(w http.ResponseWriter, req *http.Request) {
    templ.Execute(w, req.FormValue("s"))
}

const templateStr = `
<html>
<head>
<title>QR Link Generator</title>
</head>
<body>
{{if .}}
<img src="http://chart.apis.google.com/chart?chs=300x300&cht=qr&choe=UTF-8&chl={{.}}" />
<br>
{{.}}
<br>
<br>
{{end}}
<form action="/" name=f method="GET">
    <input maxLength=1024 size=70 name=s value="" title="Text to QR Encode" />
    <input type=submit value="Show QR" name=qr />
</form>
</body>
</html>
`
```

`main` 前面的部分应该很容易理解。一个标志设置了服务器的默认端口。模板变量 `templ` 是乐趣所在。它构建了一个 HTML 模板，服务器将执行该模板来显示页面；稍后详细介绍。

`main` 函数解析标志，并使用我们上面讨论的机制将函数 `QR` 绑定到服务器的根路径上。然后调用 `http.ListenAndServe` 启动服务器；它在服务器运行时阻塞。

`QR` 只是接收请求，该请求包含表单数据，并在名为 `s` 的表单值上执行模板。

模板包 `html/template` 很强大；该程序只是触及了其能力的皮毛。本质上，它通过替换从数据项派生的元素来动态重写一段 HTML 文本，在本例中是传递给 `templ.Execute` 的表单值。在模板文本（`templateStr`）中，双大括号分隔的部分表示模板动作。从 `{{"{{if .}}"}}` 到 `{{"{{end}}"}}` 的部分仅在当前数据项的值（称为 `.`（点））非空时才执行。也就是说，当 string 为空时，模板的这一部分被抑制。

两个代码片段 `{{"{{.}}"}}` 表示在网页上显示传递给模板的数据——查询字符串。HTML 模板包自动提供适当的转义，因此文本可以安全地显示。

模板字符串的其余部分只是页面加载时显示的 HTML。如果解释起来太快，请参阅模板包的[文档](https://go.dev/pkg/html/template/)以进行更全面的讨论。

好了，这就是你需要的：一个有用的 Web 服务器，仅需几行代码，还带有一些数据驱动的 HTML 文本。Go 足够强大，可以在几行代码中完成很多事情。

---

> **原文**：[go.dev/doc/effective_go](https://go.dev/doc/effective_go) | **仓库**：[golang/website](https://github.com/golang/website/blob/master/_content/doc/effective_go.html)
