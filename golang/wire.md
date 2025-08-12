## HOW TO USE WIRE

##### Build a small program to demonstrate how to use wire

```go
type Message string

type Greeter struct {
    Message Message // <- adding a Message field
}
// Greeter's methods
func (g Greeter) Greet() Message {
    return g.Message
}

type Event struct {
    Greeter Greeter // <- adding a Greeter field
}
// Event's methods
func (e Event) Start() {
    msg := e.Greeter.Greet()
    fmt.Println(msg)
}
// Initialization methods
func NewMessage() Message {
    return Message("Hi there!")
}

func NewGreeter(m Message) Greeter {
    return Greeter{Message: m}
}
func NewEvent(g Greeter) Event {
    return Event{Greeter: g}
}
```
##### See how this program works without wire
```go
func main() {
    message := NewMessage()
    greeter := NewGreeter(message)
    event := NewEvent(greeter)
    event.Start()
}
```

##### Using Wire to Generate Code
```go
func main() {
    e := InitializeEvent()
    e.Start()
}
```

**Create a separate file called wire.go**
```go
func InitializeEvent() Event {
    wire.Build(NewEvent, NewGreeter, NewMessage)
    return Event{}
}
```
> Use wire.Build() and pass in initializers we want to use, in this case that is ==NewEvent, NewGreeter, NewMessage== 

> In Wire, initializers are known as ==**"providers"**==, functions which provide a particular type.

> The type Event here is to provide information about which providers to use to construct Event. If we add values to Event{} wire will ignore them.