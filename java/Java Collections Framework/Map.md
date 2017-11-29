### HashMap LinkedHashMap
1，可以用null做为key和value  
2，capacity和load factor影响性能  
3，线程不安全，外部用同步对象控制，还可以用synchronizedMap  
4，当key的hashcode都相同时，如果支持Comparable，则进行排序降低影响  
5，迭代时并发更改了map，则会抛出ConcurrentModificationException  
6，当元素拥挤时，会将列表转变成树，优化性能，缩减时会恢复为链表  
7，hash(key)核心hash算法，在长度为2的幂前提下，hash值高位下移16位，参与取模，降低碰撞，另外取模部分利用2的幂减1来做&操作提高了性能。这里是取模的优化方案，在HashMap中约定长度必然是2的幂，然后在取模时才有&操作以提高性能。  
8，链表状态下碰撞元素加入时是放入链表尾部，联想到redis放入元素是放入头部，因为它认为最近放入的元素可能最容易被使用。  
9，继承Serializable接口，可是字段使用transient修饰，比如table,entrySet。原因是hashcode操作依赖jvm所处的环境因素,不同环境可能有不同的hash值，做一现成存储的内容既是序列化也无法通用.所以hashmap自己实现了writeObject和readObject，这里就需要知道java在序列化和反序列化一个类时是先调用writeObject和readObject,如果没有默认调用的是ObjectOutputStream的defaultWriteObject以及ObjectInputStream的defaultReadObject方法。
（http://www.cnblogs.com/zhilu-doc/p/5338462.html）  
10，优化部分中的树，查找自己位置时折半查找效率高于链表，而删除操作效率低于链表。树中当两个节点的hash一样，会利用compareTo方法比较，如果还是相同，就使用identityHashCode（http://blog.csdn.net/tbdp6411/article/details/46915981）进行比较。  
11,LinkedHashMap作为它的子类用模版方法的方式实现了排序的功能，可以看见LinkedHashMap源码中Entry定义了before，after来定义自己元素的前后。存储结构完全使用hashmap一套，只是用另外一个线路链接起全部元素。  

### SortedMap NavigableMap TreeMap：  
1,继承结构：SortedMap <—NavigableMap <— TreeMap 从TreeMap看，它实现了Map,SortedMap,NavigableMap  
2,对于顺序来说，必然是有比较，一个基础的知识Comparable是一个接口，继承后实现int compareTo(T o)方法，再比较时直接调用这个object的这个方法为依据来确认。Comparators是给集合对象外部强加的比较方法，它可以被传给一些排序方法，比如：Collections.sort，Arrays.sort。Comparators还用于某些对象的排序，比如：SortedSet SortedMap，或者提供给一些没有实现Comparable接口的集合拥有排序的能力。  
3，TreeMap故名思义就是用树结构实现的，进源代码第一句话，TreeMap是红黑树的实现方式。树这个结构以及各种演变的结构都是前人为了解决问题发明的（https://www.cnblogs.com/maybe2030/p/4732377.htm）  
4，所有SortedMap的实现都应该实现4种构造方法，这个在TreeMap中可以得到验证。  
5，TreeMap继承了NavigableMap，NavigableMap是SortedMap的扩展接口，提供了查询相关的方法列表以供子类实现，比如 lowerEntry floorEntry。其中pollFirstEntry，pollLastEntry获取并删除元素，有点队列的作用。  
6，TreeMap和HashMap一样它也实现了两个私有方法writeObject和readObject，用于自定义序列化，以及也是线程不安全的，可以使用SortedMap m = Collections.synchronizedSortedMap(new TreeMap(...))。关于synchronizedMap,我们查看Collections其实很简单就是使用装饰模式把包装进来的map全部方法在调用时都先用synchronized (mutex)这种方式来进行同步化。  
7，TreeMap的实现的操作比如：containsKey get put remove 算法时间复杂度是log(n)，算法可以参考《算法导论》。看来有必要好好学习下了。  
8，为了维持tree的平衡，在TreeMap.put方法部分操作，一个把新加或删除的节点找到然后操作，第二个是在新加或删除的节点基础上将树进行平衡操作。  
9，所以TreeMap是通过key.compareTo()或Comparator来定位key的坑位置，HashMap是用hashcode,两套不同的实现的map结构。但是它们都是用equals来决定key的唯一性的。

### WeakHashMap  
1,WeakHashMap使用WeakReference作为Entry，是交weak的原因，内部结构实现和HashMap差异不大，不过没有HashMap的转变成树这么复杂，就是简单链表就实现了。
关键还是WeakReference，java中的引用类型有：强引用，软引用，弱引用，虚引用。这个设计是java对内存回收的部分能力提供给开发者了。
2.一直疑惑WeakHashMap的使用场景，找到一个很好的例子，tomcat源码中使用WeakHashMap：（https://github.com/apache/tomcat/blob/trunk/java/org/apache/tomcat/util/collections/ConcurrentCache.java）

### IdentityHashMap
1，无论是TreeMap还是HashMap，都是用equals来确定key的唯一性，而IdentityHashMap的特殊指出是它用==。


### ConCurrentHashMap
1，无论synchronizedMap，还是HashTable，在操作map的时候都是锁住整个map的，达到了线程安全的效果，效果不佳，不过我觉得在小概率少量并发的场景里，简单使用是可行的。ConCurrentHashMap从结构上进行了设计，将一个map分段成多个Segment，这些Segment作为整个map的一部分在并发操作时进行锁操作，这样就不用锁整个map了，说白了就是把原来要锁整个map优化成锁map的一小段。


### ConcurrentReaderHashMap



### ConcurrentSkipListMap
