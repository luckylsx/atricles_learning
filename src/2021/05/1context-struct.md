> 原文地址：https://blog.golang.org/context-and-structs
> 作者：Jean de Klerk, Matt T. Proud

## Contexts and structs

### 介绍

在许多Go API中，尤其是现代的API中，函数和方法的第一个参数通常是context.Context。 context提供了一种跨API边界以及在流程之间传输截止日期，调用者取消以及其他请求范围的值的方法。

[context的文档](https://golang.org/pkg/context/)说明：
> Contexts should not be stored inside a struct type, but instead passed to each function that needs it.

context 不应该存储在一个struct类型的实例中，而应该传递给每一个需要它的函数中。

本文在此建议的基础上扩展了原因和示例，描述了为什么传递Context而不是将其存储在另一种类型中为什么很重要。 它还强调了一种罕见的情况，即以结构类型存储Context以及如何安全地存储Context是有意义的。

### 首选将context作为参数传递

为了理解不要将context存储在结构中的建议，让我们考虑首选的“上下文作为参数”方法：

```
type Worker struct { /* … */ }

type Work struct { /* … */ }

func New() *Worker {
  return &Worker{}
}

func (w *Worker) Fetch(ctx context.Context) (*Work, error) {
  _ = ctx // A per-call ctx is used for cancellation, deadlines, and metadata.
}

func (w *Worker) Process(ctx context.Context, work *Work) error {
  _ = ctx // A per-call ctx is used for cancellation, deadlines, and metadata.
}
```

在这里，(*Worker).Fetch和(*Worker).Process方法都直接接受context。 通过这种按参数传递设计，用户可以设置每次呼叫的截止时间，取消和元数据。 并且，很清楚如何使用传递给每个方法的context.Context：不会期望将context.Context传递给一个方法的任何其他方法都将使用它。 这是因为上下文的范围仅限于所需的操作，这大大提高了此程序包中context的实用性和清晰度。

### 将context存储在结构中会导致混乱

让我们使用不好的结构体存储context的方法再次检查上面的Worker示例。 它的问题是，当您将context存储在结构中时，调用者的生命周期就变得晦涩难懂，或者更糟的是，这两种范围以不可预测的方式混合在一起：
```
type Worker struct {
  ctx context.Context
}

func New(ctx context.Context) *Worker {
  return &Worker{ctx: ctx}
}

func (w *Worker) Fetch() (*Work, error) {
  _ = w.ctx // A shared w.ctx is used for cancellation, deadlines, and metadata.
}

func (w *Worker) Process(work *Work) error {
  _ = w.ctx // A shared w.ctx is used for cancellation, deadlines, and metadata.
}
```

(*Worker).Fetch和(*Worker).Process方法都使用存储在Worker中的context。 这样可以防止Fetch and Process（本身可能具有不同的context）的调用方指定截止日期，请求取消以及按每次调用附加元数据。 例如：用户无法仅提供(*Worker).Process调用的截止时间。 调用者的生存期与共享context混合在一起，并且context的范围仅限于创建Worker的生存期。 

与通过参数传递方法相比，该API对用户也更加令人困惑。 用户可能会问自己：

* 自从实例化一个context.Context，构造者是否要做需要取消或截止时间的工作吗？
* 传递给实例的context.Context是否适用于(*Worker).Fetch和（*Worker).Process中的工作？ 两者都不？ 还是适用于一个而不是另一个？

该API需要大量文档来明确告诉用户确切的context.Context是用于什么的。 用户可能还必须阅读代码，而不是能够依赖于API传达的结构。

最后，设计一个生产级服务器可能非常危险，该服务器的请求每个都不具有context，因此不能充分满足取消要求。 如果无法设置按呼叫的截止时间，您的流程可能会积压([Handling Overload](https://sre.google/sre-book/handling-overload/))并耗尽其资源（如内存）！

### 规则之外：保留向后兼容性

当Go 1.7（引入了context.Context）发布时，大量的API必须以向后兼容的方式添加上下文支持。 例如，net / http的Client方法（例如Get和Do）是上下文的理想选择。 使用这些方法发送的每个外部请求都将受益于context.Context附带的期限，取消和元数据支持。

有两种方法可以增加对上下文的支持,以向后兼容的方式提供上下文：将context 包含在结构中，稍后会看到；以及复制函数，其中重复项接受context.Context并将Context作为其函数名后缀 。 与context相比，应采用重复方法，并在保持模块兼容中进一步讨论。 但是，在某些情况下这是不切实际的：例如，如果您的API公开了大量功能，则将它们全部复制可能是不可行的。

net/http包选择了结构中包含 context的方法，这提供了一个有用的案例研究。让我们看看net/http的做法。在引入context.context之前，Do的定义如下：

```
func (c *Client) Do(req *Request) (*Response, error)
```

在Go 1.7之后，如果不是因为它会破坏向后兼容性的事实，Do可能看起来如下所示：
```
func (c *Client) Do(ctx context.Context, req *Request) (*Response, error)
```

但是，保留向后兼容性并遵守Go 1兼容性承诺对于标准库至关重要。 因此，相反，维护者选择在http.Request结构上添加一个context.Context，以允许支持context.Context而不破坏向后兼容性：
```
type Request struct {
  ctx context.Context

  // ...
}

func NewRequestWithContext(ctx context.Context, method, url string, body io.Reader) (*Request, error) {
  // Simplified for brevity of this article.
  return &Request{
    ctx: ctx,
    // ...
  }
}

func (c *Client) Do(req *Request) (*Response, error)
```

如上所述，在对API进行改造以支持上下文时，可以将context.Context添加到结构中。 但是，请记住首先考虑复制函数，这可以在不牺牲实用性和理解力的情况下向后兼容上下文中的上下文。 例如：
```
func (c *Client) Call() error {
  return c.CallContext(context.Background())
}

func (c *Client) CallContext(ctx context.Context) error {
  // ...
}
```

### 总结

通过context，可以轻松地在调用堆栈中传播重要的跨库和跨API信息。 但是，必须一致，清晰地使用它，以使其易于理解，易于调试且有效。

当作为方法中的第一个参数传递而不是存储在结构类型中时，用户可以充分利用其可扩展性，以便通过调用堆栈构建强大的取消，截止时间和元数据信息。 而且，最重要的是，当将其作为参数传入时，可以清楚地了解其范围，从而可以在堆栈中上下轻松地理解和调试。

当设计带有context的API时，请记住以下建议：将context.Context作为参数传递； 不要将其存储在结构中。