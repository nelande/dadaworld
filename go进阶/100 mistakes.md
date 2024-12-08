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

