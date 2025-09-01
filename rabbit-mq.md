## 1) RabbitMQ: exchanges, queues, bindings, routing keys, consumers (post office analogy)

**Answer:**

- **Exchange** = post office (routes)
- **Queue** = mailbox (stores)
- **Binding** = rule connecting post office → mailbox
- **Routing key** = address on envelope (e.g., `orders.created`)
- **Consumer** = person who opens the mailbox

---

## 2) Direct exchange with selective routing (Billing, Shipping, Analytics)

**Answer:**  
Use a **direct exchange** and bind queues with **exact routing keys**.

---

![RabbitMQ Diagram](Resources\rb-mq\routing-direct.png)

### Publisher

```csharp
// Declare a direct exchange
channel.ExchangeDeclare(
    exchange: "orders-direct-exchange",
    type: ExchangeType.Direct,
    durable: true
);

void Publish(string routingKey, string message)
{
    var body = Encoding.UTF8.GetBytes(message);

    channel.BasicPublish(
        exchange: "orders-direct-exchange",
        routingKey: routingKey,
        basicProperties: null,
        body: body
    );
}

// Example publishes
Publish(routingKey: "orders.created",   message: "Order 123 created");
Publish(routingKey: "orders.shipped",   message: "Order 123 shipped");
Publish(routingKey: "orders.cancelled", message: "Order 123 cancelled");
```

---

### Consumers

```csharp
// Billing: only created
channel.QueueDeclare(
    queue: "billing-queue",
    durable: true,
    exclusive: false,
    autoDelete: false
);
channel.QueueBind(
    queue: "billing-queue",
    exchange: "orders-direct-exchange",
    routingKey: "orders.created"
);

// Shipping: only shipped
channel.QueueDeclare(
    queue: "shipping-queue",
    durable: true,
    exclusive: false,
    autoDelete: false
);
channel.QueueBind(
    queue: "shipping-queue",
    exchange: "orders-direct-exchange",
    routingKey: "orders.shipped"
);

// Analytics: bind to ALL routing keys
channel.QueueDeclare(
    queue: "analytics-queue",
    durable: true,
    exclusive: false,
    autoDelete: false
);
channel.QueueBind(
    queue: "analytics-queue",
    exchange: "orders-direct-exchange",
    routingKey: "orders.created"
);
channel.QueueBind(
    queue: "analytics-queue",
    exchange: "orders-direct-exchange",
    routingKey: "orders.shipped"
);
channel.QueueBind(
    queue: "analytics-queue",
    exchange: "orders-direct-exchange",
    routingKey: "orders.cancelled"
);
```

## 3) Topic exchange with selective routing (Billing, Shipping, Analytics)

**Answer:**  
Use a **topic** exchange and bind queues with keys/wildcards.

![RabbitMQ Diagram](Resources\rb-mq\routing-topic.png)

**Publisher:**

```csharp
channel.ExchangeDeclare(
    exchange: "orders-exchange",
    type: ExchangeType.Topic,
    durable: true
);

void Publish(string routingKey, string message)
{
    var body = Encoding.UTF8.GetBytes(message);
    channel.BasicPublish(
        exchange: "orders-exchange",
        routingKey: routingKey,
        basicProperties: null,
        body: body
    );
}

Publish(routingKey: "orders.created",   message: "Order 123 created");
Publish(routingKey: "orders.shipped",   message: "Order 123 shipped");
Publish(routingKey: "orders.cancelled", message: "Order 123 cancelled");
```

**Consumers:**

```csharp
// Billing: only created
channel.QueueDeclare(
    queue: "billing-queue",
    durable: true,
    exclusive: false,
    autoDelete: false
);
channel.QueueBind(
    queue: "billing-queue",
    exchange: "orders-exchange",
    routingKey: "orders.created"
);

// Shipping: only shipped
channel.QueueDeclare(
    queue: "shipping-queue",
    durable: true,
    exclusive: false,
    autoDelete: false
);
channel.QueueBind(
    queue: "shipping-queue",
    exchange: "orders-exchange",
    routingKey: "orders.shipped"
);

// Analytics: all order events
channel.QueueDeclare(
    queue: "analytics-queue",
    durable: true,
    exclusive: false,
    autoDelete: false
);
channel.QueueBind(
    queue: "analytics-queue",
    exchange: "orders-exchange",
    routingKey: "orders.*"
);
```

---

## 4) Fanout exchange (pub/sub broadcast)

**Answer:**  
Use a **fanout** exchange to broadcast every message to **all** bound queues.  
The routing key is ignored for fanout exchanges.

---

![RabbitMQ Diagram](Resources\rb-mq\pub-sub.png)

### Real-world Example: **User Events Broadcast**

Imagine a system where every **user event** (signup, login, profile update) must be sent to multiple services:

- **Email Service** → sends welcome/login emails
- **Activity Feed Service** → updates the user's feed
- **Audit Service** → records all events for compliance

---

### Publisher

```csharp
channel.ExchangeDeclare(
    exchange: "user-events-exchange",
    type: ExchangeType.Fanout,
    durable: true
);

void Publish(string message)
{
    var body = Encoding.UTF8.GetBytes(message);
    channel.BasicPublish(
        exchange: "user-events-exchange",
        routingKey: "",
        basicProperties: null,
        body: body
    );
}

// Example publishes
Publish(message: "User 42 signed up");
Publish(message: "User 42 logged in");
Publish(message: "User 42 updated profile");
```

---

### Consumers

```csharp
// Email Service: receives all user events
channel.QueueDeclare(
    queue: "email-service-queue",
    durable: true,
    exclusive: false,
    autoDelete: false
);
channel.QueueBind(
    queue: "email-service-queue",
    exchange: "user-events-exchange",
    routingKey: ""
);

// Activity Feed Service: receives all user events
channel.QueueDeclare(
    queue: "activity-feed-queue",
    durable: true,
    exclusive: false,
    autoDelete: false
);
channel.QueueBind(
    queue: "activity-feed-queue",
    exchange: "user-events-exchange",
    routingKey: ""
);

// Audit Service: receives all user events
channel.QueueDeclare(
    queue: "audit-service-queue",
    durable: true,
    exclusive: false,
    autoDelete: false
);
channel.QueueBind(
    queue: "audit-service-queue",
    exchange: "user-events-exchange",
    routingKey: ""
);
```

## 5) Request/Reply pattern (RPC with correlation IDs)

**Answer:**  
Use a **direct exchange** with a **reply queue**.  
The client sends a request message with a `CorrelationId` and `ReplyTo` property.  
The server processes the request and publishes the reply to the `ReplyTo` queue with the same `CorrelationId`.

---

![RabbitMQ Diagram](Resources\rb-mq\request-reply-pattern.png)

### Publisher (Client)

```csharp
// Declare request exchange
channel.ExchangeDeclare(
    exchange: "rpc-exchange",
    type: ExchangeType.Direct,
    durable: true
);

// Declare a temporary reply queue
var replyQueueName = channel.QueueDeclare().QueueName;

// Consumer to receive replies
var consumer = new EventingBasicConsumer(channel);
string correlationId = Guid.NewGuid().ToString();

string response = null;
consumer.Received += (model, ea) =>
{
    if (ea.BasicProperties.CorrelationId == correlationId)
    {
        response = Encoding.UTF8.GetString(ea.Body.ToArray());
        Console.WriteLine($"[Client] Received response: {response}");
    }
};

channel.BasicConsume(
    queue: replyQueueName,
    autoAck: true,
    consumer: consumer
);

// Publish request
var props = channel.CreateBasicProperties();
props.CorrelationId = correlationId;
props.ReplyTo = replyQueueName;

var messageBytes = Encoding.UTF8.GetBytes("GetOrderStatus:123");
channel.BasicPublish(
    exchange: "rpc-exchange",
    routingKey: "order-service",
    basicProperties: props,
    body: messageBytes
);

Console.WriteLine("[Client] Sent request: GetOrderStatus:123");
```

---

### Consumer (Server)

```csharp
// Declare request queue
channel.QueueDeclare(
    queue: "order-service-queue",
    durable: true,
    exclusive: false,
    autoDelete: false
);
channel.QueueBind(
    queue: "order-service-queue",
    exchange: "rpc-exchange",
    routingKey: "order-service"
);

var consumer = new EventingBasicConsumer(channel);
consumer.Received += (model, ea) =>
{
    var body = ea.Body.ToArray();
    var message = Encoding.UTF8.GetString(body);
    Console.WriteLine($"[Server] Received request: {message}");

    // Simulate processing
    string responseMessage = "Order 123 is SHIPPED";

    var replyProps = channel.CreateBasicProperties();
    replyProps.CorrelationId = ea.BasicProperties.CorrelationId;

    var responseBytes = Encoding.UTF8.GetBytes(responseMessage);

    // Send reply to the reply queue
    channel.BasicPublish(
        exchange: "",
        routingKey: ea.BasicProperties.ReplyTo,
        basicProperties: replyProps,
        body: responseBytes
    );

    channel.BasicAck(
        deliveryTag: ea.DeliveryTag,
        multiple: false
    );
};

channel.BasicConsume(
    queue: "order-service-queue",
    autoAck: false,
    consumer: consumer
);
```

---

✅ **Result:**

- Client sends `GetOrderStatus:123` with a `CorrelationId` and `ReplyTo`.
- Server processes and replies back to the `ReplyTo` queue with the same `CorrelationId`.
- Client matches the correlation ID and reads the response.
