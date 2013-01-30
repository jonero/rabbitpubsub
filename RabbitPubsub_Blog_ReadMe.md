In the previous blog [Scaling Real-time Apps on Cloud Foundry Using Node.js and Redis](http://blog.cloudfoundry.com/2013/01/24/scaling-real-time-apps-on-cloud-foundry-using-node-js-and-redis/), we used Redis as Session Store and also as a pub-sub service for chat messages. But in many enterprise grade real-time apps, you may want to use RabbitMQ instead of Redis to do pub-sub. This is especially true for financial or Bank apps like Stock Quote apps where it is critical to *protect* and deliver each-and-every message AND do it as quickly as possible.

So, in this blog, we will start from [Scaling Real-time Apps on Cloud Foundry Using Node.js and Redis](http://blog.cloudfoundry.com/2013/01/24/scaling-real-time-apps-on-cloud-foundry-using-node-js-and-redis/) and simply replace Redis with RabbitMQ pubsub. 

The app architecture (before):

<p align='center'>
  <img src="https://github.com/rajaraodv/redispubsub/raw/master/pics/redisAsSSAndPS.png" height="300px" width="500px" />
</p>



The app architecture w/ RabbitMQ (after):

<p align='center'>
  <img src="https://raw.github.com/rajaraodv/rabbitpubsub/master/pics/finalArchitecture.png" height="300px" width="500px" />
</p>


***
## Intro to RabbitMQ
Node.js community may not be familiar with RabbitMQ. So here are some of the high-level intro of RabbitMQ.

RabbitMQ is a message broker. It simply accepts message from one or more producers and sends it to one or more consumers. 

RabbitMQ is more sophisticated and flexible than just that. Depending on the configuration, it can also figure out what needs to be done when a consumer crashes(store and re-deliver message), when consumer is slow (queue messages), when there are multiple consumers (distribute work load), or even when RabbitMQ itself crashes (durable). For more please see: [RabbitMQ tutorials](http://www.rabbitmq.com/tutorials/tutorial-one-python.html).

RabbitMQ is also very fast. It implements AMQP protocol [built by and for Wall Street firms like J.P. Morgan Chase, Goldman Sachs etc](http://en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol#History) for trading stocks and related activities. RabbitMQ is an Erlang (also well-known for concurrency & speed) implementation of that protocol.
 

For more please go through [RabbitMQ's website](http://www.rabbitmq.com).

***
## RabbitMQ Basics 
<p align="center">
<img src="https://raw.github.com/rajaraodv/rabbitpubsub/master/pics/rabbitmq.png" />
</p>

RabbitMQ has 4 pieces.

1. Producer ("P") - Sends messages to an exhange along with "Routing key" indicating how to route the message.
2. Exchange ("X") - Recieves message and Routing key from Producers and figures out what to do with the message.
3. Queues("Q") - A temporary place where the messages are stored based on Queue's "binding key" until a consumer is ready to recieve the message. Note: While a Queue physically resides inside RabbitMQ, A consumer is the one that actually creates it by providing a "Binding Key".
4. Consumer("C") - Subscribes to a Queue to recieve messages.

***
## Routing Key, Binding Key and types of Exchanges
To allow various work-flows like pub-sub, work queues, topics, RPC etc., RabbitMQ allows us to independently configure the type of the Exchange, Routing Key and Binding Key.

#### Routing Key:
A string/constraint from Producer instructing Exchange how to route the message. A Routing key looks like: "logs", "errors.logs", "warnings.logs" "tweets" etc. 

#### Binding Key:

Another string/constraint added by a Consumer to a queue to which it is binding/listening to. A Binding key looks like: "logs", "*.logs", "#.logs" etc. 

Note: In RabbitMQ, Binding keys can have "patterns" (but not Routing keys).   

#### Types of Exchange:

Exchanges can be of 4 types:

1. Direct - Sends messages from producer to consumer if Routing Key and Binding key match exactly.
2. Fanout - Sends any message from a producer to ALL consumers (i.e ignores both routing key & binding key)
3. Topic - Sends a message from producer to consumer based on pattern-matching.
4. Headers - If more complicated routing is required beyond simple Routing key string, you can use headers exchange.

As you can see the combination of `the type of Exchange`, `Routing Key` and `Binding Key` makes RabbitMQ behaves completely differently. For example: A `Fanout` Exchange ignores `Routing Key` and `Binding Key` and sends messages to all consumers. A `Topic` Exchange sends a copy of a message to zero, one or more consumers. 

Going into more details is beyond the scope of this blog, but here is another good blog that goes into more details: [AMQP 0-9-1 Model Explained](http://www.rabbitmq.com/tutorials/amqp-concepts.html) 

***
## Creating Exchanges, Producers & Consumers in Node.js
Now that we know some of the basics of RabbitMQ, and all the 4 pieces, let's see how to actually build them in RabbitMQ for our Chat app.

#### Chat App:
<p align='center'>
  <img src="https://github.com/rajaraodv/rabbitpubsub/raw/master/pics/chatAppPage2.png" height="300px" width="500px" />
</p>

#### Connecting to RabbitMQ and creating an Exchange
For our chat application, we will create a `fanout` exchange called `chatExchange`. And will be using [node-amqp module](https://github.com/postwait/node-amqp) to talk to RabbitMQ service.

<pre>
//Connect to RabbitMQ and get reference to the connection.
var rabbitConn = amqp.createConnection({});

//Create an exchange with a name 'chatExchange' and of type 'fanout'
var chatExchange;
rabbitConn.on('ready', function () {
    chatExchange = rabbitConn.exchange('chatExchange', {'type': 'fanout'});
});
</pre>
 
#### Creating Producers (So Users can send chat messages)
In our chat app, users are both producers(i.e. sends chat messages to others) and also consumers (i.e. recieves messages from others). Let's focus on users being 'producers'.

When a user sends a chat message, publish it to chatExchange w/o a Routing Key (Routing Key doesn't matter because chatExchange is a 'fanout').

<pre>
    /**
     * When a user sends a chat message, publish it to chatExchange w/o a Routing Key (Routing Key doesn't matter
     * because chatExchange is a 'fanout').
     *
     * Notice that we are getting user's name from session.
     */
    socket.on('chat', function (data) {
        var msg = JSON.parse(data);
        var reply = {action: 'message', user: session.user, msg: msg.msg };
        chatExchange.publish('', reply);
    });
</pre>

Similarly, when a user joins, publish it to chatExchange w/o Routing key.
<pre>
   /**
     * When a user joins, publish it to chatExchange w/o Routing key (Routing doesn't matter
     * because chatExchange is a 'fanout').
     *
     * Note: that we are getting user's name from session.
     */
    socket.on('join', function () {
        var reply = {action: 'control', user: session.user, msg: ' joined the channel' };
        chatExchange.publish('', reply);
    });
</pre>

#### Creating Consumers (So Users can receive chat messages)
Creating consumers involves 3 steps:

1. Create a queue with some options.
2. Bind queue to exchange using some "Binding Key"
3. Create a subscriber (usually a callback function) to actually obtain messages sent to the queue.

For our chat app,

1. Let's create a queue w/o any name. This forces RabbitMQ to create new queue for every socket.io connection w/ a new random queue name. Let's also set `exclusive` flag to ensure only one connection exists.
<pre>
 rabbitConn.queue('', {exclusive: true}, function (q) {
 ..
 }
</pre>

2. Then bind the queue to chatExchange  w/ "#" or "" 'Binding key' and listen to ALL messages.
<pre>
 q.bind('chatExchange', "");
</pre>
3. Lastly, create a consumer (via q.subscribe) that waits for messages from RabbitMQ. And when a message comes, send it to the browser.
<pre>
 q.subscribe(function (message) {
   //When a message comes, send it back to browser
   socket.emit('chat', JSON.stringify(message));
 });
</pre>

Putting it all togeather.
<pre>
sessionSockets.on('connection', function (err, socket, session) {
    /**
     * When a user sends a chat message, publish it to chatExchange w/o a Routing Key (Routing Key doesn't matter
     * because chatExchange is a 'fanout').
     *
     * Notice that we are getting user's name from session.
     */
    socket.on('chat', function (data) {
        var msg = JSON.parse(data);
        var reply = {action: 'message', user: session.user, msg: msg.msg };
        chatExchange.publish('', reply);
    });

   /**
     * When a user joins, publish it to chatExchange w/o Routing key (Routing doesn't matter
     * because chatExchange is a 'fanout').
     *
     * Note: that we are getting user's name from session.
     */
    socket.on('join', function () {
        var reply = {action: 'control', user: session.user, msg: ' joined the channel' };
        chatExchange.publish('', reply);
    });


   /**
     * Initialize subscriber queue.
     * 1. First create a queue w/o any name. This forces RabbitMQ to create new queue for every socket.io connection w/ a new random queue name.
     * 2. Then bind the queue to chatExchange  w/ "#" or "" 'Binding key' and listen to ALL messages
     * 3. Lastly, create a consumer (via .subscribe) that waits for messages from RabbitMQ. And when
     * a message comes, send it to the browser.
     *
     * Note: we are creating this w/in sessionSockets.on('connection'..) to create NEW queue for every connection
   */
    rabbitConn.queue('', {exclusive: true}, function (q) {
        //Bind to chatExchange w/ "#" or "" binding key to listen to all messages.
        q.bind('chatExchange', "");

   //Subscribe When a message comes, send it back to browser
        q.subscribe(function (message) {
            socket.emit('chat', JSON.stringify(message));
        });
    });
 });
</pre>