---
date: 2017-07-20T12:58:00Z
description: ""
disqus_identifier: "20170720001"
slug: "golang-mian-shi-ti-da-an-yu=jie-xi"
tags:
- 面试题
- Go
title: Go面试题答案与解析
topics:
- 编程语言与开发
---

昨天AstaXie发布[GoCN每日新闻(2017-07-19)](https://gocn.io/question/947)含一篇[Go面试题](https://zhuanlan.zhihu.com/p/26972862)。阅读和评论量挺高，是测试面试者对Go本身基础概念理解掌握程度，以及Go实战经验。这也是在Go中容易遇到的坑，我也曾遇到过。于是快马加鞭，抢在原作者前发布Go面试题答案和解析说明，供大家参考。如有错误请指出，谢谢。

###  1、写出下面代码输出内容。
```go
package main

import (
	"fmt"
)

func main() {
	defer_call()
}

func defer_call() {
	defer func() { fmt.Println("打印前") }()
	defer func() { fmt.Println("打印中") }()
	defer func() { fmt.Println("打印后") }()

	panic("触发异常")
}     
```
[在线运行](https://wide.b3log.org/playground/309ba15c7e2cde23076852f21838ac0f.go)

**答：**
输出内容为：
```shell
打印后
打印中
打印前
panic: 触发异常
```
**解析：**

考察对defer的理解，**defer**函数属延迟执行，延迟到调用者函数执行 return 命令前被执行。多个defer之间按LIFO先进后出顺序执行。

故考题中，在Panic触发时结束函数运行，在return前先依次打印:打印后、打印中、打印前 。最后由runtime运行时抛出打印panic异常信息。

需要注意的是，函数的`return value` 不是原子操作.而是在编译器中分解为两部分：返回值赋值 和 return 。而defer刚好被插入到末尾的return前执行。故可以在derfer函数中修改返回值。如下示例：
```go
package main

import (
	"fmt"
)

func main() {
	fmt.Println(doubleScore(0))    //0
	fmt.Println(doubleScore(20.0)) //40
	fmt.Println(doubleScore(50.0)) //50
}
func doubleScore(source float32) (score float32) {
	defer func() {
		if score < 1 || score >= 100 {
			//将影响返回值
			score = source
		}
	}()
	score = source * 2
	return

	//或者
	//return source * 2
}
```
[在线运行](https://wide.b3log.org/playground/fd3defcc67256fefc60c63c135c5b64f.go)

该实例可以在defer中修改返回值score的值。具体参见[官方文档](https://golang.org/doc/effective_go.html#defer)


###  2、以下代码有什么问题，说明原因
```go
package main

import (
	"fmt"
)

type student struct {
	Name string
	Age  int
}

func pase_student() map[string]*student {
	m := make(map[string]*student)
	stus := []student{
		{Name: "zhou", Age: 24},
		{Name: "li", Age: 23},
		{Name: "wang", Age: 22},
	}
	for _, stu := range stus {
		m[stu.Name] = &stu
	}
	return m
}
func main() {
	students := pase_student()
	for k, v := range students {
		fmt.Printf("key=%s,value=%v \n", k, v)
	}
}

```
[在线运行](https://wide.b3log.org/playground/362fde0b97440e9354efe63eefb3a1ae.go)

**答：**输出的均是相同的值：&{wang 22} 

**解析**
因为for遍历时，变量`stu`指针不变，每次遍历仅进行struct值拷贝，故`m[stu.Name]=&stu`实际上一致指向同一个指针，最终该指针的值为遍历的最后一个struct的值拷贝。形同如下代码：
```go
var stu student 
for _, stu = range stus {
	m[stu.Name] = &stu
} 
```
修正方案，取数组中原始值的指针：
```go
for i, _ := range stus {
	stu:=stus[i]
	m[stu.Name] = &stu
}
```
 

### 3、下面的代码会输出什么，并说明原因
```go
func main() {
	runtime.GOMAXPROCS(1)
	wg := sync.WaitGroup{}
	wg.Add(20)
	for i := 0; i < 10; i++ {
		go func() {
			fmt.Println("i: ", i)
			wg.Done()
		}()
	}
	for i := 0; i < 10; i++ {
		go func(i int) {
			fmt.Println("i: ", i)
			wg.Done()
		}(i)
	}
	wg.Wait()
}
```
[在线运行](https://wide.b3log.org/playground/0bc5162cf88188d75a2e59ef537085d0.go)

**答：**
将随机输出数字，但前面一个循环中并不会输出所有值。

**解析：**

实际上第一行是否设置CPU为1都不会影响后续代码。两个for循环内部go func 调用参数`i`的方式是不同的，导致结果完全不同。这也是新手容易遇到的坑。

第一个go func中`i`是外部for的一个变量，地址不变化。遍历完成后，最终i=10。故go func执行时，`i`的值始终是`10`（10次遍历很快完成）。

第二个go func中`i`是函数参数，与外部for中的`i`完全是两个变量。尾部`(i)`将发生值拷贝，go func内部指向值拷贝地址。
 
### 4、下面代码会输出什么？
```go
type People struct{}

func (p *People) ShowA() {
	fmt.Println("showA")
	p.ShowB()
}
func (p *People) ShowB() {
	fmt.Println("showB")
}

type Teacher struct {
	People
}

func (t *Teacher) ShowB() {
	fmt.Println("teacher showB")
}

func main() {
	t := Teacher{}
	t.ShowA()
}
```
[在线运行](https://wide.b3log.org/playground/f5949e74a9f2fb40b0003cd690ffdb10.go)

**答：**
将输出：
```shell
showA
showB
```
**解析**

Go中**没有继承！** 没有继承！没有继承！是叫**组合**！组合！组合！

这里People是匿名组合People。被组合的类型People所包含的方法虽然升级成了外部类型Teacher这个组合类型的方法，但他们的方法(`ShowA()`)调用时接受者并没有发生变化。

这里仍然是People。毕竟这个People类型并不知道自己会被什么类型组合，当然也就无法调用方法时去使用未知的组合者Teacher类型的功能。 

因此这里执行t.ShowA()时，在执行ShowB()时该函数的接受者是People，而非Teacher。[具体参见官方文档](https://golang.org/doc/effective_go.html#Embedding)

### 5、下面代码会触发异常吗？请详细说明
```go
func main() {
	runtime.GOMAXPROCS(1)
	int_chan := make(chan int, 1)
	string_chan := make(chan string, 1)
	int_chan <- 1
	string_chan <- "hello"
	select {
	case value := <-int_chan:
		fmt.Println(value)
	case value := <-string_chan:
		panic(value)
	}
}
```
[在线运行](https://wide.b3log.org/playground/4bc46159a2b1b499bf496ec1e5b174d9.go)

**答：** 
有可能触发异常，是随机事件。

**解析**

单个chan如果无缓冲时，将会阻塞。但结合 `select`可以在多个chan间等待执行。有三点原则：

+ select 中只要有一个case能return，则立刻执行。
+ 当如果同一时间有多个case均能return则伪随机方式抽取任意一个执行。
+ 如果没有一个case能return则可以执行"default"块。

此考题中的两个case中的两个chan均能return，则会随机执行某个case块。故在执行程序时，有可能执行第二个case，触发异常。[具体参见官方文档](https://golang.org/ref/spec#Select_statements)

### 6、下面代码输出什么？
```go
func calc(index string, a, b int) int {
	ret := a + b
	fmt.Println(index, a, b, ret)
	return ret
}

func main() {
	a := 1                                             //line 1
	b := 2                                             //2
	defer calc("1", a, calc("10", a, b))  //3
	a = 0                                              //4
	defer calc("2", a, calc("20", a, b))  //5
	b = 1                                              //6
}
```
[在线运行](https://wide.b3log.org/playground/b234243ee983929ae51ba77874a4f3d6.go)

**答** 
输出结果为：
```shell
10 1 2 3
20 0 2 2
2 0 2 2
1 1 3 4
```
**解析**

在解题前需要明确两个概念：
+ defer是在函数末尾的return前执行，先进后执行，具体见问题1。
+ 函数调用时 int 参数发生值拷贝。

不管代码顺序如何，defer calc func中参数b必须先计算，故会在运行到第三行时，执行`calc("10",a,b)`输出：`10 1 2 3`得到值`3`，将`cal("1",1,3)`存放到延后执执行函数队列中。

执行到第五行时，现行计算`calc("20", a, b)`即`calc("20", 0, 2)`输出：`20 0 2 2`得到值`2`,将`cal("2",0,2)`存放到延后执行函数队列中。

执行到末尾行，按队列先进后出原则依次执行：`cal("2",0,2)`、`cal("1",1,3)` ，依次输出：`2 0 2 2`、`1 1 3 4` 。


### 7、请写出以下输入内容
```go
func main() {
	s := make([]int, 5)
	s = append(s, 1, 2, 3)
	fmt.Println(s)
}
```
[在线运行](https://wide.b3log.org/playground/ef2548849a64ecae5475612cdbcca59e.go)

**答：**
将输出：[0 0 0 0 0 1 2 3]

**解析**

make可用于初始化数组，第二个可选参数表示数组的长度。数组是不可变的。

当执行`make([]int,5)`时返回的是一个含义默认值(int的默认值为0)的数组:`[0,0,0,0,0]`。而append函数是便是在一个数组或slice后面追加新的元素，并返回一个新的数组或slice。

这里`append(s,1,2,3)`是在数组s的继承上追加三个新元素:1、2、3，故返回的新数组为`[0 0 0 0 0 1 2 3]`
  
 
### 8、下面的代码有什么问题?
```go
type UserAges struct {
	ages map[string]int
	sync.Mutex
}

func (ua *UserAges) Add(name string, age int) {
	ua.Lock()
	defer ua.Unlock()
	ua.ages[name] = age
}

func (ua *UserAges) Get(name string) int {
	if age, ok := ua.ages[name]; ok {
		return age
	}
	return -1
}
```
[在线运行](https://wide.b3log.org/playground/ec2c79ef867858648b0adb1dedeb4e2b.go)

**答：** 
在执行 Get方法时可能被panic

**解析**

虽然有使用`sync.Mutex`做写锁，但是`map`是并发读写不安全的。map属于引用类型，并发读写时多个协程见是通过指针访问同一个地址，即访问共享变量，此时同时读写资源存在竞争关系。会报错误信息:"fatal error: concurrent map read and map write"。

可以在在线运行中执行，复现该问题。那么如何改善呢? 当然Go1.9新版本中将提供并发安全的map。首先需要了解两种锁的不同：

1. `sync.Mutex`互斥锁
2. `sync.RWMutex`读写锁，基于互斥锁的实现，可以加多个读锁或者一个写锁。

利用读写锁可实现对map的安全访问，[在线运行改进版](https://wide.b3log.org/playground/b2c07ccddc52fc8d408f63ea37786754.go)
。利用RWutex进行读锁。
```go
type RWMutex
    func (rw *RWMutex) Lock() 
    func (rw *RWMutex) RLock()
    func (rw *RWMutex) RLocker() Locker
    func (rw *RWMutex) RUnlock()
    func (rw *RWMutex) Unlock()
```

### 9、下面的迭代会有什么问题？
```go
func (set *threadSafeSet) Iter() <-chan interface{} {
	ch := make(chan interface{})
	go func() {
		set.RLock()

		for elem := range set.s {
			ch <- elem
		}

		close(ch)
		set.RUnlock()

	}()
	return ch
}
```
[在线运行](https://wide.b3log.org/playground/08a39230e163cb8c9ecd0073e4f6e93d.go)

**答：** 内部迭代出现阻塞。ch 初始化的容量默认为1，如果外部不及时读取，则内部迭代是阻塞状态。
**解析**
chan在使用make初始化时可附带一个可选参数来设置缓冲区。默认无缓冲，题目中便初始化的是无缓冲区的chan，这样只有写入的元素直到被读取后才能继续写入，不然就一直阻塞。

设置缓冲区大小后，写入数据时可连续写入到缓冲区中，直到缓冲区被占满。从chan中接收一次便可从缓冲区中释放一次。可以理解为chan是可以设置吞吐量的处理池。


###  10、以下代码能编译过去吗？为什么？
```go
package main

import (
	"fmt"
)

type People interface {
	Speak(string) string
}

type Stduent struct{}

func (stu *Stduent) Speak(think string) (talk string) {
	if think == "bitch" {
		talk = "You are a good boy"
	} else {
		talk = "hi"
	}
	return
}

func main() {
	var peo People = Stduent{}
	think := "bitch"
	fmt.Println(peo.Speak(think))
}
```
[在线运行](https://wide.b3log.org/playground/adf1dc3339b87b8f1d1a49f2f4ebb2d7.go)

**答：** 编译失败，值类型 Student{} 未实现接口People的方法，不能定义为 People类型。

**解析**

考题中的 `func (stu *Stduent) Speak(think string) (talk string)` 是表示结构类型`*Student`的指针有提供该方法，但该方法并不属于结构类型`Student`的方法。因为struct是值类型。

修改方法：

+ 定义为指针
```go
var peo People = &Stduent{}
```
+ 方法定义在值类型上,指针类型本身是包含值类型的方法。
```go
func (stu Stduent) Speak(think string) (talk string) { 
	//...
}
```

--------------------------完--------------------------------------

想评论? 请到：[GoCN社区](https://gocn.io/article/395)
