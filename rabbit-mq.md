

## 1) RabbitMQ: exchanges, queues, bindings, routing keys, consumers (post office analogy)
**Answer:**  
- **Exchange** = post office (routes)  
- **Queue** = mailbox (stores)  
- **Binding** = rule connecting post office → mailbox  
- **Routing key** = address on envelope (e.g., `orders.created`)  
- **Consumer** = person who opens the mailbox

---

## 2) Topic exchange with selective routing (Billing, Shipping, Analytics)
**Answer:**  
Use a **topic** exchange and bind queues with keys/wildcards.

**Publisher:**

```csharp
channel.ExchangeDeclare("orders-exchange", ExchangeType.Topic, durable: true);

void Publish(string routingKey, string message)
{
    var body = Encoding.UTF8.GetBytes(message);
    channel.BasicPublish(exchange: "orders-exchange",
                         routingKey: routingKey,
                         basicProperties: null,
                         body: body);
}

Publish("orders.created",  "Order 123 created");
Publish("orders.shipped",  "Order 123 shipped");
Publish("orders.cancelled","Order 123 cancelled");
```

**Consumers:**
```csharp
// Billing: only created
channel.QueueDeclare("billing-queue", durable: true, exclusive: false, autoDelete: false);
channel.QueueBind("billing-queue", "orders-exchange", "orders.created");

// Shipping: only shipped
channel.QueueDeclare("shipping-queue", durable: true, exclusive: false, autoDelete: false);
channel.QueueBind("shipping-queue", "orders-exchange", "orders.shipped");

// Analytics: all order events
channel.QueueDeclare("analytics-queue", durable: true, exclusive: false, autoDelete: false);
channel.QueueBind("analytics-queue", "orders-exchange", "orders.*");
```

---



## 3) Fanout exchange (pub/sub broadcast)

**Answer:**  
Use a **fanout** exchange to broadcast every message to **all** bound queues. The routing key is ignored for fanout exchanges.

**Publisher:**

```csharp
channel.ExchangeDeclare("pubsub-exchange", ExchangeType.Fanout, durable: true);

void Publish(string message)
{
    var body = Encoding.UTF8.GetBytes(message);
    channel.BasicPublish(exchange: "pubsub-exchange",
                         routingKey: "",
                         basicProperties: null,
                         body: body);
}

Publish("Order 123 created");
Publish("Order 123 shipped");
Publish("Order 123 cancelled");
```

**Consumers:**

```csharp
// Notifications: receives all messages
channel.QueueDeclare("notifications-queue", durable: true, exclusive: false, autoDelete: false);
channel.QueueBind("notifications-queue", "pubsub-exchange", "");

// Activity: receives all messages
channel.QueueDeclare("activity-queue", durable: true, exclusive: false, autoDelete: false);
channel.QueueBind("activity-queue", "pubsub-exchange", "");

// Audit: receives all messages
channel.QueueDeclare("audit-queue", durable: true, exclusive: false, autoDelete: false);
channel.QueueBind("audit-queue", "pubsub-exchange", "");
```





// Pending Direct Exchahnge routing
