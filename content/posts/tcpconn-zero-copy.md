---
title: "net.TCPConn Zero-Copy"
date: 2024-07-30
tags: ["go", "networking"]
---

If you have worked on network programming with Go, you probably used `net.TCPConn` at some point. Maybe you didn't know you were using it—if you just called `net.Listen("tcp", addr)` it returns `net.Conn` interface, and `net.TCPConn` is the type implementing it.

Usually, you read data from the connection, do some processing with it, and then send it further on. But there are cases—for example, working with proxies—when you just need to copy data from one TCP connection to another.

In Go, a simple TCP proxy might look like this:

```go
func main() {
	clientListener, err := net.Listen("tcp", addr)
	if err != nil { /*...*/}
	defer clientListener.Close()
	for {
		clientConn, err := clientListener.Accept()
		if err != nil { /*...*/}

		targetConn, err := net.Dial("tcp", targetAddr)
		if err != nil { /*...*/}

		go proxy(targetConn, clientConn)
	}
}

func proxy(targetConn, clientConn net.Conn) {
	defer clientConn.Close()
	defer targetConn.Close()

	errC := make(chan error, 2)
	go copyConn(clientConn, targetConn, errC)
	go copyConn(targetConn, clientConn, errC)
	<-errC
}

func copyConn(dst, src net.Conn, errC chan<- error) {
	_, err := io.Copy(dst, src)
	errC <- err
}

```

If you have seen this kind of code or written it, you might not have realized that it is using zero-copy[^1].

## **Zero-Copy**

From [Wikipedia](https://en.wikipedia.org/wiki/Zero-copy):
> "Zero-copy" describes computer operations in which the CPU does not perform the task of copying data from one memory area to another or in which unnecessary data copies are avoided. 

Sounds nice. But what does it mean 'from one memory to another'? It's talking about user and kernel space.

When you run your applications, you run them in user space—a separate memory location isolated from OS with limited privileges. When your application needs to access the system's resources, it makes system calls to the kernel, which has direct access to the hardware resources and runs in its own space.

In our case, system calls are executed to make I/O operations on the network interface. This means that when the application reads from a `net.TCPConn` object, the Go runtime, running in user space, makes a syscall to the kernel. The data read from the TCP connection is then copied from kernel space to user space, where the application can access it. The application then writes this data to the destination `net.TCPConn` by copying it from user space into kernel space and finally writing it to the network interface.
![read-write](/images/posts/tcpconn-zero-copy/standard-read-write.png)

So, how does zero-copy help us in this context? The data never leaves kernel space. Since you will be just writing the data to another `net.TCPConn`, you don't need to access it in your application. The kernel will read data from one connection and write it to another. No unnecessary data copies will be executed, just like the 'zero-copy' name implies.
![zero-Copy](/images/posts/tcpconn-zero-copy/zero-copy.png)

## **Using Zero-Copy in Go**

To use zero-copy on a `net.TCPConn`, there is a [ReadFrom](https://go.googlesource.com/go/+/refs/tags/go1.22.5/src/net/tcpsock.go#126) method.

`func (c *TCPConn) ReadFrom(r io.Reader) (int64, error)`

In essence, it forwards the call to an unexported [readFrom](https://go.googlesource.com/go/+/refs/tags/go1.22.5/src/net/tcpsock_posix.go#47) method, which then forwards the call to [spliceFrom](https://go.googlesource.com/go/+/refs/tags/go1.22.5/src/net/splice_linux.go#17) method. 

Even though you can pass any `io.Reader`, it's important that you pass the actual `net.TCPConn` object because `spliceFrom` checks the received `io.Reader` type. If it's not the expected type[^2], it just returns `false`, meaning `spliceFrom` cannot handle it and so `readFrom` falls back to the standard read-write[^3].

If it is a `net.TCPConn`, then a syscall [Splice](https://en.wikipedia.org/wiki/Splice_(system_call)) is used, which is the zero-copy implementation on Linux. What happens when we are running on an OS other than Linux? A stub is used, and `spliceFrom` just [returns](https://go.googlesource.com/go/+/refs/tags/go1.22.5/src/net/splice_stub.go) `false`.

## **io.Copy**

In my proxy example, I said the code was using zero-copy. That is because `io.Copy` internally uses `ReadFrom` if the type implements it[^4]. I'm sure there are plenty of benchmarks proving that zero-copy in Go does improve performance, but I still wanted to see it for myself.

I created two TCP connections: `srcServer ↔ srcClient` and `dstServer ↔ dstClient`. Benchmark tests will call `io.Copy` and measure how long it takes to copy 1MB of data with and without zero-copy.

![benchmark](/images/posts/tcpconn-zero-copy/benchmark.png)


At first, helper function to setup client-server connections.
```go
func initConns(addr string) (*net.TCPConn, *net.TCPConn) {
	listener, _ := net.Listen("tcp", addr)
	defer listener.Close()

	client, _ := net.Dial("tcp", listener.Addr().String())
	server, _ := listener.Accept()
	return client.(*net.TCPConn), server.(*net.TCPConn)
}
```

Then a test setup.

```go
var msg = make([]byte, 1024*1024)

type setup struct {
	src *net.TCPConn
	dst *net.TCPConn
}

func setupServers() *setup {
	srcClient, srcServer := initConns(":8080")
	dstClient, dstServer := initConns(":9090")

	go func() {
		srcClient.Write(msg)
		srcClient.Close()
	}()

	go func() {
		io.Copy(io.Discard, dstServer)
		dstServer.Close()
	}()

	return &setup{src: srcServer, dst: dstClient}
}

func (s *setup) Close() {
	s.src.Close()
	s.dst.Close()
}
```

Then a benchmark test that runs `io.Copy` with zero-copy:
```go
func BenchmarkZeroCopy(b *testing.B) {
	for i := 0; i <= b.N; i++ {
		b.StopTimer()
		s := setupServers()
		b.StartTimer()

		io.Copy(s.dst, s.src)

		b.StopTimer()
		s.Close()
		b.StartTimer()
	}
}
```

And without zero-copy. For this, I placed `net.TCPConn` in a wrapper without `ReadFrom` and `WriteTo` methods. 

```go
func BenchmarkNoZeroCopy(b *testing.B) {
	for i := 0; i <= b.N; i++ {
		b.StopTimer()
		s := setupServers()
		b.StartTimer()

		io.Copy(&noZeroCopyConn{s.dst}, &noZeroCopyConn{s.src})

		b.StopTimer()
		s.Close()
		b.StartTimer()
	}
}

type noZeroCopyConn struct {
	conn *net.TCPConn
}

func (nzc *noZeroCopyConn) Read(b []byte) (int, error) {
	return nzc.conn.Read(b)
}
func (nzc *noZeroCopyConn) Write(b []byte) (int, error) {
	return nzc.conn.Write(b)
}
```

And the results I got were as expected: zero-copy was indeed faster and used 99.93% less memory.
```
goos: linux
goarch: amd64
cpu: Intel(R) Core(TM) i7-5500U CPU @ 2.40GHz
BenchmarkZeroCopy-4       3154     410900 ns/op       23 B/op      1 allocs/op
BenchmarkNoZeroCopy-4     1504     737926 ns/op    32859 B/op      3 allocs/op
```

[^1]: Assuming it's running on Linux. I will explain why later.
[^2]: `spliceFrom` also works with `net.UnixConn`—Unix domain sockets. You can use it to copy data from Unix domain sockets to TCP connections. The reverse, `TCPConn → UnixConn` uses `spliceTo` instead.
[^3]: If you look at the source code, there's also a call to `sendFile` after trying `spliceFrom`. `sendFile` is a different zero-copy syscall that is used for working with `os.File` and is implemented on more than just Linux.
[^4]: `io.Copy` has a little more complexity actually. If the provided `src` implements a `WriteTo` (and `net.TCPConn` does) it is called first and not `ReadFrom`. But in the case of `net.TCPConn`, `WriteTo` cannot handle the call if `dst` is also a `net.TCPConn` and falls back to `ReadFrom`. `WriteTo` is used explicitly for writing to `net.UnixConn` (by using `spliceTo`). [Here's](https://github.com/golang/go/issues/58808) why.