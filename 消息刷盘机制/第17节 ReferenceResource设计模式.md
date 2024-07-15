# 第17节 ReferenceResource设计模式

## 初识ReferenceResource设计模式

在讲解 *MappedFile* 时，我们提到 `mappedByteBuffer` 是通过 `fileChannel.map(MapMode.READ_WRITE, 0, fileSize)` 方式获取的文件内存映射 *buffer*。但是，该 *buffer* 为堆外内存，是不受 *JVM* 管理的内存区域，也就是不会被垃圾收集器回收。

在如下情况下，需要手动执行 `mappedByteBuffer` 的回收，防止堆外内存泄漏。

1. TODO
2. TODO
3. TODO

如果只是简单的手动回收该堆外内存，这会造成很严重的问题，比如消息丢失，这是消息中间件所不能容忍的，所以才有了该设计模式。

该设计模式就是保证在资源（可以理解成 *MappedFile*）被占用时，如果此时你对堆外内存执行回收，该回收操作是不会被立刻执行的，它会被推迟到等所有的资源占用都释放后再执行堆外内存的回收。

之所以在这里讲解，是想着结合消息提交操作的代码实现，大家更容易理解。

## 源码深入剖析

### 成员变量

按照惯例，我们先来了解一下 *ReferenceResource* 的成员变量，以便在后面的代码阅读中对这些变量有一个清晰的认识。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| refCount | AtomicLong | 资源引用计数。需要注意的是当引用计数小于等于 0（*refCount <= 0*）或者资源不可用（*available = false*）时，就不再允许后续再占用该资源，因为此时可能正在执行堆外内存的回收。 |
| available | volatile boolean | 资源是否可用，默认为 *true* |
| cleanupOver | volatile boolean | 是否已回收堆外内存，默认 *false* |
| firstShutdownTimestamp | volatile long | 首次执行关闭资源的时间 |

### 占用资源

`hold()` 方法执行资源占用（`refCount` + 1）。需要注意的是当引用计数小于等于 0 或者资源不可用时，就不再允许该操作，因为此时可能正在执行堆外内存的回收。

```java
/**
 * @return 是否成功占用资源
 */
public synchronized boolean hold() {
    if (this.isAvailable()) { // 资源可用
        if (this.refCount.getAndIncrement() > 0) {
            return true;
        } else { // 引用计数小于等于0，不再允许该操作，回滚引用计数
            this.refCount.getAndDecrement();
        }
    }

    return false;
}
```

### 释放资源

`release()` 方法执行释放资源占用（`refCount` - 1）。

注意，通常情况下该方法只仅仅是执行释放资源占用，在所有的资源占用都释放后，`refCount` 恢复初始值 1，最终所有执行释放资源占用的 `release()` 方法都不会执行堆外内存的回收。

但是如果执行 `shutdown(final long intervalForcibly)` 方法，它也会执行一次释放资源占用，这就破坏了上述的和谐，导致后面执行释放资源占用的 `release()` 方法会执行堆外内存的回收。

```java
public void release() {
    long value = this.refCount.decrementAndGet();
    if (value > 0)
        return;

    // 回收堆外内存
    synchronized (this) {

        this.cleanupOver = this.cleanup(value);
    }
}
```

### 触发资源回收

第一次调用该方法时设置 `available` 为 *false*，设置首次执行关闭资源的时间，并释放资源占用；之后再次调用该方法，如果 *`refCount` > 0*，且超过了强制间隔，则设置 `refCount` 为一个负数，并释放资源占用；

> 注：如果在 `intervalForcibly` 期间内再次调用该方法，它不会执行任何逻辑。

- 第一次调用该方法，会等待所有的资源占用都释放后，最后执行回收堆外内存；
- 第二次及以后的调用则会立刻执行回收堆外内存；

```java
/**
 * @param intervalForcibly 强制间隔，即两次生效的间隔至少要有这么大(不是至多!)
 */
public void shutdown(final long intervalForcibly) {
    if (this.available) {
        this.available = false; // 设置资源不可用
        this.firstShutdownTimestamp = System.currentTimeMillis(); // 设置首次执行关闭资源的时间
        this.release(); // 释放资源占用
    } else if (this.getRefCount() > 0) { // 若资源引用计数还大于0
        if ((System.currentTimeMillis() - this.firstShutdownTimestamp) >= intervalForcibly) { // 要超过强制间隔的阈值才行
            this.refCount.set(-1000 - this.getRefCount());
            this.release(); // 再次释放资源占用
        }
    }
}
```

### 回收堆外内存

执行回收逻辑的是 `cleanup(final long currentRef)` 方法，该方法为抽象方法，*MappedFile* 提供了实现。用来执行回收当前 *MappedFile* 的堆外内存 `mappedByteBuffer`，并更新 `TOTAL_MAPPED_VIRTUAL_MEMORY` 和 `TOTAL_MAPPED_FILES`。

```java
// MappedFile.java
@Override
public boolean cleanup(final long currentRef) {
    if (this.isAvailable()) { // 当前资源可用，停止回收堆外内存，返回false
        log.error("this file[REF:" + currentRef + "] " + this.fileName
                + " have not shutdown, stop unmapping.");
        return false;
    }

    if (this.isCleanupOver()) { // 已回收堆外内存，直接返回true
        log.error("this file[REF:" + currentRef + "] " + this.fileName
                + " have cleanup, do not do it again.");
        return true;
    }

    clean(this.mappedByteBuffer); // @1
    TOTAL_MAPPED_VIRTUAL_MEMORY.addAndGet(this.fileSize * (-1));
    TOTAL_MAPPED_FILES.decrementAndGet();
    log.info("unmap file[REF:" + currentRef + "] " + this.fileName + " OK");
    return true;
}
```

**(1) 代码@1**

执行回收当前 *MappedFile* 的堆外内存 `mappedByteBuffer`。

```java
// MappedFile.java
public static void clean(final ByteBuffer buffer) {
    if (buffer == null || !buffer.isDirect() || buffer.capacity() == 0)
        return;
    invoke(invoke(viewed(buffer), "cleaner"), "clean");
}
```

将上述代码拆解一下，

```java
viewed(buffer); // @@1

invoke(viewed(buffer), "cleaner"); // @@2

invoke(invoke(viewed(buffer), "cleaner"), "clean"); // @@3
```

**代码@@1**

```java
// MappedFile.java
private static ByteBuffer viewed(ByteBuffer buffer) {
    String methodName = "viewedBuffer";

    // 通过反射获取所有方法
    Method[] methods = buffer.getClass().getMethods();
    for (int i = 0; i < methods.length; i++) {
        if (methods[i].getName().equals("attachment")) {
            methodName = "attachment";
            break;
        }
    }

    ByteBuffer viewedBuffer = (ByteBuffer) invoke(buffer, methodName);
    if (viewedBuffer == null)
        return buffer;
    else
        return viewed(viewedBuffer);
}
```

这里简单讲解一下，`attachment` 代表附加到此缓冲区的对象。如果此缓冲区是另一个缓冲区的视图（*view*），那么我们使用此字段来保持对那个缓冲区的引用，以确保在操作完之前不释放其内存。

这里的循环调用就是为了获取最初的 *buffer*。

可能大家对上面的概念还不是很理解，那就从 *JDK* 的源码角度讲解一下吧。

我们在前面《MappedFile》一节中讲过**缓冲区分片**，`slice()` 方法根据现有的缓冲区 *A* 创建一个子缓冲区 *A'*。也就是它创建一个新的缓冲区，新缓冲区共享原来缓冲区的一部分数据。这里的 *A'* 的 `attachment` 就为 *A* ，而 *A* 的 `attachment` 则为 *null*。

```java
// DirectByteBuffer
private final Object att;

public Object attachment() {
    return att;
}

public ByteBuffer slice() {
    int pos = this.position(); // position
    int lim = this.limit(); // limit
    assert (pos <= lim);
    int rem = (pos <= lim ? lim - pos : 0);
    int off = (pos << 0);
    assert (off >= 0);
    return new DirectByteBuffer(this, -1, 0, rem, rem, off);
}

DirectByteBuffer(DirectBuffer db,         // package-private
                               int mark, int pos, int lim, int cap,
                               int off)
{

    super(mark, pos, lim, cap);
    address = db.address() + off;

    cleaner = null;

    att = db;
}
```

**代码@@2**

通过反射调用 *buffer* 的 `cleaner()` 方法获取 *Cleaner* 对象，每个堆外内存都有一个 *Cleaner* 用于释放堆外内存。

```java
private static Object invoke(final Object target, final String methodName, final Class<?>... args) {
    return AccessController.doPrivileged(new PrivilegedAction<Object>() {
        public Object run() {
            try {
                Method method = method(target, methodName, args);
                method.setAccessible(true);
                return method.invoke(target);
            } catch (Exception e) {
                throw new IllegalStateException(e);
            }
        }
    });
}

private static Method method(Object target, String methodName, Class<?>[] args)
        throws NoSuchMethodException {
    try {
        return target.getClass().getMethod(methodName, args);
    } catch (NoSuchMethodException e) {
        return target.getClass().getDeclaredMethod(methodName, args);
    }
}
```

**代码@@3**

调用 *Cleaner* 的 `clean()` 方法释放堆外内存。