
### 前言

提到 ThreadLocal， Java 开发者并不陌生。在面试中，也经常被面试官提及，对 Java 开发者而言也是一个必须掌握的知识点，所以将它理解透彻是很有必要的。

文章稍微有点长，不过介绍的还是比较细致。

### ThreadLocal 是什么

ThreadLocal 是一个关于创建线程局部变量的类，主要作用是做数据隔离，保存到 ThreadLocal 中的数据只属于当前线程，该数据对其他线程而言是隔离的。也就是说，使用 ThreadLocal 保存的数据只能被当前线程访问，其他线程无法访问和修改。在多线程环境下，防止自己的变量被其他线程篡改。

>注意：ThreadLocal 设计的目的就是为了能够在当前线程中有属于自己的变量，并不是为了解决并发或者共享变量的问题。

下面，我们来看看这个例子：主线程初始化了一个 ThreadLocal 对象 threadLocal，并通过 threadLocal.set() 方法保存了一个值：“value1”，然后使用 threadLocal.get() 拿到设置的值。其中，子线程也使用 threadLocal.get() 去拿值，但是拿到的值是 null。

```java
public class Main {

    private static ThreadLocal<String> threadLocal = new ThreadLocal<>();

    public static void main(String[] args) {
        // 主线程设置值
        threadLocal.set("value1");
        System.out.println(Thread.currentThread().getName() + " = " + threadLocal.get());
        // 子线程
        new Thread(new Runnable() {
            @Override
            public void run() {
                // 子线程获取的值是：null
                System.out.println(Thread.currentThread().getName() + " = " + threadLocal.get());
            }
        }).start();

    }
}
```
结果：

```
main = value1
Thread-0 = null
```

但是，上面例子中，如果我们把变量换成是一个共享的对象保存到 ThreadLocal 中，那么多个线程的 ThreadLocal.get() 取得的还是这个共享对象本身，还是有并发访问问题。 所以要在保存到 ThreadLocal 之前，通过克隆或者 new 来创建新的对象，然后再进行保存。

```java
public class Main {

    private static ThreadLocal<User> threadLocal = new ThreadLocal<>();

    public static void main(String[] args) {
        // 主线程设置值
        User user = new User();
        threadLocal.set(user);
        System.out.println(Thread.currentThread().getName() + " = " + threadLocal.get());
        // 子线程
        new Thread(new Runnable() {
            @Override
            public void run() {
                threadLocal.set(user);
                // 获取到的对象地址是一样的，如果对它的值进行了修改，那么其他线程拿到的值也会改变。
                System.out.println(Thread.currentThread().getName() + " = " + threadLocal.get());
            }
        }).start();

    }
}
```
结果：

```
main = top.yifan.User@17f052a3
Thread-0 = top.yifan.User@17f052a3
```

从上面的例子，我们可以看出 ThreadLocal 的使用很简单，主要就是 set 和 get 方法，没有其他花里胡哨的操作。

下面，我们来看看 ThreadLocal 是如何做到线程间数据隔离的？

### ThreadLocal 源码分析

首先，我们先来看看 ThreadLocal 的 set 方法的源码。理解了 set 方法的实现，就明白 ThreadLocal 是如何做到数据隔离的。

set 源码：
```java
public void set(T value) {
  // 获取当前线程
  Thread t = Thread.currentThread();
  // 利用当前线程获取一个 ThreadLocalMap 的对象
  ThreadLocalMap map = getMap(t);
  // 如果上面获取的 ThreadLocalMap 对象不为空，则设置值，否则创建这个 ThreadLocalMap 对象并设置值
  if (map != null)
    map.set(this, value);
  else
    createMap(t, value);
}
  
  /**
  * Create the map associated with a ThreadLocal. Overridden in
  * InheritableThreadLocal.
  *
  * @param t the current thread
  * @param firstValue value for the initial entry of the map
  */
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

表面上看，ThreadLocal 的 set 方法源码很简单，其实不然。我们继续深入，这里需要关注一下 ThreadLocalMap。

从 set 源码中，我们得知 ThreadLocalMap 是利用当前线程 Thread 作为参数获取的。源码如下：

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```
其实，上面的代码获取的是 Thread 对象的 threadLocals 变量。源码如下：
```java
public class Thread implements Runnable {
    省略其他内容...

    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    省略其他内容...
}
```

到这里，我们基本可以知道 ThreadLocal 的数据是放入了当前线程的一个ThreadLocalMap 实例中，key 就是 ThreadLocal 对象本身，所以只能在本线程中访问，其他线程无法访问，从而实现了数据隔离。

下面，我们再来看看 ThreadLocal 的 get 方法源码。

```java
public T get() {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取 ThreadLocalMap 对象
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 以 ThreadLocal 对象本身作为 key，获取值
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // 如果 ThreadLocalMap 对象不存在，就设置初始值并返回
    // 从下面 setInitialValue() 的源码可知，设置的初始值是一个 null
    return setInitialValue();
}

/**
  * Variant of set() to establish initialValue. Used instead
  * of set() in case user has overridden the set() method.
  *
  * @return the initial value
  */
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}

protected T initialValue() {
    return null;
}
```

这里，我们应该知道 ThreadLocal 的工作原理了。

- Thread 类中维护着一个 ThreadLocalMap 类型的成员变量。
- ThreadLocalMap 是一个定义在 ThreadLocal 类中的内部类，是一个 map，用Entry 来进行数据存储。
- 当调用 ThreadLocal 的 set() 方法时，先获取当前线程的 ThreadLocalMap 对象，然后以 ThreadLocal 对象作为 key 往 ThreadLocalMap 中设置值。
- 当调用 ThreadLocal 的 get() 方法时，也是先获取当前线程的 ThreadLocalMap 对象，以 ThreadLocal 对象作为 key 从 ThreadLocalMap 中获取值。
- ThreadLocal 本身并不存储值，它只是作为一个 key 来让线程在ThreadLocalMap 中获取或设置值。

**ThreadLocal 的工作原理决定了，ThreadLocal 活动范围仅限于某个线程，每个线程独自拥有自己的变量**。

我们都知道，在 Java 中，栈内存归属于单个线程，每个线程都会有一个栈内存，其存储的变量只能在其所属线程中可见，即栈内存可以理解成线程的私有内存，而堆内存中的对象对所有线程可见，堆内存中的对象可以被所有线程访问。**那么是不是说 ThreadLocal 的实例以及其值存放在栈上呢？**  
其实不是的，因为 ThreadLocal 实例实际上也是被其创建的类持有（更顶端应该是被线程持有），而 ThreadLocal 的值其实也是被线程实例持有。它们都是位于堆上，只是通过一些技巧将可见性修改成了线程可见。（摘自 [理解Java中的ThreadLocal](https://droidyue.com/blog/2016/03/13/learning-threadlocal-in-java/) ）

下面，我们聊聊 ThreadLocal 的内部类 ThreadLocalMap。

### ThreadLocalMap 源码分析

初看，ThreadLocalMap 是一个类似 HashMap 的数据结构，但是在 ThreadLocal 中，并没实现 Map 接口。ThreadLoalMap 的 Entry 是继承 WeakReference（弱引用），在 Entry 的构造方法中，调用了 super(k) 方法，只是将传入的 key 包装成一个弱引用对象。同时 Entry 中没有 next 字段，所以就不存在链表的情况了。ThreadLocalMap 和 HashMap 还是有很大的区别的。

ThreadLocalMap 部分源码：

```java
static class ThreadLocalMap {

        /**
         * Entry 是 ThreadLocalMap 的内部类，是一个弱引用对象
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
              // Entry 的 key 为 ThreadLocal 对象，
                super(k);
                value = v;
            }
        }

        /**
         * 初始长度是 16，必须是 2 的幂
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * Entry 数组，用来存放一个线程的多个 ThreadLocal 变量，长度必须是 2 的幂
         */
        private Entry[] table;
        
        /**
         * 扩容的阈值，默认是数组大小的 2/3
         */
        private int threshold;

        /**
         * 设置扩容阈值，数组大小的 2/3
         */
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }

      省略其他内容...
}   
```

从源码中，我们可以得知 ThreadLocalMap 的结构大致如下：
![ThreadLocalMap](https://img-blog.csdnimg.cn/2020081715500060.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L29zY2hpbmFfNDE3OTA5MDU=,size_16,color_FFFFFF,t_70#pic_center)

在创建 ThreadLoalMap 对象时会初始化一个大小是 16 的 Entry 数组，扩容阈值是数组大小的 2/3，扩容后 Entry 数组大小是原来的 2 倍，Entry 对象用来保存每一个键值对(key-value)，而这里的 key 永远都是 ThreadLocal 对象本身，通过 ThreadLocal 对象的 set 方法，把 ThreadLocal 对象自己当做 key，放进了 ThreadLoalMap 中。

### ThreadLocalMap 的 Hash 冲突问题

上面，我们知道 **ThreadLocalMap 底层数据结构是一个数组**，元素是 Entry，那么它是如何解决 Hash 冲突的呢？

我们先看看 ThreadLocalMap 的 set 方法源码：

```java
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    // 根据 ThreadLocal 对象的 hashCode 确定 Entry 应该存放的位置
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        // 如果当前位置的 Entry 对象的 key 正好与传入的 key 相同，那么就覆盖 Entry 中的 value
        if (k == key) {
            e.value = value;
            return;
        }
        // 当 key 为 null 时，说明 ThreadLocal 弱引用已经被释放掉，
        // 那么就无法再通过这个 key 获取 ThreadLocalMap 中对应的 Entry 对象中的 value，这里就存在内存泄漏的可能性
        if (k == null) {
            // 用当前传入的 key 和 value 替换掉这个 key 为 null 的 Entry 对象，并清除过期的对象
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    // 如果位置为空，新建 Entry 对象并插入 table 中 i 处
    tab[i] = new Entry(key, value);
    int sz = ++size;
    // 插入后再次清除一些 key 为 null 的 Entry 对象，如果大于阈值就需要扩容
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

从 set 方法的源码中，可以知道 ThreadLocalMap 在存储数据的大致过程如下：
- 首先，利用 ThreadLocal 对象的 threadLocalHashCode 计算一个 hash 值作为数据插入的位置 i（int i = key.threadLocalHashCode & (len-1);）。
- 如果位置 i 存在有 Entry 对象且 Entry 的 key 刚好与传入的 key 相等，就使用传入的 value 覆盖原有的 value。
- 如果位置 i 存在有 Entry 对象但是 Entry 的 key 与传入的 key 不相关，就继续往下找。
- 如果位置 i 存在有 Entry 对象但是 Entry 的 key 为 null，则使用传入的 key 和 value 替换
- 如果位置 i 刚好是空的，就新建 Entry 对象插入到该位置。

到这里，我们知道 ThreadLocalMap 是通过 `nextIndex(i, len)` 方法解决 Hash 冲突的问题。通过遍历，从冲突的位置依次往后搜索空单元，如果到数组尾部，再从头开始搜索，形成环形查找。其实，这就是**线性搜索法**，开放地址法的一种实现手段。

这里需要注意一下 `threadLocalHashCode`，请看 ThreadLocal 部分源码：

```java
public class ThreadLocal<T> {
    // ThreadLocal 对象的 hash 值
    private final int threadLocalHashCode = nextHashCode();

    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }

    ...
}
```
根据源码可知，每个 ThreadLocal 对象都有一个 hash 值 threadLocalHashCode，每初始化一个 ThreadLocal 对象，hash 值就递增 `0x61c88647` 大小。查资料得知，`0x61c88647` 这个数是有特殊意义的，它能够保证 hash 表的每个散列桶能够均匀的分布，这是**斐波那契散列**。

说完 ThreadLocalMap 的 set 方法，我们再来看看它的 getEntry 方法。

```java
private Entry getEntry(ThreadLocal<?> key) {
    // 计算 Entry 的位置
    int i = key.threadLocalHashCode & (table.length - 1);
    // 根据计算出的位置 i 获取 Entry 对象
    Entry e = table[i];
    // 满足条件则返回该 Entry
    if (e != null && e.get() == key)
        return e;
    else
        // 未查找到满足条件的 Entry，继续进行处理
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            // 找到和查询的 key 相同的 Entry 则返回， 不相等就继续往下查找
            return e;
        if (k == null)
            // 如果遇到 key 是 null 的 Entry 对象，则进行清除
            expungeStaleEntry(i);
        else
            // 继续往下环形查找
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```
可以看出，在 getEntry 的时候，也会利用 ThreadLocal 对象的 `threadLocalHashCode` 计算 hash 值以确定 Entry 的位置，然后判断 Entry 的 key 和传入的 key 是否相等。如果相等，就直接返回这个 Entry；如果不相等，说明在 set 值的时候存在 Hash 冲突的情况，然后调用 getEntryAfterMiss 方法继续往下进行环形查找。

通过 set 和 getEntry 源码可以看出，如果 Hash 冲突严重的话，它们的效率都很低。

### ThreadLocal 的内存泄露问题

通过上面一系列源码的分析，我们可以知道 ThreadLocal 其实存在内存泄漏问题。

我们知道 ThreadLocalMap 使用 ThreadLocal 的弱引用作为 key，如果一个 ThreadLocal 没有外部强引用时，那么系统 GC 的时候，这个 ThreadLocal 势必会被回收，这样 ThreadLocalMap 中就会出现 key 是 null 的 Entry 对象，那么这些 key 是 null 的 Entry 对象中的 value 就无法访问到，一直存在内存中。如果当前线程一直处于运行中，那么这些 Entry 对象中的 value 就可能一直无法回收，就会发生内存泄漏。

> 关于**弱引用**的解释：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程，因此不一定会很快发现那些只具有弱引用的对象。

在实际开发中，会使用线程池去维护线程的创建和复用，比如固定大小的线程池，线程为了复用是不会主动结束的，那么 ThreadLocal 设置的 value 值就一直被引用，就会发生内存泄漏。

其实，为了避免内存泄漏问题， ThreadLocalMap 也是做了很多努力的。细心的小伙伴应该已经注意到了，在 ThreadLocalMap 的 set 和 getEntry 方法中，出现了 `replaceStaleEntry`、`cleanSomeSlots` 以及 `expungeStaleEntry` 方法。这三个方法其实就是清除 key 是 null 的 Entry 对象的。 然而，核心还是 `expungeStaleEntry` 方法，其他两个方法都调用了它。expungeStaleEntry 方法会将 key 是 null 的 Entry 对象的 value 置为 null，以便垃圾回收时能够清理，同时也会将 Entry 对象置为 null。

expungeStaleEntry 方法源码：

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
          (e = tab[i]) != null;
          i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;
                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

所以，从 ThreadLocal 的设计上看，在调用 set 和 get 方法时，都会对 key 是 null 的 Entry 对象进行清除处理，解决隐藏的内存泄漏的问题。这里不得不佩服 Josh Bloch 和 Doug Lea 大师的厉害之处。

但是，光这样还是不够的，因为只有在调用 ThreadLocal 的 set 或 get 方法时，才会对 key 是 null 的 Entry 对象进行清除处理，这是一个前提条件，但我们不可能在任何情况都调用 set 或 get 方法。所以，为了在任何情况下都能防止内存泄漏，我们最好手动调用 ThreadLocal 的 remove 方法，清除过期的数据。

```java
ThreadLocal<String> threadLocal = new ThreadLocal();
try {
    threadLocal.set("value");
    // 其它业务逻辑
} finally {
    threadLocal.remove();
}
```

remove 方法源码如下：

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        // 其实是调用了 ThreadLocalMap 的 remove 方法
        m.remove(this);
}

// ThreadLocalMap 的 remove 方法
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
          e != null;
          e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```
上面，我们看到 ThreadLocalMap 的 remove 方法也调用了 expungeStaleEntry 方法。

### ThreadLocal 的使用场景

**Hibernate 中的 Session 管理**

Hibernate 中通过 ThreadLocal 管理 session 就是一个典型的案例，不同的请求线程（用户）拥有自己的 session，若将 session 共享出去被多线程访问，必然会带来线程安全问题。

**解决 SimpleDateFormat 线程不安全问题**

SimpleDateFormat.parse() 方法存在线程安全问题，该方法内部有一个 Calendar 对象。在调用 parse 方法时会先调用 Calendar.clear() 方法，然后调用 Calendar.add() 方法。如果在此期间，有一个线程先调用了 Calendar.add() 方法，然后另一个线程又调用了 Calendar.clear() 方法，那么最后 parse 方法解析的时间就是错误的。这里我们就可以使用 ThreadLocal 包装 SimpleDateFormat 来解决线程安全问题。
```java
private static final ThreadLocal <DateFormat> df = new ThreadLocal<DateFormat>() {
        @Override
        protected DateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        }
    };
```

**管理数据库连接池中的 Connection 对象**

ThreadLocal 能够实现当前线程的操作都是用同一个 Connection，保证了事务。

**避免一个线程内过度传递参数的问题**

项目中经常会遇到在一个线程中横跨若干方法调用，需要传递的对象，通常叫上下文（Context），它是一种状态，可以是用户身份、任务信息等，这样就会存在过度传参问题。如果给每个方法增加一个 context 参数非常麻烦，而且有些时候存在调用链有无法修改源码的第三方库，context 参数就传不进去了。这个时候，就可以使用 ThreadLocal 在这些方法之间进行参数传递。只需在之前将参数设置到 ThreadLocal 中，其他方法使用参数时，只需调用 ThreadLocal 的 get 方法就可以拿到参数。

未使用 ThreadLocal：

```java
public void process(Context context) {
    step1(context);
    step2(context);
    step3(context);
}
```

使用 ThreadLocal：

```java
public void process(Context context) {
    
    try {
        threadLocal.set(context);
        // 在方法内部调用 threadLocal.get() 就可以拿到参数
        step1();
        step2();
        step3();
    } finally {
        threadLocal.remove();
    }
}

private void step1() {
  Context context = threadLocal.get();

  // do something
}
```

### 思考 

到这里，小伙伴们应该对 ThreadLocal 有一定的了解了。那么，我们来思考下面问题。

**当一个线程中有多个 ThreadLocal 对象，每一个 ThreadLocal 对象是如何区分的呢？**

其实，通过 ThreadLocal 的源码，我们可以看到：

```java
public class ThreadLocal<T> {
    // ThreadLocal 对象的 hash 值
    private final int threadLocalHashCode = nextHashCode();

    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }

    ...
}
```

对于每一个 ThreadLocal 对象，都有一个 final 修饰的 threadLocalHashCode 不可变属性，初始化后便不可修改，所以可以唯一确定一个 ThreadLocal 对象。同时，通过 nextHashCode() 方法，每次初始化的 ThreadLocal 对象的 threadLocalHashCode 都会递增，所以保证了每一个 ThreadLocal 都是不同的。


**为什么不直接用线程 id 来作为 ThreadLocalMap 的 key？**

这一点其实比较容易理解，如果直接用线程 id 来作为 ThreadLocalMap 的 key 的话，在同一个线程中就无法区分放入 ThreadLocalMap 中的多个 value。在获取 value 时，就无法知道获取的是哪个。

代码示列：
```java
public static void main(String[] args) {
        // 线程
        new Thread(new Runnable() {
            @Override
            public void run() {
                ThreadLocal<String> threadLocal1 = new ThreadLocal<>();
                threadLocal1.set("value1");
                ThreadLocal<String> threadLocal2 = new ThreadLocal<>();
                threadLocal2.set("value2");

                System.out.println(threadLocal1.get());
                System.out.println(threadLocal2.get());
            }
        }).start();
    }
```
通过之前的源码，我们可以知道 ThreadLocalMap 在定位数据时，使用的 ThreadLocal 对象的 threadLocalHashCode 值计算出的 hash 值，而每个 ThreadLocal 的 threadLocalHashCode 值又是唯一的，所以使用 ThreadLocal 作为 key 是最好的选择。

### 总结

这次阅读 ThreadLocal 的源码，给了我不少的震撼，就像一个好奇宝宝见到很多新鲜的事物一样。同时，也让我体会到了阅读源码的快乐，收获挺多的。希望也能给看到这篇文章的读者，带来一些收获。共勉！

### 参考

[1] 一文搞懂 ThreadLocal 原理: [https://www.cnblogs.com/wupeixuan/p/12638203.html](https://www.cnblogs.com/wupeixuan/p/12638203.html)  
[2] 使用ThreadLocal: [https://www.liaoxuefeng.com/wiki/1252599548343744/1306581251653666](https://www.liaoxuefeng.com/wiki/1252599548343744/1306581251653666)  
[3] Java面试必问，ThreadLocal终极篇: [https://www.jianshu.com/p/377bb840802f](https://www.jianshu.com/p/377bb840802f)  
[4] 理解Java中的ThreadLocal: [https://droidyue.com/blog/2016/03/13/learning-threadlocal-in-java/](https://droidyue.com/blog/2016/03/13/learning-threadlocal-in-java/)








