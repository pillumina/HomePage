---
title: "Design Pattern"
date: 2020-11-11T11:22:18+08:00
hero: /images/posts/golang_banner.jpg
menu:
  sidebar:
    name: Design Pattern
    identifier: design-pattern
    parent: golang odyssey
    weight: 10
draft: false
---
# Design pattern

## Builder Pattern

### scenario：build complicated object

```go
package msg

type Message struct {
	Header *Header
	Body   *Body
}
type Header struct {
	SrcAddr  string
	SrcPort  uint64
	DestAddr string
	DestPort uint64
	Items    map[string]string
}
type Body struct {
	Items []string
}

// Message对象的复杂对象
type builder struct{
  once *sync.Once
  msg *Message
}

// 返回Builder对象
func Builder() *builder{
  return &builder{
    once: &sync.Once{},
    msg: &Message{Header: &Header{}, Body: &Body{}},
  }
}

func (b *builder) WithSrcAddr(srcAddr string) *builder{
  b.msg.Header.SrcAddr = srcAddr
  return b
}

//......
func (b *builder) WithHeaderItem(key, value string) *builder{
  //map只初始化一次
  b.once.Do(func(){
    b.msg.Header.Items = make(map[string]string)
  })
  b.msg.Header.Items[key] = value
  return b
}

func (b *builder) WithBodyItem(record string) *builder{
  b.msg.Body.Items = append(b.msg.Body.Items, record)
  return b
} 

func (b *builder) Build() *Message{
  return b.msg
}
```

### Test code

```go
package test
func TestMessageBuilder(t *testing.T) {
  // 使用消息建造者进行对象创建
	message := msg.Builder().
		WithSrcAddr("192.168.0.1").
		WithSrcPort(1234).
		WithDestAddr("192.168.0.2").
		WithDestPort(8080).
		WithHeaderItem("contents", "application/json").
		WithBodyItem("record1").
		WithBodyItem("record2").
		Build()
	if message.Header.SrcAddr != "192.168.0.1" {
		t.Errorf("expect src address 192.168.0.1, but actual %s.", message.Header.SrcAddr)
	}
	if message.Body.Items[0] != "record1" {
		t.Errorf("expect body item0 record1, but actual %s.", message.Body.Items[0])
	}
}
```



##  Abstract Factory Pattern

![abstract factory](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghkw23e4r3j31bs0nge82.jpg?imageslim)

常规的工厂模式，如果新增一个对象，需要修改原来的工厂对象代码，违反单一职责原则，最好增加一个抽象层。



```go
package plugin

//插件抽象接口定义
type Plugin interface{}

// 输入插件，用于接收消息
type Input interface{
  Plugin
  Receive() string
}

// 过滤插件，用于处理消息
type Filter interface{
  Plugin
  Process(msg string) string
}

// 输出插件，用于发送消息
type Output interface{
  Plugin
  Send(msg string)
}
```

管道由上述三种插件定义

```go
package pipeline
...
// 消息管道的定义
type Pipeline struct {
	input  plugin.Input
	filter plugin.Filter
	output plugin.Output
}
// 一个消息的处理流程为 input -> filter -> output
func (p *Pipeline) Exec() {
	msg := p.input.Receive()
	msg = p.filter.Process(msg)
	p.output.Send(msg)
}
```

定义三种插件的具体实现

```go
package plugin
...
// input插件名称与类型的映射关系，主要用于通过反射创建input对象
var inputNames = make(map[string]reflect.Type)
// Hello input插件，接收“Hello World”消息
type HelloInput struct {}

func (h *HelloInput) Receive() string {
	return "Hello World"
}
// 初始化input插件映射关系表
func init() {
	inputNames["hello"] = reflect.TypeOf(HelloInput{})
}
```

```go
package plugin
...
// filter插件名称与类型的映射关系，主要用于通过反射创建filter对象
var filterNames = make(map[string]reflect.Type)
// Upper filter插件，将消息全部字母转成大写
type UpperFilter struct {}

func (u *UpperFilter) Process(msg string) string {
	return strings.ToUpper(msg)
}
// 初始化filter插件映射关系表
func init() {
	filterNames["upper"] = reflect.TypeOf(UpperFilter{})
}
```

```go
package plugin
...
// output插件名称与类型的映射关系，主要用于通过反射创建output对象
var outputNames = make(map[string]reflect.Type)
// Console output插件，将消息输出到控制台上
type ConsoleOutput struct {}

func (c *ConsoleOutput) Send(msg string) {
	fmt.Println(msg)
}
// 初始化output插件映射关系表
func init() {
	outputNames["console"] = reflect.TypeOf(ConsoleOutput{})
}
```

定义抽象工厂接口，和对应插件的工厂实现

```go
package plugin
...
// 插件抽象工厂接口
type Factory interface {
	Create(conf Config) Plugin
}
// input插件工厂对象，实现Factory接口
type InputFactory struct{}
// 读取配置，通过反射机制进行对象实例化
func (i *InputFactory) Create(conf Config) Plugin {
	t, _ := inputNames[conf.Name]
	return reflect.New(t).Interface().(Plugin)
}
// filter和output插件工厂实现类似
type FilterFactory struct{}
func (f *FilterFactory) Create(conf Config) Plugin {
	t, _ := filterNames[conf.Name]
	return reflect.New(t).Interface().(Plugin)
}
type OutputFactory struct{}
func (o *OutputFactory) Create(conf Config) Plugin {
	t, _ := outputNames[conf.Name]
	return reflect.New(t).Interface().(Plugin)
}

```

最后定义pipeline工厂方法，调用plugin.Factory 抽象工厂完成pipeline对象的实例化

```go
package pipeline
...
// 保存用于创建Plugin的工厂实例，其中map的key为插件类型，value为抽象工厂接口
var pluginFactories = make(map[plugin.Type]plugin.Factory)
// 根据plugin.Type返回对应Plugin类型的工厂实例
func factoryOf(t plugin.Type) plugin.Factory {
	factory, _ := pluginFactories[t]
	return factory
}
// pipeline工厂方法，根据配置创建一个Pipeline实例
func Of(conf Config) *Pipeline {
	p := &Pipeline{}
	p.input = factoryOf(plugin.InputType).Create(conf.Input).(plugin.Input)
	p.filter = factoryOf(plugin.FilterType).Create(conf.Filter).(plugin.Filter)
	p.output = factoryOf(plugin.OutputType).Create(conf.Output).(plugin.Output)
	return p
}
// 初始化插件工厂对象
func init() {
	pluginFactories[plugin.InputType] = &plugin.InputFactory{}
	pluginFactories[plugin.FilterType] = &plugin.FilterFactory{}
	pluginFactories[plugin.OutputType] = &plugin.OutputFactory{}
}
```



## Prototype Pattern

场景：对象的复制，如果对象成员变量复杂，或者对象有不可见变量，即会有问题

```go
package prototype
...
// 原型复制抽象接口
type Prototype interface {
	clone() Prototype
}

type Message struct {
	Header *Header
	Body   *Body
}

func (m *Message) clone() Prototype {
	msg := *m
	return &msg
}
```


