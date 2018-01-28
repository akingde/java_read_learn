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

### ArrayDeque
1，就是用数组实现一个Deque（双端队列），实现了Deque要求的方法。和linkedList一样作为链表，这就简单了下标加减就天然连接在一起了。

2，实现中，规定这个数组的长度保持2的幂次，当调用add相关的方法时，在数组放满触发扩容操作。
因为是数组实现，头尾必然需要一个下标进行记录：
```java
  transient Object[] elements;
/**
  * The index of the element at the head of the deque (which is the
  * element that would be removed by remove() or pop()); or an
  * arbitrary number equal to tail if the deque is empty.
  */
 transient int head;

 /**
  * The index at which the next element would be added to the tail
  * of the deque (via addLast(E), add(E), or push(E)).
  */
 transient int tail;
```
所以说一个常规的add操作只要在数组下标tail的位置上放数据即可。然后判断一下是不是数组放满了，扩容的操作方法也可以得到印证，两倍（doubleCapacity方法）的倍数不断增加。
```java
public void addLast(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[tail] = e;
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)
        doubleCapacity();
}

/**
   * Doubles the capacity of this deque.  Call only when full, i.e.,
   * when head and tail have wrapped around to become equal.
   */
  private void doubleCapacity() {
      assert head == tail;
      int p = head;
      int n = elements.length;
      int r = n - p; // number of elements to the right of p
      int newCapacity = n << 1;
      if (newCapacity < 0)
          throw new IllegalStateException("Sorry, deque too big");
      Object[] a = new Object[newCapacity];
      System.arraycopy(elements, p, a, 0, r);
      System.arraycopy(elements, 0, a, r, p);
      elements = a;
      head = 0;
      tail = n;
  }
```
从上面的代码可以注意到，判断是否需要扩容的依据是(tail + 1) & (elements.length - 1) == head，
elements.length始终是2的幂次，比如是8，elements.length - 1 == 0000 0111，假设设时候tail也到了数组的length==8，那么tail + 1==9 == 0000 1001，所以，
(tail + 1) & (elements.length - 1) == 0000 1001 & 0000 0111 == 0000 0001 == 1，判断依据是 1==head。head这个下标是可以向前移动的，比如调用了poll方法。所以这个数组在head和first的下标标记下，是一个头尾不相连的环结构。  
要用数组实现一个环，可以判断head 和 tail是不是到达elements.length 如果到达就设置成0，ArrayBlockingQueue有这样的实现，而这里的实现利用了长度是2的幂次的约定，通过&来进行判断，也是一个巧妙的技巧。
另外在扩容的代码中我们看见数组copy需要调用两次，一次是拷贝head 到 elements.length的元素，二次是拷贝0到head的元素。
在addFirst代码中也有相似的判断：
```java
public void addFirst(E e) {
       if (e == null)
           throw new NullPointerException();
       elements[head = (head - 1) & (elements.length - 1)] = e;
       if (head == tail)
           doubleCapacity();
   }
```
poll的操作代码也是一样，利用和elements.length - 1进行&操作达到下标在有限数组内循环：
```java
public E pollFirst() {
    int h = head;
    @SuppressWarnings("unchecked")
    E result = (E) elements[h];
    // Element is null if deque empty
    if (result == null)
        return null;
    elements[h] = null;     // Must null out slot
    head = (h + 1) & (elements.length - 1);
    return result;
}
```
3，java.util.Queue 和 java.util.Deque  
* java.util.Queue 定义了FIFO的单向队列
* java.util.Deque 继承Queue, 定义了双向队列 可以FIFO 和 LIFO
