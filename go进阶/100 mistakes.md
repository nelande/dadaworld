[toc]



### 1、影子变量

错：

```go
var client *http.Client
if tracing {
client, err := createClientWithTracing()
if err != nil {
return err
}
log.Println(client)
} else {
client, err := createDefaultClient()
if err != nil {
return err
}
log.Println(client)
}
// Use client
```

对：

```go
var client *http.Client
var err error
if tracing {
client, err = createClientWithTracing()
if err != nil {
return err
}
} else {
// Same logic
}
```



### 2、滥用嵌套代码

提前使用return

错：

```go
func join(s1, s2 string, max int) (string, error) {
	if s1 == "" {
		return "", errors.New("s1 is empty")
	} else {
		if s2 == "" {
			return "", errors.New("s2 is empty")
		} else {
			concat, err := concatenate(s1, s2)
			if err != nil {
				return "", err
			} else {
				if len(concat) > max {
					return concat[:max], nil
				} else {
					return concat, nil
				}
			}
		}
	}
}
```



对：

```go
func join(s1, s2 string, max int) (string, error) {
	if s1 == "" {
		return "", errors.New("s1 is empty")
	}
	if s2 == "" {
		return "", errors.New("s2 is empty")
	}
	concat, err := concatenate(s1, s2)
	if err != nil {
		return "", err
	}
	if len(concat) > max {
		return concat[:max], nil
	}
	return concat, nil
}
```



### 3、错误地使用init函数

错：

```go
var db *sql.DB

func init() {
	dataSourceName :=
		os.Getenv("MYSQL_DATA_SOURCE_NAME")
	d, err := sql.Open("mysql", dataSourceName)
	if err != nil {
		log.Panic(err)
	}
	err = d.Ping()
	if err != nil {
		log.Panic(err)
	}
	db = d
}
```



对：

```go
func createClient(dsn string) (*sql.DB, error) {
    db, err := sql.Open("mysql", dsn)
    if err != nil {
       return nil, err
    }
    if err = db.Ping(); err != nil {
       return nil, err
    }
    return db, nil
}
```



init不应该处理错误，而应该留给调用者

init直接设置了全局变量

使得测试功能是否可用变得复杂，因为你不知道 连接数据库的功能是否好用



### 4、过度使用Getter和Setter



### 5、接口污染

*The bigger the interface, the weaker the abstraction.*

接口越大，越不抽象



*Don’t design with interfaces, discover them.*

基于当前发现接口，而不是提前抽象



何时使用抽象：

当多种类型的具体实现拥有同一种行为时，抽象行为。如`sort.Interface`



解耦：

```go
type CustomerService1 struct {
	store mysql.Store
}

func (cs CustomerService1) CreateNewCustomer(id string) error {
	customer := Customer{id: id}
	return cs.store.StoreCustomer(customer)
}

type customerStorer interface {
	StoreCustomer(Customer) error
}

type CustomerService2 struct { // 利用interface解耦 CustomerService 和 具体实现 cs.store.StoreCustomer
	storer customerStorer
}

func (cs CustomerService2) CreateNewCustomer(id string) error {
	customer := Customer{id: id}
	return cs.storer.StoreCustomer(customer)
}
```



限制行为

```go

type IntConfig struct {
	// ...
}

func (c *IntConfig) Get() int {
	// Retrieve configuration
	return 1
}
func (c *IntConfig) Set(value int) {
	// Update configuration
}

type IntConfigGetter interface {
	Get() int
}

type Foo struct { // 通过 IntConfigGetter 接口，通过注入方式，在不改动IntConfig的情况下，限制Foo只能获取IntConfig的值，而不使用Set
	threshold IntConfigGetter
}

func NewFoo(threshold IntConfigGetter) Foo {
	return Foo{threshold: threshold}
}

func (f Foo) Bar() int {
	threshold := f.threshold.Get()
	return threshold
}
```



### 6、接口定义在生产者侧

以下为接口定义在生产者侧。

```go
package store

type Customer struct {
}

type CustomerStorage interface {
	StoreCustomer(customer Customer) error
	GetCustomer(id string) (Customer, error)
	UpdateCustomer(customer Customer) error
	GetAllCustomers() ([]Customer, error)
	GetCustomersWithoutContract() ([]Customer, error)
	GetCustomersWithNegativeBalance() ([]Customer, error)
}

```



```go
package client

import store "learningGo/producer"

type customersGetter interface {
	GetAllCustomers() ([]store.Customer, error)
}

```

实际上应该定义在消费者侧。因为抽象是被发现的，而不是创造的。接口应该由消费者定义。所以在上方中，让消费者直接暴露具体实现，让消费者决定是否使用它，或者自己是否抽象。



### 7、返回接口

正常情况下，不应该返回接口，否则会让所有调用者都依赖 定义接口的包。

除了，一些标准库以外。

或者，调用者和接口都定义在一个包里





### 8、any 没有意思

尽可能避免使用any，因为使用者不知道any可能是什么。需要翻看手册。如果明确 参数使用范围。多写点冗余代码还比较合适呢。





### 10、没意识到嵌套结构体可能带来的问题

错：

```go
type InMem struct {
    sync.Mutex
    m map[string]int
}
```



对：

```go
type InMem struct {
    mu sync.Mutex
    m map[string]int
}
```



错：

```go
type Logger struct {
    writeCloser io.WriteCloser
}
func (l Logger) Write(p []byte) (int, error) {
    return l.writeCloser.Write(p)
}
func (l Logger) Close() error {
    return l.writeCloser.Close()
}
func main() {
    l := Logger{writeCloser: os.Stdout}
    _, _ = l.Write([]byte("foo"))
    _ = l.Close()
}
```



对：

```go
type Logger struct {
    io.WriteCloser
}
func main() {
    l := Logger{WriteCloser: os.Stdout}
    _, _ = l.Write([]byte("foo"))
    _ = l.Close()
}
```



结构体嵌套不仅仅是为了简化访问字段。也不应该促使我们暴露那些我们希望从外部隐藏的数据



### 11、不适用功能选项模式

想象我们有个options结构体，这个结构体保存了配置，配置项会随需求变化

```go
type options struct {
	port    *int // 解决了port是人为设置为0，还是默认值是0 的情况
	timeout *time.Duration
}

type Option func(options *options) error

func WithPort(port int) Option {
	return func(options *options) error {
		if port < 0 {
			return errors.New("port should be positive")
		}
		options.port = &port
		return nil
	}
}

func WithTimeout(timeout time.Duration) Option { // 使用With方便变更配置项
	return func(options *options) error {
		if timeout < 0 {
			return errors.New("timeout should be positive")
		}
		options.timeout = &timeout
		return nil
	}
}

func NewServer(addr string, opts ...Option) (*http.Server, error) {
	var options options
	for _, opt := range opts {
		err := opt(&options)
		if err != nil {
			return nil, err // 由于闭包，With函数抛出错误 被放到NewServer/Build函数里。而不是With里
		}
	}
    
	return nil, nil
}
```





### 17、使用八进制时，产生误解

```go
sum := 100 + 010 // sum = 108, 010 = 0o10 = 10(base 8) = 8
```



### 18、忽略了整数型溢出	

```go
func Inc32(counter int32) int32 { // 整数型增加检查
	if counter == math.MaxInt32 {
		panic("int32 overflow")
	}
	return counter + 1
}

func IncInt(counter int) int { // 增加检查
	if counter == math.MaxInt {
		panic("int overflow")
	}
	return counter + 1
}

func AddInt(a, b int) int { // 加运算检查
	if a > math.MaxInt-b {
		panic("int overflow")
	}
	return a + b
}

func MultiplyInt(a, b int) int { // 乘运算检查
	if a == 0 || b == 0 {
		return 0
	}
	result := a * b
	if a == 1 || b == 1 {
		return result
	}
	if a == math.MinInt || b == math.MinInt {
		panic("integer overflow")
	}
	if result/b != a {
		panic("integer overflow")
	}
	return result
}
```



### 19、不了解浮点数

**When comparing two floating-point numbers, check that their difference is within an acceptable range.**
在比较两个浮点数时，应检查它们的差值是否在可接受范围内。

**When performing additions or subtractions, group operations with a similar order of magnitude for better accuracy.**
在进行加法或减法运算时，应将数量级相近的操作分组以提高计算的准确性。

**To favor accuracy, if a sequence of operations requires addition, subtraction, multiplication, or division, perform the multiplication and division operations first.**
为了提高准确性，如果一系列运算中涉及加法、减法、乘法或除法，应优先进行乘法和除法运算。



### 20、不了解切片的长度和容量

To summarize, the *slice length* is the number of available elements in the slice, whereas the *slice capacity* is the number of elements in the backing array. Adding an element to a full slice (length == capacity) leads to creating a new backing array with a new capacity, copying all the elements from the previous array, and updating the slice pointer to the new array.

简而言之，切片的长度是切片中可用元素的数量，而切片的容量是切片背后数组的元素数量。当向一个已满的切片（长度 == 容量）添加元素时，会创建一个新的背后数组，并为其分配新的容量，之后会将所有元素从原来的数组复制到新数组中，并更新切片指针指向新的数组。

```go
	s1 := make([]int, 3, 6)
	s2 := s1[1:3]
	fmt.Println(len(s2), cap(s2)) // 2, 5
```







### 21、低效地声明切片

如果事先知道切片的长度，可以直接初始化容量

```go
func convert(foos []Foo) []Bar {
	n := len(foos)
	bars := make([]Bar, 0, n)
	for _, foo := range foos {
		bars = append(bars, fooToBar(foo))
	}
	return bars
}
```



### 22、混淆 empty 和 nil 切片

空切片，指长度为0，

nil切片，值没做内存分配

nil切片一定是空切片

```go
func main() {
	var s []string // nil
	log(1, s)
	s = []string(nil) // nil
	log(2, s)
	s = []string{} // empty, not nil
	log(3, s)
	s = make([]string, 0) // empty, not nil
	log(4, s)
}
```

`var s []string`，如果我们不确定最终长度并且切片可以为空。

`[]string(nil)`，作为语法糖，用于创建一个 `nil` 且为空的切片。

`make([]string, length)`，如果已知未来的长度。

不提倡使用 `[]string{}`

一些包，会由于切片是 nil 还是 empty 结果不一样，如 `encoding/json`



### 23、不正确的检查切片是否为空

检查 len 来判断是否为空，而不是 nil



### 24、未正确地复制切片

错

```go
	src := []int{0, 1, 2}
	var dst []int
	copy(dst, src) // copy 结果取决于 两个切片的最小长度
	fmt.Println(dst) // []
```

对

```go
	src := []int{0, 1, 2}
	dst := make([]int, len(src))
	copy(dst, src)
	fmt.Println(dst)
	
	// another alternatives
	src := []int{0, 1, 2}
	dst := append([]int(nil), src...)
```





### 25、append的副作用

使用append时，会更新切片的底层数组的元素，然后返回一个长度+1的切片

```go
	s1 := []int{1, 2, 3}
	s2 := s1[1:2]
	s3 := append(s2, 10)
	fmt.Println(s1, s2, s3) // [1 2 10] [2] [2 10]
```

通过复制或者传递一个capacity一定的变量进入函数，防止函数内部修改切片从而影响外部切片

错

```go
func main() {
	s := []int{1, 2, 3}
	f(s[:2])
	fmt.Println(s) // [1 2 10]
}
func f(s []int) {
	_ = append(s, 10)
}

```

对

```go
func main() {
	s := []int{1, 2, 3}
	f(s[:2:2]) // slice with capacity 2 same backing array
	// Use s
}
func f(s []int) {
	// Update s
}
```



### 26、切片容量内存泄露

函数返回值 返回切片，但是老切片仍为回收，因为引用同一个backing array

错

```go
func consumeMessages() {
	for {
		msg := receiveMessage()
		// Do something with msg
		storeMessageType(getMessageType(msg))
	}
}
func getMessageType(msg []byte) []byte {
	return msg[:5] // 外部引用未删除
}
	
```

对

```go
func getMessageType(msg []byte) []byte {
	msgType := make([]byte, 5)
	copy(msgType, msg)
	return msgType
}
```



如果 Go 中的元素是指针类型，或者是一个结构体包含指针字段，那么这些元素将不会被垃圾回收器 (GC) 回收。

对

```go
type Foo struct {
	v []byte
}

func main() {
	foos := make([]Foo, 1_000)
	printAlloc()
	for i := 0; i < len(foos); i++ {
		foos[i] = Foo{
			v: make([]byte, 1024*1024),
		}
	}
	printAlloc()
	two := keepFirstTwoElementsOnly(foos)
	runtime.GC()
	printAlloc()
	
	fmt.Println(cap(foos))
	
	runtime.KeepAlive(two)
}

func keepFirstTwoElementsOnly(foos []Foo) []Foo {
	for i := 2; i < len(foos); i++ { // 自动置nil，或者 copy一个slice，老的切片就会被hui
		foos[i].v = nil
	}
	return foos[:2]
}

func printAlloc() {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Printf("%d KB\n", m.Alloc/1024)
}
```

### 27、低效地声明map

map的底层数据结构是 哈希表+桶。map增长条件：

平均每个桶中的元素数量（称为**负载因子**）大于某个常数值。这个常数目前等于6.5（但在未来版本中可能会发生变化，因为它是Go语言内部的参数）。

**桶溢出**过多（即包含超过8个元素的桶数量过多）。

所以如果能提前知道map的大小，就设置它



### 28、map内存溢出

go中的map的大小只会增，不会减小。如果一个map删除了键值对，但是其桶数仍然不变。

因此，我们必须记住，Go 的 `map` 只能增加大小，其内存消耗也会随之增加，并且没有自动的策略来缩小 `map` 的大小。如果这导致了高内存消耗，我们可以尝试一些不同的解决方案，例如强制 Go 重新创建 `map`，或者使用指针来检查是否可以优化内存使用。

使用指针优化

```go
	m1 := make(map[int][128]byte) 
	m2 := make(map[int]*[128]byte) // 优化之后
```



### 29、错误地比较值

在go中以下类型可以进行比较

*Booleans*—Compare whether two Booleans are equal.

*Numerics (int, float, and complex types)*—Compare whether two numerics are

equal.

*Strings*—Compare whether two strings are equal.

*Channels*—Compare whether two channels were created by the same call to make or if both are nil.

*Interfaces*—Compare whether two interfaces have identical dynamic types and equal dynamic values or if both are nil.

*Pointers*—Compare whether two pointers point to the same value in memory or if both are nil.

*Structs and arrays*—Compare whether they are composed of similar types.

那么如何比较 切片、map、或者带有切片、map的结构体呢

使用 reflect.DeepEqual，

自己写个比较函数





## 数据结构

### 30、忽略了 range 循环中，是复制赋值。

在 Go 中，所有赋值操作都会产生一个副本（拷贝）。具体来说：

1. **如果我们将一个返回结构体（struct）的函数的结果赋值给一个变量，Go 会拷贝该结构体的内容。**

   即：赋值时会将结构体的每个字段的值都复制到目标变量中，而不是直接引用原始数据。

2. **如果我们将一个返回指针（pointer）的函数的结果赋值给一个变量，Go 会拷贝这个指针的内存地址。**

   即：拷贝的是指针所指向的地址（在 64 位架构上，这个地址的长度是 64 位），而不是指针所指向的数据本身。

例子

```go
type account struct {
	balance float32
}

func main() {
	accounts := []account{
		{balance: 100.},
		{balance: 200.},
		{balance: 300.},
	}
	for _, a := range accounts {
		a.balance += 1000
	}

	for i := range accounts {
		accounts[i].balance += 1000 // 使用 for i
	}

	for i := 0; i < len(accounts); i++ {
		accounts[i].balance += 1000 // 使用 classic for loop
	}
}
```



也可以讲切片改为 指针类型的切片，但是效率会很低下



31、不清楚 range 循环中 参数是如何赋值的

range 中，参数是copy的，而且只 执行一次。classic 循环中，参数是每次执行的

所以

```go
	s1 := []int{1, 2, 3}
	fmt.Printf("%p\n", s1)
	for range s1 {
		fmt.Printf("%p\n", s1)
		s1 = append(s1, 10)
	}
	fmt.Println(s1) // [1 2 3 10 10 10]

	s2 := []int{1, 2, 3}
	for i := 0; i < len(s2); i++ {
		s2 = append(s2, s2[i]) // never end
	}
```



33、在迭代过程中错误的假设

- go中，字典不储存顺序。
- 如果在迭代过程中，插入新键值对。键值对可能会出现在下一个迭代中

```go
	m := map[int]bool{
		0: true,
		1: false,
		2: true,
	}
	for k, v := range m {
		if v {
			m[10+k] = true
		}
	}
	fmt.Println(m) // map[0:true 1:false 2:true 10:true 12:true 20:true 22:true]
					// map[0:true 1:false 2:true 10:true 12:true 20:true 22:true 30:true 32:true]
```



34、不清楚 break 的用法

break 作用于最近一层的 for switch select

所以以下代码还是会一直死循环

```go
// 错
	for i := 0; i < 5; i++ {
		fmt.Printf("%d ", i)
		switch i {
		default:
		case 2:
			break
		}
	}
// 对
loop:
	for i := 0; i < 5; i++ {
		fmt.Printf("%d ", i)
		switch i {
		default:
		case 2:
			break loop
		}
	}
```





35、在循环中使用 defer

defer只有在return语句执行时，会执行。所以以下代码优化

```go
// 优化前
func readFiles(ch <-chan string) error {
	for path := range ch {
		file, err := os.Open(path)
		if err != nil {
			return err
		}
		defer file.Close()
		// Do something with file
	}
	return nil
}

// 优化后，新创建一个函数用于执行 defer 语句
func readFiles(ch <-chan string) error {
	for path := range ch {
		if err := readFile(path); err != nil {
			return err
		}
	}
	return nil
}
func readFile(path string) error {
	file, err := os.Open(path)
	if err != nil {
		return err
	}
	defer file.Close()
	// Do something with file
	return nil
}
```





## 方法和函数

### 42 、不知道使用什么类型的接收者

**A receiver *must* be a pointer**

- If the method needs to mutate the receiver. This rule is also valid if the receiver

is a slice and a method needs to append elements:

type slice []int

func (s *slice) add(element int) {

*s = append(*s, element)

}

- If the method receiver contains a field that cannot be copied: for example, a

type part of the sync package (we will discuss this point in mistake #74, “Copy

ing a sync type”).

**A receiver *should* be a pointer**

- If the receiver is a large object. Using a pointer can make the call more effi

cient, as doing so prevents making an extensive copy. When in doubt about how

large is large, benchmarking can be the solution; it’s pretty much impossible to

state a specific size, because it depends on many factors.

**A receiver *must* be a value**

- If we have to enforce a receiver’s immutability.

- If the receiver is a map, function, or channel. Otherwise, a compilation error

occurs.

**A receiver *should* be a value**

- If the receiver is a slice that doesn’t have to be mutated.

- If the receiver is a small array or struct that is naturally a value type without

mutable fields, such as time.Time.

- If the receiver is a basic type such as int, float64, or string.

### 43、从不使用命名返回值

推荐使用命名返回值的情况，使得代码清晰。两个参数返回值都是同种类型

```go
type locator interface {
	getCoordinates(address string) (float32, float32, error)
}

type locator interface {
	getCoordinates(address string) (lat, lng float32, err error) // 接口添加返回值，清晰描述
}
```



### 44、命名返回值带来的意外影响

如果命名了 返回值，返回值初始值时zero value

另外，并不是说使用了命名返回值，就一定要使用 naked return statement，可以单纯只是让代码变得清晰

```go
if err = ctx.Err(); err != nil { // 不好的例子
	return
}

if err := ctx.Err(); err != nil { // 可以这样，这里的err是影子变量
	return 0, 0, err
}
```



### 45、返回 nil 结果

一个nil接收者 转换成 接口 会变成一个non-nil 接口

```go
type MultiError struct {
	errs []string
}

func (m *MultiError) Add(err error) {
	m.errs = append(m.errs, err.Error())
}

func (m *MultiError) Error() string {
	return strings.Join(m.errs, ";")
}

type Customer struct {
	 Age int
	 Name string
}

func (c Customer) Validate() error {
	var m *MultiError
	if c.Age < 0 {
		m = &MultiError{}
		m.Add(errors.New("age is negative"))
	}
	if c.Name == "" {
		if m == nil {
			m = &MultiError{}
		}
		m.Add(errors.New("name is nil"))
	}
	return m // nil 的MultiError指针 转换为 error 后，为 non-nil error
}

func main() {
	c := &Customer{Age: 10, Name: "John"}
	if err := c.Validate(); err != nil { // 永远返回non-nil error
		fmt.Printf("customer is invalid: %v", err)
	}
}
```

应该改成

```go
func (c Customer) Validate() error {
	var m *MultiError
	if c.Age < 0 {
		// ...
	}
	if c.Name == "" {
		// ...
	}
    if m != nil {
        return m
    }
    return nil
}
```



### 46、将文件名作为入函数的输入

假设有一个函数的功能是计算文件的空行数，可能会像以下实现

```go
func countEmptyLinesInFile(filename string) (int, error) {
	file, err := os.Open(filename)
	if err != nil {
		return 0, err
	}
	// Handle file closure
	scanner := bufio.NewScanner(file)
	for scanner.Scan() {
		// ...
	}
}
```

但是这个函数复用性低，如果需要计算比如请求的空行数，就不能复用了。应该善用接口来实现

```go
func countEmptyLines(reader io.Reader) (int, error) { // 可以传入任何实现了 io.Reader的东西
	scanner := bufio.NewScanner(reader)
	for scanner.Scan() {
		// ...
	}
}
```



### 47、疏忽了defer参数和接收者的赋值机制

当使用defer 调用一个方法或函数时，方法和函数的参数会被立即赋值。可以通过参数传入指针，或者闭包的形式使得 defer 跟上参数的变化。

对于方法，方法接收者也会被立即赋值。

```go
func main() {
	i := 1
	defer fmt.Println(i) // 1
	i = 2
}

func main() {  // 闭包
	i := 1
	defer func() {
		fmt.Println(i) // 2
	}()
	i = 2
}
```



## 错误处理

#### 48、何时使用panic

当遇上存粹的程序错误时，使用。



