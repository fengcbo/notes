# Executor

## javadoc
Executor是一个执行提交的Runnable任务的对象。这个接口提供了任务提交从每个任务的执行机制包括线程的使用细节、调度等中解耦的一种方式。通常，执行器被用来替代显示的创建线程。比如，你可能使用一下方式，而不是为每个任务执行new Thread(new(RunnableTask())).start()：

```
Executor executor = anExecutor;
executor.execute(new RunnableTask1());
executor.execute(new RunnableTask2());
```

但是，Executor并不严格要求任务的执行一定是异步的。在简单情境下，执行器可以在调用者线程中立即直接执行提交的任务。

```
class DirectExecutor implements Executor {
    public void execute(Runnable r) {
        r.run();
    }
}
```

更常见的情况是，任务在其他线程执行，而不是在调用者的线程中。下面的执行器为每个任务创建了一个新的线程。

```
class ThreadPerTaskExecutor implements Executor {
    public void execute(Runnable r) {
        new Thread(r).start();
    }
}
```

许多Executor实现对如何以及何时调度任务强加了一些限制。下面的执行器阐述了一个组合处理器，它直接将提交的任务交给了第二个处理器。

```
class SerialExecutor implements Executor {
    final Queue<Runnable> tasks = new ArrayDeque<Runnable>();
    final Executor executor;
    Runnable active;

    SerialExecutor(Executor executor) {
        this.executor = executor;
    }

    public synchronized void execute(final Runnable r) {
        tasks.offer(new Runnable() {
        public void run() {
            try {
                r.run();
            } finally {
                scheduleNext();
                }
            }
        });
        if (active == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
        if ((active = tasks.poll()) != null) {
            executor.execute(active);
        }
    }
}
```

当前包中提供的Executor实现类都实现了 ExecutorService，ExecutorService是一个应用更广泛的接口。ThreadPoolExecutor提供了一个可扩展的线程池的实现。Executors 为这些执行器提供了方便的工厂方法。

内存一致性的影响：向Executor提交Runnable对象之前的操作  happen-before 可能在其他线程的任务执行。

## 唯一的方法 void execute(Runnable command)

在未来的某个时间执行提交的命令。任务的执行可能在一个新的线程、可能在线程池、或者在调用者的线程，这由Executor的具体实现来决定。
