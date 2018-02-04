```JAVA
public interface RandomAccess {
}
```
用于标记能快速随机访问的列表，核心目的是为了在随机或顺序访问列表时有良好的性能而允许通用算法来改变他们的行为。  
这里有两种类型的List，一种是随机访问List（ArrayList），一种是连续访问List（LinkedList）。推荐使用instanceof方法进行判断是否实现了RandomAccess接口用于区分这两种List。  
实现本接口的List，存储大量数据的时候，使用以下代码：
```JAVA
for (int i=0, n=list.size(); i &lt; n; i++)
        list.get(i);
```
要快于以下代码      
```JAVA
for (Iterator i=list.iterator(); i.hasNext(); )
        i.next();
在jdk的代码中可以看到这样的代码，用来得到更好的性能。比如：
```JAVA
public static <T> void fill(List<? super T> list, T obj) {
       int size = list.size();

       if (size < FILL_THRESHOLD || list instanceof RandomAccess) {
           for (int i=0; i<size; i++)
               list.set(i, obj);
       } else {
           ListIterator<? super T> itr = list.listIterator();
           for (int i=0; i<size; i++) {
               itr.next();
               itr.set(obj);
           }
       }
   }
```
