>来源：[Thread ThreadLocal和ThreadLocalMap，用法+原理，我懵圈了？](https://blog.csdn.net/u011583316/article/details/107129450)
>
### 内部结构图
![image-20250107151826181](https://raw.githubusercontent.com/mendax92/pic/main/20250107151826181.png)
- 每个Thread线程内部都有一个ThreadLocalMap。
- Map里面存储线程本地对象ThreadLocal（key）和线程的所需存在的对象（value）。
- Thread内部的Map是由ThreadLocal维护，ThreadLocal负责向map获取和设置线程的变量值。
- 一个Thread可以有多个ThreadLocal。

```
public class ThreadTest {
    private static ThreadLocal<String> sThreadLocal = new ThreadLocal<>();
    private static ThreadLocal<Integer> sThreadLocal2 = new ThreadLocal<>();

    public static void main(String args[]) {
        sThreadLocal.set("这是在主线程中");
        System.out.println("线程名字：" + Thread.currentThread().getName() + "---" + sThreadLocal.get());
        //线程a
        new Thread(new Runnable() {
            @Override
            public void run() {
                sThreadLocal.set("这是在线程a中");
                sThreadLocal2.set(3);
                System.out.println("线程名字：" + Thread.currentThread().getName() + "---" + sThreadLocal.get());
                System.out.println("线程名字：" + Thread.currentThread().getName() + "---" + sThreadLocal2.get());
            }
        }, "线程a").start();
        //线程b
        new Thread(new Runnable() {
            @Override
            public void run() {
                sThreadLocal.set("这是在线程b中");
                System.out.println("线程名字：" + Thread.currentThread().getName() + "---" + sThreadLocal.get());
            }
        }, "线程b").start();
        //线程c
        new Thread(() -> {
            sThreadLocal.set("这是在线程c中");
            System.out.println("线程名字：" + Thread.currentThread().getName() + "---" + sThreadLocal.get());
        }, "线程c").start();
    }
}
```

打印
```
线程名字：main---这是在主线程中
线程名字：线程a---这是在线程a中
线程名字：线程a---3
线程名字：线程b---这是在线程b中
线程名字：线程c---这是在线程c中
```

### ThreadLocal探析
- set()方法用于保存当前线程的所需存在的对象
- get()方法用于获取当前线程的存储的对象
- initialValue()为当前线程初始存储的对象,即在get方法不存在值的时候返回null
- remove()方法移除当前线程的对象

#### set方法
1. 第一步会去获取调用当前方法的线程Thread。
2. 然后顺其自然的拿到当前线程内部的ThreadLocalMap容器。
3. 最后就把对象给丢进去。
```
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

#### get方法
1. 获取当前线程的ThreadLocalMap对象
2. 从map中根据this（当前的threadlocal对象）获取线程存储的Entry节点
3. 从Entry节点获取存储的对应Value返回
4. map为空的话返回初始值null,并初始化ThreadLocalMap
```
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null)
            return (T)e.value;
    }
    return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

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

#### remove方法
```
public void remove() {
 ThreadLocalMap m = getMap(Thread.currentThread());
 if (m != null)
     m.remove(this);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

 /**
  * Remove the entry for key.
  */
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
ThreadLocalMap是ThreadLocal的内部类，实现了一套自己的Map结构，内部维护了一个数组
![image-20250107152056463](https://raw.githubusercontent.com/mendax92/pic/main/20250107152056463.png)
#### ThreadLocalMap的成员变量
```
static class ThreadLocalMap {
    /**
     * The initial capacity -- MUST be a power of two.
     */
    private static final int INITIAL_CAPACITY = 16;

    /**
     * The table, resized as necessary.
     * table.length MUST always be a power of two.
     */
    private Entry[] table;

    /**
     * The number of entries in the table.
     */
    private int size = 0;

    /**
     * The next size value at which to resize.
     */
    private int threshold; // Default to 0
}
```

#### HashCode 计算
ThreaLocalMap中没有采用传统的调用ThreadLocal的hashcode方法（继承自object的hashcode），而是调用nexthashcode，源码如下：
```
private final int threadLocalHashCode = nextHashCode();
private static AtomicInteger nextHashCode = new AtomicInteger();
 //1640531527 能够让hash槽位分布相当均匀
private static final int HASH_INCREMENT = 0x61c88647; 
private static int nextHashCode() {
      return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

#### Hash冲突
和HashMap的最大的不同在于，ThreadLocalMap解决Hash冲突的方式就是简单的步长加1或减1及线性探测，寻找下一个相邻的位置。
```
/**
 * Increment i modulo len.
 */
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}
/**
 * Decrement i modulo len.
 */
private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}
```
ThreadLocalMap采用线性探测的方式解决Hash冲突的效率很低，如有大量不同的ThreadLocal对象放入map中时发送冲突。所以建议每个线程只存一个变量（一个ThreadLocal）就不存在Hash冲突的问题，如果一个线程要保存set多个变量，就需要创建多个ThreadLocal，多个ThreadLocal放入Map中时会极大的增加Hash冲突的可能。

清楚意思吗？当你在一个线程需要保存多个变量时，你以为是多次set？你错了你得创建多个ThreadLocal，多次set的达不到存储多个变量的目的。
```
sThreadLocal.set("这是在线程a中");
```