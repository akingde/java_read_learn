### 概念
序列化，就是将对象的状态信息转换为可以存储或传输的形式的过程，反序列化，就是将对象状态信息恢复回来。
### Serializable
java世界里，类继承Serializable接口，就标记了该类可以被序列化和反序列化。  
序列化的过程也很清晰：   
* 在序列化过程中，如果被序列化的类中定义了writeObject 和 readObject 方法，虚拟机会试图调用对象类里的 writeObject 和 readObject 方法，进行用户自定义的序列化和反序列化。用户自定义的 writeObject 和 readObject 方法可以允许用户控制序列化的过程，比如可以在序列化的过程中动态改变序列化的数值。

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
        Person.TYPE="no"; //modify the static value

        //serialize
        File file = new File("/test.txt");
        try {
            OutputStream out = new FileOutputStream(file);
            ObjectOutputStream objout = new ObjectOutputStream(out);
            objout.writeObject(person);
            objout.close();
        } catch (IOException e) {
            e.printStackTrace();
        }

        //deserialize
        Person perobj = null;
        try {
            InputStream in = new FileInputStream(file);
            ObjectInputStream objin = new ObjectInputStream(in);
            perobj = (Person)objin.readObject();
            System.out.println(perobj.TYPE);
            in.close();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```
