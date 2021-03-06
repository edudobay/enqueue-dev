---
layout: default
title: AMQP
parent: Transports
nav_order: 3
---
{% include support.md %}

# AMQP transport

Implements [AMQP specifications](https://www.rabbitmq.com/specification.html) and implements [amqp interop](https://github.com/queue-interop/amqp-interop) interfaces.
Build on top of [php amqp extension](https://github.com/pdezwart/php-amqp).

Drawbacks:
* [heartbeats will not work properly](https://github.com/pdezwart/php-amqp#persistent-connection)
* [signals will not be properly handled](https://github.com/pdezwart/php-amqp#keeping-track-of-the-workers)

Parts:
* [Installation](#installation)
* [Create context](#create-context)
* [Declare topic](#declare-topic)
* [Declare queue](#declare-queue)
* [Bind queue to topic](#bind-queue-to-topic)
* [Send message to topic](#send-message-to-topic)
* [Send message to queue](#send-message-to-queue)
* [Send priority message](#send-priority-message)
* [Send expiration message](#send-expiration-message)
* [Send delayed message](#send-delayed-message)
* [Consume message](#consume-message)
* [Subscription consumer](#subscription-consumer)
* [Purge queue messages](#purge-queue-messages)

## Installation

_**Warning**: You need amqp extension of at least 1.9.3. Here's how you can [compile](https://github.com/php-enqueue/enqueue-dev/blob/09d209447b9dbdf118bff7d983fcb8b0f919e789/docker/Dockerfile#L8) the extension from the [source code](https://github.com/pdezwart/php-amqp)._

```bash
$ composer require enqueue/amqp-ext
```

## Create context

```php
<?php
use Enqueue\AmqpExt\AmqpConnectionFactory;

// connects to localhost
$connectionFactory = new AmqpConnectionFactory();

// same as above
$factory = new AmqpConnectionFactory('amqp:');

// same as above
$factory = new AmqpConnectionFactory([]);

// connect to AMQP broker at example.com
$factory = new AmqpConnectionFactory([
    'host' => 'example.com',
    'port' => 1000,
    'vhost' => '/',
    'user' => 'user',
    'pass' => 'pass',
    'persisted' => false,
]);

// same as above but given as DSN string
$factory = new AmqpConnectionFactory('amqp://user:pass@example.com:10000/%2f');

// SSL or secure connection
$factory = new AmqpConnectionFactory([
    'dsn' => 'amqps:',
    'ssl_cacert' => '/path/to/cacert.pem',
    'ssl_cert' => '/path/to/cert.pem',
    'ssl_key' => '/path/to/key.pem',
]);

$context = $factory->createContext();

// if you have enqueue/enqueue library installed you can use a factory to build context from DSN
$context = (new \Enqueue\ConnectionFactoryFactory())->create('amqp:')->createContext();
$context = (new \Enqueue\ConnectionFactoryFactory())->create('amqp+ext:')->createContext();
```

## Declare topic.

Declare topic operation creates a topic on a broker side.

```php
<?php
use Interop\Amqp\AmqpTopic;

/** @var \Enqueue\AmqpExt\AmqpContext $context */

$fooTopic = $context->createTopic('foo');
$fooTopic->setType(AmqpTopic::TYPE_FANOUT);
$context->declareTopic($fooTopic);

// to remove topic use delete topic method
//$context->deleteTopic($fooTopic);
```

## Declare queue.

Declare queue operation creates a queue on a broker side.

```php
<?php
use Interop\Amqp\AmqpQueue;

/** @var \Enqueue\AmqpExt\AmqpContext $context */

$fooQueue = $context->createQueue('foo');
$fooQueue->addFlag(AmqpQueue::FLAG_DURABLE);
$context->declareQueue($fooQueue);

// to remove queue use delete queue method
//$context->deleteQueue($fooQueue);
```

## Bind queue to topic

Connects a queue to the topic. So messages from that topic comes to the queue and could be processed.

```php
<?php
use Interop\Amqp\Impl\AmqpBind;

/** @var \Enqueue\AmqpExt\AmqpContext $context */
/** @var \Interop\Amqp\Impl\AmqpQueue $fooQueue */
/** @var \Interop\Amqp\Impl\AmqpTopic $fooTopic */

$context->bind(new AmqpBind($fooTopic, $fooQueue));
```

## Send message to topic

```php
<?php
/** @var \Enqueue\AmqpExt\AmqpContext $context */
/** @var \Interop\Amqp\Impl\AmqpTopic $fooTopic */

$message = $context->createMessage('Hello world!');

$context->createProducer()->send($fooTopic, $message);
```

## Send message to queue

```php
<?php
/** @var \Enqueue\AmqpExt\AmqpContext $context */
/** @var \Interop\Amqp\Impl\AmqpQueue $fooQueue */

$message = $context->createMessage('Hello world!');

$context->createProducer()->send($fooQueue, $message);
```

## Send priority message

```php
<?php
/** @var \Enqueue\AmqpExt\AmqpContext $context */

$fooQueue = $context->createQueue('foo');
$fooQueue->addFlag(AmqpQueue::FLAG_DURABLE);
$fooQueue->setArguments(['x-max-priority' => 10]);
$context->declareQueue($fooQueue);

$message = $context->createMessage('Hello world!');

$context->createProducer()
    ->setPriority(5) // the higher priority the sooner a message gets to a consumer
    //
    ->send($fooQueue, $message)
;
```

## Send expiration message

```php
<?php
/** @var \Enqueue\AmqpExt\AmqpContext $context */
/** @var \Interop\Amqp\Impl\AmqpQueue $fooQueue */

$message = $context->createMessage('Hello world!');

$context->createProducer()
    ->setTimeToLive(60000) // 60 sec
    //
    ->send($fooQueue, $message)
;
```

## Send delayed message

AMQP specification says nothing about message delaying hence the producer throws `DeliveryDelayNotSupportedException`.
Though the producer (and the context) accepts a delivery delay strategy and if it is set it uses it to send delayed message.
The `enqueue/amqp-tools` package provides two RabbitMQ delay strategies, to use them you have to install that package

```php
<?php
use Enqueue\AmqpTools\RabbitMqDlxDelayStrategy;

/** @var \Enqueue\AmqpExt\AmqpContext $context */
/** @var \Interop\Amqp\Impl\AmqpQueue $fooQueue */

// make sure you run "composer require enqueue/amqp-tools".

$message = $context->createMessage('Hello world!');

$context->createProducer()
    ->setDelayStrategy(new RabbitMqDlxDelayStrategy())
    ->setDeliveryDelay(5000) // 5 sec

    ->send($fooQueue, $message)
;
````

## Consume message:

```php
<?php
/** @var \Enqueue\AmqpExt\AmqpContext $context */
/** @var \Interop\Amqp\Impl\AmqpQueue $fooQueue */

$consumer = $context->createConsumer($fooQueue);

$message = $consumer->receive();

// process a message

$consumer->acknowledge($message);
// $consumer->reject($message);
```

## Subscription consumer

```php
<?php
use Interop\Queue\Message;
use Interop\Queue\Consumer;

/** @var \Enqueue\AmqpExt\AmqpContext $context */
/** @var \Interop\Amqp\Impl\AmqpQueue $fooQueue */
/** @var \Interop\Amqp\Impl\AmqpQueue $barQueue */

$fooConsumer = $context->createConsumer($fooQueue);
$barConsumer = $context->createConsumer($barQueue);

$subscriptionConsumer = $context->createSubscriptionConsumer();
$subscriptionConsumer->subscribe($fooConsumer, function(Message $message, Consumer $consumer) {
    // process message

    $consumer->acknowledge($message);

    return true;
});
$subscriptionConsumer->subscribe($barConsumer, function(Message $message, Consumer $consumer) {
    // process message

    $consumer->acknowledge($message);

    return true;
});

$subscriptionConsumer->consume(2000); // 2 sec
```

## Purge queue messages:

```php
<?php
/** @var \Enqueue\AmqpExt\AmqpContext $context */
/** @var \Interop\Amqp\Impl\AmqpQueue $fooQueue */

$queue = $context->createQueue('aQueue');

$context->purgeQueue($queue);
```

[back to index](../index.md)
