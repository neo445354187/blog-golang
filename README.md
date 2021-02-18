# golang

参考资料：<https://studygolang.com/pkgdoc>
参考资料：<http://www.topgoer.com/>
参考资料：<https://gorm.io/zh_CN/docs/>
参考资料：<https://www.bilibili.com/video/BV1A7411x7S2?p=4>
## todo


## 遗忘特性
### 1. 在子协程中`return`，作用是退出子协程，无论子协程的代码是否在main()中
```
func main() {
	go func() {
		fmt.Println("子协程")
		return
		fmt.Println("子协程2")
	}()
	time.Sleep(time.Second*2)
	fmt.Println("主协程")
}
```

### 2. `defer`代码
1. `defer`代码需要被代码执行到，才能绑定在退出时执行，没有机会被执行到的`defer`代码，如果放在`if{}else{}`或者`return`后
的`defer`并不会在退出时被调用
2. `defer`执行时间是`return xxx`将返回值赋值给返回变量后，`return`返回前
例子
```
func test1() (i int) {
	defer func() {
		fmt.Println(i)
	}()
	return 100
}
```
备注：`defer`执行在100赋值给变量`i`后
3. `defer xxx(yy)`声明，一般后面接执行函数，即使匿名函数也是执行的，不会只是声明函数；此时，如果有变量`yy`作为实参，
    那么会以 **声明位置 此刻 的变量值**赋值给`defer`后的最外层函数或者方法的参数（即复制，根据形参或实参的类型，
    真复制和假复制都可以），此后无论`yy`如何变化，`defer`后的执行函数或方法中获取到的这个实参是不变的；
    即相当于形参被赋值后，声明结束，等待被执行
    但是除了上面所说的形参被赋值外，`defer`后其他所有用到的变量或者函数接收者变量，都是`defer`真正执行时的变量值
例子：
```go
package main

import "fmt"

type Test struct {
	name string
}

func (t *Test) Close(i int) {
	fmt.Println(t.name, " closed", "-------", i)
}
func main() {
	ts := []Test{{"a"}, {"b"}, {"c"}}

	for i, t := range ts {
		// i变量作为实参传入，无论后面变量i如何变化，那个`defer`后的实参值不变
		// 变量t是实际执行时才赋值的
		defer t.Close(i)
	}
}
```

### 3. EOF理解以及文件上传事项
EOF：是不存在的，只是底层函数通过已获取的字节和已知文件大小做对比，判断文件是否获取完整，如果大小相等，给返回值为-1，-1即为EOF的值
文件上传注意：本地会得到EOF的反馈，但是服务端不会，所以，上传文件时，需要一个协议，用现成的或自己实现一种都可以。
客户端根据需要传输的文件，构造并发送一个数据包（包含文件名，文件大小等），而服务端读取包头便知文件大小，继而进行文件末尾的判断

### 4. json解压码
`json.Unmarshal(info,&res)`
备注：解码是注意第二个参数是引用
备注：压缩和解压的格式需要一致，压缩的数据是struct，那么解压也是struct，压缩是map，则解压也是map

### 5. SOCKET：TCP/IP网络的API。
Socket是应用层与TCP/IP协议族通信的中间软件抽象层，它是一组接口。在设计模式中，Socket其实就是一个门面模式，
它把复杂的TCP/IP协议族隐藏在Socket接口后面，对用户来说，一组简单的接口就是全部，让Socket去组织数据，以符合指定的协议。

Socket 接口是TCP/IP网络的API，Socket接口定义了许多函数或例程，用以开发TCP/IP网络上的应用程序。

这是为了实现以上的通信过程而建立成来的通信管道，其真实的代表是客户端和服务器端的一个通信进程，
双方进程通过socket进行通信，而通信的规则
采用指定的协议。socket只是一种连接模式，不是协议,tcp,udp，简单的说（虽然不准确）是两个最基本的协议,
很多其它协议都是基于这两个协议如

### 6. var,new,make
a. var declaration: 对于 基本类型 申请空间并初始化，对于map, slice, chan 只是声明，值为nil
`var var1 = xxx` ：声明申请并初始化空间。这个是自定义值初始化，而上一行是默认值初始化
b. new : 申请空间并初始化，和declaration 相同，只不过返回的是指针。扩展：结构体初始化： os.NewFile(...), xxx.New(...), xxx.NewYYY(...)
c. make: 用于slice, map, chan的创建
d. type：声明接口或者struct
备注：如果没有等号和等号右边内容，则一般初始化该类型的默认值，0 空字符串 nil等
例子
```golang
var firt []int
var second = new(int)       //括号里只接收类型，不能自定义初始化
var third chan []byte       //没有空间的管道，只能同步
var fouth = xxx             //可以接收一切，但是具体值，得看右边
fifth := xxx                //可以接收一切，但是具体值，得看右边
var ch chan int             //注意这种管道声明，管道不能被赋值，还需要用make()初始化
```
等号右边建议：`map, slice, chan`采用make，其余正常操作

### 7. select case
a. 有`default`，`select case`中执行`case`条件满足的分支，多个满足，就随机执行其中一个，否则执行`default`
b. 没有`default`，执行满足`case`条件的分支，没有满足条件的，就阻塞，知道满足其中一个`case`为止。多个`case`满足，随机
    执行器中一个
例子：多个满足条件
```go
package main

import (
	"fmt"
	"time"
)

var ch1 = make(chan int)
var ch2 = make(chan int)

func main() {
	go func() {
		ch1 <- 1
	}()

	go func() {
		ch2 <- 1
	}()

	time.Sleep(time.Second)
	select {
	case <-ch1:
		fmt.Println("1th case is selected.")		//被随机执行之一
	case <-ch2:
		fmt.Println("2th case is selected.")		//被随机执行之一
	case <-time.After(time.Second * 10):
		fmt.Println("After!.")
	default:
		fmt.Println("default!.")

	}
}
```
例子：定时器阻塞
```
package main

import (
	"fmt"
	"time"
)

var ch1 = make(chan int)
var ch2 = make(chan int)

func main() {
	go func() {
		time.Sleep(time.Second * 15)
		ch1 <- 1	//15s后往通道给值
	}()

	go func() {
		time.Sleep(time.Second * 11)
		ch2 <- 1	//11s后往通道给值
	}()

	for {
		select {//阻塞5s，等待<-time.After(time.Second * 5)后返回相应的chan，执行该分支；注意阻塞位置在select而不是具体case；
		case <-ch1:
			fmt.Println("1th case is selected.")
		case <-ch2:
			fmt.Println("2th case is selected.")
		case <-time.After(time.Second * 5):		 
			fmt.Println("After!.")
		}
	}
}

```
### 8. 全局变量和局部变量
a. 小范围可以使用其外部的大范围变量，直接使用，而不是复制哦，即小范围更改了大范围的变量，大范围也会更改，协程也适用，
    即协程内使用了协程外的变量，协程内更改了变量，协程外的该变量也会被更改；猜测：原理应该是协程并不单独维护内存
b. 全局变量定义后不用，不会报错，局部变量定义后不用，会报错
示意图1
```
{
    {
        go func(){
            
        }()
    }
}
```
总结：示意图1这种，全局变量 > 大范围局部变量 > 下范围局部变量(包含协程)，只维护了一份变量；
示意图2：
```
{
    {
        var a xxx
    }
    {
        var b xxx
    }
}
```
总结：示意图2这种，局部a和局部b才维护了多份变量，即不相关的局部变量也可以重名
参考资料：<https://www.jianshu.com/p/78f10bdbac73>
例子
```go
package main

import (
	"fmt"
	"time"
)

//全局变量定义后不用，不会报错，局部变量定义后不用，会报错
var globle int

func main() {
	var aa = 1

	{
		var bb = 10
		{
			bb = 20
			fmt.Println("in-in bb：", bb)
		}
		aa = 20
		fmt.Println("in aa,bb：", aa, bb)
	}
	go func() {
		aa = 50
		fmt.Println("Goruntine aa：", aa)
	}()
	time.Sleep(time.Second)
	fmt.Println("main aa：", aa) //这里不能获取局部变量bb

	//这里的i相当于在for{}的括号内被声明和赋值
	sum := 0
	for i := 0; i < 5; i++ {
		sum += i
	}

	//fmt.Println(i)	//不能获取到局部变量i
	fmt.Println("main sum：", sum)
}

```

### 9. 利用管道防止子协程在没有执行完成时，主协程执行完后，子协程被强制关闭
说明：这里的主协程可以理解为父协程，即父协程产生子协程的关系
```
package main

import (
	"fmt"
)

func main() {
	//声明管道，主要用于阻塞
	var ch = make(chan bool)
	for i := 0; i < 5; i++ {
		go func(i int) {
			fmt.Println("goruntine ", i)
			//在子协程执行完成后，再给管道赋值
			ch <- true
		}(i)
	}

	for i := 0; i < 5; i++ {
		//主协程一直阻塞
		<-ch
	}
}

```
### 10 包名 文件夹名 文件名，package名理解
猜测：文件夹可以理解为包，包名在文件夹下的文件里的package声明，文件名不重要；以下操作都可以执行
文件位置在`/example01/a/b/lydia.go`
```
package lyd     //只需要保证在同一个文件夹下，package一致即可；文件夹上下级可以为不同包，不然就没法完了；

func Say() int {
	return 11
}__
```
使用包
```
import	sss "example01/a/b"     给包起了sss的别名，sss.Say()可以调用
import	lyd "example01/a/b"     给包起了lyd的别名，lyd.Say()可以调用
import	"example01/a/b"         直接采用原始包名，lyd.Say()可以调用
```
### 11 fallthrough
强制跳过一个switch中case的break，执行下一个case或者default
```
package main

import "fmt"

func main() {
	a := 10
	switch a {
	case 10:
		fmt.Println("The integer was <= 4")
		fallthrough
	case 2:
		fmt.Println("The integer was <= 5")
		fallthrough
	default:
		fmt.Println("default case")
	}
}
```

### 12 函数也是一种数据类型
`type FuncType func(int, int) int`

### 13 `go build`和`go install`区别
`go install/build`
共同点：
- 都是用来编译包和其依赖的包的
- 一般生成静态库文件放在`$GOPATH/pkg`目录下
不同点：
- `go install`如果为main包，则会在`$GOPATH/bin` 生成一个可执行的二进制文件。也可以编译非main包
- `go build` 在当前目录编译生成一个可执行的二进制文件，只对main包有效，

### 14 静态库和动态库
两者区别：
a，静态库的使用需要：
1 包含一个对应的头文件告知编译器lib文件里面的具体内容
2 设置lib文件允许编译器去查找已经编译好的二进制代码
备注：静态用.a为后缀，如：libhello.a

b，动态库的使用：
程序运行时需要加载动态库，对动态库有依赖性，需要手动加入动态库
备注：动态通常用.so为后缀，如：libhello.so，windows动态库后缀为.dll

c，依赖性：
静态链接表示静态性，在编译链接之后， **lib库中需要的资源已经在可执行程序中了**， 也就是静态存在，没有依赖性了
动态，就是实时性，**在运行的时候载入需要的资源，那么必须在运行的时候提供 需要的 动态库，有依赖性， 
运行时候没有找到库就不能运行了**

d，区别：
简单讲，静态库就是直接将需要的代码连接进可执行程序；动态库就是在需要调用其中的函数时，
根据函数映射表找到该函数然后调入堆栈执行。
做成静态库可执行文件本身比较大，但不必附带动态库
做成动态库可执行文件本身比较小，但需要附带动态库
链接静态库，编译的可执行文件比较大，当然可以用strip命令精简一下（如：strip libtest.a），
但还是要比链接动态库的可执行文件大。程序运行时间速度稍微快一点。
静态库是程序运行的时候已经调入内存，不管有没有调用，都会在内存里头。静态库在程序编译时会被连接到目标代码中，
程序运行时将不再需要该静态库。
其在编译程序时若链接,程序运行时会在系统指定的路径下搜索，然后导入内存，程序一般执行时间稍微长一点，
但编译的可执行文件比较小；动态库是程序运行的时候需要调用的时候才装入内存，不需要的时候是不会装入内存的。
动态库在程序编译时并不会被连接到目标代码中，而是在程序运行是才被载入，因此在程序运行时还需要动态库存在。
参考：<https://zhidao.baidu.com/question/940565610931659332.html>
动态库编译参考：<http://reborncodinglife.com/2018/04/29/how-to-create-dynamic-lib-in-golang/>
静态库编译参考：<http://reborncodinglife.com/2018/04/27/how-to-create-static-lib-in-golang/>

### 15 for-range赋值理解
示例
```
for index,value :=range oldData {
}
```
流程：
1. oldData数据会for-range开始时刻，复制一份自身为newData，即for-range里无论怎么更改oldData数据，都保证迭代按原来进行，
    即用newData进行迭代。复制时根据自身类型，引用类型（如：切片，映射）就浅复制，值类型（数组）就真复制
2. index和value根据newData的键和值类型，都对应初始化自己的一块内存（整个过程没有改变过），
    用于迭代中储存newData键的值和newData值的值
3. 注意第2步中，index和value会根据newData键和值的类型，判断出是真复制还是浅复制，由于index一般都是标量值类型，所以一般
   真复制，而复制到value时根据newData子元素自身类型，引用类型（如：切片，映射）就浅复制，值类型（数组）就真复制
说明1：newData是隐形的，有维持迭代功能
说明2：这样明确的目的，就是搞清楚for-range过程中，改变其中的值，对于oldData的影响

例子
```go
package main

import (
	"fmt"
)

func main() {
	m := make(map[string]*[]string)
	stus := map[string][]string{
		"a": []string{"A", "B"},
		"b": []string{"X", "Y"},
	}
	for k, stu := range stus {
		stu[0] = "HAHA"
		m[k] = &stu
		fmt.Println(stu, stus[k])
	}
}

```
注意，range 会复制对象。
```go
package main

import "fmt"

func main() {
    a := [3]int{0, 1, 2}
    // 说明：for-range开始那一刻，已经注定了index,value在整个迭代过程中的值，即a有个复制品去填充index，value，
    // 无论迭代过程中a的值如何变化
    for i, v := range a { // index、value 都是从复制品中取出。
        if i == 0 { // 在修改前，我们先修改原数组。
            a[1], a[2] = 999, 999
            fmt.Println(a) // 确认修改有效，输出 [0, 999, 999]。
        }
        a[i] = v + 100 // 使用复制品中取出的 value 修改原数组。
    }
    fmt.Println(a) // 输出 [100, 101, 102]。
}
```
实例3
```go
package main

func main() {
	s := []int{1, 2, 3, 4, 5}
	for i, v := range s { // 复制 struct slice { pointer, len, cap }。
		if i == 0 {
			s = s[2:]  // 对 slice 的修改，不会影响 range。
			s[2] = 100 // 对底层数据的修改。
		}
		println(i, v)
	}
}
```

### 16 匿名结构体以及编译结构体操作
```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	//注意匿名结构体声明和初始化方式
	var user struct {
		Name string
		Age  int
	}
	user.Name = "pprof.cn"
	user.Age = 18
    // 利用reflect获取值
	value := reflect.ValueOf(user)
	for i := 0; i < value.NumField(); i++ {
		fmt.Printf("Field %d: %v\n", i, value.Field(i))
	}
}

```

### 17 继承结构和继承接口
说明1：**子结构体继承父结构体，建议初始化时，也将父结构初始化，以免不必要麻烦**
说明2：**结构体中不明确声明接口的方式，将错误留在编码阶段，更不容易出错，建议采用，
        当然可以多个结构体(一般为继承关系)共同实现接口所有方法**

参考：<https://blog.csdn.net/wanwan880818/article/details/80592467>
参考：<http://www.topgoer.com/%E6%96%B9%E6%B3%95/%E6%96%B9%E6%B3%95%E9%9B%86.html>

#### 结构继承结构
说明：结构继承结构相对随意，一般可以互相调用
1. 结构体`T`和对应的结构体指针`*T`的接收者方法，调用者无论是结构体还是结构体指针都可以互相调用
2. 子结构体`S`继承了结构体`T`或者对应的结构体指针`*T`，都可以调用`T` + `*T` 的全部方法
    注意：这里有个坑，即`S`继承`T`时，初始化`S`时，不用初始化继承的`T`，也能编译通过，并可以调用`T` + `*T` 的全部方法
         但是不建议采用，因为`T` + `*T` 的方法可能会调用`T`自身数据，而此时多半`T`为0值，但并不会报错，不宜察觉；
         **所以，建议初始化时，也将父结构初始化，以免不必要麻烦**
    注意2：即使是子结构调用父结构方法，所用到的属性并不会后向引用，即不会引用子结构的属性值；这一点与php是不同的
3. 子结构类型值是不能赋值给**父结构**（或祖先结构）类型变量的，即使父结构继承了接口；注意：是父结构类型变量，不是父接口类型变量哦
例子1

```go
package main

import (
	"fmt"
)

type T struct {
	name string //设置元素名称
}

type S struct {
	T
	name string //设置元素名称
}

func (h *T) eat() { //指针接收者
	fmt.Println("T eat cake!!", h.name) 
}

func main() {
	//a := S{T:&T{name: "neo"}, name: "chao"} //继承父结构指针`*T`时的初始化方式
	a := S{T:T{name: "neo"}, name: "chao"} // 在继承父结构`T`时的初始化
	//a := S{ name: "chao"} // 在继承父结构T时，没有初始化父结构T的，也是可以调用`T` + `*T` 的全部方法
	a.eat()     // 注意即使是子结构调用父结构方法，所用到的属性并不会后向引用，即不会引用子结构的属性值
}

```

#### 结构继承接口
##### 结构体声明中明确继承接口`I`
1. 子结构`S`声明的接收者方法属于子结构`S`或者子结构指针`*S`共有（两者的方法集数目都+1），而子结构指针`*S`声明的接收者方法
    属于子结构指针`*S`独有（只有`*S`的方法集数目+1）；`*S`深刻阐述“你的就是我的，我的还是我的”格言
2. 子结构`S`或者结构指针`*S`赋值给`I`接口类型的变量的条件是，子结构`S`或者结构指针`*S`哪个实现接口`I`的方法集数目多
    哪个就可以赋值，两者数目一样多（包括实现0个方法），那就都可以赋值，又根据上面的那一句格言，我们知道，要么子结构`S`
    不能赋值，要么俩都能赋值；即子结构指针`*S`一直可以赋值给`I`接口类型的变量
3. 如果不用赋值给`I`接口类型的变量，那么即使明确声明了继承接口`I`，那么结构体和结构体指针调用方法规则，就回到结构体继承
    结构体中的调用方法规则；
例子
```
package main

import "fmt"

type I interface {
	eat()
	//sleep()
	run()
}

type T struct {
	I           
	name string //设置元素名称
}

type S struct {
	T
	name string //设置元素名称
}

func (h S) eat() {    //接收者
	fmt.Println("S eat cake!!", h.name)
}

func (h *S) run() {    //指针接收者
	fmt.Println("S runing!!", h.name)
}

func main() {
	var a I
	//a = S{T:T{name: "neo"}, name: "chao"} //无法通过编译，因为`S`的方法集数目小于`*S`的方法集数目
	//a = &S{T:T{name: "neo"}, name: "chao"} //可以通过
	a = &S{ name: "chao"}		// 可以通过
	//a.eat()
	fmt.Println(a)
}

```    

结构体调用结构体指针接收者方法更改值会有效么？反之呢？
##### 结构体没有明确声明继承接口`I`
- 子结构没有继承接口，子结构`S`或者子结构指针`*S`则需要实现全部接口方法，哪个实现了全部方法哪个就可以赋值给接口`I`类型的变量
说明：**结构体中不明确声明接口方式，编码更不容易出错，建议采用**
```go
package main

import "fmt"

type I interface {
	eat()
	run()
}

type T struct {
	//I
	name string //设置元素名称
}

type S struct {
	T
	name string //设置元素名称
}

func (h S) eat() {    //接收者
	fmt.Println("S eat cake!!", h.name)
}

func (h *S) run() {    //指针接收者
	fmt.Println("S runing!!", h.name)
}

func main() {
	var a I
	a = S{T:T{name: "neo"}, name: "chao"} //无法通过编译，因为`S`没有实现`I`全部方法
	//a = &S{T:T{name: "neo"}, name: "chao"} //`*S`实现了`I`全部方法，可以通过
	//a = &S{ name: "chao"}		// 可以通过
	a.eat()
	fmt.Println(a)
}

```

#### 更改得以保留
说明：方法中改变调用者元素值，是否在外层得以保留更改，取决于是否为指针接收者方法，而不是调用者是否为指针
1. 因为调用者即使是指针，去调用普通的接收者方法，更改也不会保存的，例子1
2. 因为调用者即使不是指针，去调用指针接收者方法，更改也会被保留，例子2
例子1：
```go
package main

import "fmt"

type I interface {
	eat()
	run()
	get()
}

type T struct {
	I
	name string //设置元素名称
}

type S struct {
	T
	name string //设置元素名称
	Age  int
}
// 说明：方法中改变调用者，是否在外层得以保留更改，取决于是否为指针接收者方法，而不是调用者是否为指针
// 因为调用者即使是指针，去调用普通的接收者方法，更改也不会保存的
func (h *S) eat() { //指针接收者
	h.Age +=  10
}

func (h S) run() { //接收者
	h.Age += 100
}
func (h S) get() { //接收者
	fmt.Println("get ", h.Age)
}
func main() {
	var a I
	a = &S{T: T{name: "neo"}, name: "chao", Age: 1}
	a.eat()
	a.run()
	a.get()
}

```
例子2：

```go
package main

import "fmt"

type I interface {
	eat()
	run()
	get()
}

type T struct {
	I
	name string //设置元素名称
}

type S struct {
	T
	name string //设置元素名称
	Age  int
}

func (h *S) eat() { //指针接收者
	h.Age +=  10
}

func (h S) run() { //接收者
	h.Age += 100
}
func (h S) get() { //接收者
	fmt.Println("get ", h.Age)
}
func main() {
	a := S{T: T{name: "neo"}, name: "chao", Age: 1}
	a.eat()
	a.run()
	a.get()
}
```

#### 多个结构体共同实现接口方法
说明：一个接口的所有方法，不一定需要由一个类型完全实现，接口的方法可以通过在类型中嵌入其他类型或者结构体来实现
```go
package main

import "fmt"

type I interface {
	eat()
	run()
	get()
}

type T struct {
	name string //设置元素名称
	Age  int
}

type S struct {
	T
	name string //设置元素名称

}

func (h T) run() { //接收者
	h.Age += 100
}

func (h *S) eat() { //指针接收者
	h.Age +=  10
}

func (h S) get() { //接收者
	fmt.Println("get ", h.Age)
}
func main() {
	a := S{T: T{name: "neo", Age: 1}, name: "chao"}
	a.eat()
	a.run()
	a.get()
}
```

#### 继承方法调用以及访问属性
1. 通过继承方法调用和访问属性规则采用就近原则，即子结构有，则不会向父结构寻找
2. 父结构方法并不具备后向引用属性功能，即父结构方法不能调用子结构属性，只能调用父结构及更早的祖先属性

### 18 包里`init()`
两种情况
1. 同一个包，在程序的各个文件中，多次被调用，那么这个包`init()`只会执行一次
2. 同一个包里，多个文件都有`init()`，那么只要这个包被调用，那么全部`init()`方法都会被执行，当然只会执行一次

### 19 指针和变量
a. 变量就是一块具体内存，至于内存里放什么，有多大，就涉及到变量类型
变量名是给编译器看的，编译器根据变量是局部还是全局分配内存地址或栈空间，
所谓的变量名在内存中不存在，操作时转换成地址数存放在寄存器中了。其实可以理解为是符号表起到了连接作用。
参考：<https://zhuanlan.zhihu.com/p/101934152>

b. 指针的本质是地址，表现在其本质就是一堆数字。指针变量本质是一个变量，只不过他内部存贮的是地址（即指针）。
c. 符号表在编译程序工作的过程中需要不断收集、记录和使用源程序中一些语法符号的类型和特征等相关信息。
这些信息一般以表格形式存储于系统中。如常数表、变量名表、数组名表、过程名表、标号表等等，统称为符号表。
参考：<https://blog.csdn.net/u011555996/article/details/79781316>

### 20 rune

byte 等同于int8，常用来处理ascii字符
rune 等同于int32,常用来处理unicode或utf-8字符，

```
func main()  {
	str:="这是 haha"
	fmt.Println(len([]rune(str)))   // 有点像php的mb_strlen系列的功能
}
```

### 21 ()()两个括号
(*int)(0)。只有当两个类型的底层基础类型相同时，才允许这种转型操作，或者是两者都是指向相同底层结构的指针类型，
这些转换只改变类型而不会影响值本身。如果x是可以赋值给T类型的值，那么x必然也可以被转为T类型，但是一般没有这个必要。
如：`fmt.Println(([]byte)(str))`或者`fmt.Println(string(str))`
说明：上面的string为类型，可以为自定义类型的（通过type声明的）

### 22 字符串
说明：**字符串不能更改；** 意味如果两个字符串共享相同的底层数据的话也是安全的，这使得复制任何长度的字符串代价是低廉的。
同样，一个字符串s和对应的子字符串切片s[7:]的操作也可以安全地共享相同的内存，因此字符串切片操作代价也是低廉的。
在这两种情况下都没有必要分配新的内存。 
```go
package main

import "fmt"

func main() {
	s:="abcdef"
	//s[0] = "A" // 错误
	t := "A"+s[1:]
	fmt.Println(t)
}

```

### 23 指针声明
```go
package main

func main() {
	s := "sdf"
	position := &s
	*position = "Senior " + *position
	//*n := s		// 报错，因为用法不对，n需要是已经声明过的指针变量
}
```

### 24 数据竞争器
说明：为了帮助诊断此类错误，Go 包含一个内置的 data race detector。
要使用它，请在go命令中添加-race标志：
```
go test -race mypkg    // to test the package
go run -race mysrc.go  // to run the source file
go build -race mycmd   // to build the command
go install -race mypkg // to install the package
```
备注：竞争检查器会报告所有的已经发生的数据竞争。然而，它只能检测到运行时的竞争条件；
并不能证明之后不会发生数据竞争。所以为了使结果尽量正确，请保证你的测试并发地覆盖到了你到包。


## 函数类型和闭包
```go
package main

import "fmt"
//声明自定义函数类型，下面的`diyFunc`为自定义的函数类型，`func() int`可以声明具体的形参类型和返回值类型，如:`func(int,int) int`
//注意type这一行代码位置，并没有在main()中哦
type diyFunc func() int

//这里的diyFunc即为`func() int`，可以用`func() int`代替。即return后的`func() int`去掉形参名称，即为diyFunc
func test() diyFunc {
    //这里是声明在闭包外，所以x值被全周期保留了，如果在闭包里面被声明的话，每次进入该闭包，x都将进行初始化；x是闭包形参的话
    //每次都将被实参赋值
	var x int
	return func() int {
		x++
		return x * x
	}
}

func main() {
	a := test()
	b := test()
    //a()和b()是两个函数，里面的x是不同的地址；
    //匿名函数中x的值被保留全生命周期，有且只有一种情况，即x并不在闭包内声明
    //一旦x由实参赋值或者在闭包内部声明初始化，则按初始化或者赋值的值来；即这个与php的函数内static变量是不同的
	fmt.Println(a())    //1
	fmt.Println(a())    //4
	fmt.Println(b())    //1
	fmt.Println(b())    //4
}

```
额外：闭包里获取和更改的闭包外的值，就是起引用。即闭包里面值被更改，闭包外也会被更改

## 工作区介绍
Go代码必须放在工作区中。工作区其实就是一个对应于特定工程的目录，它应包含3个子目录：src目录、pkg目录和bin目录。

   src目录：用于以代码包的形式组织并保存Go源码文件。（比如：.go .c .h .s等）
   pkg目录：用于存放经由go install命令构建安装后的代码包（包含Go库源码文件）的“.a”归档文件。
   bin目录：与pkg目录类似，在通过go install命令完成安装后，保存由Go命令源码文件生成的可执行文件。
   
   目录src用于包含所有的源代码，是Go命令行工具一个强制的规则，而pkg和bin则无需手动创建，
   如果必要Go命令行工具在构建过程中会自动创建这些目录。
   
   需要特别注意的是，只有当环境变量GOPATH中只包含一个工作区的目录路径时，
   go install命令才会把命令源码安装到当前工作区的bin目录下。若环境变量GOPATH中包含多个工作区的目录路径，
   像这样执行go install命令就会失效，此时必须设置环境变量GOBIN。
   
### 工作区注意事项
1. 分文件编程(多个源文件)，必须放在src目录
2. 设置GOPATH环境变量
3. 同一级目录，包名必须一样；不同级目录，包名不一样
4. `go env`命令可以查看相关的环境变量
5. 同一个目录，调用别的文件函数，直接调用即可，无需包名引用
6. 调用不同包的函数，格式为：包名.函数名()
7. 调用别的包的函数，这个包函数名字如果首字母是小写，无法让别人调用，要想别人调用，必须首字母大写

### 文件执行顺序
const -> var -> init() -> main()
备注：如果前面调用包，则先执行包的const -> var -> init()

```
import _ fmt    //这个调用的意义在于，虽然不能用fmt，但是会调用fmt.init()
```

### GO开发目录
```
go_project     // go_project为GOPATH目录
  -- bin
     -- myApp1  // 编译生成
     -- myApp2  // 编译生成
     -- myApp3  // 编译生成
  -- pkg
  -- src
     -- myApp1     // project1
        -- models
        -- controllers
        -- others
        -- main.go 
     -- myApp2     // project2
        -- models
        -- controllers
        -- others
        -- main.go 
     -- myApp3     // project3
        -- models
        -- controllers
        -- others
        -- main.go 
```

### import和包使用
```
//main.go
package main

import (
	"fmt"
	"test/calculate"	//这里import的是目录名, 不是包名字, 在golang中, 包名可以和目录名不一致的
	"test/hello"		//这里import的是目录名, 不是包名字, 在golang中, 包名可以和目录名不一致的
)

func main() {
	hello.Hello()		//注意注意注意: 这里应用的是包名, 不是目录名
	fmt.Println(calculate.Mysqrt(5))
}
```
说明：import引用的是目录，使用的是包名.方法()；即从始至终，从来没有用到 文件名

## 指针pointer
猜测：
1. 变量名只是给人看的，所有的变量编译运行后都会变成地址。 指针是存储变量地址的变量
2. 
```
var p *int
var a int = 10
p = &a  //指针p被变量a的值的地址 赋值；猜测：编译后变量名a和p是不存在的，只是地址。
        //猜测：&a编译后也是地址，只是人为阅读和操作时，进行了获取a的地址，才有&a的操作
```

## 切片
类似于数组，但是声明的`[]`或者`[...]`，与数组创建方式中，只有`[]`的区别；
但是数组长度len和容量cap都是固定的，而切片可以调整
切片：[low:high:max]
长度len = high - low
容量cap = max - low
备注：如果max在切割时省略，则max为原来的被切割数组的max，即原来数组的个数，注意默认不是high哦
备注：切片只有在容量扩充时，产生新的切片（采用新的内存地址），除此之外，都是操作原来的数据，包括append()，如果容量不扩充，
     那么append()相当于替换了切片在原来数组的后一位元素
```
//直接创建切片
s1 := []int{1,2,3,4} 
//借助make创建
s2 := make([]int,5,10)
//没有指定容量，即容量和长度一样
s3 := make([]int,5)
```
重点说明：切片的容量作用是作为一个是否扩容的判断依据；
golang中slice的扩容机制。当切片的长度超出cap容量的时候，就会引发切片扩容，
每次增加一倍的容量（当容量达到1024以后，每次增加的量大约是之前的25%~35%）。
经扩容后的切片会复制到新的内存地址中。

举例1
```
	s1 := []int{1,2,3,4}
	s2 := s1
	s1[0] = 100
	fmt.Printf("s1=%v,s2=%v", s1, s2)
```
结果：`s1=[100 2 3 4],s2=[100 2 3 4]`
说明：切片切割和复制默认是 共用 被切割和复制的数组（也可以说切片）内存地址的；
除非后面切片进行了扩容，扩容后整个切片地址将与扩容前的地址不同
举例2：
```
	s1 := []int{1, 2, 3, 4} //声明一个长度为4，容量为4的切片
	s2 := s1                //默认复制，s2和s1现在是同一个内存地址
	s1 = append(s1, 5)      //s1进行了扩容，s1不再与s2同一个内存地址
    fmt.Printf("扩容后的s1长度：%d，容量：%d\n",len(s1),cap(s1))   //扩容后s1长度为5，容量为8
	s3 := s1                //默认复制，s3与s1是同一个内存地址
	s1 = append(s1, 6)      //由于容量为8，无需扩容，所以依然是同一个内存地址
	s1[0] = 100
	fmt.Printf("s1=%v,s2=%v,s3=%v", s1, s2, s3)
```
结果：`s1=[100 2 3 4 5 6],s2=[1 2 3 4],s3=[100 2 3 4 5]`

例子3
```
	s1 := []int{1, 2, 3, 4, 5, 6, 7}
	s2 := s1[1:2:4]         //这里截取了部分容量，但是s2依然和s1共用着内存地址
	s3 := s2[0:3]           //虽然s2本身元素不能满足供s3截取，但别人共用着s1地址，所以可以
	fmt.Println(s2,cap(s2))
	fmt.Println(s3)
```
## 数组,切片和map区别
2. 数组元素个数不变，而切片是可变的
3. 复制数组或者数组作为实参进入函数，都是真复制，即内存地址完全不同；而复制切片或者切片作为实参，是复制地址，即建立引用
名称    未初始化值         声明初始化一体           元素个数是否可变    是否真复制       备注
int       0                 是
str       ""                是
bool      false             是
数组      组成元素默认值       是          固定             真复制       
结构体    空结构体            是            固定            真复制       
切片      nil               否           可变             浅复制       切片默认是浅复制，但是扩容时，又会重新生成一份内容   
映射      nil               否           可变             浅复制       映射即使扩充元素，依然是浅复制
指针      nil               否
管道      nil               否
备注0：变量声明在哪里，决定了它是全局变量还是局部；而不是初始化位置
备注1：数组未初始化时，里面的元素值，是[]后声明的类型未初始化时的默认值，`int->0，string->""，bool->false`
备注2：结构体只有声明，没有赋值时，等于空结构体，即 `结构体名{}`；结构体不能直接遍历，需要利用`reflect.ValueOf()`
备注3：map可以任意添加下标，而数组，切片，struct不行；map获取不存在的值，不会报错，会获取到默认0值
```go
package main

import "fmt"

func main() {
	var m = make(map[string]int)
	res, err := m["unkown"]
	fmt.Println(res, err)
}

```
## map
声明：`var m01=map[int]string{1:"aaa",2:"bbb"}`
说明：map数组自始至终的复制或者以实参进入函数，都是浅复制，即建立引用
与切片的区别是：切片扩容后，会和扩容前的切片使用完全不同的内存地址。而map不会

## 可见性
如果想要使用别的包里的函数，结构体类型，结构体成员，那么他们必须首字母大写
如果小写，只能在同一个包里使用。注意：是同一个包里，不一定在同一个文件里
```
type Stu struct {   //结构体名必须首字母大写
	Id int          //结构体成员名必须首字母大写
	Name string
	addr string     //首字母小写不能访问
}

func Say() {        //函数名首字母必须大写
	fmt.Println("lydia say\n")
}
```

## 面向对象编程
- 封装：通过方法实现
```
package person

import "fmt"
//定义不能导出的结构体
type person struct {
   Name string
   age  int         //小写，其他包不能调用
   sal float64      //小写，其他包不能调用
}
//定义工厂模式的函数 首字母大写 类似构造函数
func NewPerson(name string) *person{
   return &person{
      Name:name,
   }
}
//提供一个Set方法 设置年龄
func (user *person) Setage(age int) {
   if age >0  && age < 150 {
      user.age = age
   }else {
      fmt.Println("年龄数值不对！")
   }
}
//获取年龄
func (user *person) Getage() int{
   return  user.age
}
//更改年龄
func (user *person) Updateage(age int) int{
    user.age =age
   return  user.age
}
//更改姓名
func  (user *person) Updatename(name string) string{
   user.Name=name
   return  user.Name
}
```

- 继承：通过匿名字段实现
```
package main

import "fmt"

/*
继承和组合的区别
如果一个struct嵌套了另一个匿名结构体，那么这个结构可以直接访问匿名结构体的方法，从而实现继承
如果一个struct嵌套了另一个【有名】的结构体，那么这个模式叫做组合；组合不能直接使用【有名】结构体的元素和方法，只能通过直接指定【有名】结构体访问
如：`A.B.dosomethings()`和`A.B.name`访问有名结构体的方法和元素
如果一个struct嵌套了多个匿名结构体，那么这个结构可以直接访问多个匿名结构体的方法，从而实现多重继承
重点说明：在多重继承中，即一个结构体同时继承匿名结构体。那么匿名结构体，那么被继承的多个匿名结构体不能出现同名元素名和方法名，不然程序无法判断
访问具体的是哪个里面的元素和方法；当然如果不用同名的元素或者方法，整个程序还是可以编译过的；建议此时采用单线路多次进行继承，如A->B->C，这样
访问时可以采用就近原则；
*/

type Car struct {
    weight int
    name   string
}

func (p *Car) Run() {
    fmt.Println("running")
}

type Bike struct {
    Car             //如果这里给Car取一个名称作为元素名，则是【有名】结构体
    lunzi int
}
type Train struct {
    Car
}

func (p *Train) String() string {
    str := fmt.Sprintf("name=[%s] weight=[%d]", p.name, p.weight)
    return str
}

func main() {
    var a Bike
    a.weight = 100
    a.name = "bike"
    a.lunzi = 2
    fmt.Println(a)
    a.Run()

    var b Train
    b.weight = 100
    b.name = "train"
    b.Run()
    fmt.Printf("%s", &b)
}
```
继承例子2
```
package main

import "fmt"

type annimal interface {
	eat()               //这些方法并不需要全部实现，这一点跟php不一样
	sleep()
	run()
}

type cat interface {
	annimal
	Climb()
}

type HelloKitty struct {
	cat
}

func (h HelloKitty)eat(){
	fmt.Println("eat cake!!")
}

func main() {
	var a annimal
    //超集（类似于php子类，拥有更多的元素和方法）可以赋值给子集（类似于php父类）；
    //重点注意：子集必须是interface，而不能为struct；
    //
	a = HelloKitty{}
	a.eat()
}
```
```
package main

import (
	"fmt"
)

type animal interface {
	eat()
	sleep()
	run()
}

type cat interface {
	animal
	Climb()
}
type HelloKitty struct {
	cat
	name string //设置元素名称
}

func (h *HelloKitty) eat() {    //指针接收者
	fmt.Println("eat cake!!", h.name)
}

func test(re animal) {
	re.eat()
}

func main() {
	var a animal        //猜测：animal可以被HelloKitty或者&HelloKitty赋值
	a = &HelloKitty{name: "chao"}   //由于HelloKitty实现了指针接收者方法，所以这里必须加`&`，如果不加会报下面的错
	test(a)
}
```
不加`&`，报错
```
cannot use XXX literal (type XXX) as type XXX in assignment:XXX does not implement XXX
```
参考：<https://blog.csdn.net/qq_27068845/article/details/86520487>
参考：<https://blog.csdn.net/yipie/article/details/104437625/>

- 多态：通过接口实现
```
package main

import "fmt"

type annimal interface {
	eat()
	sleep()
	run()
}

type cat interface {
	annimal
	Climb()
}
type dog interface {
	annimal
}

type HelloKitty struct {
	cat
}
type husky struct {
	dog
}

func (h HelloKitty)eat(){
	fmt.Println("eat cake!!")
}

func (h husky)eat(){
	fmt.Println("eat bone!!")
}

func test(a annimal){
	a.eat()
}

func main() {
	var a annimal
	a = HelloKitty{}
	test(a)
	a = husky{}
	test(a)
}
```
参考：<https://blog.csdn.net/weixin_37910453/article/details/86645233>

## 空接口
猜测：`interface{}`会将其他类型转化为`interface{}`类型，其他类型的一些操作方法会丢失，
如果数组或切片转化为`interface{}`后，不能被range
```
func test(i []interface{}) {    //这里需要一个切片参数
	fmt.Printf("%T", i)
	for k, v := range i {
		fmt.Println(k, v)
	}
}

func main() {
	var a []int = []int{1, 2, 3}
	val := make([]interface{}, len(a))  //创建一定数量的空接口
	for k, v := range a {
		val[k] = v              //只能一个一个的单独赋值，更改空接口
	}
	test(val)
}
```

## 类型断言

注意：
- 类型断言，只能是`interface{}`类型
- 我们将`interface{}`数据转化时，有时我们是能看到数据结构的，但千万注意，不能想当然的认为golang能完美断言多层
  嵌套数据，需要根据`reflect.TypeOf(XXX)`或者`fmt.Sprintf("%T", xxx)`获取出来看，它能获取什么类型，就转化那个类型，嵌套的只能再次进行类型
  转化
1. 
```
package main

import (
	"fmt"
)

type Student struct {
	id   int
	name string
}

func main() {
	i := make([]interface{}, 3)
	i[0] = 22
	i[1] = "chao"
	i[2] = Student{id: 1, name: "chao"}
    //方式1
	for _, data := range i {
		if value, ok := data.(int); ok == true {            //注意断言格式
			fmt.Println("类型为：int，值为：v%", value)
		} else if value, ok := data.(string); ok == true {
			fmt.Println("类型为：string，值为：v%", value)
		} else if value, ok := data.(Student); ok == true {
			fmt.Println("类型为：Student，值为：v%", value)
		}
	}
    //方式2
    for _, data := range i {
        switch value := data.(type) {           //注意用法
        case int:                               //注意用法
            fmt.Println("类型为：int，值为：v%", value)
        case string:
            fmt.Println("类型为：string，值为：v%", value)
        case Student:
            fmt.Println("类型为：Student，值为：v%", value)
        }
    }
}

```

## 复制文件代码
```
package main

import (
	"io"
	"fmt"
	"os"
)

func main() {
	args := os.Args             //获取cli参数

	if len(args) != 3 {
		fmt.Println("usage:xxx.exe source dest")    
		return
	}

	if args[1] == args[2] {
		fmt.Println("源文件和目标文件不要相同！")
		return
	}
	sf, _ := os.Open(args[1])       //打开文件
	df, _ := os.Create(args[2])     //新建文件
	buf := make([]byte, 4*1024)     //开启4k缓存
	defer sf.Close()
	defer df.Close()
	for {
		n, err := sf.Read(buf)      //将文件内容 依次 注入缓存
		if err != nil || err == io.EOF {    //注意这里 `err == io.EOF`
			fmt.Println(err)
			break
		}
		df.Write(buf[:n])           //将缓存放入文件中
	}
}
```

## 协程
```
package main

import (
	"fmt"
	"runtime"       
	"time"
)

func task() {
	for j := 0; j < 5; j++ {
		fmt.Printf("task j=%d\n",j)
		fmt.Println("this is a task!")
		time.Sleep(time.Second)         //time.Sleep()作用是让cpu在一段时间内不执行该协程
	}
}
//main()为主协程
func main() {
	go task()
	
	for i := 0; i < 2; i++ {
        //runtime.Gosched()作用是代码执行到`runtime.Gosched()`，那么该协程出让一次cpu执行机会。注意下次cpu调度到该协程时，
        //自`runtime.Gosched()`后开始执行，当然如果循环，那就执行到下一次`runtime.Gosched()`位置时，又进行让渡一次
		runtime.Gosched()               
		fmt.Printf("task i=%d\n",i)
		fmt.Println("this is a main!")
		//time.Sleep(time.Second)
	}
}

```
- `runtime.Goexit()` 结束当前协程，注意结束的是协程，不是所处函数，当然所处函数的defer系列代码，依然会被执行完
- `runtime.GOMAXPROCS(1)` 设置go代码运行cpu核数

## channel
说明：goroutine奉行通过通信来共享内存，而不是共享内存来通信
- `make(chan TYPE,0)`声明无缓冲channel，无缓冲channel在数据'放入'和'拿出'channel是同步的，如果缺少其中一方，则会阻塞
```
package main

import (
	"fmt"
	"time"
)

var ch = make(chan int)     //放入和拿出数据是同步的，缺少任何一方，另一方都会阻塞

func printer(str string) {
	for _, v := range str {
		fmt.Printf("%c", v)
		time.Sleep(time.Second)
	}
}

func person01() {
	printer("hello")
	ch <- 666                   //如果没有person02中的<-ch，这里依然会阻塞
	fmt.Println("测试ch阻塞")
}
func person02() {
	<-ch
	printer("world")
}

func main() {
	go person01()
	time.Sleep(time.Second*10)
	go person02()
	for {
	}
}
```
- `make(chan TYPE,CAP)`声明有缓冲channel，在以下情况会阻塞
a. 缓冲区满了，则放入数据会阻塞
b. 缓冲区没有数据，则拿取数据会阻塞
```
package main

import (
	"fmt"
	"time"
)

func main() {
	var ch = make(chan int, 3)
	go func() {
		for i := 0; i < 10; i++ {
			ch <- i
			fmt.Printf("子协程i:%d\n", i)
		}
	}()

	time.Sleep(time.Second * 4)

	for i := 0; i < 10; i++ {
		val := <- ch
		fmt.Printf("主协程val:%d\n", val)
	}

	for{

	}
}
输出结果：
子协程i:0
子协程i:1
子协程i:2      //先执行了子协程，并填满了3个数据缓冲
主协程val:0
主协程val:1
主协程val:2    //这里在拿出了最后一个缓冲，即数据2后，cpu切换回了子协程，将数据3放入了channel，但还没来得及打印，就切换回了主协程
主协程val:3    //主协程获取到了数据3，并打印了
子协程i:3      //又切换会了子协程，并打印数据
子协程i:4      //子协程往channel中放入数据4
子协程i:5      //子协程往channel中放入数据5
子协程i:6      //重点：子协程往channel中放入数据6，channel满了，这里切换到了主协程，主协程从channel中拿出了数据4，还没打印，就切换回子协程
子协程i:7      //由于主协程拿出了数据4，则现在可以放入数据7了
主协程val:4    //切换回了主协程
主协程val:5
主协程val:6
主协程val:7
主协程val:8
子协程i:8
子协程i:9
主协程val:9
```

关闭channel
```
package main

import (
	"fmt"
	"time"
)

func main() {
	var ch = make(chan int, 3)
	go func() {
		for i := 0; i < 10; i++ {
			ch <- i
			fmt.Printf("子协程i:%d\n", i)
		}
		//关闭channel后，可以读取channel数据，但是不能再写入
		close(ch)
	}()

	time.Sleep(time.Second * 4)
    //注意这种用法
    for val := range ch {
		fmt.Printf("主协程val:%d\n", val)
	}
    
}

```
注意：channel和range的配合使用
注意2：如果关闭了channel，那么再次获取该管道，则不会再阻塞，而是获取到0值，以及管道状态为false
      `res,status := <-ch`，res的值为管道类型的0值，status为`false`

## 单向channel的应用
```
package main

import "fmt"

func main() {

	channel := make(chan int)
    //这里是子协程，需要注意的是，子协程和主协程是同时进行的，而不是代码位置的先后哦
	go producer(channel)
    //channel作为参数，是建立的引用
	consumer(channel)
}

func producer(send chan<- int) {
	for i := 0; i < 10; i++ {
		send <- i
	}
	close(send)
}

func consumer(receive <-chan int) {
	for num := range receive {          //注意这里的获取channel数据，会依照channel特性阻塞哦
		fmt.Println("receive", num)
	}
}

```

## timer
```
package main

import (
	"fmt"
	"time"
)

func main() {
	timer := time.NewTimer(3 * time.Second) //只是一次
	fmt.Println(time.Now())
	val := <-timer.C		//注意`timer.C`的用法
	fmt.Println(val)
}
```
## ticker
```
package main

import (
	"fmt"
	"time"
)

func main() {
	ticker := time.NewTicker(1 * time.Second)       //定时循环参数
	i := 0
	for {
		if i > 5 {
			ticker.Stop()       //终止ticker
			break               //跳出循环，不然执行到`<-ticker`代码，会死锁
		}
		i++
		fmt.Println(<-ticker.C)
	}
}

```

## 关键字select
说明：每个case语句必须是一个IO操作
```
select {
case <-chan1:
    // 如果chan1成功读到数据，则进行该case处理语句
case chan2 <- x:
    // 如果成功向chan2写入数据，则进行该case处理语句
default:
    // 如果上面都没有成功，则进入default处理流程
}
```
备注：没有default则可能会阻塞，有default则不会阻塞
备注：如果其中的多个case语句可以继续执行(即没有被阻塞)，那么就从那些可以执行的语句中任意选择一条来使用。
例子：实现斐波那契数
```
package main

import (
	"fmt"
	"time"
)

func fibonacci(ch chan<- int, quit <-chan bool) {
	x, y := 1, 1
	for {
		select {
		case ch <- x:
			x, y = y, x+y
		case result := <-quit:
			fmt.Printf("标签：%v 使生产斐波那契工厂结束",result)
			return
		}
	}
}

func main() {
	var ch = make(chan int)
	var quit = make(chan bool)

	//生产斐波那契数
	go fibonacci(ch, quit)

	//消费者
	for i := 0; i < 8; i++ {
		val := <-ch
		fmt.Printf("主线程读取到的数据：%d\n", val)
	}
	quit <- true
	//因为`quit <- true`执行很快，很有可能fibonacci()还没完成，主协程已经结束了
	time.Sleep(time.Second*2)
}

```
select实现超时
```
package main

import (
	"fmt"
	"time"
)

func main() {
	var ch = make(chan int)
	var quit = make(chan bool)

	go func() {
		for {
			select {
			case num := <-ch:
				fmt.Println("子协程读取到的数据：", num)
			case <-time.After(3 * time.Second):
				fmt.Println("子协程超时")
				quit <- true
			}****
		}
	}()

	for i := 0; i < 5; i++ {
		ch <- i
		time.Sleep(time.Second)
	}
	//取quit数据，不管左边有没有变量接收，其成功与否只取决于quit里是否有数据，跟左边是否有变量没有关系
	<-quit
}

```

## socket编程
### 服务器
```
package main

import (
	"fmt"
	"net"
)

func main() {
	//监听端口
	listener, err := net.Listen("tcp", ":8000")
	if err != nil {
		fmt.Println("监听错误：", err)
		return
	}
	defer listener.Close()

	//阻塞等待用户连接
	conn, err2 := listener.Accept()
	if err2 != nil {
		fmt.Println("接受连接错误：", err2)
		return
	}
	defer conn.Close()
	//接收用户请求
	buf := make([]byte, 1024)
	n, err3 := conn.Read(buf)
	if err3 != nil {
		fmt.Println("接受信息错误：", err3)
		return
	}
	fmt.Println("获取到的信息是：", string(buf[:n]))
}
```
### 客户端
```
package main

import (
	"fmt"
	"net"
)

func main() {
	//拨号
	conn, err := net.Dial("tcp", "127.0.0.1:8000")
	if err != nil {
		fmt.Println("拨号错误：", err)
		return
	}
	_,err2:=conn.Write([]byte("this is a client"))
	if err2 != nil {
		fmt.Println("写入错误：", err2)
		return
	}
	defer conn.Close()
}
```

### 持续监听
服务端
```
package main

import (
	"fmt"
	"net"
	//"runtime"
	"strings"
)

//处理连接
func handleConn(conn net.Conn) {
	defer conn.Close()
	//接收用户请求
	addr := conn.RemoteAddr().String()
	buf := make([]byte, 1024)
	fmt.Printf("与%s建立了连接", addr)

	for {
		n, err3 := conn.Read(buf)
		//注意：如果采用netcat工具，传输字符串，该工具会将回车键一起传入，所以字符串末尾会多一个字符
		//注意2：如果客户端采用os.Stdin.Read(str)，那么服务段接收信息，末尾会多两个字符'\r\n'
		//注意3：如果客户端采用fmt.Scan(&str)，那么服务段接收信息是标准的
		str := string(buf[:n])
		//fmt.Println("检查收到信息长度：",len(str))
		if err3 != nil {
			fmt.Println("接受信息错误：", err3)
			return
		}
		//
		if str == "exit" {
			fmt.Println("收到终止信号：", str)
			return
			//runtime.Goexit()
		}

		fmt.Printf("来自%s的信息：%s\n", addr, string(str))
		//猜测：服务器写入后，客户端默认打印在标准输出上
		conn.Write([]byte(strings.ToUpper(str)))
	}
}

func main() {
	//监听端口
	listener, err := net.Listen("tcp", ":8000")
	if err != nil {
		fmt.Println("监听错误：", err)
		return
	}
	defer listener.Close()

	for {
		//阻塞等待用户连接
		conn, err2 := listener.Accept()
		if err2 != nil {
			fmt.Println("接受连接错误：", err2)
			return
		}
		//开启子协程处理连接
		go handleConn(conn)
	}

}
```
客户端：输入信息和打印服务端信息
```
package main

import (
	"fmt"
	"net"
	//"os"
)

func main() {
	//拨号
	conn, err := net.Dial("tcp", "127.0.0.1:8000")
	if err != nil {
		fmt.Println("拨号错误：", err)
		return
	}
	//关闭连接
	defer conn.Close()

	var str = make([]byte, 1024)
	var buf = make([]byte, 1024)
	//子协程：获取不断的键盘输入
	go func() {
		for {
			fmt.Scan(&str)
			//n,_:=os.Stdin.Read(str)
			_, err2 := conn.Write(str)
			if err2 != nil {
				fmt.Println("写入错误：", err2)
				return
			}
		}
	}()

	for {
		n, err3 := conn.Read(buf)
		if err3 != nil {
			fmt.Println("读取错误：", err3)
			return
		}
		fmt.Println("来自服务的的信息：", string(buf[:n]))
	}
}

```

## 客户端上传文件到服务端
思路：客户端从文件读取内容buf，写入网络；服务的从网络读取内容，写入文件
备注：先传请求头，包含文件名，文件大小等信息
客户端代码
```
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net"
	"os"
)

type Header struct {
	Name string
	Size int64
}

func main() {
	//提示信息
	fmt.Println("请输入完整文件路径：\n")
	var path string
	fmt.Scan(&path)

	//设置请求头信息，包含文件名和文件大小
	finfo,err0:=os.Stat(path)
	if err0 != nil {
		fmt.Println("os.Open err0：", err0)
		return
	}
	var h Header = Header{finfo.Name(),finfo.Size()}
	hInfo,_:=json.Marshal(h)
	//fmt.Println(string(h_info))

	//建立连接网络
	conn, err03 := net.Dial("tcp", "127.0.0.1:8000")
	if err03 != nil {
		fmt.Println("net.Dial err03：", err03)
		return
	}
	defer conn.Close()

	//发送请求头到服务端
	conn.Write(hInfo)

	//获取文件
	file, err := os.Open(path)
	if err != nil {
		fmt.Println("os.Open err：", err)
		return
	}
	defer file.Close()

	//设置缓冲
	buf := make([]byte, 1024*4)

	for {
		//从文件读取数据到缓冲
		n, err02 := file.Read(buf)
		if err02 == io.EOF {
			fmt.Println("文件发送完毕")
			return
		}else if err02 != nil {
			fmt.Println("file.Read err02：", err02)
			return
		}
		//发送
		conn.Write(buf[:n])
	}

}

```
服务端代码
```
package main

import (
	"encoding/json"
	"fmt"
	"net"
	"os"

	//"runtime"
	//"strings"
)


type Header struct {
	Name string
	Size int64
}

func main() {
	//监听端口
	listener, err := net.Listen("tcp", ":8000")
	if err != nil {
		fmt.Println("监听错误：", err)
		return
	}
	defer listener.Close()

	//获取请求头信息
	conn, err01 := listener.Accept()
	if err != nil {
		fmt.Println("listener.Accept err01：", err01)
		return
	}
	defer conn.Close()

	buf := make([]byte, 4*1024)
	n, err02 := conn.Read(buf)
	if err02 != nil {
		fmt.Println("conn.Read err02：", err02)
		return
	}
	//json解析结构，必须和原来一样的结构
	var hInfo Header
	//注意参数为引用
	json.Unmarshal(buf[:n], &hInfo)
	//fmt.Println(hInfo)
	//新建文件
	file, err03 := os.Create("server-" + hInfo.Name)
	if err03 != nil {
		fmt.Println("os.Create err03：", err03)
		return
	}
	defer file.Close()

	//根据头信息，设置文件大小判断
	var totalSize int64 = hInfo.Size
	var sum int = 0
	for {
		n, err02 = conn.Read(buf)
		if err02 != nil {
			fmt.Println("conn.Read2 err02：", err02)
			return
		}
		file.Write(buf[:n])

		sum += n
		//表明文件传输完毕
		if int64(sum) == totalSize {
			fmt.Println("获取文件完毕")
			return
		}

	}
}

```

## 并发聊天室
```
package main

import (
	"fmt"
	"net"
	"time"
)

type Object interface {
	recvMsg()
	sendMsg(user string, msg string)
	sendMsgToAll(msg string, except string)
}

//维护客户端信息
type Client struct {
	Object
	Name    string
	Addr    string
	Conn    net.Conn
	Msg     chan string //注意这里不能采用make()
	HasData chan bool   //用于判断超时断开
	IsQuit  chan bool   //用于判断客户端主动端口
}


//接收消息
func (cli *Client) recvMsg() {
	buf := make([]byte, 1024)
	for {
		n, err := cli.Conn.Read(buf)
		if err != nil {
			cli.IsQuit <- true
			fmt.Println("conn.Read err：", err)
			return
		}
		//这里有大文章可以做，比如分析请求数据，进行路由，或者判断是否针对特定用户发送消息等
		msg := string(buf[:n])
		//对特定个人，以后实现
		//cli.sendMsg()

		fmt.Println("mark:", msg)
		cli.sendMsgToAll(msg, cli.Addr)

		//执行到这里表示有数据，反向超时断开
		cli.HasData <- true
		//has <- true

	}
}

//向除发送消息用户外的所有人，发送消息
func (cli *Client) sendMsgToAll(msg string, except string) {
	for _, c := range clientList {
		if c.Addr != except {
			c.Conn.Write([]byte(msg))
		}
	}
}

//处理连接日常操作:客户端断开，超时断开等
func (cli *Client) handle() {
	//将管道初始化
	cli.HasData = make(chan bool)
	cli.IsQuit = make(chan bool)

	//向其他人推送登录成功消息
	cli.sendMsgToAll(cli.Addr+" login in！\n", cli.Addr)

	//从连接中读取数据，之所以放在这里，是因为handle协程结束时，需要将接收数据协程也结束掉
	go cli.recvMsg()

	for {
		select {
		case <-cli.IsQuit:
			fmt.Printf("%s Active disconnect\n", cli.Addr)
			cli.sendMsgToAll(cli.Addr+" Active disconnect\n", "")
			delete(clientList, cli.Addr)
			return
		case <-cli.HasData:
			fmt.Println("hasData")
			//<-time.After
		case <-time.After(time.Second * 60):
			cli.Conn.Close()
			fmt.Printf("%s Connection timed out\n", cli.Addr)
			cli.sendMsgToAll(cli.Addr+" Connection timed out\n", "")
			delete(clientList, cli.Addr)
			return
		}
	}
}

//客户端连接列表，注意这里声明的map映射的Client，本身就是指针
var clientList = make(map[string]*Client)

func main() {

	listener, err := net.Listen("tcp", "127.0.0.1:8000")
	if err != nil {
		fmt.Println("net.Listen err：", err)
		return
	}
	defer listener.Close()

	for {
		conn, err02 := listener.Accept()
		if err != nil {
			fmt.Println("net.Listen err：", err02)
			continue
		}
		defer conn.Close()

		//将连接信息放入clientList列表中
		addr := conn.RemoteAddr().String()
		clientList[addr] = &Client{
			Name: addr,
			Addr: addr,
			Conn: conn,
		}

		//处理一些常规项操作
		go clientList[addr].handle()
	}
}

```

## 单元测试——测试和验证代码的框架
文件名要以 '_test' 结尾
测试函数以 'Test' 开头

被测函数 testMe.go 如下
```
package main

func s1(s string) int {
	if s == "" {
		return 0
	}

	n := 1
	for range s {
		n++
	}
	return n
}
```
被测函数 \example01\a\b\lydia.go 如下
```
package lyd

func Say() int {
	return 11
}

```
测试函数 testMe_test.go
```
package main

import (
	lyd "example01/a/b"
	"testing"
)

func TestDiy(t *testing.T) {
	if lyd.Say() == 11{
		t.Error("diy say is wrong!")
	}
}

func TestS1(t *testing.T) {
	if s1("123456789") != 9 {
		t.Error(`s1("123456789") != 9`)
	}

	if s1("") != 0 {
		t.Error(`s1("") != 0`)
	}
}
```
使用测试命令
`go test`
或 执行具体测试单元
`go test ./src/testMe.go ./src/testMe_test.go -run='S1' -v`


参考：<https://www.jianshu.com/p/2360984a47a9>
参考：<http://c.biancheng.net/view/124.html>


## 基准测试——获得代码内存占用和运行效率的性能数据
参考：<http://c.biancheng.net/view/124.html>

## `go module`包管理4
模块是相关Go包的集合。modules是源代码交换和版本控制的单元。 go命令直接支持使用modules，
包括记录和解析对其他模块的依赖性。modules替换旧的基于GOPATH的方法来指定在给定构建中使用哪些源文件。

注意：开启go module需要go1.11及以上版本

### 设置`GOPROXY`
```
go env -w GOPROXY=https://goproxy.cn,direct         //采用
```

### 打开模块
```
set GO111MODULE=on    //windows
export GO111MODULE=on //linux
```
备注：默认为`GO111MODULE=auto`，所以不用特意开启或关闭；
备注2：为`auto`时，主要判断目录下是否有`go.mod`文件，如果有，则默认开启，如果没有，则关闭

### 初始化
执行下面的命令生成`go.mod`文件
`go mod init 项目名`           
特别注意：为了没有不必要的麻烦，这里的`项目名`，即`go.mod`文件中的`module 项目名`，需要与文件夹的名字完全一致
不然在引入本地包时会有很多意想不到的麻烦，例如运行时的报错
```
main.go:4:2: cannot find package "." in:
```
报错：
```
$GOPATH/go.mod exists but should not
```
原因：开启模块支持后，并不能与$GOPATH共存,所以把项目从$GOPATH中移出即可
解决：<https://blog.csdn.net/WatermelonMk/article/details/104789411>

`go.mod`--用来管理包的说明文件,现阶段非常简单,文件内一共四种指令

    module: 模块名称
    require: 依赖包列表以及版本
    exclude: 禁止依赖包列表(仅在当前模块为主模块时生效)
    replace: 替换依赖包列表(仅在当前模块为主模块时生效)
例子：
```
module my/thing
require other/thing 	v1.0.2
require new/thing 		v2.3.4
exclude old/thing 		v1.2.3
replace bad/thing 		v1.4.5 	=> good/thing v1.4.5
```
参考：<https://blog.csdn.net/benben_2015/article/details/82227338>

执行下面的命令创建vendor目录存放并下载依赖
`go mod vendor`     //如果没有这一步，依然可以使用，但是没有找到下载包，goland无法跳转；建议采用，方便git版本控制

执行完成会生成`go.sum`文件来记录所依赖的项目的版本的锁定
然后在需要使用包的文件中正常`import`即可

### 引入新的包
在需要使用包的文件中import，然后再次执行下面的命令即可
`go mod vendor`

### 依赖包整理
执行下面的命令可以将没用到的依赖包清除
`go mod tidy`

### 其他命令
`go mod` 有以下命令：

    命令	        说明
    download	download modules to local cache(下载依赖包)
    edit	    edit go.mod from tools or scripts（编辑go.mod
    graph	    print module requirement graph (打印模块依赖图)
    init	    initialize new module in current directory（在当前目录初始化mod）
    tidy	    add missing and remove unused modules(拉取缺少的模块，移除不用的模块)  删除`go.mod`中的依赖声明
                如果需要删除本地文件，那么还需要执行`go mod vendor`
    vendor	    make vendored copy of dependencies(将依赖复制到vendor下)
    verify	    verify dependencies have expected content (验证依赖是否正确）
    why	        explain why packages or modules are needed(解释为什么需要依赖)

参考：<https://www.jianshu.com/p/dcb2065436e3>
参考：<https://shockerli.net/post/go-get-golang-org-x-solution/>
参考：<https://blog.csdn.net/benben_2015/article/details/82227338>

### 将本地包导入vendor目录
a. 需要在包文件夹下执行`go mod init 包名`
b. 需要在项目目录下的go.mod文件进行声明，如下
```
require go-blog/handler/health-check v0.0.0
replace go-blog/handler/health-check => ./go-blog/handler/health-check  //本地包相对与go.mod的路径或绝对路径
```
c. 用`go mod vendor`使本地包进入vendor目录
备注:这种导入适用于已经开完完成的包，如果依然在开发状态的包，不建议导入，因为不会实时更新的

## 常规使用
对于不存在的包，一般也直接使用代码引入，如：
```
import (
	"database/sql"
	_ "github.com/go-sql-driver/mysql"  //这个包本地暂不存在
)
```
直接在命令行使用
```
go mod tidy         //拉取需要的模块，移除不要的模块                  
go mod vendor       //将模块放入vendor
```
备注：一般不用`go mod download`


## 测试
### 命令
```
go test
go test -run=test文件名 -v 文件位置
go test -run=IndexPostRouter -v ./user_test.go
```
备注：`go test`运行目录为*_test.go文件所在目录，但是很多时候并不是项目目录，所以会报错，例如gin框架报错
```
--- FAIL: TestIndexGetRouter (0.00s)
panic: html/template: pattern matches no files: `templates/**/*` [recovered]
        panic: html/template: pattern matches no files: `templates/**/*`
```
修改：这时我们应该修改*_test.go的运行目录，所以
```
os.Chdir("E:\\chao\\sync2\\WWW\\go-sample\\src\\testMod01")     //设置项目目录路径，可以相对路径的
```
备注：将上面代码放在测试方法前执行即可，当然也可以放在其他包里，先被调用即可
参考：<https://ask.csdn.net/questions/1012589>

## Air实时加载工具
参考：<https://www.liwenzhou.com/posts/Go/live_reload_with_air/>
配置文件：air.conf
```
# [Air](https://github.com/cosmtrek/air) TOML 格式的配置文件

# 工作目录
# 使用 . 或绝对路径，请注意 `tmp_dir` 目录必须在 `root` 目录下
root = '.'
tmp_dir = "tmp"

[build]
# 只需要写你平常编译使用的shell命令。你也可以使用 `make`
cmd = "go build -o ./tmp/main.exe ."
# 由`cmd`命令得到的二进制文件名
bin = "tmp/main.exe"
# 自定义的二进制，可以添加额外的编译标识例如添加 GIN_MODE=release
#full_bin = "APP_ENV=dev APP_USER=air ./"
# 监听以下文件扩展名的文件.
include_ext = ["go", "tpl", "tmpl", "html"]
# 忽略这些文件扩展名或目录
exclude_dir = ["go", "templates", "statics", "vendor"]
# 监听以下指定目录的文件
include_dir = []
# 排除以下文件
exclude_file = []
# 如果文件更改过于频繁，则没有必要在每次更改时都触发构建。可以设置触发构建的延迟时间
delay = 1000 # ms
# 发生构建错误时，停止运行旧的二进制文件。
stop_on_error = true
# air的日志文件名，该日志文件放置在你的`tmp_dir`中
log = "air_errors.log"

[log]
# 显示日志时间
time = true

[color]
# 自定义每个部分显示的颜色。如果找不到颜色，使用原始的应用程序日志。
main = "magenta"
watcher = "cyan"
build = "yellow"
runner = "green"

[misc]
# 退出时删除tmp目录
clean_on_exit = true
```

## reflect
reflect包
在Go语言的反射机制中，任何接口值都由是一个具体类型和具体类型的值两部分组成的(我们在上一篇接口的博客中有介绍相关概念)。
在Go语言中反射的相关功能由内置的reflect包提供，任意接口值在反射中都可以理解为
由reflect.Type和reflect.Value两部分组成，
并且reflect包提供了reflect.TypeOf和reflect.ValueOf两个函数来获取任意对象的Value和Type。
参考：<https://www.liwenzhou.com/posts/Go/13_reflect/>
参考：<https://www.cnblogs.com/ksir16/p/9040656.html>

## 中间件
go的中间件：将辅助服务与核心业务解耦，在设计框架的过程中，应该留出可以扩展的空间。
比如：日志记录、故障恢复等功能，如果我们把这些业务逻辑全都塞进Controller/Handler中，会显得代码特别的冗余，杂乱。
备注：其模式类似于yii2框架中的事件event，先将处理函数储存起来，然后在特定位置调用这些处理函数
参考：<https://zhuanlan.zhihu.com/p/134661627>

## sync.pool
说明：储存大对象，以便复用；因为大对象在新建时，开销很大
关于sync.pool 猜测 如下：
1. sync.pool的localSize为核数（猜测），猜测应该是有4个储存池，这里4个储存池不是表明系统只能4个储存池，而是放置某个对象用了4个储存池
    另外的对象还可以用更多的4个储存池存储
2. 储存不同对象建议采用不同的pool，每个对象放在localSize个储存池里；虽然同一个储存池可以放不同类型对象
```
package main

import (
	"fmt"
	"sync"
	"time"
)

type Person struct {
	Name string
	pool sync.Pool
}

type Student struct {
	Name string
	pool sync.Pool
}

func main() {
	p := Person{Name: "chao"}
	// 这一步，可以理解为声明了一个sync.pool，一个sync.pool对应localSize储存池，这里的储存池可以共用
    // 但不能和s.pool.New的储存池共用
	p.pool.New = func() interface{} {
		return 1        //代表着大对象
	}
	s := Student{Name: "lydia"}
	s.pool.New = func() interface{} {
		return 2
	}

	go func() {
		for i := 0; i < 10; i++ {
			p.pool.Put(1)
		}
		for i:=0;i<100000;i++{
			// 如果没有储存值，则只有New()，但这个方法已经是别的Pool里没有值的情况下才执行的
			val := p.pool.Get()
			if val != 1 {
				fmt.Println("1不等")
				return
			}
			//if i %1000==0{
			//	fmt.Println("1相等")
			//}
		}
	}()
	go func() {
		for i := 0; i < 100000; i++ {
			s.pool.Put(2)
		}
		for i := 0; i < 100000; i++ {
			val := s.pool.Get()
			if val != 2 {
				fmt.Println("2不等")
				return
			}
			//if i %1000==0{
			//	fmt.Println("2相等")
			//}
		}
	}()

	time.Sleep(time.Second * 5)
}
```
备注：这个sync.pool应该是

## sync.Mutex和sync.WaitGroup
golang 中的 sync 包实现了两种锁：

    - Mutex：互斥锁
    - RWMutex：读写锁，RWMutex 基于 Mutex 实现
### Mutex（互斥锁）

    - Mutex 为互斥锁，Lock() 加锁，Unlock() 解锁
    - 在一个 goroutine 获得 Mutex 后，其他 goroutine 只能等到这个 goroutine 释放该 Mutex
    - 使用 Lock() 加锁后，不能再继续对其加锁，直到利用 Unlock() 解锁后才能再加锁
    - 在 Lock() 之前使用 Unlock() 会导致 panic 异常
    - 已经锁定的 Mutex 并不与特定的 goroutine 相关联，这样可以利用一个 goroutine 对其加锁，
        再利用其他 goroutine 对其解锁
    - 在同一个 goroutine 中的 Mutex 解锁之前再次进行加锁，会导致死锁
    - 适用于读写不确定，并且只有一个读或者写的场景
备注：相当于串行化操作

### RWMutex（读写锁）

    - RWMutex 是单写多读锁，该锁可以加多个读锁或者一个写锁
    - 读锁占用的情况下会阻止写，不会阻止读，多个 goroutine 可以同时获取读锁
    - 写锁会阻止后面的其他 goroutine（无论读和写）进来，整个锁由该 goroutine 独占
    - 适用于读多写少的场景
Lock() 和 Unlock()

    - Lock() 加写锁，Unlock() 解写锁
    - 如果在加写锁之前已经有其他的读锁和写锁，则 Lock() 会阻塞直到该锁可用，为确保该锁可用，
      已经阻塞的 Lock() 调用会从获得的锁中排除新的读取器，即写锁权限高于读锁，有写锁时优先进行写锁定
    - 在 Lock() 之前使用 Unlock() 会导致 panic 异常
RLock() 和 RUnlock()

    - RLock() 加读锁，RUnlock() 解读锁
    - RLock() 加读锁时，如果存在写锁，则无法加读锁；当只有读锁或者没有锁时，可以加读锁，读锁可以加载多个
    - RUnlock() 解读锁，RUnlock() 撤销单词 RLock() 调用，对于其他同时存在的读锁则没有效果
    - 在没有读锁的情况下调用 RUnlock() 会导致 panic 错误
    - RUnlock() 的个数不得多余 RLock()，否则会导致 panic 错误

互斥锁相当于mysql的排他锁
读写锁相当于mysql的共享锁

包括互斥锁和读写锁补充：上面的`Lock()，UnLock()，Rlock()，RUnlock()`起作用的前提是调用他们的锁是
来自于同一个实例变量mu01，而不能是各自`new(sync.WaitGroup)`的多个变量mu01,mu02,mu03
例子
```
package main

import (
	"fmt"
	"sync"
	"time"
)
type Person struct {
	name string
	mu sync.RWMutex		// 这就像gin框架里的声明了
}
var a int = 0

func main() {
	var swg=new(sync.WaitGroup)
	p01 := Person{}		// 实际已经初始化mu了
	swg.Add(3)
	start:=time.Now()
	go func() {
		p01.mu.RLock()
		time.Sleep(time.Second * 3)
		fmt.Println("first:",a)
		p01.mu.RUnlock()
		swg.Done()
	}()
	go func() {
		time.Sleep(time.Second * 1)		// 延迟1s去拿读锁，没阻碍，直接获取到值
		p01.mu.RLock()
		fmt.Println("second:",a)
		p01.mu.RUnlock()
		swg.Done()
	}()
	go func() {
		time.Sleep(time.Second * 2)
		p01.mu.Lock()
		a = 10
		p01.mu.Unlock()
		swg.Done()
	}()
	swg.Wait()
	end:=time.Now()
	fmt.Println(start,end)
}

```

## 并发编程
1. 同一个包被多次import导入，`init()`和全局变量只会执行一次
    - 全局变量在主协程或者开始时期，只初始化一次，后期都不再修改，那么可以不加锁
    - 除上面的情况外，只要在协程并发期，依然有对全局变量的更改，那么读操作和写操作都必须加锁，互斥锁和读写锁都可以
      猜测：因为赋值或者说更改值操作并不是原子性的，有可能赋值到一半，地址被其他协程拿走了
      参考：<https://www.v2ex.com/t/544205>
2. 共用的东西尽量采用池的概念，大对象保存在sync.pool；连接用连接池

## context和channel
参考：<https://segmentfault.com/a/1190000017394302>
参考：<https://blog.csdn.net/u011957758/article/details/82948750>
```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"sync"
	"time"
)

type myKey struct{}
type myVal struct {
	N int
	S string
}

var quit = make(chan bool)
var wg = new(sync.WaitGroup)

func parent() {
	//ctx, cancel := context.WithCancel(context.Background())
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	//cancel()是可以多次调用的，除了第一次`<-context.Done()`有效，后面都是无效的

	// 设置键值对在context上，传给子协程
	ctx = context.WithValue(ctx, myKey{}, &myVal{0, "abc"})
	wg.Add(2)
	go child(ctx, "child01")
	go child(ctx, "child02")

	//for-range-chan必须预留跳出循环方式，不然就会导致死锁
	//fatal error: all goroutines are asleep - deadlock!
	//第一种：close(chan)关闭管道，但是这种方式需要注意，关闭管道后，不能给管道赋值或者不能再次关闭管道
	//第二种：用break跳出for-range-chan循环
	for r := range quit {
		if r == true {
			fmt.Printf("比赛结束 %t\n", r)
			cancel()
			break //可以跳出for-range-chan的循环
		}
	}
	wg.Wait()
	fmt.Println("完成")
}

func child(context context.Context, name string) {
	// 设置起点，根据键获取值
	num := context.Value(myKey{}).(*myVal).N
	var rInt int
	var ok = false

	for {
		// select-case多个条件都成立，是无法判断执行哪个case的哦
		select {
		// `context.Done()`需要父协程给相应的时间，简单说就是如果父协程跑得快，直接执行完，子协程中的`context.Done()`
		// 可能执行不了，需要sync.WaitGroup
		case <-context.Done():
			fmt.Printf("%s done\n", name)
			wg.Done()
			//这步是让for-range-chan能通过，不然它就死锁了。由于有两个子协程，所以这里有个协程可能会阻塞，
			//不过影响不大，最终可以被回收
			quit <- true
			return
			// case quit <- ok 这种写法，管道quit关闭了，代码执行到这里了，会报错的；
			// 即最好不要有close(chan)操作
		case quit <- ok:
			rInt = rand.Intn(5)
			num += rInt
			fmt.Printf("%s arrived %d\n", name, num)
			if num > 10 && !ok {
				ok = true
				fmt.Println(name, " return ", time.Now())
				wg.Done()
			}
		}
		time.Sleep(time.Second)
	}

}

func main() {
	parent()
}

```

## reflect
说明：反射是指在程序运行期对程序本身进行访问和修改的能力
参考：<http://www.topgoer.com/%E5%B8%B8%E7%94%A8%E6%A0%87%E5%87%86%E5%BA%93/%E5%8F%8D%E5%B0%84.html>
参考：<https://blog.csdn.net/lanyang123456/article/details/95238197>

不是所有的反射值都可以修改。对于一个反射值是否可以修改，可以通过CanSet()进行检查。
要修改值，必须满足:

- 可以寻址，可寻址的类型：

    + 指针指向的具体元素
    + slice的元素
    + 可寻址的结构体的字段(指向结构体的指针)
    + 可寻址的数组的元素(指向数组的指针)
    
- 不是结构体没有导出的字段

示例1
```go
package main

import (
	"fmt"
	"reflect"
)

type Person struct {
	Name string `json:"haha" encode="hehe"`
	Age  int    `json:"haha1" encode="hehe2"`
}

func main() {
	// 如果需要更改其值，就只能传入指针，不然不能更改
	var a = &Person{"chao", 12}
	to := reflect.TypeOf(a).Elem()  // 获取结构类型相关信息，Elem()获取指针指向的元素
	vo := reflect.ValueOf(a).Elem() // 获取结构值相关信息
	var tmp bool
	for i := 0; i < to.NumField(); i++ {
		f := to.Field(i)
		fmt.Println(f.Name, f.Type, f.Tag.Get("json"))
		r := vo.Field(i)
		switch r.Kind() {
		case reflect.String:
			tmp = r.CanSet()
			r.SetString("lydia")
		case reflect.Int:
			r.SetInt(88)
		}
		fmt.Println(r, tmp)
		fmt.Println(r.Kind())
	}
}

```

示例2
```go
package main

import (
	"fmt"
	"reflect"
)

type Person struct {
	Name string `json:"haha" encode="hehe"`
	Age  int    `json:"haha1" encode="hehe2"`
}

func (p Person) Say(str string) {
	fmt.Printf("I am %s, %s", p.Name, str)
}

func main() {
	// 如果需要更改其值，就只能传入指针，不然不能更改
	var a = Person{"chao", 12}
	v := reflect.ValueOf(a)
	// 获取方法
	m := v.MethodByName("Say")
	// 构建一些参数
	args := []reflect.Value{reflect.ValueOf("666")}
	// 没参数的情况下：var args2 []reflect.Value
	// 调用方法，需要传入方法的参数
	m.Call(args)
}

```

## 误区

### 协程共享和sync.WaitGroup
说明：php的一般运行是多进程的，所以程序之间可以理解为没啥联系，变量也不共享，但是golang的协程外变量是在协程间共享的
猜测：定义`协程里`和`协程外`（不一定正确的）
`协程里`：`go`关键字后执行的程序A，包括通过`import`引入另外的文件x的方法B，A程序调用了程序B，在这些程序里（包括A,B）
          *声明* 的变量，属于该协程独有，当然具体方法里的声明的变量只能自己方法用，这个范围暂时不讨论；
          但是，如果B使用了x的全局变量或者其他包文件全局变量，这个变量并不在`协程里`
          即必须在程序里直接运行声明的变量才在`协程里`
`协程外`：与`协程里`相反，`协程外`的变量容易成为脏数据，一般需要加锁，如果协程共享数据，建议channel
备注：golang的缺点是没有协程内的全局变量
main.go
```
package main

import "example01/a"

func main() {
	go func() {
		a.Incr1()
	}()

	go func() {
		a.Incr100()
	}()

	for {
	}
}

```
a/client1.go
```
package a

import (
	"example01/demo"
	"fmt"
	"time"
)

func Incr1() int {
	for{
		time.Sleep(time.Second)
		demo.Mark = demo.Mark + 1
		fmt.Println(demo.Mark)
	}
}

```
a/client100.go
```
package a

import (
	"example01/demo"
	"fmt"
	"time"
)

func Incr100()  {
	for{
		time.Sleep(time.Second)
		demo.Mark = demo.Mark + 100
		fmt.Println(demo.Mark)
	}
}
```
demo/demo.go
```
package demo

var Mark int =0
```
备注：代码运行后，`demo.Mark`是共享的，但是这种共享是不保证是否有脏数据的，如下例
```
package main

import (
	"fmt"
	"sync"
)

var ex int

func main() {
	ex = 0
	var ewg sync.WaitGroup      // go等待一组协程结束的实现方式
	ewg.Add(100000)
	for i := 0; i < 100000; i++ {
		go func() {
			ex += 1
			ewg.Done()
		}()
	}
	ewg.Wait()
	fmt.Println(ex)
}
```
结果：一般不为100000的，因为有些协程读取了脏数据；如果要修改协程外共享数据，要么用channel，要么加锁`sync.Mutex`

## 千万注意

### 换行符
说明：windows和unix系列系统换行符是不同的，前者为`\r\n`，后者则是`\n`
开始编码前，一定要将编辑器的换行符调整为unix系列，即`\n`
- windows下调整方式：先调整编辑器的默认换行符，参考<https://blog.csdn.net/x356982611/article/details/84883267>
  然后用编辑器新建一个文件，将文件内容复制进入新文件即可
- linux下调整方式：dos2unix，当然还有一个相对应的unix2dos，或者vim编辑`set fileformat=unix`或者简写`set ff=unix`

## 代码开发和交叉编译
### 代码开发
1. 依然采用`go mod`进行包管理，但由于其引入本地包的不足，采用GOPATH进行补充
2. 利用goland将项目的GOPATH设置在项目的上一级的src上，即 E:\xxx\src\具体项目，其中E:\xxx\src为多个项目的GOPATH
3. 这样既能达到引入线上包，又能利用GOPATH获取本地包，以及代码高亮

### 交叉编译
1. 环境准备
```
set GOARCH=amd64        # 线上cpu架构，通过命令 `lscpu`
set GOOS=linux          # 线上系统
go build mian.go
```

## 报错

### 交叉编译文件在linux上报windows文件错误
说明：在window上交叉编译后，上线报错，提示报错是windows的文件位置，**千万注意，这里不是交叉编译的问题**，而是代码那儿报错
为啥会报错呢？因为那个位置引入了redis服务，redis密码不正确
```
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x28 pc=0xa0fc4a]

goroutine 1 [running]:
blog/initRouter.SetupRouter(0xc0002b6780)
	E:/chao/sync2/WWW/go-sample/src/blog/initRouter/initRouter.go:31 +0x49a
main.main()
	E:/chao/sync2/WWW/go-sample/src/blog/main.go:21 +0x48
```


## 面试

### 求质数
说明：利用一个定理——如果一个数是合数，那么它的最小质因数肯定小于等于他的平方根。例如：50，最小质因数是2，2<50的开根号
再比如：15，最小质因数是3，3<15的开根号
合数是与质数相对应的自然数。一个大于1的自然数如果它不是合数，则它是质数。
上面的定理是说，如果一个数能被它的最小质因数整除的话，那它肯定是合数，即不是质数。所以判断一个数是否是质数，
只需判断它是否能被小于它开跟后后的所有数整除，这样做的运算就会少了很多，因此效率也高了很多。
参考：<https://www.cnblogs.com/chucklu/p/4627058.html>
```
package main

import "fmt"

func main() {
	/* 定义局部变量 */
	var i, j int
	var res = make([]int, 0)
	for i = 2; i < 100; i++ {
		// 第二个for，其实是判断数不是合数
		for j = 2; j <= (i / j); j++ {
			if i%j == 0 {
				break // 如果发现因子，则不是素数
			}
		}
		if j > (i / j) {
			res = append(res, i)
			fmt.Printf("%d  是素数\n", i)
		}
	}
}

```

## gorm
说明：虽然gorm关联必须声明外键，但是停留在代码层面，而数据库可以没有实际的外键。
说明2：Has One和belongs to是站在不同表的角度看问题，hasOne 正向关联，belongsTo 反向关联。
      简单的讲就是，没有太大的区别，只是在逻辑上出现的思想的偏差（逻辑的合理性）。
      belongsTo：可以理解为属于
      hasOne：可以理解为拥有
说明3：尽管Has One和belongs to关系不同，但是foreignkey和association_foreignkey指向是一样的
- foreignkey：指定拥有者标记（一般为ID数据）在被拥有者表中的保存字段名称（一般为`表名_id`）
- association_foreignkey：拥有者标记（在拥有者表中的字段），一般为ID数据

说明4：建立关联关系后，
- 增是自动的向关联表添加记录
- 删和改都不会自动更改被拥有者表
- 查询需要采用显示声明
```
var card CreditCard
// 这样是根据user的关联id，只查询表CreditCard
db.Model(&user).Related(&card, "CreditCard")
```

说明5：多对多关系，记录插入顺序注意点是最后插入中间表记录
 
说明6：
预加载：在获取拥有者表记录后，立即通过关联的id，查出被拥有者表的相关记录(采用这个进行关联查询)


## 包
### 预定义的名字
内建常量: true false iota nil

内建类型: int int8 int16 int32 int64
          uint uint8 uint16 uint32 uint64 uintptr
          float32 float64 complex128 complex64
          bool byte rune string error

内建函数: make len cap new append copy close delete
          complex real imag
          panic recover
说明：这些内部预先定义的名字并不是关键字，你可以再定义中重新使用它们。
在一些特殊的场景中重新定义它们也是有意义的，但是也要注意避免过度而引起语义混乱。

- copy() 复制函数，对于切片来说，真复制0层数据，更多层的数据是浅复制，如果需要全部层都真复制，需要迭代，每层单独copy()

### 数组
#### 数组声明
```go
package main

import "fmt"

func main() {
	s := [6]int{0, 1, 2, 3, 4, 5}
	fmt.Println(s)
	zero(&s)
	fmt.Println(s)
	a := [2]int{1, 2}
	b := [...]int{1, 2}					// ...也是数组，不是切片；表示数组的长度是根据初始化值的个数来计算
	c := [2]int{1, 3}
	fmt.Println(a == b, a == c, b == c) // "true false false"
	r := [...]int{99: -1}				// 定义了一个含有100个元素的数组r，最后一个元素被初始化为-1，其它元素都是用0初始化。
	fmt.Println(r)
}

func zero(s *[6]int) {
	for i, _ := range s {
		s[i] = 0
	}
}
```
#### 数组操作

### 切片
常见操作：

    将切片 b 的元素追加到切片 a 之后：a = append(a, b...)
    
    复制切片 a 的元素到新的切片 b 上：
    
    b = make([]T, len(a))
    copy(b, a)
    删除位于索引 i 的元素：a = append(a[:i], a[i+1:]...)
    
    切除切片 a 中从索引 i 至 j 位置的元素：a = append(a[:i], a[j:]...)
    
    为切片 a 扩展 j 个元素长度：a = append(a, make([]T, j)...)
    
    在索引 i 的位置插入元素 x：a = append(a[:i], append([]T{x}, a[i:]...)...)
    
    在索引 i 的位置插入长度为 j 的新切片：a = append(a[:i], append(make([]T, j), a[i:]...)...)
    
    在索引 i 的位置插入切片 b 的所有元素：a = append(a[:i], append(b, a[i:]...)...)
    
    取出位于切片 a 最末尾的元素 x：x, a = a[len(a)-1], a[:len(a)-1]
    
    将元素 x 追加到切片 a：a = append(a, x)
### map
说明：map只有len，表示map的元素个数，map没有cap，更没有数量限制；即`make(map[int]string)`和`make(map[int]string,0)`都可以被赋值
- delete(m,key)


### json

解析包含任意层级的数组和对象的JSON数据和用 Decoder解析数据流
参考：<https://blog.csdn.net/kevin_tech/article/details/105213359?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param>

说明：golang里使用`encoding/json`对json数据进行处理
```
json.Marshal(arg1)  // 对arg1进行json编码
json.MarshalIndent(arg1, "", "    ")  // 对arg1进行json编码，两个额外的参数用于表示每一行输出的前缀和每一个层级的缩进
json.Unmarshal(arg1, &arg2) // 对json字符串进行解码到arg2，arg2需要提前声明`var arg2 []Movie`
```

备注：也有其他的扩展包<https://github.com/tidwall/gjson>


例
```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
)

type Movie struct {
    // **注意：用于json解析的struct属性名必须大写**
	Title  string
	Year   int  `json:"released"`
	Color  bool `json:"color,omitempty"`	// color作为转化后的下标，omitempty表示当Go语言结构体成员为空或零值时不生成JSON对象（这里false为零值）
	Actors []string
}

var movies = []Movie{
	{Title: "Casablanca", Year: 1942, Color: false,
		Actors: []string{"Humphrey Bogart", "Ingrid Bergman"}},
	{Title: "Cool Hand Luke", Year: 1967, Color: true,
		Actors: []string{"Paul Newman"}},
	{Title: "Bullitt", Year: 1968, Color: true,
		Actors: []string{"Steve McQueen", "Jacqueline Bisset"}},
	// ...
}

func main() {
	data, err := json.MarshalIndent(movies, "", "	")
	if err != nil {
		log.Fatalf("JSON marshaling failed: %s", err)
	}
	//fmt.Printf("%s\n", data)

	var res []Movie
	json.Unmarshal(data, &res)
	fmt.Println(res)
}

```
例2

```go
package main

import (
	"encoding/json"
	"fmt"
)

type Code struct {
	// **注意：用于json解析的struct属性名必须大写**
	Pct       float64 `json:"pct"`
	Symbol    string  `json:"symbol"`
	Current   float64 `json:"current"`
	Mc        int     `json:"mc"`
	Name      string  `json:"name"`
	Exchange  string  `json:"exchange"`
	T         int     `json:"type"`
	Areacode  int     `json:"areacode"`
	TickSize  float64 `json:"tick_size"`
	Niota     float64 `json:"niota"`
	HasFollow bool    `json:"has_follow"`
	Indcode   string  `json:"indcode"`
}

type Data struct {
	List  []Code `json:"list"`
	Count int    `json:"count"`
}
type Result struct {
	Data             Data
	ErrorCode        int    `json:"error_code"`
	ErrorDescription string `json:"error_description"`
}

func main() {
	var r Result
	str := `{"data":{"count":298,"list":[{"pct":-1.97,"symbol":"SZ300869","current":98.03,"mc":39384122336,"name":"康泰医学","exchange":"sh_sz","type":11,"areacode":"130000","tick_size":0.01,"niota":11.7619,"has_follow":false,"indcode":"S3705"},{"pct":0.01,"symbol":"SZ300866","current":136.8,"mc":55599241918,"name":"安克创新","exchange":"sh_sz","type":11,"areacode":"430000","tick_size":0.01,"niota":28.5092,"has_follow":false,"indcode":"S2704"},{"pct":7.79,"symbol":"SZ300861","current":53.14,"mc":21268531700,"name":"美畅股份","exchange":"sh_sz","type":11,"areacode":"610000","tick_size":0.01,"niota":25.776,"has_follow":false,"indcode":"S6401"},{"pct":4.8,"symbol":"SZ300860","current":147,"mc":10630923408,"name":"锋尚文化","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":23.6956,"has_follow":false,"indcode":"S4605"},{"pct":13.04,"symbol":"SZ300856","current":101.4,"mc":11391849600,"name":"科思股份","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":16.0894,"has_follow":false,"indcode":"S2203"},{"pct":-5.29,"symbol":"SZ300841","current":480.82,"mc":28830000000,"name":"康华生物","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":31.7707,"has_follow":false,"indcode":"S3703"},{"pct":-1.47,"symbol":"SZ300832","current":149.77,"mc":61757659500,"name":"新产业","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":25.3376,"has_follow":false,"indcode":"S3705"},{"pct":-3.35,"symbol":"SZ300829","current":96.7,"mc":10918309196,"name":"金丹科技","exchange":"sh_sz","type":11,"areacode":"410000","tick_size":0.01,"niota":10.1507,"has_follow":false,"indcode":"S1105"},{"pct":-3.01,"symbol":"SZ300821","current":13.2,"mc":15816000000,"name":"东岳硅材","exchange":"sh_sz","type":11,"areacode":"370000","tick_size":0.01,"niota":22.7955,"has_follow":false,"indcode":"S2203"},{"pct":0.64,"symbol":"SZ300815","current":120.21,"mc":16646752000,"name":"玉禾田","exchange":"sh_sz","type":11,"areacode":"340000","tick_size":0.01,"niota":14.2904,"has_follow":false,"indcode":"S4104"},{"pct":1.25,"symbol":"SZ300805","current":24.3,"mc":10271792100,"name":"电声股份","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":10.8518,"has_follow":false,"indcode":"S7202"},{"pct":-2.02,"symbol":"SZ300792","current":134.64,"mc":19421846928,"name":"壹网壹创","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":22.7602,"has_follow":false,"indcode":"S7202"},{"pct":-0.24,"symbol":"SZ300782","current":367.08,"mc":66110400000,"name":"卓胜微","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":40.1244,"has_follow":false,"indcode":"S2701"},{"pct":-0.59,"symbol":"SZ300777","current":47.26,"mc":18956473900,"name":"中简科技","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":12.7379,"has_follow":false,"indcode":"S2204"},{"pct":0.06,"symbol":"SZ300776","current":140.97,"mc":14914659833,"name":"帝尔激光","exchange":"sh_sz","type":11,"areacode":"420000","tick_size":0.01,"niota":19.7262,"has_follow":false,"indcode":"S6402"},{"pct":-0.43,"symbol":"SZ300773","current":41.7,"mc":33360834000,"name":"拉卡拉","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":10.07,"has_follow":false,"indcode":"S7102"},{"pct":-1.73,"symbol":"SZ300770","current":95.45,"mc":22068363524,"name":"新媒股份","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":21.01,"has_follow":false,"indcode":"S7203"},{"pct":-3.02,"symbol":"SZ300768","current":41.72,"mc":16676416900,"name":"迪普科技","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":14.4548,"has_follow":false,"indcode":"S7102"},{"pct":8.85,"symbol":"SZ300763","current":111.83,"mc":15452615656,"name":"锦浪科技","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":13.0965,"has_follow":false,"indcode":"S6303"},{"pct":-0.26,"symbol":"SZ300761","current":38.48,"mc":15573612800,"name":"立华股份","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":27.5535,"has_follow":false,"indcode":"S1107"},{"pct":-2.96,"symbol":"SZ300760","current":328.11,"mc":398953402763,"name":"迈瑞医疗","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":19.8251,"has_follow":false,"indcode":"S3705"},{"pct":-2.48,"symbol":"SZ300747","current":68.37,"mc":19684800000,"name":"锐科激光","exchange":"sh_sz","type":11,"areacode":"420000","tick_size":0.01,"niota":12.5612,"has_follow":false,"indcode":"S2704"},{"pct":-5.28,"symbol":"SZ300741","current":51.99,"mc":31976489600,"name":"华宝股份","exchange":"sh_sz","type":11,"areacode":"540000","tick_size":0.01,"niota":13.9285,"has_follow":false,"indcode":"S3404"},{"pct":-3.1,"symbol":"SZ300735","current":17.21,"mc":13331229475,"name":"光弘科技","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":17.3879,"has_follow":false,"indcode":"S2705"},{"pct":-0.77,"symbol":"SZ300726","current":40.07,"mc":16008001000,"name":"宏达电子","exchange":"sh_sz","type":11,"areacode":"430000","tick_size":0.01,"niota":16.9203,"has_follow":false,"indcode":"S2702"},{"pct":-0.64,"symbol":"SZ300725","current":119.67,"mc":17337574849,"name":"药石科技","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":16.9885,"has_follow":false,"indcode":"S3701"},{"pct":-0.36,"symbol":"SZ300702","current":106.27,"mc":19363075486,"name":"天宇股份","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":21.4177,"has_follow":false,"indcode":"S3701"},{"pct":-2.36,"symbol":"SZ300699","current":70.28,"mc":36434821500,"name":"光威复材","exchange":"sh_sz","type":11,"areacode":"370000","tick_size":0.01,"niota":13.642,"has_follow":false,"indcode":"S2204"},{"pct":-0.7,"symbol":"SZ300685","current":75.62,"mc":16781053523,"name":"艾德生物","exchange":"sh_sz","type":11,"areacode":"350000","tick_size":0.01,"niota":14.524,"has_follow":false,"indcode":"S3705"},{"pct":2.76,"symbol":"SZ300682","current":22.33,"mc":22789380040,"name":"朗新科技","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":26.8115,"has_follow":false,"indcode":"S7102"},{"pct":0.07,"symbol":"SZ300662","current":55.88,"mc":10182551820,"name":"科锐国际","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":11.6767,"has_follow":false,"indcode":"S4605"},{"pct":-0.6,"symbol":"SZ300661","current":287.94,"mc":44798221476,"name":"圣邦股份","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":14.23,"has_follow":false,"indcode":"S2701"},{"pct":-0.88,"symbol":"SZ300659","current":50.45,"mc":11432080031,"name":"中孚信息","exchange":"sh_sz","type":11,"areacode":"370000","tick_size":0.01,"niota":18.6604,"has_follow":false,"indcode":"S7102"},{"pct":-1.3,"symbol":"SZ300639","current":44.68,"mc":10520029933,"name":"凯普生物","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":11.0216,"has_follow":false,"indcode":"S3705"},{"pct":-0.82,"symbol":"SZ300638","current":58.97,"mc":14243982775,"name":"广和通","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":11.1266,"has_follow":false,"indcode":"S7302"},{"pct":1.04,"symbol":"SZ300630","current":44.6,"mc":19400586361,"name":"普利制药","exchange":"sh_sz","type":11,"areacode":"460000","tick_size":0.01,"niota":21.288,"has_follow":false,"indcode":"S3701"},{"pct":-2,"symbol":"SZ300628","current":55.98,"mc":50530277365,"name":"亿联网络","exchange":"sh_sz","type":11,"areacode":"350000","tick_size":0.01,"niota":28.8039,"has_follow":false,"indcode":"S7302"},{"pct":-1.15,"symbol":"SZ300602","current":26.58,"mc":13444562046,"name":"飞荣达","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":12.7808,"has_follow":false,"indcode":"S2705"},{"pct":-2.2,"symbol":"SZ300601","current":174.66,"mc":117444288766,"name":"康泰生物","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":15.7639,"has_follow":false,"indcode":"S3703"},{"pct":-4.07,"symbol":"SZ300598","current":132.46,"mc":13977648088,"name":"诚迈科技","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":23.1603,"has_follow":false,"indcode":"S7102"},{"pct":-2.41,"symbol":"SZ300595","current":57.85,"mc":35051825655,"name":"欧普康视","exchange":"sh_sz","type":11,"areacode":"340000","tick_size":0.01,"niota":20.8111,"has_follow":false,"indcode":"S3705"},{"pct":-5.76,"symbol":"SZ300529","current":66.58,"mc":53132766852,"name":"健帆生物","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":25.0161,"has_follow":false,"indcode":"S3705"},{"pct":-2.66,"symbol":"SZ300502","current":61.23,"mc":20255861556,"name":"新易盛","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":13.982,"has_follow":false,"indcode":"S7302"},{"pct":-1.68,"symbol":"SZ300498","current":21.12,"mc":134671290939,"name":"温氏股份","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":24.169,"has_follow":false,"indcode":"S1107"},{"pct":-1.42,"symbol":"SZ300487","current":54.96,"mc":11629901435,"name":"蓝晓科技","exchange":"sh_sz","type":11,"areacode":"610000","tick_size":0.01,"niota":12.069,"has_follow":false,"indcode":"S2203"},{"pct":0.32,"symbol":"SZ300482","current":86.5,"mc":29607568675,"name":"万孚生物","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":15.1575,"has_follow":false,"indcode":"S3705"},{"pct":-1.69,"symbol":"SZ300463","current":48.17,"mc":26870407788,"name":"迈克生物","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":11.2536,"has_follow":false,"indcode":"S3705"},{"pct":-0.49,"symbol":"SZ300454","current":201,"mc":82224221120,"name":"深信服","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":12.7494,"has_follow":false,"indcode":"S7102"},{"pct":0.66,"symbol":"SZ300418","current":29.13,"mc":34158432963,"name":"昆仑万维","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":14.533,"has_follow":false,"indcode":"S7203"},{"pct":-2.15,"symbol":"SZ300408","current":26.37,"mc":46018864383,"name":"三环集团","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":10.2479,"has_follow":false,"indcode":"S2702"},{"pct":-3.36,"symbol":"SZ300406","current":18.97,"mc":11196837627,"name":"九强生物","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":16.505,"has_follow":false,"indcode":"S3705"},{"pct":-0.17,"symbol":"SZ300395","current":40.22,"mc":13546623261,"name":"菲利华","exchange":"sh_sz","type":11,"areacode":"420000","tick_size":0.01,"niota":10.8218,"has_follow":false,"indcode":"S2402"},{"pct":-0.19,"symbol":"SZ300394","current":58.22,"mc":11581352022,"name":"天孚通信","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":13.173,"has_follow":false,"indcode":"S7302"},{"pct":-2.83,"symbol":"SZ300357","current":56.75,"mc":29687212800,"name":"我武生物","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":23.5405,"has_follow":false,"indcode":"S3703"},{"pct":-3.8,"symbol":"SZ300347","current":98.6,"mc":86027696707,"name":"泰格医药","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":16.5135,"has_follow":false,"indcode":"S3706"},{"pct":-1.5,"symbol":"SZ300285","current":39.37,"mc":37907470672,"name":"国瓷材料","exchange":"sh_sz","type":11,"areacode":"370000","tick_size":0.01,"niota":11.8363,"has_follow":false,"indcode":"S2202"},{"pct":-2.65,"symbol":"SZ300236","current":51.86,"mc":15073052784,"name":"上海新阳","exchange":"sh_sz","type":11,"areacode":"310000","tick_size":0.01,"niota":12.4231,"has_follow":false,"indcode":"S2203"},{"pct":-1.63,"symbol":"SZ300144","current":18.74,"mc":48999366310,"name":"宋城演艺","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":12.4887,"has_follow":false,"indcode":"S4601"},{"pct":-1.32,"symbol":"SZ300136","current":57.77,"mc":55586406472,"name":"信维通信","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":13.1792,"has_follow":false,"indcode":"S2705"},{"pct":1.72,"symbol":"SZ300132","current":23.64,"mc":12222303763,"name":"青松股份","exchange":"sh_sz","type":11,"areacode":"350000","tick_size":0.01,"niota":15.5061,"has_follow":false,"indcode":"S2203"},{"pct":-2.23,"symbol":"SZ300122","current":124.88,"mc":199904000000,"name":"智飞生物","exchange":"sh_sz","type":11,"areacode":"500000","tick_size":0.01,"niota":26.6602,"has_follow":false,"indcode":"S3703"},{"pct":-2.36,"symbol":"SZ300033","current":150.08,"mc":80629248000,"name":"同花顺","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":19.1674,"has_follow":false,"indcode":"S7102"},{"pct":-2.23,"symbol":"SZ300015","current":48.67,"mc":200635497944,"name":"爱尔眼科","exchange":"sh_sz","type":11,"areacode":"430000","tick_size":0.01,"niota":13.2999,"has_follow":false,"indcode":"S3706"},{"pct":-0.23,"symbol":"SZ300014","current":48.69,"mc":89628707484,"name":"亿纬锂能","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":11.7663,"has_follow":false,"indcode":"S6303"},{"pct":-3.13,"symbol":"SZ300012","current":23.54,"mc":39166172400,"name":"华测检测","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":11.396,"has_follow":false,"indcode":"S5101"},{"pct":-2.39,"symbol":"SZ300003","current":33.42,"mc":60327146741,"name":"乐普医疗","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":11.1071,"has_follow":false,"indcode":"S3705"},{"pct":-1.89,"symbol":"SZ002993","current":104,"mc":18758000000,"name":"奥海科技","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":11.4797,"has_follow":false,"indcode":"S2705"},{"pct":0.39,"symbol":"SZ002991","current":123.78,"mc":11540119878,"name":"甘源食品","exchange":"sh_sz","type":11,"areacode":"360000","tick_size":0.01,"niota":21.7423,"has_follow":false,"indcode":"S3404"},{"pct":0.65,"symbol":"SZ002990","current":85.31,"mc":10769534400,"name":"盛视科技","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":20.1652,"has_follow":false,"indcode":"S7101"},{"pct":-3.68,"symbol":"SZ002985","current":149,"mc":22373840000,"name":"北摩高科","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":17.6673,"has_follow":false,"indcode":"S6502"},{"pct":1.09,"symbol":"SZ002984","current":24.96,"mc":16215736742,"name":"森麒麟","exchange":"sh_sz","type":11,"areacode":"370000","tick_size":0.01,"niota":11.6661,"has_follow":false,"indcode":"S2206"},{"pct":-2.62,"symbol":"SZ002982","current":100.17,"mc":10205319600,"name":"湘佳股份","exchange":"sh_sz","type":11,"areacode":"430000","tick_size":0.01,"niota":18.8208,"has_follow":false,"indcode":"S1107"},{"pct":-0.83,"symbol":"SZ002978","current":34.61,"mc":13874600000,"name":"安宁股份","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":18.2697,"has_follow":false,"indcode":"S2103"},{"pct":0.13,"symbol":"SZ002975","current":77.48,"mc":10740941154,"name":"博杰股份","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":22.8358,"has_follow":false,"indcode":"S6402"},{"pct":-0.67,"symbol":"SZ002960","current":41.63,"mc":10249722300,"name":"青鸟消防","exchange":"sh_sz","type":11,"areacode":"130000","tick_size":0.01,"niota":12.167,"has_follow":false,"indcode":"S6402"},{"pct":-1.83,"symbol":"SZ002959","current":121.7,"mc":18969600000,"name":"小熊电器","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":14.9351,"has_follow":false,"indcode":"S3301"},{"pct":-1.64,"symbol":"SZ002950","current":24.6,"mc":15584661666,"name":"奥美医疗","exchange":"sh_sz","type":11,"areacode":"420000","tick_size":0.01,"niota":10.7068,"has_follow":false,"indcode":"S3705"},{"pct":0.29,"symbol":"SZ002938","current":54.41,"mc":125811179315,"name":"鹏鼎控股","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":10.4061,"has_follow":false,"indcode":"S2702"},{"pct":-1.81,"symbol":"SZ002925","current":58.08,"mc":26601286300,"name":"盈趣科技","exchange":"sh_sz","type":11,"areacode":"350000","tick_size":0.01,"niota":19.2915,"has_follow":false,"indcode":"S2705"},{"pct":-1.29,"symbol":"SZ002916","current":115.1,"mc":56331534135,"name":"深南电路","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":11.8925,"has_follow":false,"indcode":"S2702"},{"pct":0.84,"symbol":"SZ002912","current":69.88,"mc":12209483379,"name":"中新赛克","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":13.8769,"has_follow":false,"indcode":"S7101"},{"pct":-1.33,"symbol":"SZ002901","current":88.41,"mc":35537532930,"name":"大博医疗","exchange":"sh_sz","type":11,"areacode":"350000","tick_size":0.01,"niota":24.5249,"has_follow":false,"indcode":"S3705"},{"pct":0.79,"symbol":"SZ002867","current":25.58,"mc":18708879386,"name":"周大生","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":16.7888,"has_follow":false,"indcode":"S3603"},{"pct":-0.13,"symbol":"SZ002851","current":30.96,"mc":15539057253,"name":"麦格米特","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":10.2361,"has_follow":false,"indcode":"S6303"},{"pct":0.13,"symbol":"SZ002841","current":97.82,"mc":65370778955,"name":"视源股份","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":18.9988,"has_follow":false,"indcode":"S2703"},{"pct":-2.22,"symbol":"SZ002821","current":237.11,"mc":55287093846,"name":"凯莱英","exchange":"sh_sz","type":11,"areacode":"120000","tick_size":0.01,"niota":15.9521,"has_follow":false,"indcode":"S3701"},{"pct":-0.32,"symbol":"SZ002818","current":15.54,"mc":11748365797,"name":"富森美","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":13.6177,"has_follow":false,"indcode":"S4505"},{"pct":-1.2,"symbol":"SZ002815","current":17.24,"mc":15232757371,"name":"崇达技术","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":10.1745,"has_follow":false,"indcode":"S2702"},{"pct":-0.38,"symbol":"SZ002803","current":41.93,"mc":15878053724,"name":"吉宏股份","exchange":"sh_sz","type":11,"areacode":"350000","tick_size":0.01,"niota":17.1961,"has_follow":false,"indcode":"S3602"},{"pct":-2.61,"symbol":"SZ002793","current":14.94,"mc":21620038267,"name":"罗欣药业","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":14.7536,"has_follow":false,"indcode":"S3701"},{"pct":-2.26,"symbol":"SZ002773","current":46.31,"mc":40583952653,"name":"康弘药业","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":13.0321,"has_follow":false,"indcode":"S3701"},{"pct":-1.77,"symbol":"SZ002755","current":17.18,"mc":15945794830,"name":"奥赛康","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":23.4544,"has_follow":false,"indcode":"S3701"},{"pct":-4.14,"symbol":"SZ002714","current":75.45,"mc":282511738001,"name":"牧原股份","exchange":"sh_sz","type":11,"areacode":"410000","tick_size":0.01,"niota":15.3187,"has_follow":false,"indcode":"S1107"},{"pct":-0.19,"symbol":"SZ002706","current":26.1,"mc":20465364774,"name":"良信电器","exchange":"sh_sz","type":11,"areacode":"310000","tick_size":0.01,"niota":12.148,"has_follow":false,"indcode":"S6304"},{"pct":-0.85,"symbol":"SZ002697","current":8.18,"mc":11124800000,"name":"红旗连锁","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":10.4225,"has_follow":false,"indcode":"S4503"},{"pct":-3.03,"symbol":"SZ002690","current":47.95,"mc":32414200000,"name":"美亚光电","exchange":"sh_sz","type":11,"areacode":"340000","tick_size":0.01,"niota":19.8722,"has_follow":false,"indcode":"S6402"},{"pct":-0.05,"symbol":"SZ002677","current":20.4,"mc":13172993082,"name":"浙江美大","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":24.4208,"has_follow":false,"indcode":"S3301"},{"pct":-0.9,"symbol":"SZ002653","current":23.08,"mc":24788244833,"name":"海思科","exchange":"sh_sz","type":11,"areacode":"540000","tick_size":0.01,"niota":10.102,"has_follow":false,"indcode":"S3701"},{"pct":0.21,"symbol":"SZ002607","current":33.7,"mc":207841359409,"name":"中公教育","exchange":"sh_sz","type":11,"areacode":"340000","tick_size":0.01,"niota":21.0286,"has_follow":false,"indcode":"S7201"},{"pct":-0.6,"symbol":"SZ002605","current":31.38,"mc":12569329197,"name":"姚记科技","exchange":"sh_sz","type":11,"areacode":"310000","tick_size":0.01,"niota":15.585,"has_follow":false,"indcode":"S3603"},{"pct":-0.94,"symbol":"SZ002602","current":10.52,"mc":78400899303,"name":"世纪华通","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":11.1414,"has_follow":false,"indcode":"S7203"},{"pct":-1.29,"symbol":"SZ002601","current":24.41,"mc":49601629900,"name":"龙蟒佰利","exchange":"sh_sz","type":11,"areacode":"410000","tick_size":0.01,"niota":11.1148,"has_follow":false,"indcode":"S2203"},{"pct":-0.58,"symbol":"SZ002597","current":34.02,"mc":18981611039,"name":"金禾实业","exchange":"sh_sz","type":11,"areacode":"340000","tick_size":0.01,"niota":13.6501,"has_follow":false,"indcode":"S2203"},{"pct":0.83,"symbol":"SZ002595","current":27.96,"mc":22344000000,"name":"豪迈科技","exchange":"sh_sz","type":11,"areacode":"370000","tick_size":0.01,"niota":13.9616,"has_follow":false,"indcode":"S6402"},{"pct":-2.17,"symbol":"SZ002572","current":28.36,"mc":25975174982,"name":"索菲亚","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":13.4367,"has_follow":false,"indcode":"S3603"},{"pct":-3.12,"symbol":"SZ002568","current":54.31,"mc":28277264160,"name":"百润股份","exchange":"sh_sz","type":11,"areacode":"310000","tick_size":0.01,"niota":12.0406,"has_follow":false,"indcode":"S3403"},{"pct":-3.03,"symbol":"SZ002557","current":61.11,"mc":31124730000,"name":"洽洽食品","exchange":"sh_sz","type":11,"areacode":"340000","tick_size":0.01,"niota":11.4546,"has_follow":false,"indcode":"S3404"},{"pct":-1.69,"symbol":"SZ002555","current":41.25,"mc":87130382501,"name":"三七互娱","exchange":"sh_sz","type":11,"areacode":"340000","tick_size":0.01,"niota":26.0702,"has_follow":false,"indcode":"S7203"},{"pct":-2.75,"symbol":"SZ002511","current":20.86,"mc":27353646774,"name":"中顺洁柔","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":10.8096,"has_follow":false,"indcode":"S3601"},{"pct":-1.87,"symbol":"SZ002508","current":35.18,"mc":33386666079,"name":"老板电器","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":16.0563,"has_follow":false,"indcode":"S3301"},{"pct":-0.85,"symbol":"SZ002507","current":45.62,"mc":36002583762,"name":"涪陵榨菜","exchange":"sh_sz","type":11,"areacode":"500000","tick_size":0.01,"niota":19.0885,"has_follow":false,"indcode":"S3404"},{"pct":-2.34,"symbol":"SZ002475","current":54.19,"mc":378735328940,"name":"立讯精密","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":11.4832,"has_follow":false,"indcode":"S2705"},{"pct":-0.65,"symbol":"SZ002468","current":15.35,"mc":23513121270,"name":"申通快递","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":11.0351,"has_follow":false,"indcode":"S4208"},{"pct":-1.73,"symbol":"SZ002463","current":18.15,"mc":31280285272,"name":"沪电股份","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":16.2594,"has_follow":false,"indcode":"S2702"},{"pct":-0.61,"symbol":"SZ002458","current":13.06,"mc":12932813401,"name":"益生股份","exchange":"sh_sz","type":11,"areacode":"370000","tick_size":0.01,"niota":67.5428,"has_follow":false,"indcode":"S1107"},{"pct":-0.31,"symbol":"SZ002440","current":9.58,"mc":11033295000,"name":"闰土股份","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":12.8111,"has_follow":false,"indcode":"S2203"},{"pct":-1.53,"symbol":"SZ002439","current":34.66,"mc":32386020010,"name":"启明星辰","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":11.5607,"has_follow":false,"indcode":"S7102"},{"pct":-0.88,"symbol":"SZ002396","current":27.18,"mc":15836059548,"name":"星网锐捷","exchange":"sh_sz","type":11,"areacode":"350000","tick_size":0.01,"niota":11.6611,"has_follow":false,"indcode":"S7302"},{"pct":-3.77,"symbol":"SZ002372","current":14.03,"mc":22070775222,"name":"伟星新材","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":20.8663,"has_follow":false,"indcode":"S6103"},{"pct":-1.52,"symbol":"SZ002304","current":135.18,"mc":203714637840,"name":"洋河股份","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":14.3393,"has_follow":false,"indcode":"S3403"},{"pct":0.34,"symbol":"SZ002299","current":23.77,"mc":29576999400,"name":"圣农发展","exchange":"sh_sz","type":11,"areacode":"350000","tick_size":0.01,"niota":27.5208,"has_follow":false,"indcode":"S1107"},{"pct":-2.56,"symbol":"SZ002287","current":27.01,"mc":14321268643,"name":"奇正藏药","exchange":"sh_sz","type":11,"areacode":"540000","tick_size":0.01,"niota":13.7586,"has_follow":false,"indcode":"S3702"},{"pct":0.68,"symbol":"SZ002262","current":16.24,"mc":16550604795,"name":"恩华药业","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":15.9645,"has_follow":false,"indcode":"S3701"},{"pct":-1.35,"symbol":"SZ002243","current":13.19,"mc":15943657564,"name":"通产丽星","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":10.525,"has_follow":false,"indcode":"S3602"},{"pct":-2.29,"symbol":"SZ002242","current":42.28,"mc":32435905320,"name":"九阳股份","exchange":"sh_sz","type":11,"areacode":"370000","tick_size":0.01,"niota":11.4067,"has_follow":false,"indcode":"S3301"},{"pct":-6.37,"symbol":"SZ002236","current":21.91,"mc":65801780008,"name":"大华股份","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":11.3059,"has_follow":false,"indcode":"S2705"},{"pct":0.74,"symbol":"SZ002233","current":14.94,"mc":17824511489,"name":"塔牌集团","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":15.2843,"has_follow":false,"indcode":"S6101"},{"pct":-0.55,"symbol":"SZ002223","current":32.45,"mc":32510326807,"name":"鱼跃医疗","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":10.2382,"has_follow":false,"indcode":"S3705"},{"pct":-1.7,"symbol":"SZ002191","current":9.85,"mc":14414325228,"name":"劲嘉股份","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":11.453,"has_follow":false,"indcode":"S3602"},{"pct":1.16,"symbol":"SZ002190","current":27.88,"mc":10004961376,"name":"成飞集成","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":11.6474,"has_follow":false,"indcode":"S6502"},{"pct":0.62,"symbol":"SZ002128","current":9.8,"mc":18831420231,"name":"露天煤业","exchange":"sh_sz","type":11,"areacode":"150000","tick_size":0.01,"niota":10.4995,"has_follow":false,"indcode":"S2102"},{"pct":2.19,"symbol":"SZ002127","current":19.17,"mc":47010768217,"name":"南极电商","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":24.0413,"has_follow":false,"indcode":"S4504"},{"pct":0.33,"symbol":"SZ002120","current":18.42,"mc":53375143677,"name":"韵达股份","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":12.9283,"has_follow":false,"indcode":"S4208"},{"pct":-0.29,"symbol":"SZ002110","current":6.86,"mc":16842328755,"name":"三钢闽光","exchange":"sh_sz","type":11,"areacode":"350000","tick_size":0.01,"niota":12.7112,"has_follow":false,"indcode":"S2301"},{"pct":3.75,"symbol":"SZ002099","current":8.3,"mc":13435336600,"name":"海翔药业","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":11.1849,"has_follow":false,"indcode":"S3701"},{"pct":-2.67,"symbol":"SZ002064","current":7.66,"mc":35539092425,"name":"华峰氨纶","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":16.0674,"has_follow":false,"indcode":"S2204"},{"pct":-1.79,"symbol":"SZ002032","current":79.66,"mc":65391118610,"name":"苏泊尔","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":17.0423,"has_follow":false,"indcode":"S3301"},{"pct":-2.16,"symbol":"SZ002007","current":48.95,"mc":89302751238,"name":"华兰生物","exchange":"sh_sz","type":11,"areacode":"410000","tick_size":0.01,"niota":19.5915,"has_follow":false,"indcode":"S3703"},{"pct":-1,"symbol":"SZ000999","current":26.69,"mc":26126841000,"name":"华润三九","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":11.2189,"has_follow":false,"indcode":"S3702"},{"pct":-0.89,"symbol":"SZ000963","current":24.6,"mc":43027816785,"name":"华东医药","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":14.382,"has_follow":false,"indcode":"S3701"},{"pct":-0.41,"symbol":"SZ000935","current":14.41,"mc":10993540795,"name":"四川双马","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":16.0969,"has_follow":false,"indcode":"S6101"},{"pct":1.02,"symbol":"SZ000895","current":52.72,"mc":174992557057,"name":"双汇发展","exchange":"sh_sz","type":11,"areacode":"410000","tick_size":0.01,"niota":22.2277,"has_follow":false,"indcode":"S3404"},{"pct":-2.27,"symbol":"SZ000877","current":19.36,"mc":20303276486,"name":"天山股份","exchange":"sh_sz","type":11,"areacode":"650000","tick_size":0.01,"niota":11.3208,"has_follow":false,"indcode":"S6101"},{"pct":-1.93,"symbol":"SZ000876","current":30.45,"mc":131056995367,"name":"新希望","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":11.0208,"has_follow":false,"indcode":"S1104"},{"pct":-1.87,"symbol":"SZ000858","current":226.02,"mc":877359857370,"name":"五粮液","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":18.9393,"has_follow":false,"indcode":"S3403"},{"pct":-0.49,"symbol":"SZ000789","current":16.09,"mc":12829742479,"name":"万年青","exchange":"sh_sz","type":11,"areacode":"360000","tick_size":0.01,"niota":18.7978,"has_follow":false,"indcode":"S6101"},{"pct":-2.28,"symbol":"SZ000785","current":9.01,"mc":53696884501,"name":"居然之家","exchange":"sh_sz","type":11,"areacode":"420000","tick_size":0.01,"niota":17.2597,"has_follow":false,"indcode":"S4503"},{"pct":-0.71,"symbol":"SZ000717","current":4.2,"mc":10137807278,"name":"韶钢松山","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":10.8549,"has_follow":false,"indcode":"S2301"},{"pct":-3.85,"symbol":"SZ000710","current":52.38,"mc":18574255209,"name":"贝瑞基因","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":14.7566,"has_follow":false,"indcode":"S3705"},{"pct":0.28,"symbol":"SZ000708","current":17.62,"mc":88829724421,"name":"中信特钢","exchange":"sh_sz","type":11,"areacode":"420000","tick_size":0.01,"niota":13.4284,"has_follow":false,"indcode":"S2301"},{"pct":0.96,"symbol":"SZ000672","current":26.41,"mc":21487700793,"name":"上峰水泥","exchange":"sh_sz","type":11,"areacode":"620000","tick_size":0.01,"niota":28.1403,"has_follow":false,"indcode":"S6101"},{"pct":-5.72,"symbol":"SZ000661","current":361.45,"mc":146286148821,"name":"长春高新","exchange":"sh_sz","type":11,"areacode":"220000","tick_size":0.01,"niota":21.2265,"has_follow":false,"indcode":"S3703"},{"pct":0,"symbol":"SZ000629","current":2.15,"mc":18553851796,"name":"攀钢钒钛","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":11.7308,"has_follow":false,"indcode":"S2103"},{"pct":-2.54,"symbol":"SZ000603","current":15.32,"mc":10577230074,"name":"盛达资源","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":13.9018,"has_follow":false,"indcode":"S2403"},{"pct":-2.58,"symbol":"SZ000596","current":227.9,"mc":114730152000,"name":"古井贡酒","exchange":"sh_sz","type":11,"areacode":"340000","tick_size":0.01,"niota":16.3586,"has_follow":false,"indcode":"S3403"},{"pct":0.42,"symbol":"SZ000581","current":23.87,"mc":24063471095,"name":"威孚高科","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":10.2685,"has_follow":false,"indcode":"S2802"},{"pct":-1.87,"symbol":"SZ000568","current":143.48,"mc":210162685256,"name":"泸州老窖","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":18.0194,"has_follow":false,"indcode":"S3403"},{"pct":-1.02,"symbol":"SZ000538","current":109.22,"mc":139428572051,"name":"云南白药","exchange":"sh_sz","type":11,"areacode":"530000","tick_size":0.01,"niota":10.428,"has_follow":false,"indcode":"S3702"},{"pct":-1.6,"symbol":"SZ000403","current":34.45,"mc":16964374244,"name":"双林生物","exchange":"sh_sz","type":11,"areacode":"140000","tick_size":0.01,"niota":12.192,"has_follow":false,"indcode":"S3703"},{"pct":0.02,"symbol":"SZ000048","current":42.56,"mc":17033450056,"name":"京基智农","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":19.8764,"has_follow":false,"indcode":"S1104"},{"pct":0,"symbol":"SZ000029","current":10.81,"mc":10936044600,"name":"深深房A","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":11.3037,"has_follow":false,"indcode":"S4301"},{"pct":-0.67,"symbol":"SH688588","current":31.18,"mc":12472311894,"name":"凌志软件","exchange":"sh_sz","type":82,"areacode":"320000","tick_size":0.01,"niota":20.2623,"has_follow":false,"indcode":null},{"pct":-1.07,"symbol":"SH688580","current":168.69,"mc":11529399256,"name":"伟思医疗","exchange":"sh_sz","type":82,"areacode":"320000","tick_size":0.01,"niota":27.7558,"has_follow":false,"indcode":null},{"pct":-2.79,"symbol":"SH688568","current":59.18,"mc":13019600000,"name":"中科星图","exchange":"sh_sz","type":82,"areacode":"110000","tick_size":0.01,"niota":19.6641,"has_follow":false,"indcode":null},{"pct":0.49,"symbol":"SH688508","current":109.99,"mc":12406872000,"name":"芯朋微","exchange":"sh_sz","type":82,"areacode":"320000","tick_size":0.01,"niota":14.7576,"has_follow":false,"indcode":null},{"pct":-2.19,"symbol":"SH688505","current":23.62,"mc":24635660000,"name":"复旦张江","exchange":"sh_sz","type":82,"areacode":"310000","tick_size":0.01,"niota":14.543,"has_follow":false,"indcode":null},{"pct":-1.35,"symbol":"SH688399","current":194.09,"mc":11377555800,"name":"硕世生物","exchange":"sh_sz","type":82,"areacode":"320000","tick_size":0.01,"niota":10.8078,"has_follow":false,"indcode":null},{"pct":10.65,"symbol":"SH688390","current":147.5,"mc":12980000000,"name":"固德威","exchange":"sh_sz","type":82,"areacode":"320000","tick_size":0.01,"niota":11.1984,"has_follow":false,"indcode":null},{"pct":-1.89,"symbol":"SH688389","current":25.41,"mc":10728102000,"name":"普门科技","exchange":"sh_sz","type":82,"areacode":"440000","tick_size":0.01,"niota":10.102,"has_follow":false,"indcode":null},{"pct":-0.46,"symbol":"SH688388","current":51.86,"mc":11973229360,"name":"嘉元科技","exchange":"sh_sz","type":82,"areacode":"440000","tick_size":0.01,"niota":17.9828,"has_follow":false,"indcode":null},{"pct":11.88,"symbol":"SH688365","current":44.72,"mc":17932720000,"name":"光云科技","exchange":"sh_sz","type":82,"areacode":"330000","tick_size":0.01,"niota":12.5464,"has_follow":false,"indcode":null},{"pct":0.5,"symbol":"SH688363","current":121.49,"mc":58315200000,"name":"华熙生物","exchange":"sh_sz","type":82,"areacode":"370000","tick_size":0.01,"niota":16.6021,"has_follow":false,"indcode":null},{"pct":-2.5,"symbol":"SH688318","current":229.18,"mc":15279430600,"name":"财富趋势","exchange":"sh_sz","type":82,"areacode":"440000","tick_size":0.01,"niota":17.3437,"has_follow":false,"indcode":null},{"pct":-2.28,"symbol":"SH688311","current":107.69,"mc":12348812300,"name":"盟升电子","exchange":"sh_sz","type":82,"areacode":"510000","tick_size":0.01,"niota":10.0251,"has_follow":false,"indcode":null},{"pct":-1.13,"symbol":"SH688298","current":130.2,"mc":15624000000,"name":"东方生物","exchange":"sh_sz","type":82,"areacode":"330000","tick_size":0.01,"niota":22.1716,"has_follow":false,"indcode":null},{"pct":-3.7,"symbol":"SH688222","current":38.77,"mc":15534363600,"name":"成都先导","exchange":"sh_sz","type":82,"areacode":"510000","tick_size":0.01,"niota":20.9627,"has_follow":false,"indcode":null},{"pct":-2.7,"symbol":"SH688208","current":74.36,"mc":33462000000,"name":"道通科技","exchange":"sh_sz","type":82,"areacode":"440000","tick_size":0.01,"niota":23.0685,"has_follow":false,"indcode":null},{"pct":-1.01,"symbol":"SH688200","current":266,"mc":16275259476,"name":"华峰测控","exchange":"sh_sz","type":82,"areacode":"110000","tick_size":0.01,"niota":26.1618,"has_follow":false,"indcode":null},{"pct":1.15,"symbol":"SH688188","current":218.09,"mc":21809000000,"name":"柏楚电子","exchange":"sh_sz","type":82,"areacode":"310000","tick_size":0.01,"niota":19.0307,"has_follow":false,"indcode":null},{"pct":7.07,"symbol":"SH688169","current":544.98,"mc":36332000182,"name":"石头科技","exchange":"sh_sz","type":82,"areacode":"110000","tick_size":0.01,"niota":48.3162,"has_follow":false,"indcode":null},{"pct":-1.72,"symbol":"SH688106","current":34.91,"mc":16908078994,"name":"金宏气体","exchange":"sh_sz","type":82,"areacode":"320000","tick_size":0.01,"niota":10.969,"has_follow":false,"indcode":null},{"pct":-0.15,"symbol":"SH688100","current":26.34,"mc":13170000000,"name":"威胜信息","exchange":"sh_sz","type":82,"areacode":"430000","tick_size":0.01,"niota":10.0086,"has_follow":false,"indcode":null},{"pct":0.81,"symbol":"SH688095","current":342.97,"mc":16510575800,"name":"福昕软件","exchange":"sh_sz","type":82,"areacode":"350000","tick_size":0.01,"niota":17.8995,"has_follow":false,"indcode":null},{"pct":0.47,"symbol":"SH688088","current":64.3,"mc":26105800000,"name":"虹软科技","exchange":"sh_sz","type":82,"areacode":"330000","tick_size":0.01,"niota":10.6073,"has_follow":false,"indcode":null},{"pct":-4.03,"symbol":"SH688085","current":56.26,"mc":11552062710,"name":"三友医疗","exchange":"sh_sz","type":82,"areacode":"310000","tick_size":0.01,"niota":20.8625,"has_follow":false,"indcode":null},{"pct":-2.16,"symbol":"SH688050","current":203.5,"mc":21395841852,"name":"爱博医疗","exchange":"sh_sz","type":82,"areacode":"110000","tick_size":0.01,"niota":10.3373,"has_follow":false,"indcode":null},{"pct":-4.4,"symbol":"SH688036","current":96.53,"mc":77224000000,"name":"传音控股","exchange":"sh_sz","type":82,"areacode":"440000","tick_size":0.01,"niota":12.7886,"has_follow":false,"indcode":null},{"pct":-1.29,"symbol":"SH688029","current":209.75,"mc":27968065000,"name":"南微医学","exchange":"sh_sz","type":82,"areacode":"320000","tick_size":0.01,"niota":16.3951,"has_follow":false,"indcode":null},{"pct":-0.11,"symbol":"SH688018","current":167,"mc":13360000000,"name":"乐鑫科技","exchange":"sh_sz","type":82,"areacode":"310000","tick_size":0.01,"niota":15.0776,"has_follow":false,"indcode":null},{"pct":0.22,"symbol":"SH688016","current":259.85,"mc":18703521498,"name":"心脉医疗","exchange":"sh_sz","type":82,"areacode":"310000","tick_size":0.01,"niota":19.9022,"has_follow":false,"indcode":null},{"pct":-2.88,"symbol":"SH688008","current":79.37,"mc":89673328370,"name":"澜起科技","exchange":"sh_sz","type":82,"areacode":"310000","tick_size":0.01,"niota":15.5978,"has_follow":false,"indcode":null},{"pct":1.99,"symbol":"SH688002","current":81.29,"mc":36174050000,"name":"睿创微纳","exchange":"sh_sz","type":82,"areacode":"370000","tick_size":0.01,"niota":10.8203,"has_follow":false,"indcode":null},{"pct":0.41,"symbol":"SH688001","current":39.55,"mc":16970367832,"name":"华兴源创","exchange":"sh_sz","type":82,"areacode":"320000","tick_size":0.01,"niota":10.4407,"has_follow":false,"indcode":null},{"pct":0.78,"symbol":"SH605199","current":29.58,"mc":11835216884,"name":"葫芦娃","exchange":"sh_sz","type":11,"areacode":"460000","tick_size":0.01,"niota":11.0466,"has_follow":false,"indcode":"S3702"},{"pct":1.72,"symbol":"SH605168","current":183.5,"mc":12673739450,"name":"三人行","exchange":"sh_sz","type":11,"areacode":"610000","tick_size":0.01,"niota":25.7275,"has_follow":false,"indcode":"S7202"},{"pct":10,"symbol":"SH605009","current":119.33,"mc":12728931100,"name":"豪悦护理","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":29.0368,"has_follow":false,"indcode":"S3601"},{"pct":-0.95,"symbol":"SH603986","current":178.78,"mc":84166164965,"name":"兆易创新","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":13.3994,"has_follow":false,"indcode":"S2701"},{"pct":-0.86,"symbol":"SH603983","current":66.72,"mc":26754720000,"name":"丸美股份","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":18.4693,"has_follow":false,"indcode":"S2203"},{"pct":-2.44,"symbol":"SH603960","current":44,"mc":11481558000,"name":"克来机电","exchange":"sh_sz","type":11,"areacode":"310000","tick_size":0.01,"niota":11.3208,"has_follow":false,"indcode":"S6402"},{"pct":0.49,"symbol":"SH603915","current":24.71,"mc":11448820054,"name":"国茂股份","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":10.2852,"has_follow":false,"indcode":"S6401"},{"pct":-1.05,"symbol":"SH603899","current":68.87,"mc":63871938812,"name":"晨光文具","exchange":"sh_sz","type":11,"areacode":"310000","tick_size":0.01,"niota":16.2517,"has_follow":false,"indcode":"S3603"},{"pct":-0.51,"symbol":"SH603893","current":70.16,"mc":28925564800,"name":"瑞芯微","exchange":"sh_sz","type":11,"areacode":"350000","tick_size":0.01,"niota":10.761,"has_follow":false,"indcode":"S2701"},{"pct":-3.57,"symbol":"SH603868","current":50.73,"mc":22097988000,"name":"飞科电器","exchange":"sh_sz","type":11,"areacode":"310000","tick_size":0.01,"niota":18.4954,"has_follow":false,"indcode":"S3301"},{"pct":-1.11,"symbol":"SH603866","current":56.25,"mc":37061872144,"name":"桃李面包","exchange":"sh_sz","type":11,"areacode":"210000","tick_size":0.01,"niota":15.0245,"has_follow":false,"indcode":"S3404"},{"pct":0.86,"symbol":"SH603833","current":109.26,"mc":64271282023,"name":"欧派家居","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":14.1854,"has_follow":false,"indcode":"S3603"},{"pct":-0.25,"symbol":"SH603816","current":62.72,"mc":39664870918,"name":"顾家家居","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":10.7246,"has_follow":false,"indcode":"S3603"},{"pct":2.55,"symbol":"SH603806","current":68.47,"mc":52691250911,"name":"福斯特","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":12.94,"has_follow":false,"indcode":"S6303"},{"pct":-2.92,"symbol":"SH603786","current":70.85,"mc":28347085000,"name":"科博达","exchange":"sh_sz","type":11,"areacode":"310000","tick_size":0.01,"niota":16.0991,"has_follow":false,"indcode":"S2802"},{"pct":0.21,"symbol":"SH603730","current":28.06,"mc":16298989235,"name":"岱美股份","exchange":"sh_sz","type":11,"areacode":"310000","tick_size":0.01,"niota":12.039,"has_follow":false,"indcode":"S2802"},{"pct":-0.41,"symbol":"SH603707","current":43.82,"mc":40934722142,"name":"健友股份","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":14.2969,"has_follow":false,"indcode":"S3701"},{"pct":-1.7,"symbol":"SH603666","current":83.65,"mc":11597130434,"name":"亿嘉和","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":19.0579,"has_follow":false,"indcode":"S6401"},{"pct":0.64,"symbol":"SH603658","current":157,"mc":67604655143,"name":"安图生物","exchange":"sh_sz","type":11,"areacode":"410000","tick_size":0.01,"niota":22.7643,"has_follow":false,"indcode":"S3705"},{"pct":-1.17,"symbol":"SH603638","current":59.05,"mc":35359707825,"name":"艾迪精密","exchange":"sh_sz","type":11,"areacode":"370000","tick_size":0.01,"niota":15.7338,"has_follow":false,"indcode":"S6402"},{"pct":-0.31,"symbol":"SH603613","current":81.41,"mc":16622436268,"name":"国联股份","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":12.1612,"has_follow":false,"indcode":"S7203"},{"pct":-0.64,"symbol":"SH603609","current":13.98,"mc":12890397346,"name":"禾丰牧业","exchange":"sh_sz","type":11,"areacode":"210000","tick_size":0.01,"niota":18.6246,"has_follow":false,"indcode":"S1104"},{"pct":2.72,"symbol":"SH603606","current":23,"mc":15044403983,"name":"东方电缆","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":12.2944,"has_follow":false,"indcode":"S6304"},{"pct":-1.85,"symbol":"SH603605","current":144,"mc":28982816640,"name":"珀莱雅","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":12.5481,"has_follow":false,"indcode":"S2203"},{"pct":-5.97,"symbol":"SH603596","current":33.38,"mc":13637766180,"name":"伯特利","exchange":"sh_sz","type":11,"areacode":"340000","tick_size":0.01,"niota":11.5146,"has_follow":false,"indcode":"S2802"},{"pct":-0.84,"symbol":"SH603589","current":56.8,"mc":34080000000,"name":"口子窖","exchange":"sh_sz","type":11,"areacode":"340000","tick_size":0.01,"niota":18.7554,"has_follow":false,"indcode":"S3403"},{"pct":1.15,"symbol":"SH603587","current":21.97,"mc":10571964000,"name":"地素时尚","exchange":"sh_sz","type":11,"areacode":"310000","tick_size":0.01,"niota":16.6646,"has_follow":false,"indcode":"S3502"},{"pct":-2.69,"symbol":"SH603583","current":75.47,"mc":18748901235,"name":"捷昌驱动","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":13.8662,"has_follow":false,"indcode":"S6302"},{"pct":-1.04,"symbol":"SH603568","current":22.78,"mc":28624399122,"name":"伟明环保","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":15.3612,"has_follow":false,"indcode":"S4104"},{"pct":-1.1,"symbol":"SH603517","current":77.64,"mc":47254087160,"name":"绝味食品","exchange":"sh_sz","type":11,"areacode":"430000","tick_size":0.01,"niota":17.0387,"has_follow":false,"indcode":"S3404"},{"pct":-1.07,"symbol":"SH603515","current":26,"mc":19657657630,"name":"欧普照明","exchange":"sh_sz","type":11,"areacode":"310000","tick_size":0.01,"niota":11.5378,"has_follow":false,"indcode":"S2703"},{"pct":-0.58,"symbol":"SH603489","current":171.01,"mc":20521200000,"name":"八方股份","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":21.742,"has_follow":false,"indcode":"S6301"},{"pct":1.47,"symbol":"SH603444","current":586.3,"mc":42134186838,"name":"吉比特","exchange":"sh_sz","type":11,"areacode":"350000","tick_size":0.01,"niota":25.456,"has_follow":false,"indcode":"S7203"},{"pct":-1.88,"symbol":"SH603439","current":24.56,"mc":10003833625,"name":"贵州三力","exchange":"sh_sz","type":11,"areacode":"520000","tick_size":0.01,"niota":20.3547,"has_follow":false,"indcode":"S3702"},{"pct":-0.18,"symbol":"SH603429","current":38.73,"mc":14726654805,"name":"集友股份","exchange":"sh_sz","type":11,"areacode":"340000","tick_size":0.01,"niota":15.4753,"has_follow":false,"indcode":"S3602"},{"pct":-1.66,"symbol":"SH603416","current":76.25,"mc":10717700000,"name":"信捷电气","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":12.2455,"has_follow":false,"indcode":"S6302"},{"pct":-2.54,"symbol":"SH603392","current":178.55,"mc":77419280000,"name":"万泰生物","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":10.569,"has_follow":false,"indcode":"S3705"},{"pct":0.44,"symbol":"SH603379","current":20.43,"mc":12472086726,"name":"三美股份","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":14.5229,"has_follow":false,"indcode":"S2203"},{"pct":-2.23,"symbol":"SH603369","current":47.4,"mc":59463300000,"name":"今世缘","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":15.6367,"has_follow":false,"indcode":"S3403"},{"pct":1.03,"symbol":"SH603355","current":32.33,"mc":12964330000,"name":"莱克电气","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":10.1979,"has_follow":false,"indcode":"S3301"},{"pct":-0.21,"symbol":"SH603338","current":101.04,"mc":49053462094,"name":"浙江鼎力","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":16.3447,"has_follow":false,"indcode":"S6402"},{"pct":-7.21,"symbol":"SH603317","current":62,"mc":37261054500,"name":"天味食品","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":16.8297,"has_follow":false,"indcode":"S3404"},{"pct":-0.6,"symbol":"SH603298","current":14.87,"mc":12883306319,"name":"杭叉集团","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":12.0013,"has_follow":false,"indcode":"S6402"},{"pct":-2.42,"symbol":"SH603290","current":173.6,"mc":27776000000,"name":"斯达半导","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":17.1413,"has_follow":false,"indcode":"S2701"},{"pct":-1.35,"symbol":"SH603288","current":160.2,"mc":519119001922,"name":"海天味业","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":23.8598,"has_follow":false,"indcode":"S3404"},{"pct":0.61,"symbol":"SH603267","current":70.39,"mc":16293595640,"name":"鸿远电子","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":14.6181,"has_follow":false,"indcode":"S2702"},{"pct":4.3,"symbol":"SH603208","current":129,"mc":13552853391,"name":"江山欧派","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":10.7245,"has_follow":false,"indcode":"S3603"},{"pct":-0.97,"symbol":"SH603198","current":21.41,"mc":17128000000,"name":"迎驾贡酒","exchange":"sh_sz","type":11,"areacode":"340000","tick_size":0.01,"niota":13.9138,"has_follow":false,"indcode":"S3403"},{"pct":-1.44,"symbol":"SH603195","current":150.2,"mc":90212192760,"name":"公牛集团","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":36.6232,"has_follow":false,"indcode":"S3603"},{"pct":-0.12,"symbol":"SH603160","current":163.3,"mc":74615438861,"name":"汇顶科技","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":35.1274,"has_follow":false,"indcode":"S2701"},{"pct":1.79,"symbol":"SH603156","current":24.98,"mc":31612030128,"name":"养元饮品","exchange":"sh_sz","type":11,"areacode":"130000","tick_size":0.01,"niota":17.7309,"has_follow":false,"indcode":"S3403"},{"pct":-3.17,"symbol":"SH603127","current":90.25,"mc":20463729842,"name":"昭衍新药","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":13.9269,"has_follow":false,"indcode":"S3706"},{"pct":-1.71,"symbol":"SH603087","current":118.07,"mc":66301027800,"name":"甘李药业","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":21.7745,"has_follow":false,"indcode":"S3703"},{"pct":-1.63,"symbol":"SH603043","current":38.51,"mc":15557893046,"name":"广州酒家","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":14.1109,"has_follow":false,"indcode":"S3404"},{"pct":-2.84,"symbol":"SH603026","current":51.4,"mc":10417752000,"name":"石大胜华","exchange":"sh_sz","type":11,"areacode":"370000","tick_size":0.01,"niota":11.068,"has_follow":false,"indcode":"S2203"},{"pct":0,"symbol":"SH601975","current":2.78,"mc":13739083434,"name":"招商南油","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":11.1826,"has_follow":false,"indcode":"S4206"},{"pct":-2.39,"symbol":"SH601888","current":217.26,"mc":424194836689,"name":"中国中免","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":18.8224,"has_follow":false,"indcode":"S4603"},{"pct":-1.3,"symbol":"SH601636","current":7.58,"mc":20362986587,"name":"旗滨集团","exchange":"sh_sz","type":11,"areacode":"430000","tick_size":0.01,"niota":10.3918,"has_follow":false,"indcode":"S6102"},{"pct":-0.63,"symbol":"SH601360","current":17.42,"mc":117829841009,"name":"三六零","exchange":"sh_sz","type":11,"areacode":"120000","tick_size":0.01,"niota":18.8574,"has_follow":false,"indcode":"S7102"},{"pct":1.42,"symbol":"SH601225","current":8.58,"mc":85800000000,"name":"陕西煤业","exchange":"sh_sz","type":11,"areacode":"610000","tick_size":0.01,"niota":13.4559,"has_follow":false,"indcode":"S2102"},{"pct":-0.81,"symbol":"SH601100","current":71.3,"mc":93072168000,"name":"恒立液压","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":16.5388,"has_follow":false,"indcode":"S6401"},{"pct":-0.48,"symbol":"SH601019","current":6.27,"mc":13084863000,"name":"山东出版","exchange":"sh_sz","type":11,"areacode":"370000","tick_size":0.01,"niota":10.0291,"has_follow":false,"indcode":"S7201"},{"pct":1.71,"symbol":"SH601012","current":73.2,"mc":276111626612,"name":"隆基股份","exchange":"sh_sz","type":11,"areacode":"610000","tick_size":0.01,"niota":11.2308,"has_follow":false,"indcode":"S6303"},{"pct":-0.15,"symbol":"SH601006","current":6.45,"mc":95890805117,"name":"大秦铁路","exchange":"sh_sz","type":11,"areacode":"140000","tick_size":0.01,"niota":10.3204,"has_follow":false,"indcode":"S4207"},{"pct":-1.11,"symbol":"SH600989","current":10.7,"mc":78466952000,"name":"宝丰能源","exchange":"sh_sz","type":11,"areacode":"640000","tick_size":0.01,"niota":12.6468,"has_follow":false,"indcode":"S2102"},{"pct":-2.08,"symbol":"SH600887","current":39.1,"mc":237853582670,"name":"伊利股份","exchange":"sh_sz","type":11,"areacode":"150000","tick_size":0.01,"niota":12.8637,"has_follow":false,"indcode":"S3404"},{"pct":-1.27,"symbol":"SH600885","current":46.7,"mc":34780364478,"name":"宏发股份","exchange":"sh_sz","type":11,"areacode":"420000","tick_size":0.01,"niota":10.1027,"has_follow":false,"indcode":"S6304"},{"pct":-2.65,"symbol":"SH600872","current":66.79,"mc":53207398187,"name":"中炬高新","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":13.258,"has_follow":false,"indcode":"S3404"},{"pct":-1.41,"symbol":"SH600867","current":14.69,"mc":29879291315,"name":"通化东宝","exchange":"sh_sz","type":11,"areacode":"220000","tick_size":0.01,"niota":14.9751,"has_follow":false,"indcode":"S3703"},{"pct":-0.89,"symbol":"SH600809","current":207.99,"mc":181269164045,"name":"山西汾酒","exchange":"sh_sz","type":11,"areacode":"140000","tick_size":0.01,"niota":14.7227,"has_follow":false,"indcode":"S3403"},{"pct":1.92,"symbol":"SH600801","current":26.5,"mc":55559896158,"name":"华新水泥","exchange":"sh_sz","type":11,"areacode":"420000","tick_size":0.01,"niota":20.1149,"has_follow":false,"indcode":"S6101"},{"pct":-2.81,"symbol":"SH600779","current":64.6,"mc":31560052091,"name":"水井坊","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":23.1323,"has_follow":false,"indcode":"S3403"},{"pct":-1.27,"symbol":"SH600764","current":39.71,"mc":28219092918,"name":"中国海防","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":14.6823,"has_follow":false,"indcode":"S6503"},{"pct":-4.45,"symbol":"SH600763","current":186.9,"mc":59927616000,"name":"通策医疗","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":21.1903,"has_follow":false,"indcode":"S3706"},{"pct":3.79,"symbol":"SH600732","current":13.69,"mc":27877346570,"name":"爱旭股份","exchange":"sh_sz","type":11,"areacode":"310000","tick_size":0.01,"niota":13.3657,"has_follow":false,"indcode":"S4301"},{"pct":-1.27,"symbol":"SH600720","current":16.39,"mc":12723397722,"name":"祁连山","exchange":"sh_sz","type":11,"areacode":"620000","tick_size":0.01,"niota":12.8106,"has_follow":false,"indcode":"S6101"},{"pct":-0.88,"symbol":"SH600702","current":35.88,"mc":12078786720,"name":"舍得酒业","exchange":"sh_sz","type":11,"areacode":"510000","tick_size":0.01,"niota":10.1399,"has_follow":false,"indcode":"S3403"},{"pct":0.42,"symbol":"SH600612","current":50.2,"mc":26260511753,"name":"老凤祥","exchange":"sh_sz","type":11,"areacode":"310000","tick_size":0.01,"niota":11.2056,"has_follow":false,"indcode":"S3603"},{"pct":0.32,"symbol":"SH600598","current":18.76,"mc":33349275093,"name":"北大荒","exchange":"sh_sz","type":11,"areacode":"230000","tick_size":0.01,"niota":10.539,"has_follow":false,"indcode":"S1101"},{"pct":0.08,"symbol":"SH600585","current":59.24,"mc":313930684780,"name":"海螺水泥","exchange":"sh_sz","type":11,"areacode":"340000","tick_size":0.01,"niota":20.9256,"has_follow":false,"indcode":"S6101"},{"pct":-5.42,"symbol":"SH600570","current":95.05,"mc":99240826168,"name":"恒生电子","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":19.4213,"has_follow":false,"indcode":"S7102"},{"pct":-0.98,"symbol":"SH600566","current":23.27,"mc":18963324390,"name":"济川药业","exchange":"sh_sz","type":11,"areacode":"420000","tick_size":0.01,"niota":19.7179,"has_follow":false,"indcode":"S3702"},{"pct":-1.12,"symbol":"SH600563","current":65.05,"mc":14636250000,"name":"法拉电子","exchange":"sh_sz","type":11,"areacode":"350000","tick_size":0.01,"niota":14.9628,"has_follow":false,"indcode":"S2702"},{"pct":-0.94,"symbol":"SH600556","current":19.05,"mc":32012007001,"name":"天下秀","exchange":"sh_sz","type":11,"areacode":"450000","tick_size":0.01,"niota":25.0384,"has_follow":false,"indcode":"S7202"},{"pct":-1.98,"symbol":"SH600519","current":1725.1,"mc":2167066824780,"name":"贵州茅台","exchange":"sh_sz","type":11,"areacode":"520000","tick_size":0.01,"niota":25.6468,"has_follow":false,"indcode":"S3403"},{"pct":-1.24,"symbol":"SH600516","current":6.36,"mc":24205971540,"name":"方大炭素","exchange":"sh_sz","type":11,"areacode":"620000","tick_size":0.01,"niota":11.2992,"has_follow":false,"indcode":"S2402"},{"pct":-0.18,"symbol":"SH600507","current":5.6,"mc":12073321249,"name":"方大特钢","exchange":"sh_sz","type":11,"areacode":"360000","tick_size":0.01,"niota":15.1979,"has_follow":false,"indcode":"S2301"},{"pct":0.7,"symbol":"SH600486","current":90.28,"mc":27977673324,"name":"扬农化工","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":13.7366,"has_follow":false,"indcode":"S2203"},{"pct":1.95,"symbol":"SH600436","current":255.09,"mc":153900187099,"name":"片仔癀","exchange":"sh_sz","type":11,"areacode":"350000","tick_size":0.01,"niota":17.9323,"has_follow":false,"indcode":"S3702"},{"pct":0.52,"symbol":"SH600426","current":25.36,"mc":41252091260,"name":"华鲁恒升","exchange":"sh_sz","type":11,"areacode":"370000","tick_size":0.01,"niota":13.3157,"has_follow":false,"indcode":"S2203"},{"pct":0.45,"symbol":"SH600398","current":6.69,"mc":28898050142,"name":"海澜之家","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":10.828,"has_follow":false,"indcode":"S3502"},{"pct":-0.79,"symbol":"SH600352","current":13.74,"mc":44700779756,"name":"浙江龙盛","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":10.224,"has_follow":false,"indcode":"S2203"},{"pct":-0.16,"symbol":"SH600309","current":75.28,"mc":236360126005,"name":"万华化学","exchange":"sh_sz","type":11,"areacode":"370000","tick_size":0.01,"niota":12.1918,"has_follow":false,"indcode":"S2203"},{"pct":-2.99,"symbol":"SH600305","current":22.08,"mc":22145269187,"name":"恒顺醋业","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":11.3257,"has_follow":false,"indcode":"S3404"},{"pct":-1.49,"symbol":"SH600276","current":89.74,"mc":476227775601,"name":"恒瑞医药","exchange":"sh_sz","type":11,"areacode":"320000","tick_size":0.01,"niota":21.3409,"has_follow":false,"indcode":"S3701"},{"pct":1.33,"symbol":"SH600273","current":12.18,"mc":17450658014,"name":"嘉化能源","exchange":"sh_sz","type":11,"areacode":"330000","tick_size":0.01,"niota":14.4435,"has_follow":false,"indcode":"S2203"},{"pct":-0.97,"symbol":"SH600271","current":16.34,"mc":30351425979,"name":"航天信息","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":10.8301,"has_follow":false,"indcode":"S7102"},{"pct":-5.96,"symbol":"SH600211","current":74.3,"mc":18421781735,"name":"西藏药业","exchange":"sh_sz","type":11,"areacode":"540000","tick_size":0.01,"niota":11.9778,"has_follow":false,"indcode":"S3702"},{"pct":-1.74,"symbol":"SH600183","current":22.58,"mc":51584196646,"name":"生益科技","exchange":"sh_sz","type":11,"areacode":"440000","tick_size":0.01,"niota":11.0017,"has_follow":false,"indcode":"S2702"},{"pct":-0.4,"symbol":"SH600167","current":12.46,"mc":28509968659,"name":"联美控股","exchange":"sh_sz","type":11,"areacode":"210000","tick_size":0.01,"niota":14.2377,"has_follow":false,"indcode":"S4101"},{"pct":-0.52,"symbol":"SH600161","current":40.27,"mc":50516305565,"name":"天坛生物","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":15.7143,"has_follow":false,"indcode":"S3703"},{"pct":1.96,"symbol":"SH600132","current":100.48,"mc":48629425975,"name":"重庆啤酒","exchange":"sh_sz","type":11,"areacode":"500000","tick_size":0.01,"niota":21.3899,"has_follow":false,"indcode":"S3403"},{"pct":0.12,"symbol":"SH600053","current":24.94,"mc":10812507552,"name":"九鼎投资","exchange":"sh_sz","type":11,"areacode":"360000","tick_size":0.01,"niota":16.1375,"has_follow":false,"indcode":"S4903"},{"pct":0.08,"symbol":"SH600031","current":24.04,"mc":203685880759,"name":"三一重工","exchange":"sh_sz","type":11,"areacode":"110000","tick_size":0.01,"niota":13.9907,"has_follow":false,"indcode":"S6402"},{"pct":-0.54,"symbol":"SH600009","current":74.18,"mc":142941777673,"name":"上海机场","exchange":"sh_sz","type":11,"areacode":"310000","tick_size":0.01,"niota":15.4504,"has_follow":false,"indcode":"S4205"}]},"error_code":0,"error_description":""}`
	json.Unmarshal([]byte(str), &r)
	fmt.Printf("%#v", r.Data.List[1].Pct)
}

```
### fmt
参考：<https://studygolang.com/pkgdoc>

```
// fmt系列方法，基本都可以一次性打印多个数据
/*以Print进行扩展衍生
 * 前面+F：表示输出到指定的IO
 * 前面+S：表示获取返回的string
 * 末尾+ln：表示输出结束换行
 * 末尾+f：表示格式化输出
 */

fmt.Print("Print：打印字符串，输出结束不加换行符")
fmt.Println("Println：打印字符串，输出结束加换行符")
fmt.Printf("Printf：格式化打印字符串,%s", "是的")
fmt.Fprint(os.Stdout, "Fprint将os.Stdout")         // Fprint采用默认格式将其参数格式化并写入第一个参数（是一个io.Writer）
fmt.Sprint("获取指定字符串")                             // 可以进行赋值给变量
fmt.Printf("%T", fmt.Errorf("错误：%s", "这是一个测试错误")) //*errors.errorString，可以直接打印出错误结果
/*以Scan进行扩展衍生
 * 前面+F：表示指定的IO进行切割，赋值给具体的变量
 * 前面+S：表示输入字符串进行切割，赋值给具体的变量
 * 末尾+ln：表示输出结束换行
 * 末尾+f：表示格式化输出
 */
var a, b string
fmt.Scan(&a)                                // 按空格分割默认获取
fmt.Scanf("%s\t%s", &a, &b)                 // 自定义分割获取，这里以tab分割
fmt.Sscanf("haha	neo", "%s\t%s", &a, &b) // 自定义输入的字符串，自定义格式分割，然后赋值给变量
fmt.Fscanf(os.Stdin, "%s\t%s", &a, &b)		// 自定义输出io.reader，自定义格式分割，然后赋值给变量

```

### strings
```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	/*
		func EqualFold(s, t string) bool
		说明：判断两个utf-8编码字符串（将unicode大写、小写、标题三种格式字符视为相同）是否相同。
	 */
	fmt.Println(strings.EqualFold("TiTle","title"))	// 结果为：true

	/*
		func HasPrefix(s, prefix string) bool
		说明：判断s是否有前缀字符串prefix。区分大小写
	 */
	fmt.Println(strings.HasPrefix("abcd","A"))		// 结果为：false

	/*
		func HasSuffix(s, suffix string) bool
		说明：判断s是否有后缀字符串suffix。区分大小写
	 */
	fmt.Println(strings.HasSuffix("abcd","d"))		// 结果为：true
    
    /*
		func Contains(s, substr string) bool
		说明：判断字符串s是否包含子串substr。区分大小写
	 */
	fmt.Println(strings.Contains("abcde","cd"))	// 结果为true
    
    /*
        func ContainsRune(s string, r rune) bool
        说明：判断字符串s是否包含utf-8码值r。区分大小写
     */
    fmt.Println(strings.ContainsRune("中国人",'国'))	// 结果为true

	/*
		func ContainsAny(s, chars string) bool
		说明：判断字符串s是否包含字符串chars中的任一字符。区分大小写
	 */
	fmt.Println(strings.ContainsAny("abcdef","cd"))	// 结果为：true

    /*
		func Count(s, sep string) int
		说明：返回字符串s中有几个不重复的sep子串。区分大小写
	 */
	fmt.Println(strings.Count("abcbcbcb","bcb"))	// 结果：2

    /*
		func Index(s, sep string) int
		说明：子串sep在字符串s中第一次出现的位置，不存在则返回-1。。区分大小写
	 */
	fmt.Println(strings.Index("abcbcbcb","bcb"))	// 结果：1
    //func IndexByte(s string, c byte) int	// 字符c在s中第一次出现的位置，不存在则返回-1。
    //func IndexRune(s string, r rune) int	// unicode码值r在s中第一次出现的位置，不存在则返回-1。
    //func IndexAny(s, chars string) int	// 字符串chars中的任一utf-8码值在s中第一次出现的位置，如果不存在或者chars为空字符串则返回-1。
    //func IndexFunc(s string, f func(rune) bool) int	// s中第一个满足函数f的位置i（该处的utf-8码值r满足f(r)==true），不存在则返回-1。
    //func LastIndex(s, sep string) int	// 子串sep在字符串s中最后一次出现的位置，不存在则返回-1。
    //func LastIndexAny(s, chars string) int	// 字符串chars中的任一utf-8码值在s中最后一次出现的位置，如不存在或者chars为空字符串则返回-1。
    //func LastIndexFunc(s string, f func(rune) bool) int	// s中最后一个满足函数f的unicode码值的位置i，不存在则返回-1。

	/*
		func Title(s, sep string) int
		说明：返回s中每个单词的首字母都改为标题格式的字符串拷贝。
		BUG: Title用于划分单词的规则不能很好的处理Unicode标点符号。
	 */
	fmt.Println(strings.Title("this is a test!"))	// 结果：This Is A Test!

    /*
        func ToLower(s string) string
        说明：返回将所有字母都转为对应的小写版本的拷贝。
     */
    fmt.Println(strings.ToLower("This Is A Test!"))	// 结果：this is a test!
	//func ToLowerSpecial(_case unicode.SpecialCase, s string) string	// 使用_case规定的字符映射，返回将所有字母都转为对应的小写版本的拷贝。
	//func ToUpper(s string) string	// 返回将所有字母都转为对应的大写版本的拷贝。
	//func ToUpperSpecial(_case unicode.SpecialCase, s string) string	// 使用_case规定的字符映射，返回将所有字母都转为对应的大写版本的拷贝。
	//func ToTitle(s string) string	// 返回将所有字母都转为对应的标题版本的拷贝。
	//func ToTitleSpecial(_case unicode.SpecialCase, s string) string	// 使用_case规定的字符映射，返回将所有字母都转为对应的标题版本的拷贝。
	
/*
		func Repeat(s string, count int) string
		说明：返回count个s串联的字符串。
	 */
	fmt.Println(strings.Repeat("abc",3))	// 结果：abcabcabc

    /*
        func Replace(s, old, new string, n int) string
        说明：返回将s中前n个不重叠old子串都替换为new的新字符串，如果n<0会替换所有old子串。
    */
    fmt.Println(strings.Replace("abcde", "b", "B", 1)) // 结果：abcabcabc
	
    /*
		func Map(mapping func(rune) rune, s string) string
		说明：将s的每一个unicode码值r都替换为mapping(r)，返回这些新码值组成的字符串拷贝。
			 如果mapping返回一个负值，将会丢弃该码值而不会被替换。（返回值中对应位置将没有码值）
	*/
	rot13 := func(r rune) rune {
        switch {
        case r >= 'A' && r <= 'Z':
            return 'A' + (r-'A'+13)%26
        case r >= 'a' && r <= 'z':
            return 'a' + (r-'a'+13)%26
        }
        return r
    }
    fmt.Println(strings.Map(rot13, "'Twas brillig and the slithy gopher..."))

    /*
        func Trim(s string, cutset string) string
        说明：返回将s前后端所有cutset包含的utf-8码值都去掉的字符串。
    */
    fmt.Println(strings.Trim("abcef", "acf")) // 结果：bce
    //func TrimSpace(s string) string	// 返回将s前后端所有空白（unicode.IsSpace指定）都去掉的字符串。
    //func TrimFunc(s string, f func(rune) bool) string	// 返回将s前后端所有满足f的unicode码值都去掉的字符串。
    //func TrimLeft(s string, cutset string) string	// 返回将s前端所有cutset包含的utf-8码值都去掉的字符串。
    //func TrimLeftFunc(s string, f func(rune) bool) string	// 返回将s前端所有满足f的unicode码值都去掉的字符串。
    //func TrimPrefix(s, prefix string) string	// 返回去除s可能的前缀prefix的字符串。
    //func TrimRight(s string, cutset string) string	// 返回将s后端所有cutset包含的utf-8码值都去掉的字符串。
    //func TrimRightFunc(s string, f func(rune) bool) string	// 返回将s后端所有满足f的unicode码值都去掉的字符串。
    //func TrimSuffix(s, suffix string) string	// 返回去除s可能的后缀suffix的字符串。

    /*
        func Fields(s string) []string
        说明：返回将字符串按照空白（unicode.IsSpace确定，可以是一到多个连续的空白字符）分割的多个字符串。
             如果字符串全部是空白或者是空字符串的话，会返回空切片。
    */
    fmt.Println(strings.Fields("ab ce f")) // 结果：[ab ce f]
    //func FieldsFunc(s string, f func(rune) bool) []string	// 类似Fields，但使用函数f来确定分割符（满足f的unicode码值）。如果字符串全部是分隔符或者是空字符串的话，会返回空切片。
    
    /*
        func Split(s, sep string) []string
        说明：用去掉s中出现的sep的方式进行分割，会分割到结尾，并返回生成的所有片段组成的切片（每一个sep都会进行一次切割，即使两个sep相邻，
            也会进行两次切割）。如果sep为空字符，Split会将s切分成每一个unicode码值一个字符串。
    */
    fmt.Println(strings.Split("ab ce f"," ")) // 结果：[ab ce f]
    //func SplitN(s, sep string, n int) []string	// 在Split()功能基础上，增加。参数n决定返回的切片的数目
    //n > 0 : 返回的切片最多n个子字符串；最后一个子字符串包含未进行切割的部分。
    //n == 0: 返回nil
    //n < 0 : 返回所有的子字符串组成的切片
    //func SplitAfter(s, sep string) []string
    //func SplitAfterN(s, sep string, n int) []string
    
    /*
        func Join(a []string, sep string) string
        说明：将一系列字符串连接为一个字符串，之间用sep来分隔。
    */
    fmt.Println(strings.Join([]string{"a","b","c"}," ")) // 结果：a b c

    //type Reader	// 操作读取字符
    //func NewReader(s string) *Reader
    //func (r *Reader) Len() int
    //func (r *Reader) Read(b []byte) (n int, err error)
    //func (r *Reader) ReadByte() (b byte, err error)
    //func (r *Reader) UnreadByte() error
    //func (r *Reader) ReadRune() (ch rune, size int, err error)
    //func (r *Reader) UnreadRune() error
    //func (r *Reader) Seek(offset int64, whence int) (int64, error)
    //func (r *Reader) ReadAt(b []byte, off int64) (n int, err error)
    //func (r *Reader) WriteTo(w io.Writer) (n int64, err error)
    //type Replacer	// 替换字符串
    //func NewReplacer(oldnew ...string) *Replacer
    //func (r *Replacer) Replace(s string) string
    //func (r *Replacer) WriteString(w io.Writer, s string) (n int, err error)

}
```
### strconv

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	/*
	func IsPrint(r rune) bool
	说明：返回一个字符是否是可打印的，和unicode.IsPrint一样，r必须是：字母（广义）、数字、标点、符号、ASCII空格。
	 */
	fmt.Println(strconv.IsPrint('.'))
    
    /*
    func Quote(s string) string
    说明：返回字符串s在go语法下的双引号字面值表示，控制字符、不可打印字符会进行转义。（如\t，\n，\xFF，\u0100）
     */
    fmt.Println(strconv.Quote("this is	a test!"))
    //func CanBackquote(s string) bool
    //func Quote(s string) string
    //func QuoteToASCII(s string) string
    //func QuoteRune(r rune) string
    //func QuoteRuneToASCII(r rune) string
    //func Unquote(s string) (t string, err error)
    //func UnquoteChar(s string, quote byte) (value rune, multibyte bool, tail string, err error)
    
    /*
	func ParseBool(str string) (value bool, err error)
	说明：返回字符串s在go语法下的双引号字面值表示，控制字符、不可打印字符会进行转义。（如\t，\n，\xFF，\u0100）
	 */
	fmt.Println(strconv.ParseBool("1"))

	//func ParseBool(str string) (value bool, err error)
	//func ParseInt(s string, base int, bitSize int) (i int64, err error)
	//func ParseUint(s string, base int, bitSize int) (n uint64, err error)
	//func ParseFloat(s string, bitSize int) (f float64, err error)
	
    /*
    func FormatInt(i int64, base int) string
    说明：返回i的base进制的字符串表示。base 必须在2到36之间，结果中会使用小写字母'a'到'z'表示大于10的数字。
     */
    fmt.Println(strconv.FormatInt(12,2))

	//func FormatInt(i int64, base int) string
	//func FormatUint(i uint64, base int) string
	//func FormatFloat(f float64, fmt byte, prec, bitSize int) string
	
    /*
	func Atoi(s string) (i int, err error)
	说明：Atoi是ParseInt(s, 10, 0)的简写。
	 */
	fmt.Println(strconv.Atoi("12"))

    /*
    func Itoa(i int) string
    说明：Itoa是FormatInt(i, 10) 的简写。
     */
    fmt.Println(strconv.Itoa(12))
	
    //func AppendBool(dst []byte, b bool) []byte	// 等价于append(dst, FormatBool(b)...)
	//func AppendInt(dst []byte, i int64, base int) []byte
	//func AppendUint(dst []byte, i uint64, base int) []byte
	//func AppendFloat(dst []byte, f float64, fmt byte, prec int, bitSize int) []byte
	//func AppendQuote(dst []byte, s string) []byte
	//func AppendQuoteToASCII(dst []byte, s string) []byte
	//func AppendQuoteRune(dst []byte, r rune) []byte
	//func AppendQuoteRuneToASCII(dst []byte, r rune) []byte


}
```
### regexp

```go
package main

import (
	"bytes"
	"fmt"
	"regexp"
)

func main() {
	//是否匹配字符串
	// .匹配任意一个字符 ，*匹配零个或多个 ，优先匹配更多(贪婪)
	match, _ := regexp.MatchString("H(.*)d!", "Hello World!")
	fmt.Println(match) //true
	//或
	match, _ = regexp.Match("H(.*)d!", []byte("Hello World!"))
	fmt.Println(match) //true
	//或通过`Compile`来使用一个优化过的正则对象
	r, _ := regexp.Compile("H(.*)d!")
	fmt.Println(r.MatchString("Hello World!")) //true

	// 这个方法返回匹配的子串
	fmt.Println(r.FindString("Hello World! world")) //Hello World!
	//同上
	fmt.Println(string(r.Find([]byte("Hello World!")))) //Hello World!

	// 这个方法查找第一次匹配的索引
	// 的起始索引和结束索引，而不是匹配的字符串
	fmt.Println(r.FindStringIndex("Hello World! world")) //[0 12]

	// 这个方法返回全局匹配的字符串和局部匹配的字符，匹配最大的子字符串一次。
	// 它和r.FindAllStringSubmatch("Hello World! world"，1) 等价。  比如
	// 这里会返回匹配`H(.*)d!`的字符串
	// 和匹配`(.*)`的字符串
	fmt.Println(r.FindStringSubmatch("Hello World! world")) //[Hello World! ello Worl]

	// 和上面的方法一样，不同的是返回全局匹配和局部匹配的
	// 起始索引和结束索引
	fmt.Println(r.FindStringSubmatchIndex("Hello World! world")) //[0 12 1 10]
	// 这个方法返回所有正则匹配的字符，不仅仅是第一个
	fmt.Println(r.FindAllString("Hello World! Held! world", -1)) //[Hello World! Held!]

	// 这个方法返回所有全局匹配和局部匹配的字符串起始索引,只匹配最大的串
	// 和结束索引
	fmt.Println(r.FindAllStringSubmatchIndex("Hello World! world", -1))       //[[0 12 1 10]]
	fmt.Println(r.FindAllStringSubmatchIndex("Hello World! Held! world", -1)) //[[0 18 1 16]]

	// 为这个方法提供一个正整数参数来限制匹配数量
	res, _ := regexp.Compile("H([a-z]+)d!")
	fmt.Println(res.FindAllString("Hello World! Held! Hellowrld! world", 2)) //[Held! Hellowrld!]

	fmt.Println(r.FindAllString("Hello World! Held! world", 2)) //[Hello World! Held!]
	//注意上面两个不同，第二参数是一最大子串为单位计算。

	// regexp包也可以用来将字符串的一部分替换为其他的值
	fmt.Println(r.ReplaceAllString("Hello World! Held! world", "html")) //html world

	// `Func`变量可以让你将所有匹配的字符串都经过该函数处理
	// 转变为所需要的值
	in := []byte("Hello World! Held! world")
	out := r.ReplaceAllFunc(in, bytes.ToUpper)
	fmt.Println(string(out))

	// 在 b 中查找 reg 中编译好的正则表达式，并返回第一个匹配的位置
	// {起始位置, 结束位置}
	b := bytes.NewReader([]byte("Hello World!"))
	reg := regexp.MustCompile(`\w+`)
	fmt.Println(reg.FindReaderIndex(b)) //[0 5]

	// 在 字符串 中查找 r 中编译好的正则表达式，并返回所有匹配的位置
	// {{起始位置, 结束位置}, {起始位置, 结束位置}, ...}
	// 只查找前 n 个匹配项，如果 n < 0，则查找所有匹配项

	fmt.Println(r.FindAllIndex([]byte("Hello World!"), -1)) //[[0 12]]
	//同上
	fmt.Println(r.FindAllStringIndex("Hello World!", -1)) //[[0 12]]

	// 在 s 中查找 re 中编译好的正则表达式，并返回所有匹配的内容
	// 同时返回子表达式匹配的内容
	// {
	//     {完整匹配项, 子匹配项, 子匹配项, ...},
	//     {完整匹配项, 子匹配项, 子匹配项, ...},
	//     ...
	// }
	// 只查找前 n 个匹配项，如果 n < 0，则查找所有匹配项
	reg = regexp.MustCompile(`(\w)(\w)+`)                      //[[Hello H o] [World W d]]
	fmt.Println(reg.FindAllStringSubmatch("Hello World!", -1)) //[[Hello H o] [World W d]]

	// 将 template 的内容经过处理后，追加到 dst 的尾部。
	// template 中要有 $1、$2、${name1}、${name2} 这样的“分组引用符”
	// match 是由 FindSubmatchIndex 方法返回的结果，里面存放了各个分组的位置信息
	// 如果 template 中有“分组引用符”，则以 match 为标准，
	// 在 src 中取出相应的子串，替换掉 template 中的 $1、$2 等引用符号。
	reg = regexp.MustCompile(`(\w+),(\w+)`)
	src := []byte("Golang,World!")           // 源文本
	dst := []byte("Say: ")                   // 目标文本
	template := []byte("Hello $1, Hello $2") // 模板
	m := reg.FindSubmatchIndex(src)          // 解析源文本
	// 填写模板，并将模板追加到目标文本中
	fmt.Printf("%q", reg.Expand(dst, template, src, m))
	// "Say: Hello Golang, Hello World"

	// LiteralPrefix 返回所有匹配项都共同拥有的前缀（去除可变元素）
	// prefix：共同拥有的前缀
	// complete：如果 prefix 就是正则表达式本身，则返回 true，否则返回 false
	reg = regexp.MustCompile(`Hello[\w\s]+`)
	fmt.Println(reg.LiteralPrefix())
	// Hello false
	reg = regexp.MustCompile(`Hello`)
	fmt.Println(reg.LiteralPrefix())
	// Hello true

	text := `Hello World! hello world`
	// 正则标记“非贪婪模式”(?U)
	reg = regexp.MustCompile(`(?U)H[\w\s]+o`)
	fmt.Printf("%q\n", reg.FindString(text)) // Hello
	// 切换到“贪婪模式”
	reg.Longest()
	fmt.Printf("%q\n", reg.FindString(text)) // Hello Wo

	// 统计正则表达式中的分组个数（不包括“非捕获的分组”）
	fmt.Println(r.NumSubexp()) //1

	//返回 r 中的“正则表达式”字符串
	fmt.Printf("%s\n", r.String())

	// 在 字符串 中搜索匹配项，并以匹配项为分割符，将 字符串 分割成多个子串
	// 最多分割出 n 个子串，第 n 个子串不再进行分割
	// 如果 n < 0，则分割所有子串
	// 返回分割后的子串列表
	fmt.Printf("%q\n", r.Split("Hello World! Helld! hello", -1)) //["" " hello"]

	// 在 字符串 中搜索匹配项，并替换为 repl 指定的内容
	// 如果 rep 中有“分组引用符”（$1、$name），则将“分组引用符”当普通字符处理
	// 全部替换，并返回替换后的结果
	s := "Hello World, hello!"
	reg = regexp.MustCompile(`(Hell|h)o`)
	rep := "${1}"
	fmt.Printf("%q\n", reg.ReplaceAllLiteralString(s, rep)) //"${1} World, hello!"

	// 在 字符串 中搜索匹配项，然后将匹配的内容经过 repl 处理后，替换 字符串 中的匹配项
	// 如果 repb 的返回值中有“分组引用符”（$1、$name），则将“分组引用符”当普通字符处理
	// 全部替换，并返回替换后的结果
	ss := []byte("Hello World!")
	reg = regexp.MustCompile("(H)ello")
	repb := []byte("$0$1")
	fmt.Printf("%s\n", reg.ReplaceAll(ss, repb))
	// HelloH World!

	fmt.Printf("%s\n", reg.ReplaceAllFunc(ss,
		func(b []byte) []byte {
			rst := []byte{}
			rst = append(rst, b...)
			rst = append(rst, "$1"...)
			return rst
		}))
	// Hello$1 World!

}


```

### os
参考：<https://studygolang.com/pkgdoc>
Constants
Variables
func Hostname() (name string, err error)
func Getpagesize() int
func Environ() []string
func Getenv(key string) string
func Setenv(key, value string) error
func Clearenv()
func Exit(code int) // 不会执行defer哦
func Expand(s string, mapping func(string) string) string
func ExpandEnv(s string) string
func Getuid() int
func Geteuid() int
func Getgid() int
func Getegid() int
func Getgroups() ([]int, error)
func Getpid() int
func Getppid() int
type Signal
type PathError
    func (e *PathError) Error() string
type LinkError
    func (e *LinkError) Error() string
type SyscallError
    func (e *SyscallError) Error() string
    func NewSyscallError(syscall string, err error) error
type FileMode
    func (m FileMode) IsDir() bool
    func (m FileMode) IsRegular() bool
    func (m FileMode) Perm() FileMode
    func (m FileMode) String() string
type FileInfo
    func Stat(name string) (fi FileInfo, err error)
    func Lstat(name string) (fi FileInfo, err error)
func IsPathSeparator(c uint8) bool
func IsExist(err error) bool
func IsNotExist(err error) bool
func IsPermission(err error) bool
func Getwd() (dir string, err error)
func Chdir(dir string) error
func Chmod(name string, mode FileMode) error
func Chown(name string, uid, gid int) error
func Lchown(name string, uid, gid int) error
func Chtimes(name string, atime time.Time, mtime time.Time) error
func Mkdir(name string, perm FileMode) error
func MkdirAll(path string, perm FileMode) error
func Rename(oldpath, newpath string) error
func Truncate(name string, size int64) error
func Remove(name string) error
func RemoveAll(path string) error
func Readlink(name string) (string, error)
func Symlink(oldname, newname string) error
func Link(oldname, newname string) error
func SameFile(fi1, fi2 FileInfo) bool
func TempDir() string
type File
    func Create(name string) (file *File, err error)
    func Open(name string) (file *File, err error)
    func OpenFile(name string, flag int, perm FileMode) (file *File, err error)
    func NewFile(fd uintptr, name string) *File
    func Pipe() (r *File, w *File, err error)
    func (f *File) Name() string
    func (f *File) Stat() (fi FileInfo, err error)
    func (f *File) Fd() uintptr
    func (f *File) Chdir() error
    func (f *File) Chmod(mode FileMode) error
    func (f *File) Chown(uid, gid int) error
    func (f *File) Readdir(n int) (fi []FileInfo, err error)
    func (f *File) Readdirnames(n int) (names []string, err error)
    func (f *File) Truncate(size int64) error
    func (f *File) Read(b []byte) (n int, err error)
    func (f *File) ReadAt(b []byte, off int64) (n int, err error)
    func (f *File) Write(b []byte) (n int, err error)
    func (f *File) WriteString(s string) (ret int, err error)
    func (f *File) WriteAt(b []byte, off int64) (n int, err error)
    func (f *File) Seek(offset int64, whence int) (ret int64, err error)
    func (f *File) Sync() (err error)
    func (f *File) Close() error
type ProcAttr
type Process
    func FindProcess(pid int) (p *Process, err error)
    func StartProcess(name string, argv []string, attr *ProcAttr) (*Process, error)
    func (p *Process) Signal(sig Signal) error
    func (p *Process) Kill() error
    func (p *Process) Wait() (*ProcessState, error)
    func (p *Process) Release() error
type ProcessState
    func (p *ProcessState) Pid() int
    func (p *ProcessState) Exited() bool
    func (p *ProcessState) Success() bool
    func (p *ProcessState) SystemTime() time.Duration
    func (p *ProcessState) UserTime() time.Duration
    func (p *ProcessState) Sys() interface{}
    func (p *ProcessState) SysUsage() interface{}
    func (p *ProcessState) String() string


### io
说明：io包主要以定义接口为主，当然也有函数
#### 变量
- io.EOF
#### 方法
Variables
type Reader
type Writer
type Closer
type Seeker
type ReadCloser
type ReadSeeker
type WriteCloser
type WriteSeeker
type ReadWriter
type ReadWriteCloser
type ReadWriteSeeker
type ReaderAt
type WriterAt
type ByteReader
type ByteScanner
type RuneReader
type RuneScanner
type ByteWriter
type ReaderFrom
type WriterTo
type LimitedReader
    func LimitReader(r Reader, n int64) Reader
    func (l *LimitedReader) Read(p []byte) (n int, err error)
type SectionReader
    func NewSectionReader(r ReaderAt, off int64, n int64) *SectionReader
    func (s *SectionReader) Size() int64
    func (s *SectionReader) Read(p []byte) (n int, err error)
    func (s *SectionReader) ReadAt(p []byte, off int64) (n int, err error)
    func (s *SectionReader) Seek(offset int64, whence int) (int64, error)
type PipeReader
    func Pipe() (*PipeReader, *PipeWriter)
    func (r *PipeReader) Read(data []byte) (n int, err error)
    func (r *PipeReader) Close() error
    func (r *PipeReader) CloseWithError(err error) error
type PipeWriter
    func (w *PipeWriter) Write(data []byte) (n int, err error)
    func (w *PipeWriter) Close() error
    func (w *PipeWriter) CloseWithError(err error) error
func TeeReader(r Reader, w Writer) Reader TeeReader返回一个将其从r读取的数据写入w的Reader接口。所有通过该接口对r的读取都会执行对应的对w的写入。没有内部的缓冲：写入必须在读取完成前完成。写入时遇到的任何错误都会作为读取错误返回。
说明：w类似于一个钩子，可以自定义操作；参考：<https://www.twle.cn/t/385>
func MultiReader(readers ...Reader) Reader
func MultiWriter(writers ...Writer) Writer
func Copy(dst Writer, src Reader) (written int64, err error)    将src的数据拷贝到dst，直到在src上到达EOF或发生错误。返回拷贝的字节数和遇到的第一个错误。
func CopyN(dst Writer, src Reader, n int64) (written int64, err error)
func ReadAtLeast(r Reader, buf []byte, min int) (n int, err error)
func ReadFull(r Reader, buf []byte) (n int, err error)
func WriteString(w Writer, s string) (n int, err error)
参考：<https://studygolang.com/pkgdoc>


### bufio
type Reader
    func NewReader(rd io.Reader) *Reader    创造reader
    func NewReaderSize(rd io.Reader, size int) *Reader
    func (b *Reader) Reset(r io.Reader)
    func (b *Reader) Buffered() int
    func (b *Reader) Peek(n int) ([]byte, error)
    func (b *Reader) Read(p []byte) (n int, err error)  读取内容
    func (b *Reader) ReadByte() (c byte, err error)
    func (b *Reader) UnreadByte() error
    func (b *Reader) ReadRune() (r rune, size int, err error)
    func (b *Reader) UnreadRune() error
    func (b *Reader) ReadLine() (line []byte, isPrefix bool, err error)
    func (b *Reader) ReadSlice(delim byte) (line []byte, err error)
    func (b *Reader) ReadBytes(delim byte) (line []byte, err error)
    func (b *Reader) ReadString(delim byte) (line string, err error)
    func (b *Reader) WriteTo(w io.Writer) (n int64, err error)
type Writer
    func NewWriter(w io.Writer) *Writer // 创造writer
    func NewWriterSize(w io.Writer, size int) *Writer
    func (b *Writer) Reset(w io.Writer)
    func (b *Writer) Buffered() int
    func (b *Writer) Available() int
    func (b *Writer) Write(p []byte) (nn int, err error)    // 写入内容
    func (b *Writer) WriteString(s string) (int, error)
    func (b *Writer) WriteByte(c byte) error
    func (b *Writer) WriteRune(r rune) (size int, err error)
    func (b *Writer) Flush() error
    func (b *Writer) ReadFrom(r io.Reader) (n int64, err error)
type ReadWriter
    func NewReadWriter(r *Reader, w *Writer) *ReadWriter
type SplitFunc
    func ScanBytes(data []byte, atEOF bool) (advance int, token []byte, err error)
    func ScanRunes(data []byte, atEOF bool) (advance int, token []byte, err error)
    func ScanWords(data []byte, atEOF bool) (advance int, token []byte, err error)
    func ScanLines(data []byte, atEOF bool) (advance int, token []byte, err error)
type Scanner
    func NewScanner(r io.Reader) *Scanner   // 主要用于切割输入文本
    func (s *Scanner) Split(split SplitFunc)
    func (s *Scanner) Scan() bool
    func (s *Scanner) Bytes() []byte
    func (s *Scanner) Text() string
    func (s *Scanner) Err() error

### ioutil
func NopCloser(r io.Reader) io.ReadCloser
func ReadAll(r io.Reader) ([]byte, error)   与ReadFile的形参不同
func ReadFile(filename string) ([]byte, error)
func WriteFile(filename string, data []byte, perm os.FileMode) error
func ReadDir(dirname string) ([]os.FileInfo, error)
func TempDir(dir, prefix string) (name string, err error)
func TempFile(dir, prefix string) (f *os.File, err error)



### runtime
详情：<https://studygolang.com/pkgdoc>
- runtime.Stack()
```go
package main

import "runtime/debug"

func main() {
	println(string(debug.Stack()))
}

```

### time
Constants
type ParseError
    func (e *ParseError) Error() string
type Weekday
    func (d Weekday) String() string
type Month
    func (m Month) String() string
    type Location
    func LoadLocation(name string) (*Location, error)
    func FixedZone(name string, offset int) *Location
    func (l *Location) String() string
type Time
    func Date(year int, month Month, day, hour, min, sec, nsec int, loc *Location) Time 
    func Parse(layout, value string) (Time, error)  用相应的格式解析日期字符串
    func ParseInLocation(layout, value string, loc *Location) (Time, error)
    func Now() Time
    func Unix(sec int64, nsec int64) Time   时间戳转化为time
    func (t Time) Location() *Location
    func (t Time) Zone() (name string, offset int)
    func (t Time) IsZero() bool
    func (t Time) Local() Time
    func (t Time) UTC() Time
    func (t Time) In(loc *Location) Time
    func (t Time) Unix() int64  time转化为秒级时间戳
    func (t Time) UnixNano() int64
    func (t Time) Equal(u Time) bool
    func (t Time) Before(u Time) bool
    func (t Time) After(u Time) bool
    func (t Time) Date() (year int, month Month, day int)
    func (t Time) Clock() (hour, min, sec int)
    func (t Time) Year() int
    func (t Time) Month() Month
    func (t Time) ISOWeek() (year, week int)
    func (t Time) YearDay() int
    func (t Time) Day() int
    func (t Time) Weekday() Weekday
    func (t Time) Hour() int
    func (t Time) Minute() int
    func (t Time) Second() int
    func (t Time) Nanosecond() int
    func (t Time) Add(d Duration) Time  更改时间
    func (t Time) AddDate(years int, months int, days int) Time 以年月日更改时间
    func (t Time) Sub(u Time) Duration
    func (t Time) Round(d Duration) Time
    func (t Time) Truncate(d Duration) Time
    func (t Time) Format(layout string) string 格式化时间
    func (t Time) String() string 返回time对应字符串
    func (t Time) GobEncode() ([]byte, error)
    func (t *Time) GobDecode(data []byte) error
    func (t Time) MarshalBinary() ([]byte, error)
    func (t *Time) UnmarshalBinary(data []byte) error
    func (t Time) MarshalJSON() ([]byte, error)
    func (t *Time) UnmarshalJSON(data []byte) error
    func (t Time) MarshalText() ([]byte, error)
    func (t *Time) UnmarshalText(data []byte) error
type Duration
    func ParseDuration(s string) (Duration, error)
    func Since(t Time) Duration
    func (d Duration) Hours() float64
    func (d Duration) Minutes() float64
    func (d Duration) Seconds() float64
    func (d Duration) Nanoseconds() int64
    func (d Duration) String() string
type Timer
    func NewTimer(d Duration) *Timer
    ```
    t := time.NewTimer(time.Second * 1)
    c := <-t.C      // 主要是C字段起作用
    println(c.String())
    ```
    func AfterFunc(d Duration, f func()) *Timer
    func (t *Timer) Reset(d Duration) bool
    func (t *Timer) Stop() bool
type Ticker
    func NewTicker(d Duration) *Ticker // 主要是C字段起作用
    func (t *Ticker) Stop()
func Sleep(d Duration)  程序休眠一段时间
func After(d Duration) <-chan Time 过一段时间，返回一个管道值
func Tick(d Duration) <-chan Time

### flag
说明：解析命令行参数的包

```go
package main

import (
	"flag"
	"fmt"
	"os"
)

// 实际中应该用更好的变量名
var (
	h bool

	v, V bool
	t, T bool
	q    *bool

	s string
	p string
	c string
	g string
)

func init() {
	flag.BoolVar(&h, "h", false, "this help")

	flag.BoolVar(&v, "v", false, "show version and exit")
	flag.BoolVar(&V, "V", false, "show version and configure options then exit")

	flag.BoolVar(&t, "t", false, "test configuration and exit")
	flag.BoolVar(&T, "T", false, "test configuration, dump it and exit")

	// 另一种绑定方式
	q = flag.Bool("q", false, "suppress non-error messages during configuration testing")

	// 注意 `signal`。默认是 -s string，有了 `signal` 之后，变为 -s signal
	flag.StringVar(&s, "s", "", "send `signal` to a master process: stop, quit, reopen, reload")
	flag.StringVar(&p, "p", "/usr/local/nginx/", "set `prefix` path")
	flag.StringVar(&c, "c", "conf/nginx.conf", "set configuration `file`")
	flag.StringVar(&g, "g", "conf/nginx.conf", "set global `directives` out of configuration file")

	// 改变默认的 Usage
	flag.Usage = usage
}

func main() {
	flag.Parse()
	// go run main.go -h
	if h {
		flag.Usage()
	}
	fmt.Println()
}

func usage() {
	fmt.Fprintf(os.Stderr, `nginx version: nginx/1.10.0
Usage: nginx [-hvVtTq] [-s signal] [-c filename] [-p prefix] [-g directives]

Options:
`)
	flag.PrintDefaults()
}
```
备注：

```
-flag // 只支持bool类型
-flag=x
-flag x // 只支持非bool类型
```
参考：<https://www.cnblogs.com/landv/p/11114508.html>



### math/rand
```go
package main

import (
	"math/rand"
	"time"
)

func main() {
	rand.Seed(time.Now().UnixNano())	// 设置种子，我们以当前时间的纳秒；当然也可以用毫秒，微秒等；没有随机种子，多次取值，值不变
	println(rand.Int())	// 返回随机int
	println(rand.Int31())	// 返回31为int值
	println(rand.Intn(100))	// 返回一个取值范围在[0, n)的伪随机int64值，如果n<=0会panic。
	println(rand.Float64())	// 返回一个取值范围在[0.0, 1.0)的伪随机float64值。
}

```
### sort
```go
package main

import (
	"fmt"
	"sort"
)

// 自定义一个根据每个元素的长度排序的类型
type Diy []string

func (d Diy) Len() int           { return len(d) }
func (d Diy) Swap(i, j int)      { d[i], d[j] = d[j], d[i] }
func (d Diy) Less(i, j int) bool { return len(d[i]) < len(d[j]) }

func main() {
	a := Diy{"dddd", "ffffff", "ccc", "eeeee", "a", "bb"}
	sort.Sort(a)	// sort.Sort 并不保证排序的稳定性。如果有需要, 可以使用 sort.Stable(a)
	fmt.Println(a)
	sort.Sort(sort.Reverse(a))	// sort提供的类型有：sort.StringSlice，sort.IntSlice等，可以直接将[]string,[]int转化为对应类型排序
	fmt.Println(a)


	// 备注：sort 包 在内部实现了四种基本的排序算法：
	// sort默认采用快速排序（quickSort）
	// 插入排序（insertionSort）、归并排序（symMerge）、堆排序（heapSort）和快速排序（quickSort）；
	// sort 包会依据实际数据自动选择最优的排序算法。所以我们写代码时只需要考虑实现 sort.Interface 这个类型就可以了。

}

```

### net
#### 方法
- net.Dial() // 请求网页功能采用http更好

### net/http
#### 变量
- http.ResponseWriter
- *http.Request
#### 方法
- http.HandleFunc()
- http.ListenAndServe()

参考：<https://studygolang.com/pkgdoc>


### errors
- errors.New() 函数是非常稀少的，因为有一个方便的封装函数fmt.Errorf()，它还会处理字符串格式化。

```go
package main

import (
	"fmt"
	"time"
)
// MyError is an error implementation that includes a time and message.
type MyError struct {
	When time.Time
	What string
}
// 实现Error()方法，就算继承error
func (e MyError) Error() string {
	return fmt.Sprintf("%v: %v", e.When, e.What)
}

func oops() error {
	return MyError{
		time.Date(1989, 3, 15, 22, 30, 0, 0, time.UTC),
		"the file system has gone away",
	}
}
func main() {
	if err := oops(); err != nil {
		fmt.Println(err)
	}
	// Output: 1989-03-15 22:30:00 +0000 UTC: the file system has gone away
}

```

### syscall
- syscall.Errno(2)

### path
说明：path实现了对斜杠分隔的路径的实用操作函数。
备注：应该采用path/filepath 
func IsAbs(path string) bool
func Split(path string) (dir, file string)
func Join(elem ...string) string
func Dir(path string) string    // 获取目录
func Base(path string) string   // 获取最后文件夹名（或文件名）
func Ext(path string) string    // 获取文件后缀，包括点.
func Clean(path string) string  // 纠正文件表述方式
func Match(pattern, name string) (matched bool, err error)

### path/filepath
说明：filepath包实现了兼容各操作系统的文件路径的实用操作函数。
func IsAbs(path string) bool
func Abs(path string) (string, error)
func Rel(basepath, targpath string) (string, error)
func SplitList(path string) []string
func Split(path string) (dir, file string)
func Join(elem ...string) string
func FromSlash(path string) string  // FromSlash函数将path中的斜杠（'/'）替换为路径分隔符并返回替换结果，多个斜杠会替换为多个路径分隔符。
func ToSlash(path string) string    // ToSlash函数将path中的路径分隔符替换为斜杠（'/'）并返回替换结果，多个路径分隔符会替换为多个斜杠。
func VolumeName(path string) (v string)
func Dir(path string) string
func Base(path string) string
func Ext(path string) string
func Clean(path string) string
func EvalSymlinks(path string) (string, error)  // EvalSymlinks函数返回path指向的符号链接（软链接）所包含的路径。
func Match(pattern, name string) (matched bool, err error)
func Glob(pattern string) (matches []string, err error) // 类似于php
type WalkFunc
func Walk(root string, walkFn WalkFunc) error   // 会递归获取；Walk函数会遍历root指定的目录下的文件树，对每一个该文件树中的目录和文件都会调用walkFn，包括root自身。
func HasPrefix(p, prefix string) bool


### sync
#### 属性
- sync.Once
sync.Once 是 Golang package 中使方法只执行一次的对象实现，作用与 init 函数类似。但也有所不同。

init 函数是在文件包首次被加载的时候执行，且只执行一次
sync.Once 是在代码运行中需要的时候执行，且只执行一次
当一个函数不希望程序在一开始的时候就被执行的时候，我们可以使用 sync.Once 。

例：
```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var once sync.Once
	onceBody := func() {
		fmt.Println("Only once")
	}
	done := make(chan bool)
	for i := 0; i < 10; i++ {
		go func(i int) {
			once.Do(onceBody)	// 里面的代码却只执行了一次
			fmt.Println(i)	// 打印了这里
			done <- true
		}(i)
	}
	for i := 0; i < 10; i++ {
		<-done
	}
}

```

### reflect

#### 方法
- reflect.TypeOf()
- reflect.ValueOf()

### unsafe
说明：使用unsafe包的同时也放弃了Go语言保证与未来版本的兼容性的承诺，
因为它必然会在有意无意中会使用很多实现的细节，而这些实现的细节在未来的Go语言中很可能会被改变。

- unsafe.Sizeof()   // 获取值的占位大小
- unsafe.Alignof()  // 获取变量类型对齐系数，即变量起始位只能是对齐系数倍数，而不是上一个值的下一位（有空则默认保留空位）
- unsafe.Offsetof() // 获取值的偏移量，即从0开始的index
参考：<https://zhuanlan.zhihu.com/p/53413177>
- unsafe.Pointer()  // 参数为指针，返回包含任意类型变量的地址
说明：
+ uinptr
    一个足够大的无符号整型， 用来表示任意地址。
    可以进行数值计算。
+ unsafe.Pointer
    代表一个可以指向任意类型的指针。
    不可以进行数值计算。
    有四种区别于其他类型的特殊操作：
    任意类型的指针值均可转换为 Pointer。
    Pointer 均可转换为任意类型的指针值。
    uintptr 均可转换为 Pointer。(将uintptr转为unsafe.Pointer指针可能会破坏类型系统，因为并不是所有的数字都是有效的内存地址。)
    Pointer 均可转换为 uintptr。
参考：<https://blog.csdn.net/hello_ufo/article/details/86713947>
+ Golang指针
  *类型:普通指针类型，用于传递对象地址，不能进行指针运算。
  unsafe.Pointer:通用指针类型，用于转换不同类型的指针，不能进行指针运算，不能读取内存存储的值（必须转换到某一类型的普通指针）。
  uintptr:用于指针运算，GC 不把 uintptr 当指针，uintptr 无法持有对象。uintptr 类型的目标会被回收。
          不要将uintptr的返回值赋值给具体变量，因为移动GC，可能会更改变量地址，即不能`tmp := uintptr(...)`
  unsafe.Pointer 是桥梁，可以让任意类型的指针实现相互转换，也可以将任意类型的指针转换为 uintptr 进行指针运算。
  unsafe.Pointer 不能参与指针运算，比如你要在某个指针地址上加上一个偏移量，Pointer是不能做这个运算的，那么谁可以呢?
  
  就是uintptr类型了，只要将Pointer类型转换成uintptr类型，做完加减法后，转换成Pointer，通过*操作，取值，修改值，随意。
  
  总结：unsafe.Pointer 可以让你的变量在不同的普通指针类型转来转去，也就是表示为任意可寻址的指针类型。而 uintptr 常用于与 unsafe.Pointer 打配合，用于做指针运算。
  参考:<https://www.cnblogs.com/-wenli/p/12682477.html>

```go
package main
 
import (
    "fmt"
    "reflect"
    "unsafe"
)
 
func main() {
 
    v1 := uint(12)
    v2 := int(13)
 
    fmt.Println(reflect.TypeOf(v1)) //uint
    fmt.Println(reflect.TypeOf(v2)) //int
 
    fmt.Println(reflect.TypeOf(&v1)) //*uint
    fmt.Println(reflect.TypeOf(&v2)) //*int
 
    p := &v1
    p = (*uint)(unsafe.Pointer(&v2)) //使用unsafe.Pointer进行类型的转换，作为桥梁被转化
 
    fmt.Println(reflect.TypeOf(p)) // *unit
    fmt.Println(*p) //13
}
```





