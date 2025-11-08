
# Ultra Fast TCP Server 🏎

This is an implementation of a custom TCP server written in Java, built on top of Netty Event Lopping groups to handle millions of concurrent TCP connections for ultra-low latency systems.




## Architecture 🛠

The basic architecture is very simple and straight forward.

There are mainly two main groups:

- `bossGroup`: Single threaded event loop group to accept incoming TCP connection requests. It jobs is to accpet TCP socket and hand them it to worker group.
- `workerGroup`: Multi-threaded event loops (default threads = CPU cores * 2) that handle read/write events for accepted connections. Handles actual I/O and runs your handlers.
- `childHandler`: Configures the pipeline for each accepted connection (child channels). This is executed when a new SocketChannel is created for a client.

## Binding and Synchronizing

- `bind(8080)`: Starts listening on TCP port 8080. Returns a ChannelFuture (an async result).

- `.sync()`: Blocks the current thread until the bind completes successfully (or fails).

- `f.channel().closeFuture().sync()`: waits until the server channel is closed (e.g., close() was called). This keeps the main thread alive while Netty’s event loops do the work.

## 📕 How to use it?

The class `CustomChannelHandler` is where actual data processing is happening. This is where all the inbound network events are handled. The method `channelRead()` reads the `Object msg` casts it to `ByteBuf` and echo it on terminal and then writes back to `ByteBuf`.

Inside `channelRead()` you can write your data processing and transforming logic to convert Netty's `ByteBuf` to your own Java POJO or any typed class and do your business logic processing, such network or database call or any computation.
```
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf in = (ByteBuf) msg;
        System.out.println("Received: " + in.toString(CharsetUtil.UTF_8)); // logic here
        ctx.write(in); // Echo back
    }
```
## 📌 Important: Netty's `ByteBuf`
Unlike normal Java objects that rely on Garbage Collection (GC), Netty’s ByteBuf objects often use off-heap (native) memory — meaning memory outside the JVM heap.

➡️ The JVM’s GC does not automatically free off-heap memory.
So Netty must manually track when it’s safe to free it.

That’s done via reference counting — just like in systems languages such as C++ or Rust.

### ⚙️ What “Reference Counting” Means

Every ByteBuf has an internal reference count (refCnt), which is just an integer.

- When a ByteBuf is created, refCnt = 1
- Every time someone retains it (keeps a copy, passes it forward), the count increases
- When someone releases it (done using the data), the count decreases
- When the count reaches 0, Netty frees the memory

#### What’s happening step by step:

- A message (TCP packet) arrives.
- Netty reads the bytes into a `ByteBuf` (allocated from memory pool).
- Netty calls your `channelRead()` method and gives you that `ByteBuf`.
- At this point, you own it — you are responsible for freeing it.
- When you’re done, call `release()` to decrement the reference count.
- When `refCnt` hits 0, Netty frees that memory.

If you forget to release it, that’s a memory leak.
If you release it twice, that’s a use-after-free error — reading from it after freeing memory.

### ⚠️ The Safe Shortcut — Let Netty Manage It
If you don’t need to manually keep the buffer, Netty can manage it automatically:
```
ctx.write(in); // Echo back
```
Here’s the trick:

When you pass a ByteBuf to ctx.write() or ctx.fireChannelRead(),
you transfer ownership to Netty.

Netty will release it automatically once it’s written to the socket or forwarded.
