### ArrayList
1，简单来讲ArrayList就是个可以伸缩容量的数组结构体。内部维护了一个Object[] elementData  
2，数组的伸缩，必然会使用到数组的复制操作，所以多处调用了System.arraycopy方法。还有Arrays.copyOf(elementData, size)方法实际也是调用了底层的 System.arraycopy方法。  
3，add方法新增元素，就是向数组最后一位放。那么容量不够了扩容的规则是这样的：
```JAVA
private void grow(int minCapacity) {
      // overflow-conscious code
      int oldCapacity = elementData.length;
      int newCapacity = oldCapacity + (oldCapacity >> 1);
      if (newCapacity - minCapacity < 0)
          newCapacity = minCapacity;
      if (newCapacity - MAX_ARRAY_SIZE > 0)
          newCapacity = hugeCapacity(minCapacity);
      // minCapacity is usually close to size, so this is a win:
      elementData = Arrays.copyOf(elementData, newCapacity);
  }
```
会增加oldCapacity >> 1（原容量除2整数部分）个容量出来，如果还是没达到这次加入数据后能放下的数据，那就扩容到能放下这下数据的最小容量。
4，ArrayList是线程不安全的。  
5，因为是数组实现当要找一个特定的元素时，就需要遍历比对来完成了：
```JAVA
public boolean contains(Object o) {
    return indexOf(o) >= 0;
}
public int indexOf(Object o) {
     if (o == null) {
         for (int i = 0; i < size; i++)
             if (elementData[i]==null)
                 return i;
     } else {
         for (int i = 0; i < size; i++)
             if (o.equals(elementData[i]))
                 return i;
     }
     return -1;
 }
 ```
 6，序列化实现自己实现了writeObject和readObject方法，思路是只需要将放了元素的值全部记录下来即可，因为毕竟哪些没放满的空间就没必要序列化占空间记录了。
 ```JAVA
 private void writeObject(java.io.ObjectOutputStream s)
      throws java.io.IOException{
      // Write out element count, and any hidden stuff
      int expectedModCount = modCount;
      s.defaultWriteObject();

      // Write out size as capacity for behavioural compatibility with clone()
      s.writeInt(size);

      // Write out all elements in the proper order.
      for (int i=0; i<size; i++) {
          s.writeObject(elementData[i]);
      }

      if (modCount != expectedModCount) {
          throw new ConcurrentModificationException();
      }
  }
```
从实现代码上看，在序列化过程中如果list发生修改，则序列化失败。先把数组的有效长度size放进ObjectOutputStream，然后在循环size长度将数据一个个放入即可。
### LinkedList
1，是一个标准的双向链表，双向就是一个元素可以链接到自己前面的元素也可以链接到后面的元素，如此遍历时是可以从头向尾，也可以从尾向头。
直接看元素的定义：
```JAVA
private static class Node<E> {
      E item;
      Node<E> next;
      Node<E> prev;

      Node(Node<E> prev, E element, Node<E> next) {
          this.item = element;
          this.next = next;
          this.prev = prev;
      }
  }
  ```

  2，LinkedList实现了List和Deque（双端队列）接口。
  3，因为是队列形式实现的列表，所以当要访问（get）某个下标（index）的元素时，是需要进行循环遍历操作的，应该有个比较清晰的认识：  
  ```JAVA
  public E get(int index) {
       checkElementIndex(index);
       return node(index).item;
   }
   /**
  * Returns the (non-null) Node at the specified element index.
  */
 Node<E> node(int index) {
     // assert isElementIndex(index);

     if (index < (size >> 1)) {
         Node<E> x = first;
         // 遍历
         for (int i = 0; i < index; i++)
             x = x.next;
         return x;
     } else {
         Node<E> x = last;
         for (int i = size - 1; i > index; i--)
             x = x.prev;
         return x;
     }
 }
 ```

### Vector
1，也是数组维护，实现和ArrayList相似。
2，操作方法用synchronized修饰：
```JAVA
public synchronized boolean add(E e) {
       modCount++;
       ensureCapacityHelper(elementCount + 1);
       elementData[elementCount++] = e;
       return true;
   }
```
