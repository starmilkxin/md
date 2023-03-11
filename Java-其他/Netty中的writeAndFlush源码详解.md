Netty中的writeAndFlush方法还是很搞人的，谈一下其中两个方面
1. ChannelHandlerContext和Channel的writeAndFlush方法的区别
2. writeAndFlush中的write和flush方法

# ChannelHandlerContext和Channel的writeAndFlush方法的区别
首先我们要先知道一下ChannelPipeline中的结构
![ChannelPipeline](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220117080004.png)
它其中是一种双向链表的数据结构，将所有ChannelHandlerContext连接在一起，而每个CHannelHandlerContext中又包含有一个ChannelHandler。<br/>
<br/>
我们知道，Netty中的ChannelHandlerContext和Channel它们都有writeAndFlush方法。我们来跟踪源码看看。<br/>
<br/>
首先是ChannelHandlerContext.writeAndFlush方法。看源码，可以发现ChannelHandlerContext接口的实现类是DefaultChannelHandlerContext，而DefaultChannelHandlerContext又是继承于AbstractChannelHandlerContext的，在AbstractChannelHandlerContext中，我们终于发现了writeAndFlush方法

```Java
@Override
    public ChannelFuture writeAndFlush(Object msg, ChannelPromise promise) {
        if (msg == null) {
            throw new NullPointerException("msg");
        }
        if (isNotValidPromise(promise, true)) {
            ReferenceCountUtil.release(msg);
            // cancelled
            return promise;
        }
        write(msg, true, promise);
        return promise;
    }
```

而Channel就更复杂了，它的实现类是NioSocketChannel，NioSocketChannel又继承了AbstractNioByteChannel，AbstractNioByteChannel又继承了AbstractNioChannel，AbstractNioChannel又继承了AbstractChannel，最后终于在AbstractChannel中找到了writeAndFlush方法

```Java
@Override
    public ChannelFuture writeAndFlush(Object msg) {
        return pipeline.writeAndFlush(msg);
    }
```

我们首先来看AbstractChannelHandlerContext中的writeAndFlush。<br/>
可以看到它主要的就是write(msg, true, promise)。也就是说这一步是write方法，同时还说明了需要flush是true，这也为之后的write和flush分离做铺垫。<br/>
继续看write方法

```Java
private void write(Object msg, boolean flush, ChannelPromise promise) {
        AbstractChannelHandlerContext next = findContextOutbound();
        final Object m = pipeline.touch(msg, next);
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            if (flush) {
                next.invokeWriteAndFlush(m, promise);
            } else {
                next.invokeWrite(m, promise);
            }
        } else {
            AbstractWriteTask task;
            if (flush) {
                task = WriteAndFlushTask.newInstance(next, m, promise);
            }  else {
                task = WriteTask.newInstance(next, m, promise);
            }
            safeExecute(executor, task, promise, m);
        }
    }
```

关键就是这个findContextOutbound()了。看函数名是找到上下文的Outbound。我们继续看

```Java
private AbstractChannelHandlerContext findContextOutbound() {
        AbstractChannelHandlerContext ctx = this;
        do {
            ctx = ctx.prev;
        } while (!ctx.outbound);
        return ctx;
    }
```

可以看到这一步是在找寻当前AbstractChannelHandlerContext的**之前的最近的**一个包含有ChannelOutboundHandler的AbstractChannelHandlerContext。<br/>
之后呢就是如果flush是trueinvokeWrite，那么就采用invokeWriteAndFlush方法，否则就是invokeWrite方法。具体步骤也是一目了然，就是少了个invokeFlush0()。

```Java
	private void invokeWriteAndFlush(Object msg, ChannelPromise promise) {
        if (invokeHandler()) {
            invokeWrite0(msg, promise);
            invokeFlush0();
        } else {
            writeAndFlush(msg, promise);
        }
    }

    private void invokeWrite(Object msg, ChannelPromise promise) {
        if (invokeHandler()) {
            invokeWrite0(msg, promise);
        } else {
            write(msg, promise);
        }
    }
```

继续看invokeWrite0方法。

```Java
    private void invokeWrite0(Object msg, ChannelPromise promise) {
        try {
            ((ChannelOutboundHandler) handler()).write(this, msg, promise);
        } catch (Throwable t) {
            notifyOutboundHandlerException(t, promise);
        }
    }
```

其中的handler方法也有讲究噢，如果是DefaultChannelHandlerContext类的话，那么就是

```Java
    @Override
    public ChannelHandler handler() {
        return handler;
    }
```

这不就是我们熟悉的，自己重写的handler的write方法吗？果不其然，跳到了我们自己写的handler这一步。
![自己写的handler的write方法](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220117081659.png)

这下我们知道了，原来write方法就是在有ChannelOutboundHandler的ChannelHandlerContext中反向传递。

但是我们是知道的，反向传递会一直到传递到头部ChannelHandlerContext也就是HeadContext，那么此时的handler方法则变成了

```Java
        @Override
        public ChannelHandler handler() {
            return this;
        }
```

没错，返回了HeadContext其本身。因为final class HeadContext extends AbstractChannelHandlerContext implements ChannelOutboundHandler, ChannelInboundHandler了，所以是可行的。<br/>
此时的write方法就变成了

```Java
        @Override
        public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
            unsafe.write(msg, promise);
        }
```

最后的最后

```Java
        @Override
        public final void write(Object msg, ChannelPromise promise) {
            assertEventLoop();

            ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            if (outboundBuffer == null) {
                // If the outboundBuffer is null we know the channel was closed and so
                // need to fail the future right away. If it is not null the handling of the rest
                // will be done in flush0()
                // See https://github.com/netty/netty/issues/2362
                safeSetFailure(promise, WRITE_CLOSED_CHANNEL_EXCEPTION);
                // release message now to prevent resource-leak
                ReferenceCountUtil.release(msg);
                return;
            }

            int size;
            try {
                msg = filterOutboundMessage(msg);
                size = pipeline.estimatorHandle().size(msg);
                if (size < 0) {
                    size = 0;
                }
            } catch (Throwable t) {
                safeSetFailure(promise, t);
                ReferenceCountUtil.release(msg);
                return;
            }

            outboundBuffer.addMessage(msg, size, promise);
        }
```

最终把msg加入到outboundBuffer中，完成了它的使命。<br/>
<br/>
还记得我们一开始的初衷吗。。。<br/>
ctx.channel().writeAndFlush(msg)方法与ctx.writeAndFlush(msg)异同就在这！

```Java
    @Override
    public ChannelFuture writeAndFlush(Object msg) {
        return pipeline.writeAndFlush(msg);
    }
	
    @Override
    public final ChannelFuture write(Object msg) {
        return tail.write(msg);
    }
```

原来它是直接从最后一个ChannelHandlerContext开始反向传递。<br/>
<br/>
总结：<br/>
+ ChannelHandlerContext和Channel它们都有writeAndFlush方法，但是前者是从当前的ChannelHandlerContext开始找寻含有ChannelOutboundHandler的ChannelHandlerContext来反向传递。而后者则是从尾部ChannelHandlerContext开始的。
+ 所以说，在ChannelInboundHandler中，我们一般来说都是写ctx.channel().writeAndFlush(msg)。而ChannelOutboundHandler中一般都要避免ctx.channel().writeAndFlush(msg)而采用ctx.writeAndFlush(msg)，为了避免产生死循环，还是挺容易理解的吧~

# writeAndFlush中的write和flush方法
write方法在前面已经是说的很清楚了，接下来我们继续看看flush方法。<br/>
和write方法一样，主要就是从这里开始

```Java
    private void invokeFlush0() {
        try {
            ((ChannelOutboundHandler) handler()).flush(this);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    }
```

同样也是反向传递直到头部

```Java
        @Override
        public void flush(ChannelHandlerContext ctx) throws Exception {
            unsafe.flush();
        }
		
		@Override
        public final void flush() {
            assertEventLoop();

            ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            if (outboundBuffer == null) {
                return;
            }

            outboundBuffer.addFlush();
            flush0();
        }
		
		        protected void flush0() {
            if (inFlush0) {
                // Avoid re-entrance
                return;
            }

            final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            if (outboundBuffer == null || outboundBuffer.isEmpty()) {
                return;
            }

            inFlush0 = true;

            // Mark all pending write requests as failure if the channel is inactive.
            if (!isActive()) {
                try {
                    if (isOpen()) {
                        outboundBuffer.failFlushed(FLUSH0_NOT_YET_CONNECTED_EXCEPTION, true);
                    } else {
                        // Do not trigger channelWritabilityChanged because the channel is closed already.
                        outboundBuffer.failFlushed(FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
                    }
                } finally {
                    inFlush0 = false;
                }
                return;
            }

            try {
                doWrite(outboundBuffer);
            } catch (Throwable t) {
                if (t instanceof IOException && config().isAutoClose()) {
                    /**
                     * Just call {@link #close(ChannelPromise, Throwable, boolean)} here which will take care of
                     * failing all flushed messages and also ensure the actual close of the underlying transport
                     * will happen before the promises are notified.
                     *
                     * This is needed as otherwise {@link #isActive()} , {@link #isOpen()} and {@link #isWritable()}
                     * may still return {@code true} even if the channel should be closed as result of the exception.
                     */
                    close(voidPromise(), t, FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
                } else {
                    try {
                        shutdownOutput(voidPromise(), t);
                    } catch (Throwable t2) {
                        close(voidPromise(), t2, FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
                    }
                }
            } finally {
                inFlush0 = false;
            }
        }
```

里面的doWrite就是最终的传输数据的操作。<br/>
<br/>

总结：
+ flush操作无论在哪个handler中使用都没事，因为它就是反向传递到头部ChannelHandlerContext然后将头部ChannelHandlerContext中的数据传输到客户端。