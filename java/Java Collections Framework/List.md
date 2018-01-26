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
