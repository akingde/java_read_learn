### 概念
序列化，就是将对象的状态信息转换为可以存储或传输的形式的过程，反序列化，就是将对象状态信息恢复回来。
### Serializable
java世界里，类继承Serializable接口，就标记了该类可以被序列化和反序列化。  
序列化的过程也很清晰：   
* 在序列化过程中，如果被序列化的类中定义了writeObject 和 readObject 方法，虚拟机会试图调用对象类里的 writeObject 和 readObject 方法，进行用户自定义的序列化和反序列化。用户自定义的 writeObject 和 readObject 方法可以允许用户控制序列化的过程，比如可以在序列化的过程中动态改变序列化的数值。在[HashMap](https://github.com/dchack/java_read_learn/blob/master/java/Java%20Collections%20Framework/Map.md#hashmap-linkedhashmap)，[ArrayList](https://github.com/dchack/java_read_learn/blob/master/java/Java%20Collections%20Framework/List.md#arraylist)中都有自定义的序列化规则实现，以确保更好的利用空间提高性能。

* 如果没有这样的方法，则默认调用是 ObjectOutputStream 的 defaultWriteObject 方法以及 ObjectInputStream 的 defaultReadObject 方法。


#### 静态变量
在序列化过程中不会以对象状态的信息保存，这个是比较好理解的，静态变量是属于类的数据，而不属于对象状态信息。  
* 测试代码：
```JAVA
public class WriteReadTest {

    public static void main(String[] args){
        Person person = new Person();
        person.setAge(25);
        person.setName("YXY");
        person.setType("not"); //modify the static value
        Person sPerson = writerThenRead(person);
        System.out.println(sPerson.getAge());//25
        System.out.println(sPerson.getType());//yes
    }

    public static Person writerThenRead(Person person) {
        //serialize
        File file = new File("/Users/dongchao/dc_file/test.txt");
        try {
            OutputStream out = new FileOutputStream(file);
            ObjectOutputStream objout = new ObjectOutputStream(out);
            objout.writeObject(person);
            objout.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        person.setType("yes"); //modify the static value
        //deserializ
        Person perobj = null;
        try {
            InputStream in = new FileInputStream(file);
            ObjectInputStream objin = new ObjectInputStream(in);
            perobj = (Person)objin.readObject();
            in.close();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return perobj;
    }

}
```
type是static修饰的，首先赋值no，然后进行了序列化，而此时这个信息是不存入我们序列化的文件的，因为在反序列化前又把它改成了yes，最后打印出来的就是yes。其实这个静态变量是存在栈中的数据，不参与序列化，引用也是直接引用栈的数据，所以改成什么就引用到什么。
#### Transient 关键字
变量被transient修饰，变量将不再是对象序列化的一部分，该变量内容在序列化后无法获得访问。
测试代码：
```JAVA
public static void main(String[] args) {
        Person person = new Person();
        person.setCardNo("3306111111");
        Person sPerson = WriteReadTest.writerThenRead(person);
        System.out.println(sPerson.getCardNo());//null
    }
```
字段cardNo在这里尽管设置了，但在序列化反序列化后是无法传输的。

#### Externalizable 接口
代码：
```JAVA
public interface Externalizable extends java.io.Serializable {

    void writeExternal(ObjectOutput out) throws IOException;

    void readExternal(ObjectInput in) throws IOException, ClassNotFoundException;
}
```
继承Externalizable可以明确要增加一些自定义的序列化规则，实现writeExternal和readExternal方法，操作ObjectOutput和ObjectInput。  
不过不常用，因为我们实现writeObject 和 readObject 方法也是可以达到效果的。

#### 其他序列化库
* Kryo
* FST
* Hessian
* Json
* Xml
* Protostuff
* ProtoBuf
* Thrift
* Avro
* MsgPack
