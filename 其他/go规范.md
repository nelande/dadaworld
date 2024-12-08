[toc]



# 代码风格

## 命名

使用驼峰命名法

```go
type SomeType struct{} // 大驼峰

var localVariable // 小驼峰
```



文件名：全小写，可包含数字、下划线。不能采用驼峰，因为linux系统默认区分文件大小写



包名、目录名：包名和目录名保持一致。

包名：全小写、可包含数字

目录名：全小写、可包含数字、中划线



## 注释

可使用 /* */ 和 //，推荐统一使用 //

```go
// this is the
// the example

/*
This is the
example
*/
```





合理使用注释：

- 包含有多个文件，包注释出现在其中一个文件即可
- 导出的标识符都应该有注释



## 格式

采用空行分割功能实现中相应的逻辑块



编译指令注释符和编译指令之间不需要空格

# 编程实践

## 声明和初始化

变量使用时才声明并初始化，什么时候用什么时候声明，这就和C不同



少使用全局变量，这会导致业务代码和全局变量之间产生耦合，难以跟踪数据的变化。

解决方法：使用数据进行封装。

## 常量

相关声明放一组内

```go
const (
	ServTypeSet ServType = 1
    ServTypeQuery ServType = 4
)
```



## 整数

防止溢出和回绕



## 字符串

当字符串有转移字符时，使用raw string，以提高可读性\

```go
const prefix = "import \"unsafe\"" // 错误

const prefix = `import "unsafe"`
```



## slice、map

函数返回空的切片

使用 `len(s) == 0` 判断切片是否为空



为啥，因为切片有 nil（零值） 和 空切片 两种值。 使用var 声明的 是nil切片，短声明的是 空切片

```go
	var nilSlice []int
	emptySlice := []int{}

	fmt.Println(nilSlice) // []
	fmt.Println(emptySlice) // []

	fmt.Println(nilSlice == nil) // true
	fmt.Println(emptySlice == nil) // false

	fmt.Println(len(nilSlice) == 0) // true
	fmt.Println(len(emptySlice) == 0) // true
```

相关帖子：https://qwqaq.com/2021/12/golang-slice-nil-vs-empty/



## 控制语句

每个switch分支必须包含default分支，除非所有的类型都明确了，如下所示

```go
// 类型已经确定
type color int

const (
	red color = iota
	green
	blue
)

func (c color) change() color {
	switch c {
	case red:
		return green
	case green:
		return blue
	case blue:
		return red
	}
	return red
}

```



不要在迭代集合数据结构的过程中添加或删除元素，如下所示

可能输出 Answer,10

也有可能 Answer,10 AnserNew,11

```go
func main() {
	m := make(map[string]int)
	m["Answer"] = 10
	for k, v := range m {
		m["AnswerNew"] = 11
		fmt.Println(k, v)
	}
}
```



例外：for-range迭代时可以修改或删除key对应的元素值，但不要添加key和元素值对



## 函数和方法

函数功能单一

### 选择方法接接收者的类型

- 改变内容，指针接收者，反之
- 大型结构或数组，考虑用指针类型，提升效率
- `sync.Mutex`等同步字段，**必须**指针类型
- `map func chan`，**不要**使用指针类型，入参是`slice`，不改变其容量，或者重新分配，**不使用**指针类型



## init

```
每个包可以拥有多个init函数
包的每个源文件也可以拥有多个init函数
同一个包中多个init函数的执行顺序go语言没有明确的定义(说明)
同一个源文件中定义了多个init()函数，它们的顺序将按照在源代码中的出现顺序来执行
```

`init`执行顺序：包级常量和变量 → `init` → `main`

不要在`init`中进行申请资源、打开文件等可能失败的操作，因为`init`函数不能处理错误，除非你在`init`中，调用`os.Exit()`，如下所示

```go
func init(){
    f, err := openFile()
    if err != nil {
        os.Exit(1)
    }
}
```



相关帖子：https://www.cnblogs.com/chenjiazhan/p/17473207.html



## 闭包

禁止在闭包中直接使用循环控制变量，如下所示

由于未使用参数值，闭包函数读到的值将依赖实际运行的值

```go
func foo() {
	for i := 0; i < 6; i++ {
		go func() {
			fmt.Printf("%-2d\n", i)
		}()
	}
}
```



## 结构体与接口类型

匿名嵌入，只有当被嵌入类型所有的导出方法和字段都需要被加到外部类型时使用匿名嵌入

原因：若使用匿名嵌入，外部可直接访问被嵌入类型的方法和字段



## 接口设计

像模板方法一样实现接口定义，和使用接口。以降低接口定义和实现直接的耦合性，如下所示

```go
type Producer interface {
	DoSomething()
}

// Template 使用模板方法，设计使用接口的函数
func Template(t Producer) string {
	t.DoSomething()
	return ""
}
```



### 避免定义多大的接口

设计职责单一，容易组合的接口



### 无错时，返回无类型nil值。避免错误变量和nil直接比较

当使用自定义类型，且无错时，返回类型不为nil的值，且使用`err == nil`，可能会发生以下错误

```go
func foo() error {
	var err *myError // = nil，这会指定 error 的类型为 myError
	return err
}

func main() {
	var err = foo()
	if err == nil {
		fmt.Println("err is nil") // 即使实际没错，也会走进该分支，因为error已经有了类型
	} else {
		fmt.Println("err is not nil")
	}
}

```

## 

## 类型断言

使用switch类型断言时，必须处理匹配失败情况，否则会panic

使用ok方式对变量进行类型断言，防止成功后，返回结果为空值

```go
fmt.Println("a value", a.(string)) // 错误用法

func bar(a interface{}) {
	value, ok := a.(string) // 正确用法
	if ok {
		fmt.Println("a value is", value)
	} else {
		fmt.Println("type is not string")
	}

	pvalue, ok := a.(*int) // 正确用法
	if !ok || pvalue == nil {
		fmt.Println("type is not *int")
	}
}
```



## 包

禁止使用`.`简化导入包

禁止使用相对路径导入包

非特殊情况不要使用别名



## 错误与异常处理



## 错误处理

不要其他额外信息的简单错误，用`errors.New`

如果需要传播下游函数返回的错误，用`fmt.Errorf`和`%w`

错误信息避免**大写**开头，**标点符号**结尾，因为会和其他错误信息组合输出

## 异常处理

包外可见的函数禁止向外抛出`panic`，需要内部`recover`处理

发生异常后，需要尝试将程序恢复到合理状态



## 数据竞争

缩短在临界区停留的时间

存在并发同时读写的操作时，应确保：并发时无数据竞争，不死锁。禁止拷贝锁与同步的对象

使用`chan`前，对其初始化

使用ok方式对`chan`处理

```go
for {
    select {
        case <-cc: // 错误用法
    }
}

for {
    select {
        case _, ok := <-cc: // 正确用法
    }
}
```



## 禁止解引用空指针