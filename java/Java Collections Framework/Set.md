### HashSet
1，内部维护一个HashMap来实现：
```JAVA
private transient HashMap<E,Object> map;
```
简单来说，就是HashMap的key就是set。看add方法：

```JAVA
private static final Object PRESENT = new Object();

public boolean add(E e) {
      return map.put(e, PRESENT)==null;
  }
```
2，可见学习[HashMap](https://github.com/dchack/java_read_learn/blob/master/java/Java%20Collections%20Framework/Map.md#hashmap-linkedhashmap)的重要性了。

### LinkedHashSet
1，和LinkedHashMap继承HashMap一样，LinkedHashSet继承HashSet。构造方法在HashSet上：
```JAVA
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
       map = new LinkedHashMap<>(initialCapacity, loadFactor);
   }
  ```
dummy字段是无用的字段只是为了重载出一个定制的构造函数，用LinkedHashMap代替HashMap。  
和HashSet原理一样用map的key实现了set。

### TreeSet
1，举一反三，TreeSet肯定是用[TreeMap](https://github.com/dchack/java_read_learn/blob/master/java/Java%20Collections%20Framework/Map.md#sortedmap-navigablemap-treemap)实现的了。
