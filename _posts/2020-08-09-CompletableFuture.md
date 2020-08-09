---

layout:   post

title:   CompletableFuture入门指南

subtitle:  Future与CompletableFuture | API | 与Reactor对比

date:    2020-08-09

author:   yyconstantine

header-img: img/post-bg-universe.jpg

catalog: true

tags:

  - Spring

---

### CompletableFuture

---

#### Why not `Future`

首先，我们使用异步的场景最终都是通过传递`Runnable`或`Callable`对象。当我们传递`Callable`时有时会期望获取它的返回值，以用于我们接下来操作的执行。在以前，我们都是通过`Future`来管理我们的task，使用`get()`方法等待其返回结果，如下：

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    ExecutorService executorService = Executors.newSingleThreadExecutor();
    Future<String> future = executorService.submit(() -> {
        Thread.sleep(2000);
        return "async method";
    });
    Thread.sleep(1000);
    System.out.println("main thread");
    System.out.println(future.get());
}
```

当使用`get()`获取返回结果时，我们总是在主线程中阻塞地等待其返回，或者我们通过一个忙循环不断地调用`isDone()`，这样的使用并不优雅，甚至使用`get()`一定程度上失去了异步编程的意义。这种情况下，`CompletableFuture`应运而生。

---

#### `CompletableFuture`简介

通过查看其Java Docs，我们得知：

- `CompletableFuture`可以明确得知其任务执行状态，并可以设置完成时回调的**增强版**`Future`
- 除了直接操作状态、结果之外，`CompletableFuture`还实现了`CompletionStage`接口的方法
  - `thenApply`：为线程执行结束提供了设置回调的入口
  - 所有没有显示`Executor`参数的异步方法都是使用`ForkJoinPool.commonPool()`执行的。为了简化监视、调试和跟踪，所有生成的异步任务都是`CompletableFuture.AsynchronousCompletetionTask`（一个空的接口）
    - 一个很容易产生的疑惑是，虽然`thenApply`提供了我们自定义线程池的入口，但是同样也支持不传入线程池参数的方法，这里使用的是`ForkJoinPool`，那会不会由于大量线程产生导致OOM呢？
    - 查看源码得知，其使用的是`ForkJoinPool.commonPool()`，该线程池会基于系统参数设定其大小，而实际运行也可以发现，涌入大量线程时，实际也只有十几个ForkJoinPool线程在运行：
    - ![image.png](https://i.loli.net/2020/08/09/fHxOiWF4lbvqAyr.png)
    - ![image.png](https://i.loli.net/2020/08/09/nv9GDoQgmfJ5ZHl.png)
- 所有的`CompletionStage`的方法都是未标记`@override`的，这样即使子类重写也不会产生影响
- 此外，`CompletableFuture`同样是`Future`的实现，这意味着其也支持`Future`的操作，其中
  - `cancel`与`completeexceptionally`有相同效果
  - `isCompletedExceptionally`可以确定方法是否通过任何形式结束
  - 如果使用`CompletionException`异常完成，则`get()`和`get(long, TimeUnit)`方法会抛出`ExecutionException`

---

#### `CompletableFuture` API

- 创建任务

```java
// 创建带有返回值的任务(Callable)
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)
// 创建无返回值的任务(Runnable)
public static CompletableFuture<Void> runAsync(Runnable runnable)
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor)
```

- 使用示范

```java
CompletableFuture.supplyAsync(() -> "Hello World");
CompletableFuture.runAsync(() -> System.out.println("Hello World"));
```

- 获取结果（终止操作）

```java
// 阻塞等待返回结果
T get();
// 在指定超时时间内阻塞等待返回结果
T get(long timeout, TimeUnit unit);
// 正常情况下与get无差别，取消时会抛出CancellationException，异常时会抛出CompletionException
T join();
// 立马获取返回结果，如果获取不到则返回valueIfAbsent
T getNow(T valueIfAbsent);
```

- 处理结果

```java
// 处理执行结果或异常(result, throwable) -> {}
public CompletableFuture<T> whenComplete(BiConsumer<? super T,? super Throwable> action)
// 上述场景的异步化处理
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action, Executor executor)
// 当原始CompletableFuture抛出异常，返回一个新的CompletableFuture，将该异常作为参数传递
public CompletableFuture<T> exceptionally(Function<Throwable,? extends T> fn)
```

- 使用示范

```java
CompletableFuture.supplyAsync(() -> Thread.currentThread().getName() + "running now")
    .whenComplete((result, throwable) -> System.out.println(result));

CompletableFuture.supplyAsync(() -> Thread.currentThread().getName() + "running now")
    .whenCompleteAsync((result, throwable) -> System.out.println(result));

CompletableFuture.supplyAsync(() -> Thread.currentThread().getName() + "running now")
    .exceptionally(throwable -> System.out.println(throwable.getCause());
```

- 设置回调

```java
// 线程执行完毕后设置回调
public <U> CompletionStage<U> thenApply(Function<? super T,? extends U> fn);

// 线程执行完毕后设置异步回调
public <U> CompletionStage<U> thenApplyAsync (Function<? super T,? extends U> fn);

// 线程执行完毕后设置异步回调，并使用指定线程池执行
public <U> CompletionStage<U> thenApplyAsync (Function<? super T,? extends U> fn, Executor executor);
```

- 使用示范

```java
CompletableFuture.supplyAsync(() -> 128)
                .thenApply(i -> i/2)
                .thenApplyAsync(i -> i/2)
                .whenComplete((result, throwable) -> System.out.println(result));
```

- 纯消费

```java
// 只处理结果，只有输入，没有输出
public CompletionStage<Void> thenAccept(Consumer<? super T> action);

// 异步执行
public CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action);

// 指定线程池异步执行
public CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action, Executor executor);

// 多个CompletableFuture结果聚合消费
public <U> CompletionStage<Void> thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action);

// 多个CompletableFuture结果异步聚合消费
public <U> CompletionStage<Void> thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action);

// 多个CompletableFuture结果异步聚合消费，并使用指定线程池执行
public <U> CompletionStage<Void> thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action, Executor executor);

// 或者使用无返回值的Runnable
public CompletionStage<Void> runAfterBoth(CompletionStage<?> other, Runnable action);
public CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other, Runnable action);
public CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other, Runnable action, Executor executor);

// 或者单纯执行一个Runnable
public CompletionStage<Void> thenRun(Runnable action);
public CompletionStage<Void> thenRunAsync(Runnable action);
public CompletionStage<Void> thenRunAsync(Runnable action, Executor executor);
```

- 使用示范

```java
// 创建一个CompletableFuture
CompletableFuture.supplyAsync(() -> 128)
    // 执行第一个CompletableFuture
                .thenApplyAsync(i -> i/2)
    // 传入第二个CompletableFuture, 并对两者结果进行计算
                .thenAcceptBothAsync(CompletableFuture.supplyAsync(() -> 1000)
                                .thenApplyAsync(i -> i/2), 
                        (i1, i2) -> System.out.println(i1 + i2));
```

- 组合

```java
// 用于传入一个CompletableFuture的执行结果，并作为参数传递给另一个定义的CompletableFuture
// 与thenAcceptBoth相比，其使用BiConsumer接收，无返回值，而thenCompose使用Function接收，有返回值
public <U> CompletionStage<U> thenCompose(Function<? super T, ? extends CompletionStage<U>> fn);

public <U> CompletionStage<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn);

public <U> CompletionStage<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn, Executor executor);
```

- 使用示范

```java
CompletableFuture.supplyAsync(() -> 4299)
                .thenCompose(result -> CompletableFuture.supplyAsync(() -> "Now you can get iPhone11 with price ¥" + result))
                .whenComplete((result, throwable) -> System.out.println(result));
```

- 任一结束则执行

```java
// 上述compose或both都是等所有CompletableFuture执行完才进行消费，而either是任意ComplatebleFuture执行完则开始消费
// 下述API的区别与上述同理，Function存在返回值，Consumer无返回值
public <U> CompletionStage<U> applyToEither(CompletionStage<? extends T> other, Function<? super T, U> fn);
public CompletionStage<Void> acceptEither(CompletionStage<? extends T> other, Consumer<? super T> action);
```

- 使用示范

```java
CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(2000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return "CompletableFuture 1#";
})
    // .acceptEither(CompletableFuture.supplyAsync(() -> "CompletableFuture 2#"), System.out::println);
    .applyToEither(CompletableFuture.supplyAsync(() -> "CompletableFuture 2#"), input -> input)
    .whenComplete((result, throwable) -> System.out.println(result));
```

- 组合多个`CompletableFuture`的辅助方法

```java
// 所有CompletableFuture执行完成开始计算
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs) {}
// 任一CompletableFuture执行完成开始计算
public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs) {}
```

- 使用示范

```java
// 不存在返回值，但是会等待所有CompletableFuture执行结束
CompletableFuture<Void> allFutures = CompletableFuture.allOf(
    CompletableFuture.supplyAsync(() -> 1000),
    CompletableFuture.supplyAsync(() -> 2000),
    CompletableFuture.supplyAsync(() -> 3000)
);
allFutures.get();

// 存在返回值，只要有任一CompletableFuture执行结束即进行计算
CompletableFuture.anyOf(
    CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return 100;
    }),
    CompletableFuture.supplyAsync(() -> 200)
)
    .whenComplete((r, t) -> System.out.println(r));
```

- 一些额外的技巧

```java
// 将结果组合为集合形式
public static <T> CompletableFuture<List<T>> sequence(List<CompletableFuture<T>> futures) {
    CompletableFuture<Void> allDoneFuture = CompletableFuture.allOf(futures.toArray(new CompletableFuture[futures.size()]));
    return allDoneFuture.thenApply(v -> futures.stream().map(CompletableFuture::join).collect(Collectors.<T>toList()));
}
public static <T> CompletableFuture<Stream<T>> sequence(Stream<CompletableFuture<T>> futures) {
    List<CompletableFuture<T>> futureList = futures.filter(f -> f != null).collect(Collectors.toList());
    return sequence(futureList);
}

// Future转CompletableFuture
public static <T> CompletableFuture<T> toCompletable(Future<T> future, Executor executor) {
    return CompletableFuture.supplyAsync(() -> {
        try {
            return future.get();
        } catch (InterruptedException | ExecutionException e) {
            throw new RuntimeException(e);
        }
    }, executor);
}
```

---

#### 闲话

最近其实在看Reactor的一些编程模型，发现这跟`CompletableFuture`存在一些类似之处，都是异步的容器，可以添加对结果的响应，简化我们的编码，但是它们还是存在一些不同，所以我对`CompletableFuture`进行了一些学习，总结如下：

- Reactor是事件的容器，无论0->n的`Flux`或是0->1的`Mono`，传递的都是事件；而`CompletableFuture`传递的是结果
- 正因为Reactor传递的是事件，所以在Reactor创建的时候事件不会执行，直到产生订阅；而`CompletableFuture`在创建时已经提交了异步任务执行，所以容器承载的是异步结果（当然也可能还没执行完）
- 因为Reactor发布-订阅机制，所以允许存在多个订阅方（消费多次）；`CompletableFuture`的结果只能被一次消费（或是传递给新的`CompletableFuture`，如前面提到的`exceptionally`）
