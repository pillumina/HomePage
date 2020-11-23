---
title: "Golang逃逸分析"
date: 2020-11-23T11:22:18+08:00
hero: /images/posts/golang_banner.jpg
menu:
  sidebar:
    name: Golang逃逸分析
    identifier: golang-escape-analysis
    parent: golang odyssey
    weight: 10
draft: false
---

***问题： golang函数传参是不是应该和c一样，尽量不要直接传结构体，而是要传结构体指针？***

## 逃逸分析

逃逸分析指的是，在计算机语言编译器优化原理中，分析指针动态范围的方法，和编译器优化原理的指针分析和外形分析相关联。当变量（或者对象）在方法中被分配后，其指针有可能被返回或者被全局引用，这种现象就是指针（或引用）的逃逸（Escape）。

其实在java概念中有一个误解 --- new出来的东西都在堆上，栈上存的是它的引用。 这句话在现代JVM上有问题，就是因为逃逸分析机制。简单来说，就是JVM的逃逸分析会在运行时(runtime)检测当前方法栈帧(frame)内new出来的对象的引用，是否被传出当前的栈帧。如果传出，就会发生逃逸，没有传出则不会。对于未发生逃逸的变量，则会直接在栈上分配内存。因为栈上内存由在函数返回时自动回收，而堆上的的内存需要gc去回收，如果程序中有大量逃逸的对象，那么势必会增加gc的压力。

```java
public void test(){
  List<Integer> a = new ArrayList<>();
  a.add(1); // a 未逃逸，在栈上分配
}

public List<Integer> test1(){
  List<Integer> a = new ArrayList<>();
  a.add(1);
  return a // 发生逃逸，因此分配在堆上
}
```



## 区别

- 不同于JVM运行时的逃逸分析，Golang的逃逸分析是在编译期完成。
- golang的逃逸分析只针对指针。一个值引用变量如果没有被取址，那么它永远不可能逃逸。

```
go version go1.13.4 darwin/amd64
```

验证某个函数的变量是否发生逃逸的方法：

- go run -gcflags "-m -l" (-m打印逃逸分析信息，-l禁止内联编译)

- go tool compile -S xxxx.go | grep runtime.newobject（汇编代码中搜newobject指令，这个指令用于生成堆对象）



备注： 关于-gcflags "-m -l"的输出，有两种情况：

- Moved to heap: xxx
- xxx escapes to heap

二者都表示发生了逃逸，当xxx变量为指针的时候，出现第二种；当xxx变量为值类型时，为上一种，测试代码：

```go
type S int
func main(){
  a := S(0)
  b := make([]*S, 2)
  b[0] = &a
  c := new(S)
  b[1] = c
}
```



## Golang逃逸分析

本文探究什么时候，什么情况下会发生逃逸

### case 1

最基本的情况

```
在某个函数中new或者字面量创建出的变量，将其指针作为函数返回值，则该变量一定发生逃逸
```

下面是例子:

```go
func test() *User{
  a := User{}
  return &a
}
```



### case 2

需要验证文章开头情况的正确性，也就是当某个值取指针并传给另一个函数的时候，是否有逃逸：

```go
type User struct{
  Username string
  Password string
  Age	int
}

func main(){
  a := "aaa"
  u := &User{a, "123", 12}
  Call1(u)
}

func Call1(u *User){
  fmt.Printf("%v", u)
}
```

逃逸情况:

```
-> go run -gcflags "-m -l" main.go
# command-line-arguments
./main.go:18:12: leaking param: u
./main.go:19:12: Call1... argument does bnot escape
./main.go:19:13 u escapes to heap
./main.go:14:23 &User literal escapes to heap
```

可见发生了逃逸，这里将指针传给一个函数并打印，如果不打印，只对u进行读写：

```go
func Call1(u *User) int{
  u.Username = "bbb"
  return u.Age * 20
}
```

结果:

```
-> go run -gcflags "-m -l" main.go
# command-line-arguments
./main.go:19:12: Call1 u does not escape
./main.go:14:23 main &User literal does not escape
```

并没有发生逃逸。其实如果只是对u进行读写，不管调用几次函数，传了几次指针，都不会逃逸。所以我们可以换衣fmt.Printf的源码，可以发现传入的u被赋值给了pp指针的一个成员变量

```go
// Printf formats according to a format specifier and writes to standard output.
// It returns the number of bytes written and any write error encountered.
func Printf(format string, a ...interface{}) (n int, err error) {
	return Fprintf(os.Stdout, format, a...)
}

// Fprintf formats according to a format specifier and writes to w.
// It returns the number of bytes written and any write error encountered.
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) {
	p := newPrinter()
	p.doPrintf(format, a)
	n, err = w.Write(p.buf)
	p.free()
	return
}

// doPrintf里有
// ....
p.printArg(a[argNum], rune(c))
// ....

func (p *pp) printArg(arg interface{}, verb rune) {
	p.arg = arg
	p.value = reflect.Value{}
  // ....
}
```

这个pp类型的指针p是由构造函数newPrinter返回，根据case1，p一定会发生逃逸，而p引用了传入指针，所以我们可以总结：

```
被已经逃逸的变量引用的指针，一定发生逃逸。
```



### case3

上述备注代码的例子：

```go
func main(){
  a  := make([]*int, 1)
  b := 12
  a[0] = &b
}
```

实际上这个代码中, slice a不会逃逸，而被a引用的b会逃逸。类似的情况会发生在map和chan之中

```go
func main(){
  a := make([]*int, 1)
  b := 12
  a[0] = &b
  
  c := make(map[string]*int)
  d := 14
  c["aaa"] = &d
  
  e := make(chan *int, 1)
  f := 15
  e <- &f
}
```

结果可以发现, b, d, f都逃逸了。所以我们可以得出结论：

```
被指针类型的slice, map和chan引用的指针一定会发生逃逸。
备注： stack overflow上有人提问为何使用指针的chan比使用值得chan慢%30， 答案就在这里。使用指针的chan发生逃逸，gc拖慢了速度。
```



## 总结

我们得出指针**必然逃逸**的情况：

- 在某个函数中new或者字面量创建出的变量，将其指针作为函数返回，则该变量一定发生逃逸（构造函数返回的指针变量一定逃逸）
- 被已经逃逸的变量引用的指针，一定发生逃逸
- 被指针类型slice, map和chan引用的指针，一定发生逃逸

同时我们也得出一些**必然不会逃逸**的情况：

- 指针被未发生逃逸的变量引用
- 仅仅在函数内对变量做取址操作，而未将指针传出

有些情况**可能发生逃逸，也可能不会发生逃逸** ：

- 将指针作为入参传给别的函数，这里还是要看指针在被传入的函数中的处理过程，如果发生了上述三种情况，则会逃逸；否则不会发生逃逸。

***因此，对于文章开头的问题，我们不能仅仅依据使用值引用作为函数入参可能因为copy导致额外内存开销而放弃这种值引用类型入参的写法。因为如果函数内有造成变量逃逸的操作情形，gc可能会称为程序效率不高的瓶颈。***