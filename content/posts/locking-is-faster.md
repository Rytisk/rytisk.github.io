---
title: "Locking Is Faster"
date: 2024-08-31
tags: ["go", "concurrency"]
---

These days, I've been looking into QUIC. In the Go community, [quic-go](https://github.com/quic-go/quic-go) is the most popular implementation of QUIC. This post is not about QUIC itself. This time I just wanted to analyze a specific piece of code from the `quic-go` library. And that code is inside a method [Write](https://github.com/quic-go/quic-go/blob/42f04d4e02205ee796d1fdec12bd2b46166e7cde/send_stream.go#L87):

```go
func (s *sendStream) Write(p []byte) (int, error) {
    // Concurrent use of Write is not permitted 
    // (and doesn't make any sense), but sometimes people do it anyway.
	// Make sure that we only execute one call at any given time
    // to avoid hard to debug failures.
	s.writeOnce <- struct{}{}
	defer func() { <-s.writeOnce }()

	isNewlyCompleted, n, err := s.write(p)
	if isNewlyCompleted {
		s.sender.onStreamCompleted(s.streamID)
	}
	return n, err
}
```

This method writes the passed-in byte slice `p` into the QUIC Stream. But I am only interested in the comment and lines 6-7.

Usually when working with streams or connections, concurrent writes on the object are not permitted, and the same applies to a QUIC stream. But the developers behind `quic-go` decided to put a safeguard for people making the mistake of calling `Write` concurrently.

This safeguard is a [`writeOnce`](https://github.com/quic-go/quic-go/blob/42f04d4e02205ee796d1fdec12bd2b46166e7cde/send_stream.go#L77) channel, that is created as a buffered channel of size `1`.
```go
writeOnce: make(chan struct{}, 1), // cap: 1, to protect against concurrent use of Write
```

These two lines actually create a simple [semaphore](https://en.wikipedia.org/wiki/Semaphore_(programming))—a construct for limiting concurrent access:
```go
s.writeOnce <- struct{}{}
defer func() { <-s.writeOnce }()
```
Since `writeOnce` has a size of `1`, only a single goroutine can write into the channel at a time. The others will have to wait until the first goroutine finishes the `Write` operation and the defered function is called, which will read from the `writeOnce` buffer. After this, another goroutine can write to the `writeOnce` channel.

Why did this piece of code catch my attention? Because when I saw it, I had a question: if they have already implemented a safeguard against concurrent writes, why should I need synchronization in my application code?

## **So I wrote a benchmark**

First, a benchmark with just the semaphore construct. This will make parallel access to the `writeOnce` semaphore.

```go
func BenchmarkWithoutLock(b *testing.B) {
	writeOnce := make(chan struct{}, 1)
	b.ResetTimer()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			writeOnce <- struct{}{}
			<-writeOnce
		}
	})
}
```

And another benchmmark `WithLock` that wraps the semaphore in a `sync.Mutex`. In this case, we have parallel access to mutex `mu` and serial access to `writeOnce`.

```go
func BenchmarkWithLock(b *testing.B) {
	writeOnce := make(chan struct{}, 1)
	mu := &sync.Mutex{}
	b.ResetTimer()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			mu.Lock()
			writeOnce <- struct{}{}
			<-writeOnce
			mu.Unlock()
		}
	})
}
```

And the results were a bit suprising to my not-so-experienced-in-Go-yet eyes. Why is the lockless option slower?
```txt
cpu: Intel(R) Core(TM) i5-7600K CPU @ 3.80GHz
BenchmarkWithoutLock-4         	 5010553	       234.1 ns/op
BenchmarkWithLock-4            	14815070	        80.40 ns/op
```

I ran the same benchmark without the `RunParallel`, so that the code ran serially, and I got results where the lockless approach was a bit faster:
```txt
cpu: Intel(R) Core(TM) i5-7600K CPU @ 3.80GHz
BenchmarkSeriallyWithoutLock-4   	33334258	        36.93 ns/op
BenchmarkSeriallyWithLock-4      	25462245	        47.32 ns/op
```
This makes sense because when running the code serially, there's no need for synchronization, so the `WithLock` version just adds additional overhead of locking and unlocking the mutex.

However, when we have multiple goroutines and parallel access, the situation changes because synchronization becomes necessary. The benchmarks show that parallel access to `writeOnce` is slower than parallel access to `sync.Mutex` combined with serial access to `writeOnce`:

`parallel(writeOnce) > parallel(sync.Mutex) + serial(writeOnce)`

## **And why is that?**

Channels are more complex constructs than mutexes, and they were not intended for simple locking. Channels have additional complexity due to the need for communication, buffer management and context switching, which adds overhead. The higher the parallelism, the greater the overhead for channels.

So even though there's already a safeguard in `quic-go` against concurrent writes, there's still a benefit in using a `sync.Mutex` in your application code if you are doing concurrent writes to a QUIC stream.