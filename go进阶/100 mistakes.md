1、影子变量

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



2、滥用嵌套代码

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



3、错误地使用init函数

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



4、过度使用Getter和Setter



5、接口污染

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



6、接口定义在生产者侧

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



7、返回接口

正常情况下，不应该返回接口，否则会让所有调用者都依赖 定义接口的包。

除了，一些标准库以外。

或者，调用者和接口都定义在一个包里





8、any 没有意思

尽可能避免使用any，因为使用者不知道any可能是什么。需要翻看手册。如果明确 参数使用范围。多写点冗余代码还比较合适呢。





10、没意识到嵌套结构体可能带来的问题

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



11、不适用功能选项模式

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





17、使用八进制时，产生误解

```go
sum := 100 + 010 // sum = 108, 010 = 0o10 = 10(base 8) = 8
```



18、忽略了整数型溢出	

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



19、不了解浮点数

**When comparing two floating-point numbers, check that their difference is within an acceptable range.**
在比较两个浮点数时，应检查它们的差值是否在可接受范围内。

**When performing additions or subtractions, group operations with a similar order of magnitude for better accuracy.**
在进行加法或减法运算时，应将数量级相近的操作分组以提高计算的准确性。

**To favor accuracy, if a sequence of operations requires addition, subtraction, multiplication, or division, perform the multiplication and division operations first.**
为了提高准确性，如果一系列运算中涉及加法、减法、乘法或除法，应优先进行乘法和除法运算。