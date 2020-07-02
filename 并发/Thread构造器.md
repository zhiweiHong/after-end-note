# Thread()

直接构造一个Thread，要重写里面的run方法。

```java
Thread t1 = new Thread(){
    @Overwrite
    public void run(){
        
    }
}
```

# Thread(Runnable target)

这里的target在源码里，会被run方法调用

```java
 @Override
public void run() {
    if (target != null) {
        target.run();
    }
}
```

# Thread 名称相关

Thread内部有一个静态变量threadInitNumber，用于统计当前创建的线程。默认创建名称的规则是"Thread-"+threadInitNumber

```java
public Thread() {
  init(null, null, "Thread-" + nextThreadNum(), 0);
}
```

# ThreadGroup  相关

如果没有传入ThreadGroup，默认会获取到父线程的ThreadGroup

```java
if (g == null) {
    /* Determine if it's an applet or not */

    /* If there is a security manager, ask the security manager
               what to do. */
    if (security != null) {
        g = security.getThreadGroup();
    }

    /* If the security doesn't have a strong opinion of the matter
               use the parent thread group. */
    if (g == null) {
        g = parent.getThreadGroup();
    }
}
```

#  stackSize  相关

这个参数是虚拟栈的大小，即使设置了也不一定生效。

