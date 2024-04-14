注册路由处理函数

```go
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {

})
```



handleFunc源码，其实使用默认mux注册路由处理函数

```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
  DefaultServeMux.HandleFunc(pattern, handler)
}
```



DefaultServerMux的原型 ServeMux

```go
var DefaultServeMux = &defaultServeMux

var defaultServeMux ServeMux

type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	es    []muxEntry // slice of entries sorted from longest to shortest.
	hosts bool       // whether any patterns contain hostnames
}
```



ServeMux.HandleFunc，将函数转换为HandlerFunc

```go
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
   if handler == nil {
      panic("http: nil handler")
   }
   mux.Handle(pattern, HandlerFunc(handler))
}
```



HandleFunc底层函数为`func(ResponseWriter, *Request)`

```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
   f(w, r)
}
```



HandleFunc实现了handle接口，HandleFunc只是为了方便实现接口

```go
type Handler interface {
   ServeHTTP(ResponseWriter, *Request)
}
```



回到`ServeMux.HandleFunc`中的`mux.Handle`，mux用map保存了路径和函数对应关系

```go
func (mux *ServeMux) Handle(pattern string, handler Handler) {
   mux.mu.Lock()
   defer mux.mu.Unlock()

   if pattern == "" {
      panic("http: invalid pattern")
   }
   if handler == nil {
      panic("http: nil handler")
   }
   if _, exist := mux.m[pattern]; exist {
      panic("http: multiple registrations for " + pattern)
   }

   if mux.m == nil {
      mux.m = make(map[string]muxEntry)
   }
   e := muxEntry{h: handler, pattern: pattern}
   mux.m[pattern] = e
   if pattern[len(pattern)-1] == '/' {
      mux.es = appendSorted(mux.es, e)
   }

   if pattern[0] != '/' {
      mux.hosts = true
   }
}
```



HandleFunc只是为了方便实现接口，我们当然可以直接定义一个实现`Handler`接口的类型，然后注册该类型的实例

```go
type greeting string

func (g greeting) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintln(w, g)
}

http.Handle("/greeting", greeting("Welcome, dj"))
```



http.Handle和http.HandleFunc都调用了mux.Handle，实际为两个入口。我们将通过`HandleFunc()`注册的称为处理函数，将通过`Handle()`注册的称为处理器。通过上面的源码分析不难看出，它们在底层本质上是一回事。



注册函数后，开始监听端口，处理请求

```go
func ListenAndServe(addr string, handler Handler) error {
   server := &Server{Addr: addr, Handler: handler}
   return server.ListenAndServe()
}
```

以上过程创建了一个server，server结构体如下

```go
type Server struct {
  Addr string
  Handler Handler
  TLSConfig *tls.Config
  ReadTimeout time.Duration
  ReadHeaderTimeout time.Duration
  WriteTimeout time.Duration
  IdleTimeout time.Duration
}
```



在ListenAndServe函数中，实际调用了net.Listen创建了Listener监听端口

```go
func (srv *Server) ListenAndServe() error {
   if srv.shuttingDown() {
      return ErrServerClosed
   }
   addr := srv.Addr
   if addr == "" {
      addr = ":http"
   }
   ln, err := net.Listen("tcp", addr) // <--------------------- 实际调用了net.Listen创建了Listener监听端口
   if err != nil {
      return err
   }
   return srv.Serve(ln) // <-------------- 监听
}
```





在`Server.Serve()`方法中，使用一个无限的`for`循环，不停地调用`Listener.Accept()`方法接受新连接，开启新 goroutine 处理新连接：

获得新连接后，将其封装成一个`conn`对象（`srv.newConn(rw)`），创建一个 goroutine 运行其`serve()`方法

```go
func (srv *Server) Serve(l net.Listener) error {
	if fn := testHookServerServe; fn != nil {
		fn(srv, l) // call hook with unwrapped listener
	}

	origListener := l
	l = &onceCloseListener{Listener: l}
	defer l.Close()

	if err := srv.setupHTTP2_Serve(); err != nil {
		return err
	}

	if !srv.trackListener(&l, true) {
		return ErrServerClosed
	}
	defer srv.trackListener(&l, false)

	baseCtx := context.Background()
	if srv.BaseContext != nil {
		baseCtx = srv.BaseContext(origListener)
		if baseCtx == nil {
			panic("BaseContext returned a nil context")
		}
	}

	var tempDelay time.Duration // how long to sleep on accept failure

	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
	for {
		rw, err := l.Accept() // <------------- 获取新链接
		if err != nil {
			select {
			case <-srv.getDoneChan():
				return ErrServerClosed
			default:
			}
			if ne, ok := err.(net.Error); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				srv.logf("http: Accept error: %v; retrying in %v", err, tempDelay)
				time.Sleep(tempDelay)
				continue
			}
			return err
		}
		connCtx := ctx
		if cc := srv.ConnContext; cc != nil {
			connCtx = cc(connCtx, rw)
			if connCtx == nil {
				panic("ConnContext returned nil")
			}
		}
		tempDelay = 0
		c := srv.newConn(rw) // <-------------- 封装成conn对象
		c.setState(c.rwc, StateNew, runHooks) // before Serve can return
		go c.serve(connCtx) // <--------------------- 处理链接
	}
}
```



创建一个 goroutine 运行其`serve()`方法。省略无关逻辑的代码如下：

```go
func (c *conn) serve(ctx context.Context) {
  for {
    w, err := c.readRequest(ctx)
    serverHandler{c.server}.ServeHTTP(w, w.req)
    w.finishRequest()
  }
}
```



`serve()`方法其实就是不停地读取客户端发送地请求，创建`serverHandler`对象调用其`ServeHTTP()`方法去处理请求，然后做一些清理工作。`serverHandler`只是一个中间的辅助结构，代码如下



从`Server`对象中获取`Handler`，这个`Handler`就是调用`http.ListenAndServe()`时传入的第二个参数

在`Hello World`的示例代码中，我们传入了`nil`。所以这里`handler`会取默认值`DefaultServeMux`。调用`DefaultServeMux.ServeHTTP()`方法处理请求：

```go
type serverHandler struct {
   srv *Server
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
	if handler == nil {
		handler = DefaultServeMux
	}
	handler.ServeHTTP(rw, req)
}
```



handler.ServeHTTP实现了接口

```go
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
   h, _ := mux.Handler(r)
   h.ServeHTTP(w, r)
}
```





mux的Handler，`mux.Handler(r)`通过请求的路径信息查找处理器，然后调用处理器的`ServeHTTP()`方法处理请求：

```go
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
   host := stripHostPort(r.Host)
   return mux.handler(host, r.URL.Path)
}

func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
	mux.mu.RLock()
	defer mux.mu.RUnlock()

	// Host-specific pattern takes precedence over generic ones
	if mux.hosts {
		h, pattern = mux.match(host + path)
	}
	if h == nil {
		h, pattern = mux.match(path)
	}
	if h == nil {
		h, pattern = NotFoundHandler(), ""
	}
	return
}

func (mux *ServeMux) match(path string) (h Handler, pattern string) {
	// Check for exact match first.
	v, ok := mux.m[path]
	if ok {
		return v.h, v.pattern
	}

	// Check for longest valid match.  mux.es contains all patterns
	// that end in / sorted from longest to shortest.
	for _, e := range mux.es {
		if strings.HasPrefix(path, e.pattern) {
			return e.h, e.pattern
		}
	}
	return nil, ""
}
```



在`match`方法中，首先会检查路径是否精确匹配`mux.m[path]`。如果不能精确匹配，后面的`for`循环会匹配路径的最长前缀。**只要注册了`/`根路径处理，所有未匹配到的路径最终都会交给`/`路径处理**。为了保证最长前缀优先，在注册时，会对路径进行排序。所以`mux.es`中存放的是按路径排序的处理列表：

```golang
func appendSorted(es []muxEntry, e muxEntry) []muxEntry {
  n := len(es)
  i := sort.Search(n, func(i int) bool {
    return len(es[i].pattern) < len(e.pattern)
  })
  if i == n {
    return append(es, e)
  }
  es = append(es, muxEntry{})
  copy(es[i+1:], es[i:])
  es[i] = e
  return es
}
```





我们之前都是使用DefaultMux注册函数，使用默认对象有一个问题：不可控。

一来`Server`参数都使用了默认值，二来第三方库也可能使用这个默认对象注册一些处理，容易冲突。更严重的是，我们在不知情中调用`http.ListenAndServe()`开启 Web 服务，那么第三方库注册的处理逻辑就可以通过网络访问到，有极大的安全隐患。所以，除非在示例程序中，否则建议不要使用默认对象。



我们可以使用http.NewServeMux()创建mux，然后创建server，最后调用Server.ListenAndServe()方法开启web服务

```go
func main() {
	mux := http.NewServeMux()
	mux.Handle("/greeting", greeting("Welcome to go web frameworks"))

	server := &http.Server{
		Addr:         ":8080",
		Handler:      mux,
		ReadTimeout:  20 * time.Second,
		WriteTimeout: 20 * time.Second,
	}

	server.ListenAndServe()
}
```







