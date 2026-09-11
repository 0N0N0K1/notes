# 1. Go RabbitMQ 客户端库

go文档地址： https://pkg.go.dev/github.com/rabbitmq/amqp091-go  
RabbitMQ官网: https://www.rabbitmq.com/  （看一些简单示例）

```
// 导入
go get github.com/rabbitmq/amqp091-go
```


# 2. 最佳实践
| 模式    | 交换机类型      | 适用场景            |
| ----- | ---------- | --------------- |
| 简单队列  | 默认（direct） | 一对一通信，单生产者单消费者  |
| 工作队列  | 默认（direct） | 任务分发，多消费者竞争消费   |
| 发布/订阅 | `fanout`   | 广播通知，所有订阅者都收到消息 |
| 路由    | `direct`   | 按固定 key 精确匹配投递  |
| 主题    | `topic`    | 按通配符模式灵活匹配投递    |
| 延迟队列  | DLX + TTL  | 定时任务、订单超时取消等    |
## 2.1 发布者断线重连与确定
封装了一个 Client 对象，在连接失败时自动重连，并且在连接成功之前会阻塞所有推送，还确认了所有推送消息
![[Pasted image 20260911113614.png]]
```go
package main  
  
import (  
    "context"  
    "errors"    "log"    "os"    "sync"    "time"  
    amqp "github.com/rabbitmq/amqp091-go"  
)  
  
type Client struct {  
    m          *sync.Mutex  
    queueName  string  
    logger     *log.Logger  
    connection *amqp.Connection  
    channel    *amqp.Channel  
    //连接主动关闭的chan  
    done chan bool  
    //连接关闭的错误的chan  
    notifyConnClose chan *amqp.Error  
    //信道关闭的错误的chan  
    notifyChanClose chan *amqp.Error  
    //confirm消息的chan  
    notifyConfirm chan amqp.Confirmation  
    //连接是否建立,可以发送  
    isReady bool  
}  
  
const (  
    reconnectDelay = 5 * time.Second  
  
    reInitDelay = 2 * time.Second  
  
    resendDelay = 5 * time.Second  
)  
  
var (  
    errNotConnected  = errors.New("not connected to a server")  
    errAlreadyClosed = errors.New("already closed: not connected to the server")  
    errShutdown      = errors.New("client is shutting down")  
)  
  
func main() {  
    queueName := "job_queue"  
    addr := "amqp://guest:guest@localhost:5672/"  
    queue := New(queueName, addr)  
    message := []byte("message")  
  
    // 定时关闭  
    ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(time.Second*20))  
    defer cancel()  
loop:  
    for {  
       select {  
       // Attempt to push a message every 2 seconds  
       //可以换成一个chan接收要发送的消息取代定时发送  
       case <-time.After(time.Second * 2):  
          if err := queue.Push(message); err != nil {  
             log.Printf("Push failed: %s\n", err)  
          } else {  
             log.Println("Push succeeded!")  
          }  
       case <-ctx.Done():  
          if err := queue.Close(); err != nil {  
             log.Printf("Close failed: %s\n", err)  
          }  
          break loop  
       }  
    }  
}  
  
// New creates a new consumer state instance, and automatically// attempts to connect to the server.  
func New(queueName, addr string) *Client {  
    client := Client{  
       m:         &sync.Mutex{},  
       logger:    log.New(os.Stdout, "", log.LstdFlags),  
       queueName: queueName,  
       done:      make(chan bool),  
    }  
    go client.handleReconnect(addr)  
    return &client  
}  
  
// handleReconnect will wait for a connection error on// notifyConnClose, and then continuously attempt to reconnect.  
func (client *Client) handleReconnect(addr string) {  
    for {  
       client.m.Lock()  
       client.isReady = false  
       client.m.Unlock()  
  
       client.logger.Println("Attempting to connect")  
  
       conn, err := client.connect(addr)  
       if err != nil {  
          client.logger.Println("Failed to connect. Retrying...")  
  
          select {  
          case <-client.done:  
             return  
          case <-time.After(reconnectDelay):  
          }  
          continue  
       }  
  
       if done := client.handleReInit(conn); done {  
          break  
       }  
    }  
}  
  
// connect will create a new AMQP connection  
func (client *Client) connect(addr string) (*amqp.Connection, error) {  
    conn, err := amqp.Dial(addr)  
    if err != nil {  
       return nil, err  
    }  
  
    client.changeConnection(conn)  
    client.logger.Println("Connected!")  
    return conn, nil  
}  
  
// handleReInit will wait for a channel error// and then continuously attempt to re-initialize both channels  
func (client *Client) handleReInit(conn *amqp.Connection) bool {  
    for {  
       client.m.Lock()  
       client.isReady = false  
       client.m.Unlock()  
  
       err := client.init(conn)  
       if err != nil {  
          client.logger.Println("Failed to initialize channel. Retrying...")  
  
          select {  
          case <-client.done:  
             return true  
          case <-client.notifyConnClose:  
             client.logger.Println("Connection closed. Reconnecting...")  
             return false  
          case <-time.After(reInitDelay):  
          }  
          continue  
       }  
  
       select {  
       case <-client.done:  
          return true  
       case <-client.notifyConnClose:  
          client.logger.Println("Connection closed. Reconnecting...")  
          return false  
       case <-client.notifyChanClose:  
          client.logger.Println("Channel closed. Re-running init...")  
       }  
    }  
}  
  
// init will initialize channel & declare queuefunc (client *Client) init(conn *amqp.Connection) error {  
    ch, err := conn.Channel()  
    if err != nil {  
       return err  
    }  
  
    err = ch.Confirm(false)  
    if err != nil {  
       return err  
    }  
    _, err = ch.QueueDeclare(  
       client.queueName,  
       false,  
       false,  
       false,  
       false,  
       nil,  
    )  
    if err != nil {  
       return err  
    }  
  
    client.changeChannel(ch)  
    client.m.Lock()  
    client.isReady = true  
    client.m.Unlock()  
    client.logger.Println("Setup!")  
  
    return nil  
}  
  
// changeConnection takes a new connection to the queue,// and updates the close listener to reflect this.  
func (client *Client) changeConnection(connection *amqp.Connection) {  
    client.connection = connection  
    client.notifyConnClose = make(chan *amqp.Error, 1)  
    client.connection.NotifyClose(client.notifyConnClose)  
}  
  
// changeChannel takes a new channel to the queue,// and updates the channel listeners to reflect this.func (client *Client) changeChannel(channel *amqp.Channel) {  
    client.channel = channel  
    client.notifyChanClose = make(chan *amqp.Error, 1)  
    client.notifyConfirm = make(chan amqp.Confirmation, 1)  
    client.channel.NotifyClose(client.notifyChanClose)  
    client.channel.NotifyPublish(client.notifyConfirm)  
}  
  
// Push will push data onto the queue, and wait for a confirmation.// This will block until the server sends a confirmation. Errors are  
// only returned if the push action itself fails, see UnsafePush.  
func (client *Client) Push(data []byte) error {  
    client.m.Lock()  
    if !client.isReady {  
       client.m.Unlock()  
       return errors.New("failed to push: not connected")  
    }  
    client.m.Unlock()  
    for {  
       err := client.UnsafePush(data)  
       if err != nil {  
          client.logger.Println("Push failed. Retrying...")  
          select {  
          case <-client.done:  
             return errShutdown  
          case <-time.After(resendDelay):  
          }  
          continue  
       }  
       confirm := <-client.notifyConfirm  
       if confirm.Ack {  
          client.logger.Printf("Push confirmed [%d]!", confirm.DeliveryTag)  
          return nil  
       }  
    }  
}  
  
// UnsafePush will push to the queue without checking for// confirmation. It returns an error if it fails to connect.  
// No guarantees are provided for whether the server will  
// receive the message.  
func (client *Client) UnsafePush(data []byte) error {  
    client.m.Lock()  
    if !client.isReady {  
       client.m.Unlock()  
       return errNotConnected  
    }  
    client.m.Unlock()  
  
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)  
    defer cancel()  
  
    return client.channel.PublishWithContext(  
       ctx,  
       "",  
       client.queueName,  
       false,  
       false,  
       amqp.Publishing{  
          ContentType: "text/plain",  
          Body:        data,  
       },  
    )  
}  
  
// Consume will continuously put queue items on the channel.// It is required to call delivery.Ack when it has been  
// successfully processed, or delivery.Nack when it fails.  
// Ignoring this will cause data to build up on the server.  
func (client *Client) Consume() (<-chan amqp.Delivery, error) {  
    client.m.Lock()  
    if !client.isReady {  
       client.m.Unlock()  
       return nil, errNotConnected  
    }  
    client.m.Unlock()  
  
    if err := client.channel.Qos(  
       1,  
       0,  
       false,  
    ); err != nil {  
       return nil, err  
    }  
  
    return client.channel.Consume(  
       client.queueName,  
       "",  
       false,  
       false,  
       false,  
       false,  
       nil,  
    )  
}  
  
// Close will cleanly shut down the channel and connection.
func (client *Client) Close() error {  
    client.m.Lock()  
  
    defer client.m.Unlock()  
  
    if !client.isReady {  
       return errAlreadyClosed  
    }  
    close(client.done)  
    err := client.channel.Close()  
    if err != nil {  
       return err  
    }  
    err = client.connection.Close()  
    if err != nil {  
       return err  
    }  
  
    client.isReady = false  
    return nil  
}
```
## 2.2 消费者断线重连与ack
![[Pasted image 20260911113414.png]]
```go
package main  
  
import (  
    "context"  
    "errors"    "log"    "os"    "sync"    "time"  
    amqp "github.com/rabbitmq/amqp091-go"  
)  
  
func main() {  
    queueName := "job_queue"  
    addr := "amqp://guest:guest@localhost:5672/"  
    queue := New(queueName, addr)  
  
    ctx, cancel := context.WithTimeout(context.Background(), time.Second*30)  
    defer cancel()  
  
    deliveries, err := queue.Consume()  
    if err != nil {  
       log.Printf("Could not start consuming: %s\n", err)  
       return  
    }  
  
    chClosedCh := make(chan *amqp.Error, 1)  
    queue.channel.NotifyClose(chClosedCh)  
  
    for {  
       select {  
       case <-ctx.Done():  
          err := queue.Close()  
          if err != nil {  
             log.Printf("Close failed: %s\n", err)  
          }  
          return  
  
       case amqErr := <-chClosedCh:  
          // This case handles the event of closed channel e.g. abnormal shutdown  
          log.Printf("AMQP Channel closed due to: %s\n", amqErr)  
  
          deliveries, err = queue.Consume()  
          if err != nil {  
             // If the AMQP channel is not ready, it will continue the loop. Next  
             // iteration will enter this case because chClosedCh is closed by the             // library             log.Println("Error trying to consume, will try again")  
             continue  
          }  
  
          // Re-set channel to receive notifications  
          // The library closes this channel after abnormal shutdown          chClosedCh = make(chan *amqp.Error, 1)  
          queue.channel.NotifyClose(chClosedCh)  
  
       case delivery := <-deliveries:  
          // Ack a message every 2 seconds  
          log.Printf("Received message: %s\n", delivery.Body)  
          if err := delivery.Ack(false); err != nil {  
             log.Printf("Error acknowledging message: %s\n", err)  
          }  
          <-time.After(time.Second * 2)  
       }  
    }  
}  
  
// Client is the base struct for handling connection recovery, consumption and// publishing. Note that this struct has an internal mutex to safeguard against  
// data races. As you develop and iterate over this example, you may need to add  
// further locks, or safeguards, to keep your application safe from data races  
type Client struct {  
    m               *sync.Mutex  
    queueName       string  
    logger          *log.Logger  
    connection      *amqp.Connection  
    channel         *amqp.Channel  
    done            chan bool  
    notifyConnClose chan *amqp.Error  
    notifyChanClose chan *amqp.Error  
    notifyConfirm   chan amqp.Confirmation  
    isReady         bool  
}  
  
const (  
    reconnectDelay = 5 * time.Second  
  
    reInitDelay = 2 * time.Second  
  
    resendDelay = 5 * time.Second  
)  
  
var (  
    errNotConnected  = errors.New("not connected to a server")  
    errAlreadyClosed = errors.New("already closed: not connected to the server")  
    errShutdown      = errors.New("client is shutting down")  
)  
  
// New creates a new consumer state instance, and automatically// attempts to connect to the server.  
func New(queueName, addr string) *Client {  
    client := Client{  
       m:         &sync.Mutex{},  
       logger:    log.New(os.Stdout, "", log.LstdFlags),  
       queueName: queueName,  
       done:      make(chan bool),  
    }  
    go client.handleReconnect(addr)  
    return &client  
}  
  
// handleReconnect will wait for a connection error on// notifyConnClose, and then continuously attempt to reconnect.  
func (client *Client) handleReconnect(addr string) {  
    for {  
       client.m.Lock()  
       client.isReady = false  
       client.m.Unlock()  
  
       client.logger.Println("Attempting to connect")  
  
       conn, err := client.connect(addr)  
       if err != nil {  
          client.logger.Println("Failed to connect. Retrying...")  
  
          select {  
          case <-client.done:  
             return  
          case <-time.After(reconnectDelay):  
          }  
          continue  
       }  
  
       if done := client.handleReInit(conn); done {  
          break  
       }  
    }  
}  
  
// connect will create a new AMQP connection  
func (client *Client) connect(addr string) (*amqp.Connection, error) {  
    conn, err := amqp.Dial(addr)  
    if err != nil {  
       return nil, err  
    }  
  
    client.changeConnection(conn)  
    client.logger.Println("Connected!")  
    return conn, nil  
}  
  
// handleReInit will wait for a channel error// and then continuously attempt to re-initialize both channels  
func (client *Client) handleReInit(conn *amqp.Connection) bool {  
    for {  
       client.m.Lock()  
       client.isReady = false  
       client.m.Unlock()  
  
       err := client.init(conn)  
       if err != nil {  
          client.logger.Println("Failed to initialize channel. Retrying...")  
  
          select {  
          case <-client.done:  
             return true  
          case <-client.notifyConnClose:  
             client.logger.Println("Connection closed. Reconnecting...")  
             return false  
          case <-time.After(reInitDelay):  
          }  
          continue  
       }  
  
       select {  
       case <-client.done:  
          return true  
       case <-client.notifyConnClose:  
          client.logger.Println("Connection closed. Reconnecting...")  
          return false  
       case <-client.notifyChanClose:  
          client.logger.Println("Channel closed. Re-running init...")  
       }  
    }  
}  
  
// init will initialize channel & declare queuefunc (client *Client) init(conn *amqp.Connection) error {  
    ch, err := conn.Channel()  
    if err != nil {  
       return err  
    }  
  
    err = ch.Confirm(false)  
    if err != nil {  
       return err  
    }  
    _, err = ch.QueueDeclare(  
       client.queueName,  
       false,  
       false,  
       false,  
       false,  
       nil,  
    )  
    if err != nil {  
       return err  
    }  
  
    client.changeChannel(ch)  
    client.m.Lock()  
    client.isReady = true  
    client.m.Unlock()  
    client.logger.Println("Setup!")  
  
    return nil  
}  
  
// changeConnection takes a new connection to the queue,// and updates the close listener to reflect this.  
func (client *Client) changeConnection(connection *amqp.Connection) {  
    client.connection = connection  
    client.notifyConnClose = make(chan *amqp.Error, 1)  
    client.connection.NotifyClose(client.notifyConnClose)  
}  
  
// changeChannel takes a new channel to the queue,// and updates the channel listeners to reflect this.func (client *Client) changeChannel(channel *amqp.Channel) {  
    client.channel = channel  
    client.notifyChanClose = make(chan *amqp.Error, 1)  
    client.notifyConfirm = make(chan amqp.Confirmation, 1)  
    client.channel.NotifyClose(client.notifyChanClose)  
    client.channel.NotifyPublish(client.notifyConfirm)  
}  
  
// Push will push data onto the queue, and wait for a confirmation.// This will block until the server sends a confirmation. Errors are  
// only returned if the push action itself fails, see UnsafePush.  
func (client *Client) Push(data []byte) error {  
    client.m.Lock()  
    if !client.isReady {  
       client.m.Unlock()  
       return errors.New("failed to push: not connected")  
    }  
    client.m.Unlock()  
    for {  
       err := client.UnsafePush(data)  
       if err != nil {  
          client.logger.Println("Push failed. Retrying...")  
          select {  
          case <-client.done:  
             return errShutdown  
          case <-time.After(resendDelay):  
          }  
          continue  
       }  
       confirm := <-client.notifyConfirm  
       if confirm.Ack {  
          client.logger.Printf("Push confirmed [%d]!", confirm.DeliveryTag)  
          return nil  
       }  
    }  
}  
  
// UnsafePush will push to the queue without checking for// confirmation. It returns an error if it fails to connect.  
// No guarantees are provided for whether the server will  
// receive the message.  
func (client *Client) UnsafePush(data []byte) error {  
    client.m.Lock()  
    if !client.isReady {  
       client.m.Unlock()  
       return errNotConnected  
    }  
    client.m.Unlock()  
  
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)  
    defer cancel()  
  
    return client.channel.PublishWithContext(  
       ctx,  
       "",  
       client.queueName,  
       false,  
       false,  
       amqp.Publishing{  
          ContentType: "text/plain",  
          Body:        data,  
       },  
    )  
}  
  
// Consume will continuously put queue items on the channel.// It is required to call delivery.Ack when it has been  
// successfully processed, or delivery.Nack when it fails.  
// Ignoring this will cause data to build up on the server.  
func (client *Client) Consume() (<-chan amqp.Delivery, error) {  
    client.m.Lock()  
    if !client.isReady {  
       client.m.Unlock()  
       return nil, errNotConnected  
    }  
    client.m.Unlock()  
  
    if err := client.channel.Qos(  
       1,  
       0,  
       false,  
    ); err != nil {  
       return nil, err  
    }  
  
    return client.channel.Consume(  
       client.queueName,  
       "",  
       false,  
       false,  
       false,  
       false,  
       nil,  
    )  
}  
  
// Close will cleanly shut down the channel and connection.func (client *Client) Close() error {  
    client.m.Lock()  
  
    defer client.m.Unlock()  
  
    if !client.isReady {  
       return errAlreadyClosed  
    }  
    close(client.done)  
    err := client.channel.Close()  
    if err != nil {  
       return err  
    }  
    err = client.connection.Close()  
    if err != nil {  
       return err  
    }  
  
    client.isReady = false  
    return nil  
}
```
## 2.3 延时队列
![[Pasted image 20260911122525.png]]
```go
TTL + 死信交换机（DLX） 组合实现延迟队列：
消息先进入设置了 TTL 的普通队列，过期后自动转发到死信交换机，再由死信队列的消费者处理,可再次发送到 TTL 的普通队列或者进入日志等待人工处理

// 死信交换机
err = ch.ExchangeDeclare("dlx_exchange", "direct", true, false, false, false, nil)
// 死信队列
_, err = ch.QueueDeclare("dlx_queue", true, false, false, false, nil)
// 绑定
err = ch.QueueBind("dlx_queue", "dlx_key", "dlx_exchange", false, nil)

args := amqp.Table{
    "x-message-ttl":             int32(60000),        // 消息 TTL：60 秒
    "x-dead-letter-exchange":    "dlx_exchange",      // 死信交换机
    "x-dead-letter-routing-key": "dlx_key",           // 死信路由键
}
_, err = ch.QueueDeclare("delay_queue", true, false, false, false, args)

err = ch.PublishWithContext(ctx,
    "",              // 默认交换机
    "delay_queue",   // 路由到延迟队列
    false, false,
    amqp.Publishing{
        DeliveryMode: amqp.Persistent,
        ContentType:  "text/plain",
        Body:         []byte(body),
        // 也可以在消息级别设置 TTL（会覆盖队列的 x-message-ttl）
        // Expiration: "60000",
    })

消息在 `delay_queue` 中等待 60 秒后自动过期，被转发到 `dlx_exchange`，最终进入 `dlx_queue`
```