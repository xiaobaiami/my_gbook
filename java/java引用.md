java引用分为四种: 强引用, 软引用, 弱引用, 虚引用.
## 强引用
## 软引用

软引用是在内存不够的时候, GC会回收只有软引用指向的对象, 适合作为缓存

## 弱引用

弱引用在GC触发的时候会被回收掉, 无论此时内存是否足够. 在ThreadLocal中有弱引用的使用. 

当用户调用了threadLocal = null 后, 虽然thread对象中的threadLocalMap变量的 Entry[] 中任然持有threadLocal的引用, 但是是弱引用. 虚拟机会在GC时回收掉, 那么原本的threadLocal引用就会变为null之后就有可能被清理掉, 防止内存泄漏.

## 虚引用
虚引用是最弱的一种引用, 我们没有办法通过虚引用获取到他本身, 就是说一旦创建, 虚引用就归于jvm管理, 我们就无法获取. 
在虚拟机回收虚引用的时候, 会得到一个通知.
```java
public class PhantomReference<T> extends Reference<T> {
    /**
     * 因为虚引用总是返回null, 所以这个是 return null;
     */
    public T get() {
        return null;
    }
    
    /**
     * PhantomReference 的构造方法, 两个参数, 一个是传入的虚拟用对象,
     * 第二个参数是一个队列, 当虚引用被GC时会被放入到这个队列中.
     */
    public PhantomReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
```
虚引用在java中 `Cleaner` 对象中使用到, 方便对对象的回收, 这个对象可能是不属于jvm管理的. 我们利用这个通知来触发回收的任务.  netty的 `DirectByteBuffer` 就是通过这个方式来实现堆外内存的清理. 下面就以 `DirectByteBuffer` 示例举例

```java
public class Cleaner extends PhantomReference<Object> {
    private static final ReferenceQueue<Object> dummyQueue = new ReferenceQueue();
    private Cleaner(Object var1, Runnable var2) {
        super(var1, dummyQueue);
        this.thunk = var2;
    }
	/**
     * cleaner 的构造方法私有, 只能通过create方法创建
     * 除了要传入引用, 需要传入一个Runnable 对象, 告诉cleaner 如果清理对象
     * var0 = DirectByteBuffer的实例对象
     * var1 = 清理方法, 里面有DirectByteBuffer的实例对象的起始位置, 内存大小信息
     */
    public static Cleaner create(Object var0, Runnable var1) {
        return var1 == null ? null : add(new Cleaner(var0, var1));
    }
	/**
     * 虚引用对象被回收, 有一个线程会遍历队列, 找到cleaner对象并调用clean方法
     * 线程是在父类Reference 中定义的
     */
    public void clean() {
        if (remove(this)) {
            try {
                this.thunk.run(); // 这里的thunk就是create()方法的第二个参数
            } catch (final Throwable var2) {
            }
        }
    }
}

public abstract class Reference<T> {
    /**
    * 这个就是清理对象的线程
    */
    private static class ReferenceHandler extends Thread{
        public void run() {
            while (true) {
                tryHandlePending(true);
            }
        }
    }
    static boolean tryHandlePending(boolean waitForNotify) {
        Reference<Object> r;
        Cleaner c;
        c = r instanceof Cleaner ? (Cleaner) r : null;
        if (c != null) {
            c.clean(); // 从队列中遍历元素, 判断是否是cleaner对象, 如果是则调用cleaner对象的clean() 方法清理
            return true;
        }
    }
}
```

