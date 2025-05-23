# RabbitMQ 开发笔记

## 一、六种消息模式

### 1. Simple Work Queue（简单工作队列/点对点模式）

- **定义**：生产者将消息发送到队列，多个消费者竞争消费消息，采用 **轮询（Round Robin）** 分配。
- **特点**：
  - 每条消息由一个消费者处理。
  - 消息默认自动确认（ACK）。
- **适用场景**：任务分发、负载均衡。
- **交换机**：默认交换机（无需显式声明）。

[代码](#001)

---

### 2. Work Queues（公平队列/能者多劳模式）

- **定义**：消费者需手动确认消息处理完成（ACK），未确认的消息不会被分配给其他消费者。
- **特点**：
  - 根据消费者处理能力分配任务，避免慢消费者拖慢整体进度。
  - 需显式设置 `no_ack=false` 并手动调用 `basic_ack`。
- **适用场景**：高负载任务处理、资源不均衡的集群环境。
- **交换机**：默认交换机。

[代码](#001)

---

### 3. Publish/Subscribe（发布订阅模式/Fanout）

- **定义**：生产者发送的消息会被广播到所有绑定到该交换机的队列。
- **特点**：
  - 消息被所有订阅者接收。
  - 不依赖路由键（Route Key），直接通过 `fanout` 交换机转发。
- **适用场景**：实时通知、日志同步（如监控系统）。
- **交换机**：`fanout`。

[代码](#002)

---

### 4. Routing（路由模式/Direct）

- **定义**：消息根据路由键（Route Key）精确匹配绑定到交换机的队列。
- **特点**：
  - 消息仅发送到路由键与队列绑定键完全匹配的队列。
  - 支持一对一或多对一的路由。
- **适用场景**：定向消息传递（如订单状态更新）。
- **交换机**：`direct`。

---

### 5. Topics（主题模式/通配符路由）

- **定义**：通过通配符规则匹配路由键，实现灵活的消息过滤。
- **特点**：
  - 路由键为多词组合（如 `stock.usd.nyse`）。
  - 支持通配符 `*`（匹配单个词）和 `#`（匹配多个词）。
- **适用场景**：复杂事件过滤（如股票价格监控、日志分级）。
- **交换机**：`topic`。

---

### 6. RPC（远程过程调用模式）

- **定义**：客户端发送请求并等待服务端响应，通过临时队列和消息相关联。
- **特点**：
  - 生产者（客户端）发送请求到队列，消费者处理后将结果发送回指定队列。
  - 需要 `correlation_id` 和 `reply_to` 字段关联请求与响应。
- **适用场景**：需要同步结果的服务调用（如计算任务）。
- **交换机**：默认交换机或自定义。

---

## 二、四种交换机类型

| **交换机类型** | **路由规则** | **适用场景** |
| ------------------ | ----------------------------------------------- | --------------- |
| **Direct**（直连交换机） | 路由键与绑定键完全匹配。 | 精确消息路由（如订单状态更新） |
| **Fanout**（扇形交换机） | 忽略路由键，广播到所有绑定队列。 | 实时广播（如系统通知） |
| **Topic**（主题交换机） | 路由键通过通配符（`*` 或 `#`）匹配绑定键。 | 复杂路由规则（如日志分级过滤） |
| **Headers**（头部交换机） | 根据消息头（而非路由键）匹配，需设置 `x-match` 参数（`any` 或 `all`）。 | 基于元数据的高级路由 |

---

## 三、关键概念补充

1. **ACK（手动确认）**：  
   在公平队列（Work Queues）中，需手动发送 `basic_ack`，确保消息仅被一个消费者处理。
2. **绑定（Binding）**：  
   队列需通过 `queue_bind` 显式绑定到交换机，指定路由键或规则。
3. **持久化与可靠性**：  
   - 队列和消息可设置为持久化（`durable` 和 `delivery_mode`），防止服务器重启丢失数据。
   - 生产者可通过 `Publisher Confirms` 确保消息到达交换机。

---

## 四、C#代码

### <a id="001">工作队列</a>

**生产者（发送消息到队列）**

```csharp
using RabbitMQ.Client;
using System.Text;

string queue = "work_queue";
string message = "Hello RabbitMQ Work Queue!";

ConnectionFactory factory = new ConnectionFactory { HostName = "192.168.1.17", Port = 5672, UserName = "admin", Password = "admin", VirtualHost = "/" }; // 1. 创建连接工厂
using (IConnection connection = await factory.CreateConnectionAsync()) // 2. 建立连接和通道
{
    using (IChannel channel = await connection.CreateChannelAsync())
    {
        await channel.QueueDeclareAsync(queue: queue, durable: true, exclusive: false, autoDelete: false, arguments: null); // 3. 声明队列（持久化）
        await channel.BasicPublishAsync(exchange: "", routingKey: queue, body: Encoding.UTF8.GetBytes(message)); // 4. 发送消息（持久化）使用默认交换机
        Console.WriteLine($"[x] 发送：{message}");
    }
}
```

**消费者（接收并处理消息）**

```csharp
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System.Text;

string queue = "work_queue";
IChannel channel;

ConnectionFactory factory = new ConnectionFactory { HostName = "192.168.1.17", Port = 5672, UserName = "admin", Password = "admin", VirtualHost = "/" };
using (IConnection connection = await factory.CreateConnectionAsync())
{
    using (channel = await connection.CreateChannelAsync())
    {
        await channel.QueueDeclareAsync(queue: queue, durable: true, exclusive: false, autoDelete: false, arguments: null); // 1. 声明队列（确保存在）
        await channel.BasicQosAsync(prefetchSize: 0, prefetchCount: 1, global: false); // 2. 设置公平分发（每个消费者一次只能处理一个消息）
        AsyncEventingBasicConsumer consumer = new AsyncEventingBasicConsumer(channel); // 3. 创建消费者回调
        consumer.ReceivedAsync += ConsumeAck;
        await channel.BasicConsumeAsync(queue: queue, autoAck: false, consumer: consumer);

        Console.WriteLine("[*]正在等待消息。按[enter]退出。");
        Console.ReadLine();
    }
}

/// <summary>
/// 
/// </summary>
/// <param name="model"></param>
/// <param name="ea"></param>
/// <returns></returns>
async Task ConsumeAck(object model, BasicDeliverEventArgs ea)
{
    var body = ea.Body.ToArray();
    var message = Encoding.UTF8.GetString(body);
    Console.WriteLine("[x] 收到：{0}", message);
    Thread.Sleep(1000); // 模拟耗时任务（例如睡眠1秒）
    Console.WriteLine("[x] 完成");
    await channel.BasicAckAsync(deliveryTag: ea.DeliveryTag, multiple: false); // 4. 手动确认消息（ACK）
}
```

### <a id="002">发布订阅</a>

**生产者（发送消息到交换机）**

```csharp
using RabbitMQ.Client;
using System.Text;

string exchange = "logs_exchange";
string message = "Hello RabbitMQ Work Queue!";

ConnectionFactory factory = new ConnectionFactory { HostName = "192.168.1.17", Port = 5672, UserName = "admin", Password = "admin", VirtualHost = "/" }; // 1. 创建连接工厂
using (IConnection connection = await factory.CreateConnectionAsync()) // 2. 建立连接和通道
{
    using (IChannel channel = await connection.CreateChannelAsync())
    {
        await channel.ExchangeDeclareAsync(exchange: exchange, type: ExchangeType.Fanout, durable: true); // 3. 声明 Fanout 类型的交换机
        await channel.BasicPublishAsync(exchange: exchange, routingKey: string.Empty, body: Encoding.UTF8.GetBytes(message)); // 4. 发送消息（持久化）
        Console.WriteLine($"[x] 发送：{message}");
    }
}
```

**消费者（接收并处理消息）**

```csharp
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System.Text;

string exchange = "work_queue";
IChannel channel;
QueueDeclareOk queue;

ConnectionFactory factory = new ConnectionFactory { HostName = "192.168.1.17", Port = 5672, UserName = "admin", Password = "admin", VirtualHost = "/" };
using (IConnection connection = await factory.CreateConnectionAsync())
{
    using (channel = await connection.CreateChannelAsync())
    {
        await channel.ExchangeDeclareAsync(exchange: exchange, type: ExchangeType.Fanout, durable: true); // 1. 声明 fanout 类型的交换机（确保存在）
        queue = await channel.QueueDeclareAsync(); // 2. 声明临时队列（自动删除）
        await channel.QueueBindAsync(queue: queue.QueueName, exchange: "logs_exchange", routingKey: ""); // 3. 将队列绑定到交换机
        AsyncEventingBasicConsumer consumer = new AsyncEventingBasicConsumer(channel); // 3. 创建消费者回调
        consumer.ReceivedAsync += ConsumeAck;
        await channel.BasicConsumeAsync(queue: queue.QueueName, autoAck: true, consumer: consumer); // 5. 开始消费（自动确认消息）

        Console.WriteLine("[*]正在等待消息。按[enter]退出。");
        Console.ReadLine();
    }
}

/// <summary>
/// 
/// </summary>
/// <param name="model"></param>
/// <param name="ea"></param>
/// <returns></returns>
async Task ConsumeAck(object model, BasicDeliverEventArgs ea)
{
    var body = ea.Body.ToArray();
    var message = Encoding.UTF8.GetString(body);
    Console.WriteLine($"[x] Received by {queue.QueueName}: {message}");
    await channel.BasicAckAsync(ea.DeliveryTag, false); // 手动确认消息（可选，如果需要保证消息处理可靠性）
}
```

### <a id="003">路由队列</a>

**生产者**

```csharp
```

**消费者**

```csharp
```

### <a id="004">队列</a>

**生产者**

```csharp
```

**消费者**

```csharp
```

### <a id="005">队列</a>

**生产者**

```csharp
```

**消费者**

```csharp
```
