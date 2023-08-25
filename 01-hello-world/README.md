
#"Hello World!" 


##### Prerequisites

This tutorial assumes RabbitMQ is [installed](https://www.rabbitmq.com/download.html) and running on localhost on the [standard port](https://www.rabbitmq.com/networking.html#ports) (5672). In case you use a different host, port or credentials, connections settings would require adjusting.

##### Where to get help

If you're having trouble going through this tutorial you can contact us through the [mailing list](https://groups.google.com/forum/#!forum/rabbitmq-users) or [RabbitMQ community Slack](https://rabbitmq.com/slack/).

Introduction
------------

RabbitMQ is a message broker: it accepts and forwards messages. You can think about it as a post office: when you put the mail that you want posting in a post box, you can be sure that the letter carrier will eventually deliver the mail to your recipient. In this analogy, RabbitMQ is a post box, a post office, and a letter carrier.

The major difference between RabbitMQ and the post office is that it doesn't deal with paper, instead it accepts, stores, and forwards binary blobs of data ‒ _messages_.

RabbitMQ, and messaging in general, uses some jargon.

*   _Producing_ means nothing more than sending. A program that sends messages is a _producer_ :
    
<p  align="center">
<img  src="https://www.rabbitmq.com/img/tutorials/producer.png">
</p>
    
*   _A queue_ is the name for the post box in RabbitMQ. Although messages flow through RabbitMQ and your applications, they can only be stored inside a _queue_. A _queue_ is only bound by the host's memory & disk limits, it's essentially a large message buffer. Many _producers_ can send messages that go to one queue, and many _consumers_ can try to receive data from one _queue_. This is how we represent a queue:
    
<p  align="center">
<img  src="https://www.rabbitmq.com/img/tutorials/queue.png">
</p>
    
*   _Consuming_ has a similar meaning to receiving. A _consumer_ is a program that mostly waits to receive messages:
    
<p  align="center">
<img  src="https://www.rabbitmq.com/img/tutorials/consumer.png">
</p>
    

Note that the producer, consumer, and broker do not have to reside on the same host; indeed in most applications they don't. An application can be both a producer and consumer, too.

"Hello World"
-------------
### (using the Go RabbitMQ client)

In this part of the tutorial we'll write two small programs in Go; a producer that sends a single message, and a consumer that receives messages and prints them out. We'll gloss over some of the detail in the [Go RabbitMQ](https://pkg.go.dev/github.com/rabbitmq/amqp091-go) API, concentrating on this very simple thing just to get started. It's a "Hello World" of messaging.

In the diagram below, "P" is our producer and "C" is our consumer. The box in the middle is a queue - a message buffer that RabbitMQ keeps on behalf of the consumer.

<p  align="center">
<img  src="https://www.rabbitmq.com/img/tutorials/python-one.png">
</p>

> #### The Go RabbitMQ client library
> 
> RabbitMQ speaks multiple protocols. This tutorial uses AMQP 0-9-1, which is an open, general-purpose protocol for messaging. There are a number of clients for RabbitMQ in [many different languages](http://rabbitmq.com/devtools.html). We'll use the Go amqp client in this tutorial.
> 
> First, install amqp using go get:
> 
> ```bash
> go get github.com/rabbitmq/amqp091-go
>```


Now we have *amqp* installed, we can write some code.

### Sending

<p  align="center">
<img  src="https://www.rabbitmq.com/img/tutorials/sending.png">
</p>

We'll call our message publisher (sender) send.go and our message consumer (receiver) receive.go. The publisher will connect to RabbitMQ, send a single message, then exit.

In [send.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/main/go/send.go), we need to import the library first:

```go
package main

import (
  "context"
  "log"
  "time"

  amqp "github.com/rabbitmq/amqp091-go"
)
```


We also need a helper function to check the return value for each amqp call:

```go
func failOnError(err error, msg string) {
  if err != nil {
    log.Panicf("%s: %s", msg, err)
  }
}
```


then connect to RabbitMQ server

```go
conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
failOnError(err, "Failed to connect to RabbitMQ")
defer conn.Close()
```

The connection abstracts the socket connection, and takes care of protocol version negotiation and authentication and so on for us. Next we create a channel, which is where most of the API for getting things done resides:

```go
ch, err := conn.Channel()
failOnError(err, "Failed to open a channel")
defer ch.Close()
```


To send, we must declare a queue for us to send to; then we can publish a message to the queue:

```go
q, err := ch.QueueDeclare(
  "hello", // name
  false,   // durable
  false,   // delete when unused
  false,   // exclusive
  false,   // no-wait
  nil,     // arguments
)
failOnError(err, "Failed to declare a queue")

ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

body := "Hello World!"
err = ch.PublishWithContext(ctx,
  "",     // exchange
  q.Name, // routing key
  false,  // mandatory
  false,  // immediate
  amqp.Publishing {
    ContentType: "text/plain",
    Body:        []byte(body),
  })
failOnError(err, "Failed to publish a message")
log.Printf(" [x] Sent %s\n", body)

```


Declaring a queue is idempotent - it will only be created if it doesn't exist already. The message content is a byte array, so you can encode whatever you like there.

[Here's the whole send.go script](https://github.com/rabbitmq/rabbitmq-tutorials/blob/main/go/send.go).

> #### Sending doesn't work!
> 
> If this is your first time using RabbitMQ and you don't see the "Sent" message then you may be left scratching your head wondering what could be wrong. Maybe the broker was started without enough free disk space (by default it needs at least 200 MB free) and is therefore refusing to accept messages. Check the broker logfile to confirm and reduce the limit if necessary. The [configuration file documentation](https://www.rabbitmq.com/configure.html#config-items) will show you how to set disk\_free\_limit.

### Receiving

That's it for our publisher. Our consumer listens for messages from RabbitMQ, so unlike the publisher which publishes a single message, we'll keep the consumer running to listen for messages and print them out.


<p  align="center">
<img  src="https://www.rabbitmq.com/img/tutorials/receiving.png">
</p>

The code (in [receive.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/main/go/receive.go)) has the same import and helper function as send:

```go
package main

import (
  "log"

  amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
  if err != nil {
    log.Panicf("%s: %s", msg, err)
  }
}

```


Setting up is the same as the publisher; we open a connection and a channel, and declare the queue from which we're going to consume. Note this matches up with the queue that send publishes to.

```go
conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
failOnError(err, "Failed to connect to RabbitMQ")
defer conn.Close()

ch, err := conn.Channel()
failOnError(err, "Failed to open a channel")
defer ch.Close()

q, err := ch.QueueDeclare(
  "hello", // name
  false,   // durable
  false,   // delete when unused
  false,   // exclusive
  false,   // no-wait
  nil,     // arguments
)
failOnError(err, "Failed to declare a queue")

```


Note that we declare the queue here, as well. Because we might start the consumer before the publisher, we want to make sure the queue exists before we try to consume messages from it.

We're about to tell the server to deliver us the messages from the queue. Since it will push us messages asynchronously, we will read the messages from a channel (returned by amqp::Consume) in a goroutine.

```go
msgs, err := ch.Consume(
  q.Name, // queue
  "",     // consumer
  true,   // auto-ack
  false,  // exclusive
  false,  // no-local
  false,  // no-wait
  nil,    // args
)
failOnError(err, "Failed to register a consumer")

var forever chan struct{}

go func() {
  for d := range msgs {
    log.Printf("Received a message: %s", d.Body)
  }
}()

log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
<-forever

```


[Here's the whole receive.go script](https://github.com/rabbitmq/rabbitmq-tutorials/blob/main/go/receive.go).

### Putting it all together

Now we can run both scripts. In a terminal, run the publisher:

```bash
go run send.go

```


then, run the consumer:

```bash
go run receive.go

```


The consumer will print the message it gets from the publisher via RabbitMQ. The consumer will keep running, waiting for messages (Use Ctrl-C to stop it), so try running the publisher from another terminal.

If you want to check on the queue, try using rabbitmqctl list\_queues.

Time to move on to [part 2](https://www.rabbitmq.com/tutorials/tutorial-two-go.html) and build a simple _work queue_.

Production \[Non-\]Suitability Disclaimer
-----------------------------------------

Please keep in mind that this and other tutorials are, well, tutorials. They demonstrate one new concept at a time and may intentionally oversimplify some things and leave out others. For example topics such as connection management, error handling, connection recovery, concurrency and metric collection are largely omitted for the sake of brevity. Such simplified code should not be considered production ready.

Please take a look at the rest of the [documentation](https://www.rabbitmq.com/documentation.html) before going live with your app. We particularly recommend the following guides: [Publisher Confirms and Consumer Acknowledgements](https://www.rabbitmq.com/confirms.html), [Production Checklist](https://www.rabbitmq.com/production-checklist.html) and [Monitoring](https://www.rabbitmq.com/monitoring.html).

Getting Help and Providing Feedback
-----------------------------------

If you have questions about the contents of this tutorial or any other topic related to RabbitMQ, don't hesitate to ask them on the [RabbitMQ mailing list](https://groups.google.com/forum/#!forum/rabbitmq-users).

Help Us Improve the Docs <3
---------------------------

If you'd like to contribute an improvement to the site, its source is [available on GitHub](https://github.com/rabbitmq/rabbitmq-website). Simply fork the repository and submit a pull request. Thank you!
