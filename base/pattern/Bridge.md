[git链接](https://github.com/dchack/java_read_learn/blob/master/base/pattern/Bridge.md)
##### 桥接模式
桥梁模式的用意是"将抽象化(Abstraction)与实现化(Implementation)脱耦，使得二者可以独立地变化"。这句话有三个关键词，也就是抽象化、实现化和脱耦。
* 抽象化
存在于多个实体中的共同的概念性联系，就是抽象化。作为一个过程，抽象化就是忽略一些信息，从而把不同的实体当做同样的实体对待。
* 实现化
抽象化给出的具体实现，就是实现化。
* 脱耦
所谓耦合，就是两个实体的行为的某种强关联。而将它们的强关联去掉，就是耦合的解脱，或称脱耦。在这里，脱耦是指将抽象化和实现化之间的耦合解脱开，或者说是将它们之间的强关联改换成弱关联。

桥梁模式中的所谓脱耦，就是指在一个软件系统的抽象化和实现化之间使用组合/聚合关系而不是继承关系，从而使两者可以相对独立地变化。

例子：
一个杯子，它有形状，有颜色，我们得到下面图的关系，虽然我们单独把颜色和形状抽象出来了，担当我们新出现一种颜色的时候，我们需要新增出和形状的实现进行组合的全部类。  
<img src="https://raw.githubusercontent.com/dchack/java_read_learn/master/view/桥接模式1.jpg" width="65%" height="65%">  

这个是提供的桥接模式的解决方案的类图：  
<img src="https://raw.githubusercontent.com/dchack/java_read_learn/master/view/桥接模式2.jpg" width="65%" height="65%">  

解释地址：https://www.journaldev.com/1491/bridge-design-pattern-java

因为前面提到在颜色和形状的上面是一个杯子的抽象，所以我想在实际使用中我们更多的可能是，将多个不同类型的实现部分抽象后，以组合的关系进行关联。  
<img src="https://raw.githubusercontent.com/dchack/java_read_learn/master/view/桥接模式3.jpg" width="65%" height="65%">  

我们可以看见shape 和 color （实现）可以扩展，cup的种类（抽象）也可以扩展。
