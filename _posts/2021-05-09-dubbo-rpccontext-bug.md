---
layout: article
title: Dubbo RpcContext 在异步并发场景下的一个隐藏 Bug
key: article-dubbo-rpccontext-bug
---
### 问题出现

我们团队基于 Apache Dubbo 开发了公司自用的 RPC 框架，整合了一些治理功能，比如限流、熔断等等。之前这个框架都是基于 Dubbo 2.7.3 开发的，今年才升级到 Dubbo 2.7.7。上周，有业务团队反馈，说升级到使用 Dubbo 2.7.7 的新版本框架之后，一个异步的服务调用出现了持续的信号量耗尽报错，且无法自行恢复。低版本框架则没有这个问题。

> Netflix 开源的 Hystrix 组件提供了信号量隔离的功能。每个请求在发出前都需要获取信号量，执行完成后释放信号量。这样就可以控制请求的并发数。

### 问题排查

我们在 Dubbo 中整合 Hystrix 是通过 Filter 进行的。框架在 Filter 中创建 HystrixCommand，然后把它放到 RpcContext 中并启动执行。并且在一个 Listener 中，当收到请求的结果时，将这个 Command 标记为完成，释放信号量。

大致代码是这样的：

```java
@Activate(group = {CommonConstants.CONSUMER}, order = 40010)
public class DubboClientHystrixFilter implements Filter {
  @Override
  public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    final DubboClientHystrixResult hystrixResult = new DubboClientHystrixResult();
    final DubboClientHystrixObservableCommand command =
      new DubboClientHystrixObservableCommand(setter, invoker, invocation, hystrixResult);
    RpcContext.getContext().set("command", command);
    subscribe(command, hystrixResult, invocation);
  }
}
```

```java
@Activate(group = CommonConstants.CONSUMER, order = -9999)
public class ConsumerFilter extends ListenableFilter {
  @Override
  public Listener listener(Invocation invocation) {
    Listener listener = super.listener(invocation);
    return listener != null ? listener : return listener();
  }

  @Override
  public Listener listener() {
    return new ConsumerListener();
  }
}
```

```java
public class ConsumerListener implements Listener {

  @Override
  public void onResponse(Result rpcResult, Invoker<?> invoker, Invocation invocation) {
    final DubboClientHystrixObservableCommand command
        = (DubboClientHystrixObservableCommand) RpcContext.getContext().get("command");
    command.markSuccess();
  }

  @Override
  public void onError(Throwable exception, Invoker<?> invoker, Invocation invocation) {
    final DubboClientHystrixObservableCommand command
        = (DubboClientHystrixObservableCommand) RpcContext.getContext().get("command");
    command.markError();
  }
}
```

由于用户应用表现出的行为是长时间的信号量耗尽，但请求耗时监控并无异常，所以我们怀疑是部分信号量没有正常释放。而且奇怪是，经过调试，发现每个请求都执行了 ConsumerListener 中相应的完成 Command 执行的代码。按理说不应该有问题啊。

我们在 Hystrix 中实际释放信号量的地方下了断点进行调试，发现有的请求在准备释放信号量时，对应的 Command 的信号量已经释放过了。那么问题来了，为什么会对同一个 Command 操作两次信号量释放呢？而 Command 是存放在 RpcContext 中的。这就意味着不同请求的 RpcContext 中保存了相同的 Command。难道 RpcContext 会在请求间共享？哈，这个猜测有点大胆啊。

又经过了长时间的翻代码加调试，不幸的是，上面的猜测被证实了。在一个 Dubbo 服务发起 Dubbo 请求的时候，确实会出现 RpcContext 在不同请求间共享的情况。

这是为什么呢？马上揭晓。

### 问题根因揭秘

大家都知道 RpcContext 中保存的是一个请求的上下文信息。在 Dubbo 服务方法中，它里面存放的是客户端请求中传递过来的上下文。在发起一个 Dubbo 调用时，我们也可以在里面增加需要发送给服务方的一些数据。按照我的理解，RpcContext 的生命周期应该是请求级别的。一个请求处理完成后就要销毁重新构造。

但问题来了，如果我在一个 Dubbo 服务方法中发起一个 Dubbo 调用，调用我的客户端传过来的数据存在 RpcContext 里，在调用完成之后，这个 Context 是不是就要被清理掉呢？事实是，在低版本的 2.7.3 中，确实是这样的，但在 2.7.7 中却不是。

从 Dubbo 2.7.6 起，RpcContext 中增加了一个名为 remove 的标记位。默认值为 true。在清理当前线程的 RpcContext 时候，代码会检查这个标记位的值。如果是 false，则不会执行删除操作。

```java
public class RpcContext {
    private boolean remove = true;
    
    public boolean canRemove() {
        return remove;
    }

    public void clearAfterEachInvoke(boolean remove) {
        this.remove = remove;
    }
    
    public static void removeContext() {
        removeContext(false);
    }
    
    public static void removeContext(boolean checkCanRemove) {
        if (LOCAL.get().canRemove()) {
            LOCAL.remove();
        }
    }
}
```

对这个标记位的赋值发生在 org.apache.dubbo.rpc.filter.ContextFilter 中。

```java
@Activate(group = PROVIDER, order = -10000)
public class ContextFilter implements Filter, Filter.Listener {
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        RpcContext context = RpcContext.getContext();
        try {
            context.clearAfterEachInvoke(false);
            return invoker.invoke(invocation);
        } finally {
            context.clearAfterEachInvoke(true);
            RpcContext.removeContext(true);
            RpcContext.removeServerContext();
        }
    }
}
```

从上面的代码可以看出，在执行服务端业务逻辑之前，当前 RpcContext 的 remove 标记被置为 false，业务逻辑执行完毕之后再恢复为 true，然后执行清理工作。也就是说在服务端业务逻辑执行过程中，当前线程的 RpcContext 是不会被清理掉的。

那么异步调用中处理响应结果的 Listener 中拿到 RpcContext 是从哪来的呢？因为这个 Listener 执行的线程一般不会是执行服务端的业务逻辑的线程。

我们就要看另外一个类，org.apache.dubbo.rpc.AsyncRpcResult。这个类是在发起异步 Dubbo 调用的线程中创建出来的。它会把当时的 RpcContext 对象给缓存下来。等到异步调用完成之后，再把之前缓存的 RpcContext 恢复到当前线程中，执行后续的逻辑。

```java
public class AsyncRpcResult implements Result {
    private RpcContext storedContext;
    private RpcContext storedServerContext;
    
    public AsyncRpcResult(CompletableFuture<AppResponse> future, Invocation invocation) {
        this.responseFuture = future;
        this.invocation = invocation;
        this.storedContext = RpcContext.getContext();
        this.storedServerContext = RpcContext.getServerContext();
    }
    
    private RpcContext tmpContext;
    private RpcContext tmpServerContext;
    
    private BiConsumer<Result, Throwable> beforeContext = (appResponse, t) -> {
        tmpContext = RpcContext.getContext();
        tmpServerContext = RpcContext.getServerContext();
        RpcContext.restoreContext(storedContext);
        RpcContext.restoreServerContext(storedServerContext);
    };

    private BiConsumer<Result, Throwable> afterContext = (appResponse, t) -> {
        RpcContext.restoreContext(tmpContext);
        RpcContext.restoreServerContext(tmpServerContext);
    };

    public Result whenCompleteWithContext(BiConsumer<Result, Throwable> fn) {
        this.responseFuture = this.responseFuture.whenComplete((v, t) -> {
            beforeContext.accept(v, t);
            fn.accept(v, t);
            afterContext.accept(v, t);
        });
        return this;
    }
}
```

基于以上的发现，我们就可以得出一下结论。如果在一个 Dubbo 服务的业务逻辑中连续发起多次异步的 Dubbo 服务调用，它们共用的是同一个 RpcContext 对象，也就是服务当前的 RpcContext 对象。如果有部分逻辑依赖了 RpcContext 中保存的与请求直接关联的数据，那么这段逻辑很有可能因读不到正确的数据而无法得到正确的执行结果。

### 从问题引发的思考

针对 RpcContext 的使用，在我们团队中之前就有过讨论。理论上在 Dubbo 中存在着 4 种上下文：服务端接收到客户端发过来的请求上下文，服务端要返回给客户端的响应上下文，客户端准备发给服务端的请求上下文，客户端接受到服务端返回的响应上下文。但现在 Dubbo 中只存在两个上下文，一个是 RpcContext.getContext()，一个是 RpcContext.getServerContext()。这势必就存在一个上下文应对多个使用场景的情况。

2.7.6 中引入的这个改动我猜测可能是为了解决这样一种问题：服务端的业务逻辑依赖客户端通过 RpcContext 传过来的某些数据。但在读取这些数据之前，服务端发起了一个 Dubbo 调用，导致这个 RpcContext 被清理掉了。虽然这个改动解决了这个问题，但却引入了一个新的问题，RpcContext 的存活时间被无意间拉长了，不再是请求级别的对象了。

但回过头来看如何解决这个问题，我也没有什么好办法。在现有的 RpcContext 设计上，似乎不存在完美的解决方案，只能在两害之间择其一。要想彻底解决可能只能重新设计 RpcContext，真正做到各个使用场景独立处理，杜绝数据相互干扰。但考虑到这一改动十分巨大，而且基于前面的版本中 Dubbo 的行为一直是发起调用后即清理当前 RpcContext 的既有事实，恢复到之前的行为似乎是对用户相对较为透明的解决方案。请问阅读本文的各位有什么好办法吗？