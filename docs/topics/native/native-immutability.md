[//]: # (title: Kotlin/Native 中的不可变性与并发)

> This page describes the features of the legacy memory manager. Check out [Kotlin/Native memory management](native-memory-manager.md)
> to learn about the new memory manager, which has been enabled by default since Kotlin 1.7.20.
>
> For more information on the concurrent programming in Kotlin, see the [Coroutines guide](coroutines-guide.md).
>
{type="warning"}

Kotlin/Native 实现了严格的可变性检测，确保了<!--
-->重要的不变式：对象要么不可变，要么<!--
-->在同一时刻只在单个线程中访问（`mutable XOR global`）。

在 Kotlin/Native 中不可变性是运行时属性，可以<!--
-->将 `kotlin.native.concurrent.freeze` 函数应用到任意对象子图。
它使从给定的对象递归可达的所有对象都不可变。 这样的转换是单向操作。例如即这些对象之后不能解冻。
一些天然不可变的对象（如 `kotlin.String`、 `kotlin.Int` 与<!--
-->其他原生类型，以及 `AtomicInt` 与 `AtomicReference`）
默认就是冻结的。如果对已冻结对象应用了修改操作，
那么会抛出 `InvalidMutabilityException` 异常。

为了实现 `mutable XOR global` 不变式，所有全局可见状态（目前有
`object` 单例与枚举）都会自动冻结。如果对象无需冻结<!--
-->，可以使用 `kotlin.native.ThreadLocal` 注解，这会使<!--
-->该对象状态成为线程局部的，因此可修改（但是变更后的状态对<!--
-->其他线程不可见）。

非原生类型的顶层/全局变量默认只能在<!--
-->主线程（即首先初始化 _Kotlin/Native_ 运行时的线程）中访问。
在其他线程中访问会引发 `IncorrectDereferenceException` 异常。
如需其他线程可访问这种变量，可以使用 `@ThreadLocal`
注解将该值标记为线程局部，或者使用 `@SharedImmutable` 注解，它会冻结该值并使<!--
-->其他线程可访问。 参见[全局变量与单例](#全局变量与单例)。

`AtomicReference` 类可用于将变更后的冻结状态发布给<!--
 -->其他线程，从而构建共享缓存等模式。 See [Atomic primitives and references](#atomic-primitives-and-references).

## Kotlin/Native 中的并发

Kotlin/Native 运行时并不鼓励<!--
 -->带有互斥代码块与条件变量的经典线程式并发模型，因为已知该模型<!--
 -->易出错且不可靠。相反，我们建议使用一系列<!--
 -->替代方法，让你可以使用硬件并发并实现阻塞 IO。
这些方法如下，并且分别会在后续各部分详细阐述：

### Worker

Kotlin/Native 运行时提供了 worker 的概念来取代线程：并发执行的<!--
 -->控制流以及与其关联的请求队列。Worker 非常像参与者模型<!--
 -->中的参与者。一个 worker 可以与另一个 worker 交换 Kotlin 对象，从而在任何时刻<!--
 -->每个可变对象都隶属于单个 worker，不过所有权可以转移。
请参见[对象转移与冻结](#对象转移与冻结)部分。

一旦以 `Worker.start` 函数调用启动了一个 worker，就可以使用其自身唯一的整数
worker id 来寻址。其他 worker 或者非 worker 的并发原语（如 OS 线程）可以<!--
 -->使用 `execute` 调用向 worker 发消息。

```kotlin
val future = execute(TransferMode.SAFE, { SomeDataForWorker() }) {
    // 第二个函数参数所返回的数据
    // 作为“input”参数进入 worker 例程
    input ->
    // 这里我们创建了一个当有人消费结果 future 时返回的实例
    WorkerResult(input.stringParam + " result")
}

future.consume {
    // 这里我们查看从上文例程中返回的结果。请注意 future 对象或
    // id 都可以转移给另一个 worker，所以并不是必须要在
    // 获得 future 的同一上下文中消费之。
    result -> println("result is $result")
}
```

调用 `execute` 会使用作为第二个参数传入的函数来生成一个对象子图
（例如一组相互引用的对象）然后将其作为一个整体传给该 worker。之后<!--
 -->发出该请求的线程不可以再使用该对象子图。如果第一个参数<!--
 -->是 `TransferMode.SAFE`，那么会通过图遍历来检测这一属性；而如果第一个参数是 `TransferMode.UNSAFE` 那么直接假定为 true。
`execute` 的最后一个参数是一个特殊 Kotlin lambda 表达式，不可以捕获任何状态，
并且实际上是在目标 worker 的上下文中调用。一旦处理完毕，就将结果转移给将会消费它地方<!--
 -->，并将其附加到该 worker/线程的对象图中。

如果一个对象以 `UNSAFE` 模式转移，并且依然在多个并发执行子中访问，
那么该程序可能会意外崩溃，因此考虑将 `UNSAFE` 作为最后的优化手段而不是通用<!--
 -->机制来使用。

### 对象转移与冻结

Kotlin/Native 运行时维护的一个重要的不变式是，对象要么归单个<!--
-->线程/worker 所有，要么不可变（*共享 XOR 可变*）。这确保了同一数据只有一个修改方，
因此不需要锁定。为了实现这个不变式，我们使用了非外部<!--
-->引用的对象子图的概念。
这是一个没有来自子图以外的外部引用的子图，（在 ARC 系统中）可由
O(N) 复杂度进行算法检测，其中 N 是这种子图中元素的数量。
这种子图通常是作为 lambda 表达式的结果而产生的（例如某些构建器），并且可能不<!--
-->含外部引用的对象。

冻结是一种运行时操作，通过修改对象头使给定的对象子图不可变，
这样之后的修改企图都会抛出 `InvalidMutabilityException`。它是深度冻结，因此<!--
-->如果一个对象有指向其他对象的指针，这些对象的传递闭包也都会被冻结。
冻结是单向转换；冻结的对象不能解冻。冻结的对象有一个很好的属性，
由于其不可变性，它们可以在多个 worker/线程之间自由共享，
而不会破坏“可变 XOR 共享”不变式。

一个对象是否已冻结，可以使用扩展属性 `isFrozen` 来检测，如果冻结了就可以<!--
-->共享。目前，Kotlin/Native 运行时只能在枚举对象创建后进行冻结，尽管<!--
-->将来可能实现自动冻结某些可证明不可变的对象。

### 对象子图分离

An object subgraph without external references can be disconnected using `DetachedObjectGraph<T>` to
没有外部引用的对象子图可以使用 `DetachedObjectGraph<T>`
断开到 `COpaquePointer` 值的连接，该值可以存储在 `void*` 数据中，因此断开连接的对象子图<!--
-->可以存储在 C 语言数据结构中，并且之后还能在任意线程或 worker 中通过 `DetachedObjectGraph<T>.attach()`
加回。如果 worker 机制不足以完成特定任务，那么可以将对象子图分离与[原始共享内存](#原始共享内存)相结合，能够在<!--
-->并发线程之间进行旁路对象传输。 Note that object detachment
may require an explicit leaving function holding object references and then performing cyclic garbage collection.
For example, code like:

```kotlin
val graph = DetachedObjectGraph {
    val map = mutableMapOf<String, String>()
    for (entry in map.entries) {
        // ……
    }
    map
}
```

will not work as expected and will throw runtime exception, as there are uncollected cycles in the detached graph, while:

```kotlin
val graph = DetachedObjectGraph {
    {
     val map = mutableMapOf<String, String>()
     for (entry in map.entries) {
         // ...
     }
     map
    }().also {
      kotlin.native.internal.GC.collect()
    }
 }
```

will work properly, as holding references will be released, and then cyclic garbage affecting the reference counter is
collected.

### 原始共享内存

考虑到 Kotlin/Native 与 C 语言之间通过互操作性的紧密联系，结合上文中提到的其他机制，
可以构建流行的数据结构，如并发的 hashmap 或者与
Kotlin/Native 共享缓存。可以依赖共享的 C 语言数据，并在其中存储分离的对象子图的引用。
考虑以下 .def 文件：

```c
package = global

---
typedef struct {
  int version;
  void* kotlinObject;
} SharedData;

SharedData sharedData;
```

在运行 cinterop 工具之后，可以在版本化的全局结构中共享 Kotlin 数据，
并通过自动生成的 Kotlin 代码在 Kotlin 中与其透明交互，如下所示：

```kotlin
class SharedData(rawPtr: NativePtr) : CStructVar(rawPtr) {
    var version: Int
    var kotlinObject: COpaquePointer?
}
```

因此，结合上文声明的顶层变量，可以让不同的线程看到相同的内存，
并使用平台相关的同步原语来构建传统的并发结构。

### 全局变量与单例

全局变量常常是非预期并发问题的根源，因此 _Kotlin/Native_ 实现了<!--
-->以下机制来防止意外通过全局对象共享状态：

* 全局变量（除非特别标记过）都只能在主线程（即首次初始化
  _Kotlin/Native_ 运行时的线程）中访问，如果其他线程访问这样的全局变量就会抛出 `IncorrectDereferenceException`
* 对于标有 `@kotlin.native.ThreadLocal` 注解的全局变量，每个线程都保留线程局部副本，
  因此变更在线程之间并不可见
* 对于标有 `@kotlin.native.SharedImmutable` 注解的变量，其值是共享的，但是<!--
   -->在发布之前会被冻结，因此每个线程都会看到相同的值
* 单例对象（除非标有 `@kotlin.native.ThreadLocal`）都是冻结且共享的，允许惰性值
  （除非企图创建循环冻结结构）
* 枚举总是冻结的

这些机制结合起来允许在多平台项目中跨平台复用代码的自然竞态冻结编程。

### Atomic primitives and references

Kotlin/Native standard library provides primitives for safe working with concurrently mutable data, namely
`AtomicInt`, `AtomicLong`, `AtomicNativePtr`, `AtomicReference` and `FreezableAtomicReference` in the package
`kotlin.native.concurrent`.
Atomic primitives allow concurrency-safe update operations, such as increment, decrement, and compare-and-swap,
along with value setters and getters. Atomic primitives are always considered frozen by the runtime, and
while their fields can be updated with the regular `field.value += 1`, it is not concurrency safe.
The value must be changed using dedicated operations so it is possible to perform concurrent-safe
global counters and similar data structures.

Some algorithms require shared mutable references across multiple workers. For example, the global mutable
configuration could be implemented as an immutable instance of properties list atomically replaced with the
new version on configuration update as the whole in a single transaction. This way, no inconsistent configuration
could be seen, and at the same time, the configuration could be updated as needed.
To achieve such functionality, Kotlin/Native runtime provides two related classes:
`kotlin.native.concurrent.AtomicReference` and `kotlin.native.concurrent.FreezableAtomicReference`.
Atomic reference holds a reference to a frozen or immutable object, and its value could be updated by set
or compare-and-swap operation. Thus, a dedicated set of objects could be used to create mutable shared object graphs
(of immutable objects). Cycles in the shared memory could be created using atomic references.
Kotlin/Native runtime doesn't support garbage collecting cyclic data when the reference cycle goes through
`AtomicReference` or frozen `FreezableAtomicReference`. So to avoid memory leaks, atomic references
that are potentially parts of shared cyclic data should be zeroed out once no longer needed.

If atomic reference value is attempted to be set to a non-frozen value, a runtime exception is thrown.

Freezable atomic reference is similar to the regular atomic reference until frozen behaves like a regular box
for a reference. After freezing, it behaves like an atomic reference and can only hold a reference to a frozen object.
