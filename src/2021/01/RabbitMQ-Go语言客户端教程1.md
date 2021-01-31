>- 原文作者：李文周
>- 原文地址：https://www.liwenzhou.com/posts/Go/go_rabbitmq_tutorials_01/
>- 本文作为学习笔记记录，如有侵权请联系 lucky9303@163.com

## RabbitMQ Go语言客户端教程1——HelloWorld

### 介绍
RabbitMQ是一个消息代理：它接受并转发消息。你可以把它想象成一个邮局：当你把你想要邮寄的邮件放进一个邮箱时，你可以确定邮差先生或女士最终会把邮件送到你的收件人那里。在这个比喻中，RabbitMQ是一个邮箱、一个邮局和一个邮递员。

RabbitMQ和邮局的主要区别在于它不处理纸张，而是接受、存储和转发二进制数据块——消息。

RabbitMQ和一般的消息传递都使用一些术语。
* 生产仅意味着发送。发送消息的程序是生产者：
<img style="display: block; margin: 0 auto;" src="./img/rabbit_p.png" alt="" />

* 队列是位于RabbitMQ内部的邮箱的名称。尽管消息通过RabbitMQ和你的应用程序流动，但它们只能存储在队列中。队列只受主机内存和磁盘限制的限制，实际上它是一个大的消息缓冲区。许多生产者可以向一个队列发送消息，而许多消费者可以尝试从一个队列接收数据。以下是我们表示队列的方式：
<img style="display: block; margin: 0 auto;" src="./img/rabbit_queue.png" alt="" />

* 消费与接收具有相似的含义。消费者是一个主要等待接收消息的程序：
<img style="display: block; margin: 0 auto;" src="./img/rabbit_c.png" alt="" />

**请注意，生产者，消费者和代理（broker）不必位于同一主机上。实际上，在大多数应用程序中它们不是。一个应用程序既可以是生产者，也可以是消费者。**

### RabbitMQ安装
* 参考安装教程 [安装指南](https://www.rabbitmq.com/download.html)
* 使用docker 镜像安装。[参考官方docker](https://registry.hub.docker.com/_/rabbitmq/)

**docker简单安装介绍**
采用 tag为 3-management 版本
```
// 拉取镜像
docker pull rabbitmq:3-management
// 运行docker 镜像
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management

// 此时 docker ps 即可查看运行的docker 容器

// 停止运行 rabbitmq 容器
docker stop rabbitmq

// 删除 rabbitmq 容器
docker rm rabbitmq
```

假设RabbitMQ已安装并运行在本机上的标准端口（5672）。如果你使用不同的主机、端口或凭据，则需要调整连接设置。

### golang客户端 “Hello World”的简单使用
首先介绍下目录结构
root/\
|---mq/\
-----|----amqp.go // rabbitMQ 初始化  \
-----send.go      // 向队列中发送数据 \
-----receive.go   // 接收参数 

在下图中，“ P”是我们的生产者，“ C”是我们的消费者。中间的框是一个队列——RabbitMQ代表消费者保存的消息缓冲区。
<img style="display: block; margin: 0 auto;" src="./img/mq.png" alt="" />

首先，使用go get安装amqp
```
go get github.com/streadway/amqp
```
安装好之后，就可以使用该包编写代码。

#### 初始化RabbitMQ 链接及channel 消息通道
首先在 mq 文件夹下面 创建 amqp.go 文件,进行初始化
```
package mq

import (
	"github.com/streadway/amqp"
	"log"
)

var Conn *amqp.Connection
var Ch *amqp.Channel
var err error

func NewAmqp() {
	// 1 链接RabbitMQ 建立链接
	Conn, err = amqp.Dial("amqp://guest:guest@localhost:5672/")
	FailOnError(err, "Failed to connect to RabbitMQ")
    // 因为使用的是全局链接对象和channel 对象 所以此处不能关闭
	// defer Conn.Close()
	// 2 创建通道
	Ch, err = Conn.Channel()
    // 此处与上述情况一样
    // defer Ch.close()
	FailOnError(err, "failed to open a channel")
}

// 此处定义辅助函数来检查每个amqp调用的返回值，send.go 和 receive.go 可直接调用
func FailOnError(err error, msg string) {
	if err != nil {
		log.Fatalf("msg : %s, error : %v", msg, err)
	}
}
```
#### 发送
<img style="display: block; margin: 0 auto;" src="./img/push_mq.png" alt="" />

我们将消息发布者（发送者）称为 send.go，将消息消费者（接收者）称为receive.go。发布者将连接到RabbitMQ，发送一条消息，然后退出。

要发送，我们必须声明要发送到的队列。然后我们可以将消息发布到队列：
```
// 初始化链接
mq.NewAmqp()
q, err := mq.Ch.QueueDeclare(
    "hello", // queue name
    false,   // durable
    false,   // delete when unused
    false,   // exclusive
    false,   // no-wait
    nil,
)
mq.FailOnError(err, "failed to declare queue")
body := "hello world"
err = mq.Ch.Publish(
    "",     // exchange
    q.Name, // routing key
    false,  // mandatory
    false,  // immediate
    amqp.Publishing{
        ContentType: "text/plain",
        Body: []byte(body),
    },
)
mq.FailOnError(err, "failed to publish a message")
```

声明队列是幂等的——仅当队列不存在时才创建。消息内容是一个字节数组，因此你可以在此处编码任何内容。

#### 接收
<img style="display: block; margin: 0 auto;" src="./img/rabbit_rece.png" alt="" />

设置与发布者相同；我们打开一个连接和一个通道，并声明要消耗的队列。请注意，这与send发布到的队列匹配。

```
// 初始化链接对象
mq.NewAmqp()
// 声明队列
q, err := mq.Ch.QueueDeclare(
    "hello", // name
    false,   // durable
    false,   // delete when unused
    false,   // exclusive
    false,   // no-wait
    nil,     // arguments
)
mq.FailOnError(err, "Failed to declare a queue")
```
请注意，我们也在这里声明队列。因为我们可能在发布者之前启动使用者，所以我们希望在尝试使用队列中的消息之前确保队列存在。

我们将告诉服务器将队列中的消息传递给我们。由于它将异步地向我们发送消息，因此我们将在goroutine中从通道（由amqp::Consume返回）中读取消息。

```
// 获取接收消息的Delivery通道
msgs, err := mq.Ch.Consume(
    q.Name, // queue
    "",     // consumer
    true,   // auto-ack
    false,  // exclusive
    false,  // no-local
    false,  // no-wait
    nil,    // args
)
mq.FailOnError(err, "Failed to register a consumer")

forever := make(chan bool)
go func() {
    for d := range msgs {
        log.Printf("Received a message: %s", d.Body)
    }
}()
log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
<-forever
```

**send.go 完整示例代码**
```
package main

import (
	"github.com/streadway/amqp"
	"imooc.com/learngo/mq/mq"
)

func main() {
	mq.NewAmqp()
	q, err := mq.Ch.QueueDeclare(
		"hello", // queue name
		false,   // durable
		false,   // delete when unused
		false,   // exclusive
		false,   // no-wait
		nil,
	)
	mq.FailOnError(err, "failed to declare queue")
	body := "hello world"
	err = mq.Ch.Publish(
		"", // exchange
		q.Name, // routing key
		false, // mandatory
		false, // immediate
		amqp.Publishing{
			ContentType: "text/plain",
			Body: []byte(body),
		},
	)
	mq.FailOnError(err, "failed to publish a message")
	// defer mq.Ch.Close()
}
```

**receive.go 完整示例代码**
```
package main

import (
	"imooc.com/learngo/mq/mq"
	"log"
)

func main() {
	// 初始化链接对象
	mq.NewAmqp()
	// 声明队列
	q, err := mq.Ch.QueueDeclare(
		"hello", // name
		false,   // durable
		false,   // delete when unused
		false,   // exclusive
		false,   // no-wait
		nil,     // arguments
	)
	mq.FailOnError(err, "Failed to declare a queue")

	// 获取接收消息的Delivery通道
	msgs, err := mq.Ch.Consume(
		q.Name, // queue
		"",     // consumer
		true,   // auto-ack
		false,  // exclusive
		false,  // no-local
		false,  // no-wait
		nil,    // args
	)
	mq.FailOnError(err, "Failed to register a consumer")

	forever := make(chan bool)
	go func() {
		for d := range msgs {
			log.Printf("Received a message: %s", d.Body)
		}
	}()
	log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
	<-forever
}
```

### 运行
现在我们可以运行两个脚本。在一个终端窗口，运行发布者：
```
go run send.go
```

运行消费者：
```
go run receive.go
```

消费者将打印通过RabbitMQ从发布者那里得到的消息。使用者将持续运行，等待消息（使用Ctrl-C停止它），因此请尝试从另一个终端运行发布者。

如果要检查队列，请尝试使用rabbitmqctl list_queues命令。