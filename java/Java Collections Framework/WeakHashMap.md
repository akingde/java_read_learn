
##### 概念
WeakHashMap使用WeakReference作为Entry，是交weak的原因，内部结构实现和HashMap差异不大，不过没有HashMap的转变成树这么复杂，就是简单链表就实现了。
关键还是WeakReference，java中的引用类型有：强引用，软引用，弱引用，虚引用。这个设计是java对内存回收的部分能力提供给开发者了。
##### 测试例子
以下例子就描述了WeakHashMap的特性了
```JAVA
  WeakHashMap<String, String> map = new WeakHashMap<>();
  String value = new String("value");
  map.put(new String("key"), value); // {key=value}
  System.out.println(map);
  System.gc();
  //        System.runFinalization();
  System.out.println(map);     // {}
```

##### 使用场景例子：
一直疑惑WeakHashMap的使用场景，找到一个很好的例子，tomcat源码中使用WeakHashMap：（https://github.com/apache/tomcat/blob/trunk/java/org/apache/tomcat/util/collections/ConcurrentCache.java）
