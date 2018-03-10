#### 组合模式
组合(Composite)模式的其它翻译名称也很多，比如合成模式、树模式等等。在《设计模式》一书中给出的定义是：将对象以树形结构组织起来，以达成“部分－整体”的层次结构，使得客户端对单个对象和组合对象的使用具有一致性。

* 单个对象可以理解成树结构中的叶  
* 组合对象可以理解成树结构中的分支  

基本类图：  
<img src="https://raw.githubusercontent.com/dchack/java_read_learn/master/view/组合1.jpg" width="65%" height="65%">  

这里有一点在翻看网上资料上经常提到，就是透明方式和安全方式，说白了透明方式就是把一些操作的方法（如add，remove方法）在component里进行了开放，安全模式就是这些操作的方法不对外开放，而是放入composite里去。  
两者格子利弊显而易见，透明方式对客户端来说不论你是leaf还是composite，都是一样的行为。但是leaf实现操作方法没有意义。安全方式leaf就不用关系操作的方法实现，但是对客户端来说需要判断出是leaf还是composite，才能使用操作行为。


##### 使用建议
当我们发现需求中提现出有层级关系的结构，希望用户在在使用单个对象或组合对象时，都可以统一使用它们的行为。  

代码：  
https://github.com/dchack/water_framwork/blob/master/hope-learn/src/java/com/hope/learn/patterns/composition/Client.java
