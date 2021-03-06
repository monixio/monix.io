---
layout: docs3x
title: Asynchronous Stack Traces
description: Asynchronous Stack Traces
---

Monix [Task](task.md) and [Coeval](coeval.md) come with a support for more readable stack traces.

A notable pain point of working with asynchronous code on the JVM is that stack traces no longer provide a valuable context of the execution path that a program takes. 
This limitation is even more pronounced with Scala's Future (pre- 2.13), where an asynchronous boundary is inserted after each operation. 
Task suffers a similar problem, but even a synchronous Task or Coeval program's stack trace is polluted with the run-loop details.

## Configuration

There are three stack tracing modes:
- `NONE`
- `CACHED`
- `FULL`

Each tracing mode comes with a different stack traces detail at the cost of the performance penalty.
We've found `CACHED` mode to be the best middle-ground, and it is set by default.

To prevent unbounded memory usage, stack traces for fiber are accumulated in an internal buffer as execution proceeds. 
If more traces are collected than the buffer can retain, then the older traces will be overwritten. 
The default size for the buffer is 16 but can be changed via the system property `monix.eval.traceBufferLogSize`.
Note that this property is expressed as a logarithm of a power of two!

For example, to enable full-stack tracing and a trace buffer size of 32, specify the following system properties:

`-Dmonix.eval.stackTracingMode=full -Dmonix.eval.traceBufferLogSize=5`

Note that the `stackTracingMode` option is case insensitive.

Another option is `monix.eval.enhancedExceptions` that is `true` by default and says whether exceptions caught by `Task` / `Coeval` should include enriched stack trace information.

### NONE

No tracing is instrumented by the program and so incurs a negligible impact to performance. If a trace is requested, it will be empty.

### CACHED

When cached stack tracing is enabled, a stack trace is captured and cached for selected operations, such as every `map`, `flatMap`, `create`, and `parMap2` call.

The stack trace cache is indexed by the lambda class reference, so cached tracing may produce inaccurate fiber traces under several scenarios:
- Monad transformer composition
- A named function is supplied to map, async, or flatMap at multiple call-sites

We measured about 10-30% performance hit when cached tracing is enabled in micro-benchmarks, but it will most likely be much less for any real application.

We strongly recommend benchmarking applications that use tracing and would appreciate any (good or bad!) reports.

This is the recommended mode to run in most production applications and is enabled by default.

### FULL

When full stack tracing is enabled, a stack trace is captured for most combinators, including `now`, `eval`, `suspend`, `raiseError` as well as those traced in cached mode.

Stack traces are collected on every invocation, so naturally, most programs will experience a significant performance hit. 
This mode is mainly useful for debugging in development environments.

## Showing Stack Traces

If `monix.eval.enhancedExceptions` is set to `true`, all exceptions caught by `Task` and `Coeval` will include extra information in their stack trace.

Another option is to call `Task.trace` to expose stack trace information at will.

## Example

```scala 
package test.app

import monix.eval.Task
import monix.execution.Scheduler
import cats.implicits._
import scala.concurrent.duration._

object TestTracingApp extends App {
  implicit val s = Scheduler.global

  def customMethod: Task[Unit] =
    Task.now(()).guarantee(Task.sleep(10.millis))

  val tracingTestApp: Task[Unit] = for {
    _ <- Task.shift
    _ <- Task.unit.attempt
    _ <- (Task(println("Started the program")), Task.unit).parTupled
    _ <- customMethod
    _ <- if (true) Task.raiseError(new Exception("boom")) else Task.unit
  } yield ()

  tracingTestApp.onErrorHandleWith(ex => Task(ex.printStackTrace())).runSyncUnsafe
}
```

### NONE

```
java.lang.Exception: boom
        at test.app.TestTracingApp$.$anonfun$tracingTestApp$5(TestTracingApp.scala:36)
        at monix.eval.internal.TaskRunLoop$.startFull(TaskRunLoop.scala:188)
        at monix.eval.internal.TaskRestartCallback.syncOnSuccess(TaskRestartCallback.scala:101)
        at monix.eval.internal.TaskRestartCallback$$anon$1.run(TaskRestartCallback.scala:118)
        at monix.execution.internal.Trampoline.monix$execution$internal$Trampoline$$immediateLoop(Trampoline.scala:66)
        at monix.execution.internal.Trampoline.startLoop(Trampoline.scala:32)
        at monix.execution.schedulers.TrampolineExecutionContext$JVMNormalTrampoline.super$startLoop(TrampolineExecutionContext.scala:142)
        at monix.execution.schedulers.TrampolineExecutionContext$JVMNormalTrampoline.$anonfun$startLoop$1(TrampolineExecutionContext.scala:142)
        at scala.runtime.java8.JFunction0$mcV$sp.apply(JFunction0$mcV$sp.scala:18)
        at scala.concurrent.BlockContext$.withBlockContext(BlockContext.scala:94)
        at monix.execution.schedulers.TrampolineExecutionContext$JVMNormalTrampoline.startLoop(TrampolineExecutionContext.scala:142)
        at monix.execution.internal.Trampoline.execute(Trampoline.scala:40)
        at monix.execution.schedulers.TrampolineExecutionContext.execute(TrampolineExecutionContext.scala:57)
        at monix.execution.schedulers.BatchingScheduler.execute(BatchingScheduler.scala:50)
        at monix.execution.schedulers.BatchingScheduler.execute$(BatchingScheduler.scala:47)
        at monix.execution.schedulers.AsyncScheduler.execute(AsyncScheduler.scala:31)
        at monix.eval.internal.TaskRestartCallback.onSuccess(TaskRestartCallback.scala:72)
        at monix.eval.internal.TaskRunLoop$.startFull(TaskRunLoop.scala:183)
        at monix.eval.internal.TaskRestartCallback.syncOnSuccess(TaskRestartCallback.scala:101)
        at monix.eval.internal.TaskRestartCallback.onSuccess(TaskRestartCallback.scala:74)
        at monix.eval.internal.TaskSleep$SleepRunnable.run(TaskSleep.scala:71)
        at java.util.concurrent.ForkJoinTask$RunnableExecuteAction.exec(ForkJoinTask.java:1402)
        at java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:289)
        at java.util.concurrent.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1056)
        at java.util.concurrent.ForkJoinPool.runWorker(ForkJoinPool.java:1692)
        at java.util.concurrent.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:157)
```

### CACHED

``` 
java.lang.Exception: boom
        at test.app.TestTracingApp$.$anonfun$tracingTestApp$5(TestTracingApp.scala:36)
        at guarantee @ test.app.TestTracingApp$.customMethod(TestTracingApp.scala:29)
        at flatMap @ test.app.TestTracingApp$.$anonfun$tracingTestApp$4(TestTracingApp.scala:35)
        at parTupled @ test.app.TestTracingApp$.$anonfun$tracingTestApp$2(TestTracingApp.scala:34)
        at parTupled @ test.app.TestTracingApp$.$anonfun$tracingTestApp$2(TestTracingApp.scala:34)
        at flatMap @ test.app.TestTracingApp$.$anonfun$tracingTestApp$2(TestTracingApp.scala:34)
        at flatMap @ test.app.TestTracingApp$.$anonfun$tracingTestApp$1(TestTracingApp.scala:33)
        at flatMap @ test.app.TestTracingApp$.delayedEndpoint$test$app$TestTracingApp$1(TestTracingApp.scala:32)
```

### FULL

``` 
java.lang.Exception: boom
        at test.app.TestTracingApp$.$anonfun$tracingTestApp$5(TestTracingApp.scala:36)
        at raiseError @ test.app.TestTracingApp$.$anonfun$tracingTestApp$5(TestTracingApp.scala:36)
        at now @ test.app.TestTracingApp$.customMethod(TestTracingApp.scala:29)
        at guarantee @ test.app.TestTracingApp$.customMethod(TestTracingApp.scala:29)
        at flatMap @ test.app.TestTracingApp$.$anonfun$tracingTestApp$4(TestTracingApp.scala:35)
        at <clinit> @ test.app.TestTracingApp$.delayedEndpoint$test$app$TestTracingApp$1(TestTracingApp.scala:32)
        at apply @ test.app.TestTracingApp$.$anonfun$tracingTestApp$2(TestTracingApp.scala:34)
        at parTupled @ test.app.TestTracingApp$.$anonfun$tracingTestApp$2(TestTracingApp.scala:34)
        at parTupled @ test.app.TestTracingApp$.$anonfun$tracingTestApp$2(TestTracingApp.scala:34)
        at flatMap @ test.app.TestTracingApp$.$anonfun$tracingTestApp$2(TestTracingApp.scala:34)
        at <clinit> @ test.app.TestTracingApp$.delayedEndpoint$test$app$TestTracingApp$1(TestTracingApp.scala:32)
        at flatMap @ test.app.TestTracingApp$.$anonfun$tracingTestApp$1(TestTracingApp.scala:33)
        at flatMap @ test.app.TestTracingApp$.delayedEndpoint$test$app$TestTracingApp$1(TestTracingApp.scala:32)
```
