# amqp-client-ts-clean-code-sample

### Using AMQP in Node.js

**Bad:**

```typescript
import { AMQPClient } from '@cloudamqp/amqp-client'

async function run() {
    try {
        const amqp = new AMQPClient("amqp://localhost")
        const conn = await amqp.connect()
        const ch = await conn.channel()
        const q = await ch.queue()
        const consumer = await q.subscribe({noAck: true}, async (msg) => {
            console.log(msg.bodyToString())
            await consumer.cancel()
        })
        await q.publish("Hello World", {deliveryMode: 2})
        await consumer.wait() // will block until consumer is canceled or throw an error if server closed channel/connection
        await conn.close()
    } catch (e) {
        console.error("ERROR", e)
        e.connection.close()
        setTimeout(run, 1000) // will try to reconnect in 1s
    }
}

run()
```

**Good:**

```typescript
import { AMQPClient } from '@cloudamqp/amqp-client'

class AMQPService {
    constructor(url) {
        this.url = url;
        this.conn = null;
    }

    async connect() {
        const amqp = new AMQPClient(this.url)
        this.conn = await amqp.connect()
        return this.conn;
    }

    async closeConnection() {
        if (this.conn) {
            await this.conn.close()
        }
    }
}

class MessageConsumer {
    constructor(channel, queue) {
        this.channel = channel;
        this.queue = queue;
    }

    async consumeMessage() {
        const consumer = await this.queue.subscribe({noAck: true}, async (msg) => {
            console.log(msg.bodyToString())
            await consumer.cancel()
        })
        await this.queue.publish("Hello World", {deliveryMode: 2})
        await consumer.wait() // will block until consumer is canceled or throw an error if server closed channel/connection
    }
}

async function run() {
    const service = new AMQPService("amqp://localhost");
    try {
        const conn = await service.connect();
        const ch = await conn.channel()
        const q = await ch.queue()
        const consumer = new MessageConsumer(ch, q);
        await consumer.consumeMessage();
    } catch (e) {
        console.error("ERROR", e)
        if (e.connection) {
            e.connection.close()
        }
        setTimeout(run, 1000) // will try to reconnect in 1s
    } finally {
        await service.closeConnection();
    }
}

run()

```
