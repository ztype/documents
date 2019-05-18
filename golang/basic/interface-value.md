---
title: "Golang Interface"
date: 2019-05-19T01:52:29+08:00
draft: false
---

# 怎样使用golang中的interface

在我使用golang之前，用的最多的语言是python。开始学golang之后，发现golang中的interface特别难以理解。我指的是，基础的用法是很简单的，比如我们很容易就知道怎么使用标准库里的一些interface，但是如果我们想要把自己的interface设计好，还需要一些实际的使用经验。在这篇文章中，我将通过讨论golang中的类型系统来尝试说明怎样有效地使用interface。

## interface简介

那么什么是interface？在golang中，一个interface可以代表两种东西：
* 1.方法的组合
* 2.一种类型

我们先来看第一种情况，方法的组合。

### 方法的组合

一般情况下，大家都会举一些例子来介绍interface的特性。那我们也举一些例子吧，就用Animal好了，我们现实生活中都能见到的东西。我们给这个叫`Animal`的interface定义一个行为：`speak`。那么我们就可以说，任何一个可以`speak`的东西都是一个`Animal`。这就是golang类型系统中的核心思想：抽象描述是的对象能做什么行为，而不是保存了什么数据。

让我们先来定义一下我们的`Animal` interface:

``` golang
type Animal interface {
    Speak() string
}
``` 

很简单的一个例子：任何类型，只要它有一个方法叫作`Speak`，我们就可以说它是一个`Animal`。`Speak`方法没有接收任何参数，返回值是一个string。任何定义了这个方法的类型，都满足`Animal`的定义。golang中没有继承关键字，一个类型有没有满足某个interface都是自动识别的。

我们来写几个满足`Animal`这个interface的例子：
``` golang
type Dog struct {
}

func (d Dog) Speak() string {
    return "Woof!"
}

type Cat struct {
}

func (c Cat) Speak() string {
    return "Meow!"
}

type Llama struct {
}

func (l Llama) Speak() string {
    return "?????"
}

type JavaProgrammer struct {
}

func (j JavaProgrammer) Speak() string {
    return "Design patterns!"
}
```

现在我们有了4种类型的`Animal`：`Dog`,`Cat`,`Llama`和一个`JavaProgrammer`。我们可以在我们的`main`函数中，创建一个`Animal`的切片，把4种类型都放进去，看看它们会说些什么，来，试一下：

``` golang
func main() {
    animals := []Animal{Dog{}, Cat{}, Llama{}, JavaProgrammer{}}
    for _, animal := range animals {
        fmt.Println(animal.Speak())
    }
}
```


你可以在[这里](http://play.golang.org/p/yGTd4MtgD5)试着运行一下这个例子.

好了，现在你知道golang中的interface是怎么用的了，我说完了。

是么？并不是。我们来看一些对初学者来说不太了解的例子。

### `interface{}`类型

`interface{}`，一个空interface给我们造成了很多困扰。`interface{}`类型没有定义任何方法，既然golang中没有继承关键字，是否满足某个interface都是自动判断的，那么所有的类型都满足这个interface，因为所有的类型都至少实现了`0`个方法。也就是说，如果你定义了接收`interface{}`类型参数的函数，那么你可以将任何类型的数据传进去，比如这样:

``` golang
func DoSomething(v interface{}) {
   // ...
}
```

这个函数可以接收任意类型作为它的参数。

那么问题来了：既然可以接收任意类型作为参数，那么在这个函数里面，这个参数`v`到底是什么类型的？

很多初学者会说，`v`可以是任何类型。

不，它就是`interface{}`类型。

等下！怎么肥四？！？

其实当我们传递一个参数到`DoSomething`这个函数中的时候，如果需要的话，golang的runtime(运行时)会帮我们做一个类型转换，将这个参数转成`interface{}`类型。

在golang的runtime里，所有的数据有且只有一个确定的类型。`v`的类型就是`interface{}`。*

你可能会问了：既然有类型转换，那么传进`DoSomething`函数里的到底是什么，或者说，一个`[]Animal`切片里到底放的是什么东西？

一个`interface{}`由两个字(word,指针长度)的数据组成。一个字用来指向底层类型的函数表，另一个字指向这个值实际包含的数据。我不想过度深究这个问题。如果你理解了一个`interface{}`实际包含两个字的数据，其中一个指向它实际包含的底层数据，一般来说这就够让你避开常见的坑了。如果你对interface的底层实现很感兴趣，我相信这篇[Russ Cox的文章](https://research.swtch.com/interfaces)会对你非常非常有帮助。

在我们之前的例子中，当我们创建`Animal`切片的时候，并没有用`Animal(Dog{})`之类繁琐的方式来将一个`Dog{}`类型转为`Animal`类型，因为golang会自动帮我们做这些转换。在这个`Animal`的切片中，每个元素都是`Animal`类型，但是它们都有各自的底层数据类型。

那么，知道这些东西有什么用呢？

当我们理解了golang中的interface在内存中是怎么存放的，一些捉摸不定的困惑就变得清晰起来。

例如，这个问题[`can I convert a []T to an []interface{}`](https://golang.org/doc/faq#convert_slice_of_interface)，如果你了解interface的内存结构，这个问题就很好回答了。

下面是一个典型的错误示例：

``` golang
package main

import (
    "fmt"
)

func PrintAll(vals []interface{}) {
    for _, val := range vals {
        fmt.Println(val)
    }
}

func main() {
    names := []string{"stanley", "david", "oscar"}
    PrintAll(names)
}
```

[运行一下试试](http://play.golang.org/p/4DuBoi2hJU)

运行这段代码，我们会看到下面的错误提示:
```
cannot use names (type []string) as type []interface {} in function argument
```

如果我们想把这段代码跑通，就要手动将`[]string`类型转为`[]interface{}`类型。
``` golang
package main

import (
    "fmt"
)

func PrintAll(vals []interface{}) {
    for _, val := range vals {
        fmt.Println(val)
    }
}

func main() {
    names := []string{"stanley", "david", "oscar"}
    vals := make([]interface{}, len(names))
    for i, v := range names {
        vals[i] = v
    }
    PrintAll(vals)
}
```

[跑一下试试](http://play.golang.org/p/Dhg1YS6BJS)

这样做确实很丑，但是`c'est la vie`(这就是生活)。不是所有的东西都能完美。

在实际使用中，这种场景不会经常发生，因为`[]interface{}`类型其实并没有你想的那么好用。

## 指针和interface

关于interface的另一个微妙的地方就在于，它在声明的时候没有指明实现这个interface的应该是一个指针类型还是值类型。给你一个interface类型的值，对于它指向的底层数据是一个值类型还是指针类型你并不清楚。

在之前的例子中，我们对`Animal`的所有实现都是基于值类型的。我们放到`Animal`切片中的也都是值类型。

下面，让我们来试一下将`Cat`的实现换成基于指针类型的。

``` golang
func (c *Cat) Speak() string {
    return "Meow!"
}
```

我们这样改一下，然后尝试[运行这段代码](http://play.golang.org/p/TvR758rfre)，就会看到这样的报错：

``` 
prog.go:40: cannot use Cat literal (type Cat) as type Animal in array element:
    Cat does not implement Animal (Speak method requires pointer receiver)
```

这个错误信息乍看起来有点看不懂。

它的意思是，虽然`Animal`interface并没有要求一定要基于指针类型的实现，但是如果你要将`Cat`类型做为`Animal`interface，这是不行的，因为只有`*Cat`类型满足`Animal`的条件。

要避免这个错误，你可以用一个`*Cat`类型来代替`Cat`类型，比如用`new(Cat)`来代替`Cat{}`（当然，你也可以用`&Cat{}`，我只是觉得`new(Cat)`更好看一点）。

``` golang
animals := []Animal{Dog{}, new(Cat), Llama{}, JavaProgrammer{}}
```

[现在我们的代码跑得通了](http://play.golang.org/p/x5VwyExxBM)

那么我们反向思考一下：如果我们传一个`*Dog`指针代替`Dog`类型，但是这次我们不对`Dog`类型`Speak`方法的声明做任何修改：

``` golang
animals := []Animal{new(Dog), new(Cat), Llama{}, JavaProgrammer{}}
```
[居然也能跑得通！](http://play.golang.org/p/UZ618qbPkj)

我们没有对`Dog`类型的`Speak`方法做任何修改，也能跑得通。

这是因为一个指针类型可以调用它指向的对象的相关方法，但是反过来不行。

也就是说，`*Dog`可以调用`Dog`类型的`Speak`方法，但是反过来不行，比如我们之前的例子里的`Cat`就不能调用`*Cat`的`Speak`方法。

这听起来有一点反常，但是如果你知道golang的一个哲学就是`golang中的一切都是值传递`的话，这些就可以理解了。

*调用一个函数的时候，你实际传进去的数据其实是参数的一个拷贝*

也就是说，如果这个方法的定义是基于值类型的，当你通过某个值类型的变量调用它的时候，实际上是通过这个变量的一个副本来调用的。

当你理解一个类型的方法其实也是一个特殊形式的函数的时候，这个就更容易理解了。

方法：
``` golang
func (t T)MyMethod(s string) {
    // ...
}
```

其实是一个`func(T,string)`函数，调用对象其实和其它的参数一样，是通过值传递的方式传递进函数里的。

在基于值类型的方法里对调用对象做的任何的修改对外部都是不可见的。

如：在`func(d Dog)Speak()string{...}`方法中对`d`对象所做的任何修改对外部都是不可见的。

因为一切都是值传递的。

这就解释了为什么一个`*Cat`的方法不能被`Cat`调用：任何一个`Cat`对象都可能拥有任意个`*Cat`指针指向它。如果要通过`Cat`类型来调用`*Cat`的方法，我们根本不知道要传递哪个指针进去。

相反地，如果我们在`Dog`类型上定义了一个方法，我们手上有一个`*Dog`指针，当我们要通过`*Dog`调用`Dog`的方法的时候，我们可以通过这个指针来确定到某个具体的`Dog`对象。

如果需要的话，golang的runtime可以随时将一个指针解引用得到它指向的对象。

这就意味着，如果我们想通过一个`*Dog`的值调用`Dog`类型的方法，我们可以直接像`d.Speak()`这样调用，golang的runtime会帮我做解引用，我们不必再像别的语言里一样通过`d->Speak()`的方式来调用方法。

## 实战：通过Twitter的API获取到时间戳

此Twitter API返回一个string形式的时间戳：
```
"Thu May 31 00:00:01 +0000 2012"
```

当然，在json中可以用很多种其它的方式来表示时间，json的标准里并没有定义时间的格式。为了简短起见，我并不会把整个推文的json放上来，我们只用json来处理其中的`create_at`字段：
``` golang
package main

import (
    "encoding/json"
    "fmt"
    "reflect"
)

// start with a string representation of our JSON data
var input = `
{
    "created_at": "Thu May 31 00:00:01 +0000 2012"
}
`

func main() {
    // our target will be of type map[string]interface{}, which is a
    // pretty generic type that will give us a hashtable whose keys
    // are strings, and whose values are of type interface{}
    var val map[string]interface{}

    if err := json.Unmarshal([]byte(input), &val); err != nil {
        panic(err)
    }

    fmt.Println(val)
    for k, v := range val {
        fmt.Println(k, reflect.TypeOf(v))
    }
}
```

[跑一下试试](http://play.golang.org/p/VJAyqO3hTF)

运行上面的例子，我们可以看到下面的输出：

```
map[created_at:Thu May 31 00:00:01 +0000 2012]
created_at string
```

可以看到，我们正确地取到到了这个字段，但是这个时间戳string看起来不是很有用。如果我们想用获取到的时间跟现在的时间比一下，用string是不太好比较的。

我们直接把json反序列化成golang标准库里的`time.Time`类型试一下，看看会怎么样：

``` golang

   var val map[string]time.Time

    if err := json.Unmarshal([]byte(input), &val); err != nil {
        panic(err)
    }
```

运行上面的代码，我们会得到这样的一个错误信息:

```
parsing time ""Thu May 31 00:00:01 +0000 2012"" as ""2006-01-02T15:04:05Z07:00"":
    cannot parse "Thu May 31 00:00:01 +0000 2012"" as "2006"
```

这个不太直白的报错是说，在golang的标准库将`string`与`time.Time`类型互相转换的时候产生了错误。

简单点说，我们给的string不符合golang标准库里的默认时间戳格式(因为Twitter的API最初是用Ruby实现的，Ruby中的默认的时间戳格式和golang里的默认时间戳格式不一样)。

如果想要正确地反序列化这个时间戳，我们就需要定义自己的时间类型。

`encoding/json`包是通过判断传递到`json.Unmarshal`函数中的类型有没有实现`json.Unmarshaler`interface来实现反序列化的，这个interface是这样的：

``` golang
type Unmarshaler interface {
    UnmarshalJSON([]byte) error
}
```

这个interface在[json的文档](http://golang.org/pkg/encoding/json/#Unmarshaler)中有说明。

所以我们需要的是，一个实现了`UnmarshalJSON([]byte) error`方法的`time.Time`类型:

``` golang
type Timestamp time.Time

func (t *Timestamp) UnmarshalJSON(b []byte) error {
    // ...
}
```

通过实现这个方法，我们满足了`json.Unmarshaler`interface，我们将自定义的`Timestamp`类型传到`json.Unmarshal`来进行反序列化。

这次，我们要传一个指针类型进去，因为我们想保留`json.Unmarshal`函数中对我们的对象所做的一些修改。

为了修改指针指向的对象的值，我们通过`*`操作符来将指针解引用到它所指向的对象。在`UnmarshalJSON`函数中，`t`表示一个指向`Timestamp`值的指针。通过使用`*t`，我们将指针`t`解引用，得到它所指向的对象。

记住：*golang中的一切都是值传递*

这意味着，在`UnmarshalJSON`中的`t`指针并不是我们最初调用`json.Unmarshal`的时候传进去的那个，`t`是那个指针的一个拷贝。如果你在`UnmarshalJSON`中将另一个指针赋值给`t`，你修改的也只是一个函数内的局部变量，外部并不会看到你的修改。

虽然在`UnmarshalJSON`中的`t`指针和我们最初传进去的不是同一个，但是它们指向的是同一个对象，我们可以通过解引用来访问此对象，并对它做出修改，这样外部可以看到我们做的修改了。

我们可以利用`time.Parse`方法，它的声明是这样的`func(layout, value string) (Time, error)`。

它接收两个string类型的参数，第一个layout指明时间格式，第二个是想要解析的内容。它返回一个`time.Time`类型和一个`error`。你可以在[`time`包的文档](https://golang.org/pkg/time/)中看到更多关于这个函数的细节，在这个例子里，我们不用去手写这个layout参数了，因为标准库里已经有一个`time.RubyDate`可以直接使用。

实际上，我们可以直接这样调用`time.Parse(time.RubyDate, "Thu May 31 00:00:01 +0000 2012")`来将`Thu May 31 00:00:01 +0000 2012`这个字符串解析成一个`time.Time`类型的数据。

在我们这个例子中，我们感兴趣的是`Timestamp`类型。我们可以直接用`Timestamp(v)`来将一个`time.Time`类型的变量`v`转成`Timestamp`类型。

最终我们修改之后的`UnmarshalJSON`函数看起来会是下面这样的：

``` golang
func (t *Timestamp) UnmarshalJSON(b []byte) error {
    v, err := time.Parse(time.RubyDate, string(b[1:len(b)-1]))
    if err != nil {
        return err
    }
    *t = Timestamp(v)
    return nil
}
```

在这里我们截取传入的字节切片是因为它是json元素中的原始数据，包含开始和结束的引号。在传入`time.Parse`解析之前我们需要将它们去掉。

整个时间戳的示例代码可以在[这里](http://play.golang.org/p/QpiFsJi-nZ)看到。

## 实战interface 2
将http请求转化为一个对象。让我们想一下，如何用interface解决web开发中的一个常见问题：我们想把http请求中的内容转为我们想要的一个对象。

我们可以尝试用下面的方法从http请求中取出我们想要的数据：

```golang
GetEntity(*http.Request) (interface{}, error)
```

因为一个`interface{}`可以表示任何类型，所以我们可以将我们的http请求转为任何我们想要的类型。

但这其实是一个很糟糕的策略，因为我们会需要绑定很多的业务逻辑到我们的`GetEntity`函数中，如果想要真正地使用它的返回值，我们还要做很多的类型判断。在实际使用中，其实这类似这样返回`interface{}`类型的函数是很烦人的。事实上，我们有一个更好的解决方案，那就是用传入一个`interface{}`来代替返回一个`interface{}`。

我们也可以这样设计一个返回一个具体类型的函数：

``` golang
GetUser(*http.Request) (User, error)
```

这样其实也不太好，因为可扩展性太差了。我们需要为每个具体的类型来定义一个函数。当然，我们也可以像这样：

``` golang
type Entity interface {
    UnmarshalHTTP(*http.Request) error
}
func GetEntity(r *http.Request, v Entity) error {
    return v.UnmarshalHTTP(r)
}
```

让我们的`GetEntity`函数接收一个实现了`UnmarshalHTTP`方法的类型的变量。为了达成此目的，我们还要为每个需要从http解析的数据类型实现此方法。

``` golang
func (u *User) UnmarshalHTTP(r *http.Request) error {
   // ...
}
```

在你的代码中，你可以声明一个`User`类型的变量，然后将它的地址传入`GetEntity`函数：

``` golang
var u User
if err := GetEntity(req, &u); err != nil {
    // ...
}
```

这和你解析json数据的方式很像。这种方式可以保证一致性和安全性。因为当你声明`var u User`一个`User`变量的时候，它已经自动被赋值了一个零值。golang不像别的语言，声明和初始化是分开的，可以只声明一个变量不初始化它，这有一个潜在的风险就是你可能会访问到一些无效数据。在golang里，一个变量被声明的时候，runtime会将这个变量的数据内容赋为零值。即使你的`UnmarshalHTTP`函数从http请求里反序列化数据失败，变量也会是一个有效的零值而不是无效数据。

如果你是一个Python programmer，这对你来说可能看起来有点奇怪，因为它正好和你在python里做的反过来了。这种形式的好处是，我们可以定义任意多的类型的变量，对每个类型的变量来说，它从http里解析出来的数据都是可靠的。每个类型都可以在自己的方法里决定自己需要怎样从http里被解析出来。这样我们就可以围绕这个`Entity`类型来创建通用的http响应接口。

## 总结

我希望这篇文章可以让你在使用interface的时候更自信一点。

我也希望你能记住这些：

* 要根据数据类型之间共同的功能来设计抽象，而不是它们之前共同的字段

* 一个`interface{}`变量不是任意类型，它就是`interface{}`类型。它是两个字（指针长度）宽，数据结构看起像是（key,value）

* 接收一个`interface{}`类型比返回一个`interface{}`更好

* 一个指针类型可以调用它指向的变量的方法，反过来不行

* 一切都是值传递，即使是方法的接收者

* 一个interface变量严格意义上并非只是一个指针或者不是一个指针，它就是一个interface

* 如果你想要在类型的方法内部修改对象的数据内容，用`*`操作符来手动解引用一个指针。（用指针类型做方法接收者而不是对象类型）

这些就是我个人认为interface中比较困惑的地方。

Happy coding :)

8:07 am  •  1 October 2012  •  127 notes

source:https://jordanorelli.com/post/32665860244/how-to-use-interfaces-in-go#_=_
