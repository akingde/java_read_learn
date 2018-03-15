##### 责任链模式
责任链模式(Chain of Responsibility)使多个对象都有机会处理请求，从而避免请求的发送者和接受者之间的耦合关系。将这些对象连成一条链，并沿着这条链传递该请求，直到有对象能够处理它。  

基本类图：  
<img src="https://raw.githubusercontent.com/dchack/java_read_learn/master/view/责任链模式1.jpg" width="65%" height="65%">  

代码实现如下：
```java
public abstract class Handler {

    /**
     * 持有后继的责任对象
     */
    protected Handler successor;
    /**
     * 示意处理请求的方法，虽然这个示意方法是没有传入参数的
     * 但实际是可以传入参数的，根据具体需要来选择是否传递参数
     */
    public abstract void handleRequest(int n);
    /**
     * 取值方法
     */
    public Handler getSuccessor() {
        return successor;
    }
    /**
     * 赋值方法，设置后继的责任对象
     */
    public void setSuccessor(Handler successor) {
        this.successor = successor;
    }

}

public class ConcreteHandler extends Handler {
    /**
     * 处理方法，调用此方法处理请求
     */
    @Override
    public void handleRequest(int n) {

        if(n>1000){
            /**
             * 判断是否有后继的责任对象
             * 如果有，就转发请求给后继的责任对象
             * 如果没有，则处理请求
             */
            if(getSuccessor() != null)
            {
                System.out.println("提交上级处理");
                getSuccessor().handleRequest(n);
            }else
            {
                System.out.println("上级不在 无法处理");
            }
        }else{
            System.out.println("处理请求");
        }

    }

}


public class DirectorHandler extends Handler{
    @Override
    public void handleRequest(int n) {
        System.out.println("Director 处理");
    }
}
```
client使用：  

```java
public class Client {

    public static void main(String[] args) {
        int n = 1001;
        //组装责任链
        Handler handler1 = new ConcreteHandler();
        Handler handler2 = new DirectorHandler();
        handler1.setSuccessor(handler2);
        //提交请求
        handler1.handleRequest(n);
    }

}
```

责任链的优点就是接受者和请求者进行链解耦，对于handler来说可以自由增减，对于请求者（上面例子是一个数字n）来说并不知道后续执行中有那些接受者参与。  
而对于client端来说可以通过配置不同的接受者得到定制的逻辑实现，而不需要修改接受者的代码。  
但是这个模式下就会引入大量的接受者类需要维护，而配置是一件具有风险的事情，容易出错。  
