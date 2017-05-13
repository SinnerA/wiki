---
title: golang解惑
date: 2017-05-12
---

[TOC]

## Functions

### 闭包

通过一个例子来说明

```go
var rmdirs []func()
for _, d := range tempDirs() {
	dir := d
    os.MkdirAll(dir, 0755)
    rmdirs = append(rmdirs, func() {os.RemoveAll(dir)})
}

for _, rmdir := range rmdirs {
	rmdir()
}
```

上面为什么需要用`dir: = d`赋值语句了，直接把d传进`RemoveAll`不就好了吗？

在上面的代码中，for循环创建了一个新的作用域，其中dir被声明了，所有的在这个循环内的function value都会共享同一份dir的存储空间，因此，如果不采用上述代码，那么，传进function value的dir就会全部是一样的（都会是最后一个）。

d有一个存储空间，虽然它的值一直在变化，但是RemoveAll中如果使用了d，那么RemoveAll会与d组成一个闭包，d的值在变化，闭包中的值也随之变化。

### defer

```html
需要注意的是, defer调用的函数参数的值defer被定义时就确定了.
```

例如

```go
i := 1
defer fmt.Println("Deferred print:", i)
i++
fmt.Println("Normal print:", i)
```

打印的内容如下:

```go
Normal print: 2
Deferred print: 1
```

因此我们知道，在 "defer fmt.Println("Deferred print:", i)" 调用时，i 的值已经确定了，因此相当于 **defer fmt.Println("Deferred print:", 1)**了。

```
需要强调的时, defer调用的函数参数的值在defer定义时就确定了, 而defer函数内部所使用的变量的值需要在这个函数运行时才确定.
```

例如:

```go
func f1() (r int) {
    r = 1
    defer func() {
        r++
        fmt.Println(r)
    }()
    r = 2
    return
}

func main() {
    f1()
}
```

上面的例子中, 最终打印的内容是 "3", 这是因为在 "r = 2" 赋值之后, 执行了 defer 函数, 因此在这个函数内, r 的值是2了, 自增后变为3.

```html
defer函数调用的执行时机是外层函数设置返回值之后, 并且在即将返回之前.
```

例如:

```go
func f1() (r int) {
    defer func() {
        r++
    }()
    return 0
}
func main() {
    fmt.Println(f1())
}
```

上面 **fmt.Println(f1())** 打印的是什么呢? 正确答案是 1. 这是为什么呢?
要弄明白这个问题, 我们需要牢记两点

- defer 函数调用的执行时机是外层函数设置返回值之后, 并且在即将返回之前
- return XXX 操作并不是原子的

### recover

函数中如果抛出一个panic的异常，可以在defer中通过recover捕获这个异常，那么recover函数会结束当前的panic状态，并且返回panic值。

在一个主进程，多个协程处理逻辑的结构中，这个很重要，如果不用recover捕获panic异常，会导致整个进程出错而中断。

## Method

在go中，实现面向对象编程的方法，即一个对象是指一个拥有method的值或变量，method是和指定类型关联的函数。

实际上，在GO中，支持对任意的named type（包括函数）定义method。比如：

```go
type Path []Point

func (path Path) Distance() float64 {
	sum := 0.0
    for i := range path {
    	if i > 0 {
        	sum += path[i-1].Distance(path[i])
        }
    }
}
```

由于GO中，是以值传递的方式传函数参数的，如果希望method函数能够修改对象内部的状态的话，则需要定义receiver type为对象的指针（该指针可以为nil），如下：

```go
func (p *Point) ScaleBy(factor float64) {
	p.X *= factor
    p.Y *= factor
}
```

对于一个*Pointer类型的变量，GO也会隐式地调用Point.Distance函数，下面两种使用方式是等价的

```go
pptr.Distance(q)
(*pptr).Distance(q)
```

如上，采用第一种使用方式的时候，GO也会隐式地转成第二种使用方式。

```html
因为GO支持pointer receiver，因此，对本身是pointer类型的named type，不支持定义method。
```

```go
type P *int
func (P) f() { /* do something */ } //compile error
```

### struct嵌套

和field一样，通过匿名的struct组合，大的struct可以直接调用struct内部的field的method，如下

```go
type Point struct {X, Y float64}
type ColoredPoint struct {
	Point
    Color color.RGBA
}

var cp ColoredPoint
cp.X = 1
cp.ScaleBy(2) // calls the method of Point
fmt.Println(p.Distance(q.Point))
```

### method value

通常的，我们在一个操作中select和call一个method，例如`p.Distance()`，但是，也可以把select和call分开。selector p.Distance会产生一个method value，一个method绑定到了特定的对象p，这个函数可以保存起来供后面重复使用。

```go
p := Point{1, 2}
q := Point{4, 5}
distanceFromP := p.Distance
fmt.Println(distanceFromP(q))

ScaleP := p.ScaleBy
ScaleP(2)
ScaleP(3)
ScaleP(10)
```

method value通常在你需要一个变量表示多个method value中的一个的时候，非常有用，例如

```go
type Point struct {X, Y float64}
func (p Point) Add(q Point) Point {return Point{p.X + q.X, p.Y + q.Y}}
func (p Point) Sub(q point) Point {return Point{p.X - q.X, p.Y - q.Y}}

type path []Point
func (path Path) TranslateBy(offset Point, add bool) {
	var op func(p, q Point) Point
    if add {
    	op = Point.Add
    } else {
    	op = Point.Sub
    }
    for i := range path {
    	path[i] = op(path[i], offset)
    }
}
```

### bitset

GO中没有set类型，虽然有时可以用map[T]bool来代替，但是，有时候bit set可以节省很多空间，如下

```go
package main

import (
	"fmt"
)

type IntSet struct {
	words []uint64
}

func (s *IntSet) Has(x int) bool {
	word, bit := x/64, uint(x%64)
	return word < len(s.words) && s.words[word]&(1<<bit) != 0
}

func (s *IntSet) Add(x int) {
	word, bit := x/64, uint(x%64)
	for word >= len(s.words) {
		s.words = append(s.words, 0)
	}
	s.words[word] |= 1 << bit
}

func (s *IntSet) UnionWith(t *IntSet) {
	for i, tword := range t.words {
		if i < len(s.words) {
			s.words[i] |= tword
		} else {
			s.words = append(s.words, tword)
		}
	}
}

func main() {
	var s IntSet
	s.Add(100)
	fmt.Println(s.Has(100))
}
```

## Interfaces

在GO中，Interface是一种抽象类型，它不展示任何的内部的值，只提供一些method。因此，给定一个interface，你只知道它的几个method，对于内部的实现是无法知道的。

一个具体的类型如果实现了interface里的所有方法，那么这个类型会自动地被认为是该interface的一个实例。

### 接口类型

可以基于现有的interface组合出新的interface，例如

```go
type ReadWriter interface {
	Reader
    Writer
}
type ReadWriteCloser interface {
	Reader
    Writer
    Closer
}
```

上面的跟结构体的匿名field一样，method会自动地提升到大的interface中。

### Interface Satisfication

当一个类型（通常为struct）包含某个interface中所有的methods时，则该类型satisfies这个interface。可以参考Java中的接口以及接口的实现。

一个interface变量只能使用它本身所有的method，即使赋值给它的具体类型还有额外的method，也是无法调用的，例如

```
os.Stdout.Write([]byte("hello"))
os.Stdout.Close()

var w io.Writer
w = os.Stdout
w.Write([]byte("hello"))
w.Close() //compile error: interface only has write method
```

### Interface Values

Interface value包含两个组件，分别是具体地**类型**和类型的**值**，它们被称作interface的dynamic type和dynamic value。

以一个简单的例子，来说明interface value，如下

```go
var w io.Writer
w = os.Stdout
w = new(bytes.Buffer)
w = nil
```

对于语句1，创建了type和value都为nil的interface。

对于语句2，创建了type为*os.File，value为Os.Stdout的interface

对于语句3，创建了type为*bytes.Buffer,value为[]byte的interface

[![img](http://o8m1nd933.bkt.clouddn.com/blog/go/interface_values.png)](http://o8m1nd933.bkt.clouddn.com/blog/go/interface_values.png)

两个interface相等的条件为

- 两个interface都为nil
- dynamic type和dynamic value都相等

### An interface Containing a Nil Pointer Is Non-Nil

一个nil的interface和一个interface包含一个pointer为nil情况是不同的，如下

```go
const debug = true

func main() {
	var buf *bytes.Buffer
    if debug {
    	buf = new(bytes.Buffer)
    }
    f(buf)
    if debug {
    	//.... use buf
    }
}

func f(out io.Writer) {
	if out != nil {
    	out.Write([]byte("done!\n"))
    }
}
```

上述程序会panic在out.Write函数，当main调用f时，它会把type信息*byte.Buffer赋值给out的type，nil作为value赋值给out的value，因此，此时out不是nil的interface，于是，out调用write的时候就panic了。

正确的写法是

```go
//io.Writer是interface，因此buf的type和value都为nil（interface == nil等价于type和value都为nil）
var buf io.Writer 
if debug {
	buf = new (bytes.Buffer)
}
f(buf)
```

### Type Assertions

Type Assertions一般是`x.(T)`，其中x是interface类型，T是类型，称为asserted类型。Type Assertion检查interface的dynamic type是否和类型T匹配。

- 如果asserted类型是**具体的类型**，并且dynamic type与之匹配了，那么x.(T)的返回值是x的dynamic value，否则，程序panic。

```go
var w io.Writer
w = os.Stdout
f := w.(*OS.File) //success, f == OS.Stdout
c := w.(*bytes.Buffer) // panic
```

- 如果asserted的类型T也是是**interface类型**，那么检查dynamic type是否满足T，如果检查通过，dynamic type和dynamic value保持不变，但是interface类型变为T。这种type assertion的意义是**可以改变interface的method组合**（能够新增method的数量）。

```go
var w io.Writer
w = os.Stdout
rw := w.(io.ReadWriter) // success: *os.File has both Read and Write

w = new(ByteCounter)
rw = w.(io.ReadWriter) // panic: *ByteCounter has no read method
```

## Reflection

reflection是golang中用来获取interface的type, value和method的方式。

### 类型和接口

Go 是静态类型的。每一个变量有一个静态的类型，也就是说，有一个已知类型并且在编译时就确定下来了：int，float32，*MyType，[]byte 等等。如果定义

```go
type MyInt int 
var i int 
var j MyInt
```

那么 i 的类型为 int 而 j 的类型为 MyInt。即使变量 i 和 j 有相同的底层类型，它们仍然是有不同的静态类型的。未经转换是不能相互赋值的。

**在类型中有一个重要的类别就是接口类型，表达了固定的一个方法集合。**一个接口变量可以存储任意实际值（非接口），只要这个值实现了接口的方法。例如

```go
// Reader 是包裹了基础的 Read 方法的接口。.
type Reader interface {
	Read(p []byte) (n int, err os.Error)
}
// Writer 是包裹了基础 Write 方法的接口。
type Writer interface {
	Write(p []byte) (n int, err os.Error)
}

var r io.Reader
r = os.Stdin
r = bufio.NewReader(r)
r = new(bytes.Buffer) // 等等
```

有一个事情是一定要明确的，不论 r 保存了什么值，r 的类型总是 io.Reader：**Go 是静态类型，而 r 的静态类型是 io.Reader**。

也有人说 Go 的接口是动态类型的，不过这是一种误解。它们是静态类型的：接口类型的变量总是有着相同的静态类型，这个值总是满足空接口，只是**存储在接口变量中的值运行时有可能被改变类型**。

### 接口的特色

接口类型的变量存储了两个内容：赋值给变量实际的值和这个值的类型描述。例如

```go
var r io.Reader 
tty, err = os.OpenFile("/dev/tty", os.O_RDWR, 0) 
if err != nil { 
	return nil, err 
} 
r = tty
```

用模式的形式来表达 r 包含了的是 (value, type) 对，即 (tty, *os.File)。**接口的静态类型决定了哪个方法可以通过接口变量调用，即便内部实际的值可能有一个更大的方法集**。（接口静态类型的作用）

一个很重要的细节是接口内部的对总是 (value, 实际类型) 的格式，而不会有 (value, 接口类型) 的格式。接口不能保存接口值。

### 镜像方法

```go
reflect.ValueOf(i interface{}) //从接口值到反射对象
(reflect.Value).Interface()    //从反射对象到接口值
或者
(reflect.Value).Interface().(valueType) //使用断言
```

### 值的设置

举例：

```go
var x float64 = 3.4 
v := reflect.ValueOf(x) 
v.SetFloat(7.1) // Error: will panic.

panic: reflect.Value.SetFloat using unaddressable value
```

问题不在于值 7.1 不能地址化；在于 v 不可设置。原因是，v其实是x 的一个副本，而不是 x 本身。如果在反射值内部允许更新 x 的副本v，那么 x 本身不会收到影响。这会造成混淆，并且毫无意义，因此这是非法的。

反射值需要某些**内容的地址**来修改它指向的东西。将上述例子改成：

```go
var x float64 = 3.4 
p := reflect.ValueOf(&x) // 注意：获取 X 的地址
v := p.Elem()
v.SetFloat(7.1)
```

如果要设置struct类型T的字段的值，需要T 的字段名要大写（可导出），因为只有可导出的字段是可设置的。

### 总结

反射的规则如下：

- 从接口值到反射对象的反射。
- 从反射对象到接口值的反射。
- 为了修改反射对象，其值必须可设置。

参考：[GoLang反射的规则](http://www.tuicool.com/articles/7NjaQn)

### interface的类型

- 获取interface的type

```go
t := reflect.Typeof(3)
fmt.Println(t) //"int"
```

Typeof用来获取某个interface的类型。

### interface的值

- 获取interface的value

```go
v := reflect.ValueOf(3)
fmt.Println(v) // "3"
fmt.Println("%v\n", v) // "3"
fmt.Println(v.String()) // "<int 3>"
```

- 设置interface的value

```go
x := 2
d := reflect.ValueOf(&x).Elem() // d refers to the variable x
px := d.Addr().Interface.(*int) // px := &x
*px = 3
fmt.Println(x)
d.Set(reflect.ValueOf(4))
```

### Kind

Type和Value都有Kind，就是实际类型（底层类型），如下：

```go
Bool
Int Int8 Int16 Int32 Int64 
Uint Uint8 Uint16 Uint32 Uint64 Uintptr
Float32 Float64
Complex64 Complex128
Array
Chan
Func
Interface
Map
Ptr
Slice
String
Struct
UnsafePointer
```

### interface的函数

- 遍历struct的method

```go
v := reflect.ValueOf(x)
t := v.Type()

for i := 0; i < v.NumMethod(); i++ {
	methodType := v.Method(i).Type()
}
```

通过NumMethod来遍历，获取到对应的Method，其类型是`reflect.Value`，通过其Call方法来调用该函数。

- 通过函数名来调用

```go
args := make([]reflect.Value, 0)
args = append(args, reflect.ValueOf(x))
v.MethodByName("test").Call(args)
```

通过MethodByName获取到函数然后调用Call也能完成调用。

### 完整例子

下面以一个完整的例子，来完整的展示reflect.Type,reflect.Value和reflect.Method的用法。

```go
package main

import (
	"fmt"
	"reflect"
)

type Test struct {
	i int
	f float64
	m map[string]int
	v []string
}

func NewTest() Test {
	var t Test
	t.m = make(map[string]int)
	return t
}

func (t *Test) SetI(i int) {
	t.i = i
}

func (t *Test) SetF(f float64) {
	t.f = f
}

func (t *Test) SetM(key string, value int) {
	t.m[key] = value
}

func (t *Test) SetV(value string) {
	t.v = append(t.v, value)
}

func main() {
	t := NewTest()
	fmt.Println(reflect.TypeOf(t)) //main.Test
	v := reflect.ValueOf(&t)
	fmt.Println(v.Kind()) // ptr
	typ := reflect.TypeOf(t)
	fmt.Println(v.Type()) // *main.Test
	for i := 0; i < typ.NumMethod(); i++ {
		method := typ.Method(i)
		fmt.Println("%s", method.Name)
	}
	fmt.Println(t) // {0 0 map[] []}
	args := make([]reflect.Value, 0)
	args = append(args, reflect.ValueOf(2.16))
	v.MethodByName("SetF").Call(args)
	fmt.Println(t) // {0 2.16 map[] []}
}
```
### Tips

**静态类型**：

静态类型是指有一个已知类型并且在编译时就确定下来了，并且不会再改变。

```go
type MyInt int 
var x MyInt = 7 
v := reflect.ValueOf(x)
```

这里，x的静态类型是MyInt，而不是int

**实际类型**：

```go
v.Kind()
```

通过Kind可以得到变量的实际类型，v的实际类型是int

