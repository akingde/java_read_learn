### synchronized  
synchronized用于资源进行加锁，以保证同一时间只有一个线程可以访问这个资源。

#### 使用层面的synchronized
使用java开发你都无法避免会使用synchronized关键字，而在真正使用中我们需要了解一些规则。  

##### 用synchronized修饰的资源类型来进行区分：
###### 1，修饰方法的用法
```JAVA
public synchronized void funa（）{
    //互斥代码
}
public synchronized void funb（）{
    //互斥代码
}
```
这个是锁执行这个方法的对象。这么说理解起来也比较方便，就是我们把这个对象给锁了，那么要来获取这个锁的都是互斥的，funa和funb用同个对象不能并发执行。

###### 2，修饰this
```JAVA
synchronized（this）{
    //骄傲的代码
}
```
this是这个调用的对象，那就和第一种是一个情况了，大家都是在锁一个东西所以这个代码块的执行也是和第一种情况的方法是互斥的





Java头对象，它实现synchronized的锁对象的基础，这点我们重点分析它，一般而言，synchronized使用的锁对象是存储在Java对象头里的，jvm中采用2个字来存储对象头(如果对象是数组则会分配3个字，多出来的1个字记录的是数组长度)
