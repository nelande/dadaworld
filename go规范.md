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

不要在迭代集合数据结构的过程中添加或删除元素



for-range迭代时可以修改key对应的元素值，但不要添加或删除key和元素值对