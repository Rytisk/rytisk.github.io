---
title: "Dynamically Created Redis Subscriptions"
date: 2026-01-02
tags: ["go", "redis"]
---

Recently, I had a chance to work with Redis Pub/Sub. Previously, I had only used Redis as a distributed key-value store and had never really tried other features like Pub/Sub because I usually rely on tools like Kafka for that. But this time, for my use case, I didn’t need any durability or delivery guarantees that Kafka provides — it was totally fine if published messages were dropped when there were no active subscribers. Another requirement was the number of different queues (or channels, as they are called in Redis). In my use case, the number of channels would be dynamic, and subscribers for particular channels would also be created dynamically. So for this use case, Redis Pub/Sub seemed like a good fit.

I work with Go, and I think the most popular Redis library is [go-redis](https://github.com/redis/go-redis). The API for using Pub/Sub seemed really simple, and after I found some [documentation](https://redis.uptrace.dev/guide/go-redis-pubsub.html), it just reinforced my belief that this was going to be straightforward.

To publish a message, you simply call `Publish` and pass a channel and a message payload. To subscribe to a channel, you call `Subscribe`, which returns a `PubSub` object. From that, you can either receive messages via a  Go `chan` or call the `ReceiveMessage` method.

Easy.

Well, at least the publishing part was that easy. The subscribe part, however, wasn't so simple — and I'll dig into why.

## Publish

```go
err := rdb.Publish(ctx, "mychannel1", "payload").Err()
if err != nil {
	panic(err)
}
```

So the publishing part is actually as simple as this snippet. After we setup a Redis client, it creates a connection to the server. The client supports connection pooling, so we can configure it to use more connections using `redis.Options{ PoolSize: 1000 }`. And by default, it's actually not just one connection but:
> the pool size is 10 connections per every available CPU as reported by runtime.GOMAXPROCS [^1]

When we call `Publish` on the Redis client, one of those connections from the pool is used. This makes it effective as those connections are reused. Simple.


## Subsrcibe

```go
pubsub := rdb.Subscribe(ctx, "mychannel1")
defer pubsub.Close()

ch := pubsub.Channel()

for msg := range ch {
	fmt.Println(msg.Channel, msg.Payload)
}
```

The subscription snippet is very similar: we call `Subscribe` on the redis client and pass the channel(s) we want to receive messages from. You might assume that it works the same as `Publish` — that the client uses its connection pool and reuses existing connections. However, that's not the case, because for subscriptions Redis needs to have a dedicated connection. As a result, a new connection to the Redis server is created every time `rdb.Subscribe` is called. This is fine if you have static number of channels and you don't need to subscribe to new ones dynamically.

## So when does this matter?

It matters when you have a dynamic number of channels and you subscribe/unsubscribe based on client requests. For example, imagine there's a channel for each user and you subscribe to that channel when the user opens the app. Now imagine there are 1M unique users and you have 1k RPS for subscription requests. If you called `rdb.Subscribe` each time, you would have 1k new connections per second to Redis. This wouldn't be effective, since opening and closing connections has overhead[^2]. Another point is there's a limit to how many client connections are supported — for example, ElastiCache on AWS has a client limit of 65k connections[^3].

## What's the solution?

It's actually possible to have a single connection for subscriptions. IMHO go-redis API is just not really intuitive here and could be documented better. The Redis client has a `Subscribe` [method](https://github.com/redis/go-redis/blob/1bb9e0d130f3c6acb602d6d9f1ca4acebbe96677/osscluster.go#L1949) that takes a `ctx` and `channels`. The solution here is to not pass any channels at all ¯\\\_(ツ)_/¯  
```go
func (c *ClusterClient) Subscribe(ctx context.Context, channels ...string) *PubSub 
```
The `Subscribe` call on the Redis client returns a `PubSub` object which turns out also has a `Subscribe` method of its own. So instead of only using the client's `Subscribe`, first you should call client's `Subscribe` without passing a channel, and then use `Subscribe` on the `PubSub` and provide the channel name. The difference is that the client's `Subscribe` creates a new connection to Redis server, while the `Subscribe` on `PubSub` doesn't.

The solution does sound simple at first, but there's a caveat that makes the client code less elegant. The `PubSub`'s [Subscribe](https://github.com/redis/go-redis/blob/1bb9e0d130f3c6acb602d6d9f1ca4acebbe96677/pubsub.go#L229C1-L229C76) returns only an error:
```go
func (c *PubSub) Subscribe(ctx context.Context, channels ...string) error
```
This means you don't get an object you can call `ReceiveMessage` on or get a Go `chan` for that particular Redis channel you just subscribed to. Nope. To receive messages, you have that same `PubSub` object you created with the initial `rdb.Subscribe` call. And so the `pubsub.Channel()` will return messages from all the Redis channels you subscribed to using that `PubSub` object. Which means that if you need to have different handlers for different Redis channels, you will have to implement the custom handling yourself. So you will have to do something like this:

```go
type redisSub struct {
	pubsub *redis.PubSub

	mu       sync.Mutex
	channels map[string]chan string
}

func NewRedisSub(rdb redis.UniversalClient) *redisSub {
	pubsub := rdb.Subscribe(context.Background())

	rs := &redisSub{
		pubsub:   pubsub,
		channels: make(map[string]chan string),
	}
	return rs
}

func (rs *redisSub) Run(ctx context.Context) {
	pubSubCh := rs.pubsub.Channel()

	for {
		select {
		case msg, ok := <-pubSubCh:
			if !ok {
				return
			}

			rs.mu.Lock()
			ch, found := rs.channels[msg.Channel]
			rs.mu.Unlock()

			if !found {
				continue
			}

			select {
			case ch <- msg.Payload:
			default:
			}

		case <-ctx.Done():
			return
		}
	}
}

func (rs *redisSub) Subscribe(ctx context.Context, channel string) (chan string, error) {
	err := rs.pubsub.Subscribe(ctx, channel)
	if err != nil {
		return nil, err
	}

	ch := make(chan string)

	rs.mu.Lock()
	defer rs.mu.Unlock()
	rs.channels[channel] = ch

	return ch, nil
}

func (rs *redisSub) Unsubscribe(ctx context.Context, channel string) error {
	rs.mu.Lock()
	delete(rs.channels, channel)
	rs.mu.Unlock()

	err := rs.pubsub.Unsubscribe(ctx, channel)
	if err != nil {
		return err
	}

	return nil
}

// setup and run
rs := NewRedisSub(rdb)
rs.Run(context.Background())

// lets say every call runs in it's own goroutine
func HandleReq(ctx context.Context, user string, responseWriter io.Writer) error {
	ch, err := rs.Subscribe(ctx, user)
	if err != nil {
		return err
	}
	defer rs.Unsubscribe(ctx, user)

	for {
		select {
		case msg := <-ch:
			_, err := responseWriter.Write([]byte(msg))
			if err != nil {
				return nil
			}
		case <-ctx.Done():
			return nil
		}
	}

}


```
This is a naive implementation, but you get the gist. Of course there's still things to consider like what if multiple subscribes to the same channel could occur? Then you would need to introduce synchronization for subscribes/unsubscribes and things would get even more complex real fast.

## Sharded Pub/Sub

Since version 7.0 Redis also has a sharded pub/sub[^4] which can be used by the same go-redis client with `SSubscribe` and `SPublish`. The situation regarding the API and number of connections created is still the same. But there's even more pitfalls which I might write about in a separate post.

[^1]: https://redis.uptrace.dev/guide/go-redis-debugging.html#connection-pool-size
[^2]: https://redis.io/learn/operate/redis-at-scale/talking-to-redis/client-performance-improvements 
[^3]: https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/RedisConfiguration.html
[^4]: https://redis.io/docs/latest/develop/pubsub/#sharded-pubsub