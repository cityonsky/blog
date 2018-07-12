
> 本文翻译自 Go 语言官方文档，收集了 Go 语言代码评审时的常见问题，适合掌握基本语法的初学者。
> 
> 阅读时间大约 15 分钟
> 
> [原文链接](https://github.com/golang/go/wiki/CodeReviewComments)

## Gofmt
代码提交前先跑一下 gofmt 工具，它能自动修复大多数形式化问题（对齐、换行等待）。

现在几乎所有 Go 项目都在使用 gofmt，没有使用的是因为它们在使用 goimports（它支持所有 gofmt 的功能，另外还可以规范化导入行的写法）。

下面我们讨论的都是这两个自动工具做不到的问题检查。

## Comment Sentences
用完整的句子注释声明，注释往往以声明的名称开头，以句点结束，像这样：

    // Request represents a request to run a command.
    type Request struct { ...

    // Encode writes the JSON encoding of req to w.
    func Encode(w io.Writer, req *Request) { ...

当然注释的结尾用叹号或问号也没问题。之所以这样是使用 godoc 生成的文档可读性很高。

更多信息请参考[effective go 文档](https://golang.org/doc/effective_go.html#commentary)。

## Contexts
context.Context 是 Go 语言中的一个标准类型，它内部往往包含跨越 API 和进程边界的安全证书、跟踪信息、过期时间和取消信号等信息。

Go 程序的 RPC 调用时需要显式地传递 context。

不要把 Context 放到结构类型中，把它放到参数列表里，除非你的方法名改不了（比如方法的签名必须与标准库或第三方库中的接口定义匹配）。如果放到参数列表里，通常作为第一个参数，像这样：

    func F(ctx context.Context, /* other arguments */) {}

你要是有数据需要传递，先考虑利用函数参数、接收器或全局变量。如果数据真的适合放在 Context 中，要使用 Context.Value，不要自定义 Context 类型，也不要扩展 Context 接口。

Context 是不可变的，可以将相同的上下文传递给多个调用，它们共享信息。

## Copying
这个我没有真正理解，如果你理解的话，请告诉我。
To avoid unexpected aliasing, be careful when copying a struct from another package. For example, the bytes.Buffer type contains a []byte slice and, as an optimization for small strings, a small byte array to which the slice may refer. If you copy a Buffer, the slice in the copy may alias the array in the original, causing subsequent method calls to have surprising effects.

In general, do not copy a value of type T if its methods are associated with the pointer type, *T.

## Declaring Empty Slices
定义一个空 slice，我们倾向于这么写：

    var t []string
而不是这样：

    t := []string{}
前者方式定义了一个 nil 的 slice 值，后者定义了一个不空的长度为零的 slice。这两种在很多情况下都一样，比如它们的长度和容量都是零，但还是推荐前者。
在一些特殊的地方，这两者不等价，比如 json 编码中，前者编码为 null，后者编码为 json 数组 []。

设计接口时，要避免这两者引起的不同导致的细微的程序问题。


## Crypto Rand
别用 math/rand 包生成（哪怕是一次性的）键值。没有设置随机种子时，生成器是完全可预测的，就算用 time.Nanoseconds() 设置随机种子，熵也太少了。替代的方案是使用 crypto/rand 包中的 Reader，需要随机文本的话就转换成十六进制或者base64编码：

    import (
        "crypto/rand"
        // "encoding/base64"
        // "encoding/hex"
        "fmt"
    )

    func Key() string {
        buf := make([]byte, 16)
        _, err := rand.Read(buf)
        if err != nil {
            panic(err)  // out of randomness, should never happen
        }
        return fmt.Sprintf("%x", buf)
        // or hex.EncodeToString(buf)
        // or base64.StdEncoding.EncodeToString(buf)
    }

## Doc Comments
所有顶级的、导出的名字都需要有文档注释，那些很重要的内部类型和函数也该如此。文档的规范参见[这里](https://golang.org/doc/effective_go.html#commentary)。

## Don't Panic
通常的错误处理要用 [error](https://golang.org/doc/effective_go.html#errors) 和多返回值，尽量不要用 Panic。

## Error Strings
Error 的字符串不要大写开头（特定名词除外），也不要以标点结尾，因为它们经常被打印在错误输出日志中。 也就是说，要这样定义错误：

    fmt.Errorf("something bad") 

而不是这样：
    
    fmt.Errorf("Something bad.")

这样使用方在日志打印中会自然很多：

    log.Printf("Reading %s: %v.", filename, err)

## Examples
增加一个新包时，要包含[使用的例子](https://blog.golang.org/examples)：可以是可运行的例子，也可以包含完整调用的测试演示。

## Goroutine Lifetimes
创建协程时，要清楚它是否会退出，以及什么时候退出。

不再需要的协程，如果不妥善处理，会造成一些诡异的问题，比如：
- 被收、发 channel 消息阻塞住的协程会造成泄露；
- 在结果都不需要的时候，修改输入会导致无谓的并发数据竞争；
- 发送消息给已经关闭的 channel 导致 Panic；

尽量保持并发代码简单，协程的生命周期明确。要是真的做不到，就在文档中说明协程的退出时间和原因。

## Handle Errors
不要用 “_” 方式丢弃错误，要检查它来确保函数调用成功。处理错误，返回它们，甚至在必要的时候抛出 Panic。更多详情参见[这里](https://golang.org/doc/effective_go.html#errors)。

## Imports
好的包名，在导入时一般都不会发生冲突的。要尽量避免重命名导入包名，除非发生名字冲突。要真出现了冲突问题，优先考虑重命名本地包或者特定工程的包的导入。

导入包要分组，分组之间用空行隔开，标准库放到第一分组中：

    package main
    
    import (
    	"fmt"
    	"hash/adler32"
    	"os"
    
    	"appengine/foo"
    	"appengine/user"
    
        "github.com/foo/bar"
    	"rsc.io/goversion/version"
    )
goimports 工具会帮助我们做这个事情。

## Import Dot
点导入的形式，可以方便有循环依赖的测试用例编写，比如下面的代码：

    package foo_test
    
    import (
    	"bar/testutil" // also imports "foo"
    	. "foo"
    )
测试文件 bar/testutil 导入了 foo 包，如果测试用例需要导入 bar/testutil, 那它就不能放在 foo 包下面，否则就会出现循环引用。这时候使用点导入，就可以假装测试用例是在 foo 包下面（其实是在 foo_test 包下面）。

除了上述情况，不要使用点导入，它严重影响了代码的可读性，你没办法区分一个 Quux 是当前包的一个标识符还是某个导入包的。

## In-Band Errors
在 C 或者类似的语言中，一个常见的做法是通过异常返回值（比如 -1 或者 空指针）告知调用者发生了错误，我们管这种方式叫做内联错误（In-Band Errors），像下面这样：

    // Lookup returns the value for key or "" if there is no mapping for key.
    func Lookup(key string) string
    
    // Failing to check a for an in-band error value can lead to bugs:
    Parse(Lookup(key))  // returns "parse failure for value" instead of "no value for key"
相比内联错误，在 Go 语言中我们更推荐额外返回一个值来告知错误，比如 error 或布尔值，像下面这样：

    // Lookup returns the value for key or ok=false if there is no mapping for key.
    func Lookup(key string) (value string, ok bool)
    This prevents the caller from using the result incorrectly:
    
    Parse(Lookup(key))  // compile-time error
    And encourages more robust and readable code:
    
    value, ok := Lookup(key)
    if !ok  {
        return fmt.Errorf("no value for %q", key)
    }
    return Parse(value)

这个规则对导出和非导出的函数都适用。

像 nil、""、0、-1 这样的返回值，如果它们是合法的方法调用结果（判断的标准是函数调用者可以用相同的逻辑处理这些值和其他值），那么不增加 error 返回值也是合理的。

像 strings 这样的标准库，返回了内联错误 值，这个简化了字符串处理代码，代价就是需要花费调用者更多的精力。

## Indent Error Flow
要缩进错误处理逻辑，不要缩进常规代码。这样可以改进代码的可读性，读者可以快速地浏览逻辑主干。

下面这个代码不好：

    if err != nil {
    	// error handling
    } else {
    	// normal code
    }

要改成下面这样：

    if err != nil {
    	// error handling
    	return // or continue, etc.
    }
    // normal code

如果 if 语句中有初始化逻辑，像这样：

    if x, err := f(); err != nil {
    	// error handling
    	return
    } else {
    	// use x
    }

那就把初始化移到外面，改成这样：

    x, err := f()
    if err != nil {
    	// error handling
    	return
    }
    // use x

## Initialisms
名字中的首字母缩写单词或缩略语（比如“URL”或“NATO”），要保持相同的大小写。比如“URL”可以写成“URL”或者“url”，在词组中可以是“urlPony”或者“URLPony”，但别写成Url。另一个例子是要写成“ServerHTTP”而不是“ServerHttp”。多个缩略词在一起的名字，写成“xmlHTTPRequest”或者“XMLHTTPRequest”都行。

再举个“ID"的例子，表示“identifier”缩写时，要写成“appID”这样，而不是“appId”。

protocol buffer 产生的自动化代码是个例外，写代码对人和对机器的要求不能一样。

## Interfaces
总的来说，Go 的接口要包含在使用方的包里，不应该包含在实现方的包里。实现方只需要返回具体类型（通常是指针或结构），这样可以方便地增加实现，而不需要扩展重构。

不要先定义接口再用它。脱离真实的使用场景，我们都不能确定一个接口是否有存在的价值，更别提设计接口的方法了。

测试时不要定义假接口给实现者用，反而是要定义公开的 API，用真实的实现进行测试。举个例子，下面 consumer 是接口使用方，其实现和测试代码如下：

    package consumer //consumer.go:
    
    type Thinger interface { Thing() bool }
    
    func Foo(t Thinger) string { … }
...

    package consumer //consumer_test.go:
    
    type fakeThinger struct{ … }
    func (t fakeThinger) Thing() bool { … }
    …
    if Foo(fakeThinger{…}) == "x" { … }
    
下面这个接口的实现方是不推荐的：
    
    // DO NOT DO IT!!!
    package producer
    
    type Thinger interface { Thing() bool }
    
    type defaultThinger struct{ … }
    func (t defaultThinger) Thing() bool { … }
    
    func NewThinger() Thinger { return defaultThinger{ … } }

应该返回具体的类型，让消费者来 mock 生产者的实现：

    package producer
    
    type Thinger struct{ … }
    func (t Thinger) Thing() bool { … }
    
    func NewThinger() Thinger { return Thinger{ … } }

## Line Length
在 Go 代码中没有行长度的标准规定，避免不舒服的长度就好；类似的，长一些代码可读性更强时，也不要刻意换行。

大多数非自然（在方法调用和声明的过程中）的换行，都是可以避免的，只要选择合理数量的参数列表和合适的变量名。一行代码过长，往往是因为代码中的各个名字太长了，去掉那些长名字就好了。

换句话说，在语义的分割点换行，而不是单单看行的长度。万一你发现某一行太长了，要么改名，要么调整语义，往往就解决问题了。

这里没有一个“一个代码行最多不超过多少个字符”的规定，但是一定存在一行代码功能太杂，需要改变函数边界的规则。

## Mixed Caps
参考 [mixed-caps](https://golang.org/doc/effective_go.html#mixed-caps)，Go 语言推荐驼峰式命名。有一点点地方不同于其他语言：非导出的常量要命名成 maxLength，而不是 MaxLength 或者 MAX_LENGTH。


## Named Result Parameters & Naked Returns
比较下面两个函数声明，想象在文档中看到它们的感受：

    func (n *Node) Parent1() (node *Node)
    func (n *Node) Parent2() (node *Node, err error)

这里是第二个:

    func (n *Node) Parent1() *Node
    func (n *Node) Parent2() (*Node, error)

有没有感觉第一个显得啰哩啰嗦？

我们再看另一种情况：一个函数返回两三个相同类型的参数，或者返回参数的含义在上下文中不明确，给它们加上名字其实很有用。

    func (f *Foo) Location() (float64, float64, error)

上面这个就没下面的好：

    // Location returns f's latitude and longitude.
    // Negative values mean south and west, respectively.
    func (f *Foo) Location() (lat, long float64, err error)

不要仅仅以在函数中可以少声明一个变量为由，给返回参数加上名字，偷懒是小，代码文档的可读性是大。

一些随手的简单函数，不用命名返回参数也没啥问题。一旦函数的规模大一些，就要显式地指出它的返回值含义。不要为了给返回参数命名而命名，代码文档的清晰性才是第一位要考虑的。

最后，有一些模式中需要在 defer 闭包中修改返回值，这时候给返回值命名没啥问题。


## Package Comments

包注释跟其他通过 godoc 展示的注释一样，必须临近包的声明，中间没有空行，像这样：

    // Package math provides basic constants and mathematical functions.
    package math

或这样：

    /*
    Package template implements data-driven templates for generating textual
    output such as HTML.
    ....
    */
    package template

main 包的注释，一般是按 main 所在的目录来命名，比如一个文件在 seedgen 目录下，那相关注释可以写成下面任意一种：

    // Binary seedgen ...
    package main
    or
    
    // Command seedgen ...
    package main
    or
    
    // Program seedgen ...
    package main
    or
    
    // The seedgen command ...
    package main
    or
    
    // The seedgen program ...
    package main
    or
    
    // Seedgen ..
    package main

一个细节的问题是，用二进制名做注释的开头时，首字母要不要大写？答案是要！注释是文档，所以要符合英文的文法，这就包括句子的首字母要大写。至于文档中的大小写跟实际的二进制名字没有严格匹配，那也只能这样了。所以我推荐这样的方式：

    // The seedgen command ...
    package main
    or
    
    // The seedgen program ...
    package main

## Package Names
大部分对包内元素的引用，都是通过包名实现的，比如你有一个包叫 chubby，里面的变量不必定义成 ChubbyFile（客户使用的时候引用的方式是：chubby.ChubbyFile），而应该定义成 File（客户使用的时候是 chubby.File）。

为减少引用冲突，避免用 util，common，misc，api，types 这样的通用词命包名。

## Pass Values

别为了节省几个字节而传递指针参数，如果一个函数只用到了某个指针参数的 *x  形式，那就根本不应该传指针参数。但这个建议不适用于大数据结构或可能会增长的数据结构的参数传递。

## Receiver Names

方法接收器的名字，应该是一个简写，经常是类型前缀的一两个字母，不要用 me、this、self 这样的通用名字。约定和一致性带来简洁性，如果你在一个方法中用 c 表示接收器，就别在其他方法中用 cl。

## Receiver Type

Go 的初学者往往有一个事情选择不好，就是方法的接收器是值还是用指针。一个基本的原则是把握不准的时候就用指针，但有些情况下用值接收更有道理，性能更好。这里有一些有用的细则：

- 方法需要改变接收器的内部值，那就必须用指针。
- 接收器内含有 sync.Mutex 或者类似的同步域，那就必须指针，以避免拷贝。
- 接收器是一个大数据结构或者数组，指针会效率更高。
- 如果接收者是一个结构、数组、slice，且内部的元素有指针指向一些可变元素，那更倾向于用指针接收，来提供更明确的语义。
- 如果是一个 map、func 或 chan，不要用指针接收；如果是一个 slice，并且方法中没有改变其值（reslice 或 重新分配 slice），不要用指针。
- 若接收者是一个小对象或数组，概念上是一个值类型（比如 time.Time），并且没有可变域和指针域，或者干脆就是 int、string 这种基本类型，适合用值接收器。值接收器可以减少垃圾内存，通过它传递值时，会优先尝试从栈上分配内存，而不是堆上。但保不齐在某些情况下编译器没那么聪明，所以这个在用之前要测一下。
- 最后，要是把握不准，就用指针。

## Synchronous Functions
能用同步函数就不用异步的，同步函数的含义是直接返回结果，或者在返回之前调用回调函数或完成 channel 操作。

同步函数保持协程在调用时的本地性，更容易推断协程的生命周期、内存泄漏和并发竞争，测试也更简单。

如果调用者需要并发处理，它可以很简单地开一个单独的协程；但对一个异步函数来说，调用者想改成同步的就太难了，有时候就根本不可能。

## Useful Test Failures
测试的失败分支，需要加上有用的诊断信息，包括输入、实际输出、期望输出都是什么，像这样:

    if got != tt.want {
    	t.Errorf("Foo(%q) = %d; want %d", tt.in, got, tt.want) // or Fatalf, if test can't test anything more past this point
    }

写的时候注意实际输出和期望输出别写反了。

如果你觉得代码量太大，可以尝试[表驱动测试](https://github.com/golang/go/wiki/TableDrivenTests)方案。

另外一个区分测试失败分支的方法是使用不同的测试函数名，像这样：

    func TestSingleValue(t *testing.T) { testHelper(t, []int{80}) }
    func TestNoValues(t *testing.T)    { testHelper(t, []int{}) }

不管哪种方法，目标是给以后维护你代码的人，提供足够多的诊断信息。


## Variable Names
Go 语言变量名短比长好，尤其是那些作用域很小的本地变量。用 c，不要用 lineCount；用 i，不要用 sliceIndex。

基本准则是：声明到使用的距离越远，变量名字就越详尽。方法接受者的名字，一两个字符就够了，通常的循环下标、读取器引用，一个字符（i，r）足够；那些不常见的，全局的变量，名字要起的说明性更强一些。
