# 理解Go内存分配

![image-20210722164024715](https://tva1.sinaimg.cn/large/008i3skNly1gsptgo8g1xj31ka0t2gwe.jpg)

如果你也是个go开发者，你是否关心过内存的分配和回收呢？创建的对象究竟需要由GC进行回收，还是随着调用栈被弹出，就消失了呢？GC导致的Stop The World是否导致了你程序的性能抖动呢？

本文简单章介绍了Go的内存分配逻辑，介绍了Go内存划分模型。并以代码为例子，简要介绍了几种场景下Go内存分配逻辑。随后使用go build命令验证了内存逃逸等相关验证。最后，总结出了内存分配原则。

## 需要关心内存分配问题吗？

每个工程师的时间都如此宝贵，在继续读这篇文章之前，需要你先回答几个问题，如果得到的答案是否定的，那可能本文章里写的内容对你并没有什么帮助。但是，如果你遇到了因内存分配而导致的性能问题，可能这个文章能带你理解Golang的内存分配的冰山一角，带你入个门。

问题如下：

* 你的程序是性能敏感型吗？
* GC带来的延迟影响到了你的程序性能吗？
* 你的程序有过多的堆内存分配吗？

如果你命中上面问题的其中一个或两个，那这篇文章适合你继续读下去。或你根本不知道如何回答这些问题，可能去了解下go性能观测相关的知识（pprof的使用等）对你更有帮助。

下面正文开始。

## Golang简要内存划分

![image-20210721174349861](https://tva1.sinaimg.cn/large/008i3skNly1gsptcbl1kfj30ee0ai3yr.jpg)

可以简单的认为Golang程序在启动时，会向操作系统申请一定区域的内存，分为栈（Stack）和堆（Heap）。栈内存会随着函数的调用分配和回收；堆内存由程序申请分配，由垃圾回收器（Garbage Collector）负责回收。性能上，栈内存的使用和回收更迅速一些；尽管Golang的GC很高效，但也不可避免的会带来一些性能损耗。因此，Go优先使用栈内存进行内存分配。在不得不将对象分配到堆上时，才将特定的对象放到堆中。

## 内存分配过程分析

本部分，将以代码的形式，分别介绍栈内存分配、指针作为参数情况下的栈内存分配、指针作为返回值情况下的栈内存分配并逐步引出逃逸分析和几个内存逃逸的基本原则。

正文开始，Talk is cheap，show me the code。

### 栈内存分配

我将以一段简单的代码作为示例，分析这段代码的内存分配过程。

```go
package main

import "fmt"

func main() {
	n := 4
	n2 := square(n)
	fmt.Println(n2)
}

func square(n int) int{
	return n * n
}

```

代码的功能很简单，一个`main`函数作为程序入口，定义了一个变量n，定义了另一个函数`squire`，返回乘方操作后的int值。最后，将返回的值打印到控制台。程序输出为16。

下面开始逐行进行分析，解析调用时，go运行时是如何对内存进行分配的。

![image-20210722144210080](https://tva1.sinaimg.cn/large/008i3skNly1gsptd659ooj31060j4dgw.jpg)

当代码运行到第6行，进入main函数时，会在栈上创建一个Stack frame，存放本函数中的变量信息。包括函数名称，变量等。

![image-20210722144154825](https://tva1.sinaimg.cn/large/008i3skNly1gsptd8u0mxj310q0i6dh3.jpg)

当代码运行到第7行时，go会在栈中压入一个新的Stack Frame，用于存放调用square函数的信息；包括函数名、变量n的值等。此时，计算4 * 4 的值，并返回。

![image-20210722144139751](https://tva1.sinaimg.cn/large/008i3skNly1gsptdbqdifj31440iqta5.jpg)

当square函数调用完成，返回16到main函数后，将16赋值给n2变量。注意，原来的stack frame并不会被go清理掉，而是如栈左侧的箭头所示，被标记为不合法。上图夹在红色箭头和绿色箭头之间的横线可以理解为go汇编代码中的SP栈寄存器的值，当程序申请或释放栈内存时，只需要修改SP寄存器的值，这种栈内存分配方式省掉了清理栈内存空间的耗时【1】。

![image-20210722144126127](https://tva1.sinaimg.cn/large/008i3skNly1gsptdeypxoj314y0j275q.jpg)

接下来，调用fmt.Println时，SP寄存器的值会进一步增加，覆盖掉原来square函数的stack frame，完成print后，程序正常退出。

### 指针作为参数情况下的栈内存分配

还是同样的过程，看如下这段代码。

```go
package main

import "fmt"

func main() {
	n := 4
	increase(&n)
	fmt.Println(n)
}

func increase(i *int) {
	*i++
}
```

main作为程序入口，声明了一个变量n，赋值为4。声明了一个函数increase，使用一个int类型的指针i作为参数，increase函数内，对指针i对应的值进行自增操作。最后main函数中打印了n的值。程序输出为5。

![image-20210722145620858](https://tva1.sinaimg.cn/large/008i3skNly1gsptdioby6j30zg0im0tq.jpg)

当程序运行到main函数的第6行时，go在栈上分配了一个stack frame，对变量n进行了赋值，n在内存中对应的地址为`0xc0008771`，此时程序将继续向下执行，调用increase函数。

![image-20210722150218994](https://tva1.sinaimg.cn/large/008i3skNly1gsptdvs3jxj310o0iadh0.jpg)

这时，increase函数对应的stack fream被创建，i被赋值为变量n对应的地址值0xc0008771，然后进行自增操作。

![image-20210722150535133](https://tva1.sinaimg.cn/large/008i3skNly1gsptdyoo0uj61120iu0u202.jpg)

当increase函数运行结束后，SP寄存器会上移，将之前分配的stack freme标记为不合法。此时，程序运行正常，并没有因为SP寄存器的改动而影响程序的正确性，内存中的值也被正确的修改了。

### 指针作为返回值情况下的栈内存分配

文章之前的部分分别介绍了普通变量作为参数和将指针作为参数情况下的栈内存使用，本部分来介绍将指针作为返回值，返回给调用方的情况下，内存是如何分配的，并引出内存逃逸相关内容。来看这段代码：

```go
package main

import "fmt"

func main() {
	n := initValue()
	fmt.Println(*n/2)
}

func initValue() *int {
	i := 4
	return &i
}
```

main函数中，调用了initValue函数，该函数返回一个int指针并赋值给n，指针对应的值为4。随后，main函数调用fmt.Println打印了指针n / 2对应的值。程序输出为2。

![image-20210722153419781](https://tva1.sinaimg.cn/large/008i3skNly1gspte1htktj30zq0h8q49.jpg)

程序调用initValue后，将i的地址赋值给变量n。注意，如果这时，变量i的位置在栈上，则可能会随时被覆盖掉。

![image-20210722153007366](https://tva1.sinaimg.cn/large/008i3skNly1gspte3tln9j312c0jcmyu.jpg)

在调用fmt.Println时，Stack Frame会被重新创建，变量i被赋值为*n/2也就是2，会覆盖掉原来n所指向的变量值。这会导致及其严重的问题。在面对sharing up场景时，go通常会将变量分配到堆中，如下图所示：

![image-20210722153708192](https://tva1.sinaimg.cn/large/008i3skNly1gspte6ph0oj317g0i0abn.jpg)

通过上面的分析，可以看到在面对被调用的函数返回一个指针类型时将对象分配到栈上会带来严重的问题，因此Go将变量分配到了堆上。这种分配方式保证了程序的安全性，单也不可避免的增加了堆内存创建，并需要在将来的某个时候，需要GC将不再使用的内存清理掉。

## 内存分配原则

经过上述分析，可以简单的归纳几条原则。

* Sharing down typically stays on the stack

  在调用方创建的变量或对象，通过参数的形式传递给被调用函数，这时，在调用方创建的内存空间**通常**在栈上。这种在调用方创建内存，在被调用方使用该内存的”内存共享“方式，称之为Sharing down。

* Sharing up typically escapes to the heap

  在被调用函数内创建的对象，以指针的形式返回给调用方的情况下，**通常**，创建的内存空间在堆上。这种在被调用方创建，在调用方使用的”内存共享“方式，称之为Sharing up。

* Only the compiler knows

  之所以上面两条原则都加了**通常**，因为具体的分配方式，是由编译器确定的，一些编译器后端优化，可能会突破这两个原则，因此，具体的分配逻辑，只有编译器（或开发编译器的人）知道。

## 使用go build命令确定内存逃逸情况

值得注意的是，Go在判断一个变量或对象是否需要逃逸到堆的操作，是在编译器完成的；也就是说，当代码写好后，经过编译器编译后，会在二进制中进行特定的标注，声明指定的变量要被分配到堆或栈。可以使用如下命令在编译期打印出内存分配逻辑，来具体获知特定变量或对象的内存分配位置。

查看go help可以看到go build其实是在调用go tool compile。

```bash
go help build 
... 
-gcflags '[pattern=]arg list'
        arguments to pass on each go tool compile invocation.
... 
```

```bash
go tool compile -h
...
-m    print optimization decisions
...
-l    disable inlining
...
```

其中，需要关心的参数有两个，

* -m	显示优化决策
* -l	禁止使用内联【2】

代码如下：

```go
package main

func main() {
	n := initValue()
	println(*n / 2)

	o := initObj()
	println(o)

	f := initFn()
	println(f)

	num := 5
	result := add(num)
	println(result)
}

func initValue() *int {
	i := 3								// ./main.go:19:2: moved to heap: i
	return &i
}

type Obj struct {
	i int
}

func initObj() *Obj {
	return &Obj{i: 3}			// ./main.go:28:9: &Obj literal escapes to heap
}

func initFn() func() {
	return func() {       // ./main.go:32:9: func literal escapes to heap
		println("I am a function")
	}
}

func add(i int) int {
	return i + 1
}

```

完整的构建命令和输出如下：

```bash
go build -gcflags="-m -l" 

# _/Users/rocket/workspace/stack-or-heap
./main.go:19:2: moved to heap: i
./main.go:24:9: &Obj literal escapes to heap
./main.go:28:9: func literal escapes to heap

```

可以看到，sharing up的情况（initValue，initObj，initFn）内存空间被分配到了堆上。sharing down的情况（add）内存空间在栈上。

<u>这里给读者留个问题，大家可以研究下`moved to heap`和`escapes to heap`的区别。</u>

## 总结

1. 因为栈比堆更高效，不需要GC，因此Go会尽可能的将内存分配到栈上。

2. 当分配到栈上可能引起非法内存访问等问题后，会使用堆，主要场景有：
   1. 当一个值可能在函数被调用后访问，这个**值极有可能**被分配到堆上。
   2. 当编译器检测到某个值过大，这个值会被分配到堆上。
   3. 当编译时，编译器不知道这个值的大小（slice、map...）这个值会被分配到堆上。
3. Sharing down typically stays on the stack
4. Sharing up typically escapes to the heap
5. Don't guess, Only the compiler knows

## 参考文献

【1】Go语言设计与实现 https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-stack-management/#%E5%AF%84%E5%AD%98%E5%99%A8

【2】Inlining optimisations in Go https://dave.cheney.net/2020/04/25/inlining-optimisations-in-go

【3】Golang FAQ https://golang.org/doc/faq#stack_or_heap
