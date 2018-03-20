
###### catch异常的顺序
catch 的异常需要按照从子类到父类的顺序写，也是比较好理解按照代码的执行顺序。如果父类型的异常先被catch，后面的子类型异常都没机会被catch。

###### 异常被吃掉
在依靠日志排查问题时遇到异常被吃掉的代码是比较烦人的事情，特别是没有主意到异常被吃掉的情况更是很浪费排查时间。业务开发时的最佳实践就是在catch内都留日志。

###### 异常优化
throw exception 在一般情况下对性能没什么影响，但是如果是比较极端的情况，大量异常同时被触发，或者代码中有很多利用异常来判定业务流程的情况，就需要考虑优化的手段。  
一般的，都是重载Throwable.fillInStackTrace方法返回this
stack trace(线程栈信息)。
```java
public class MyException extends Exception {
	public MyException(String message) {
		super(message);
	}
	/*
	 * 重写fillInStackTrace方法会使得这个自定义的异常不会收集线程的整个异常栈信息，会大大
	 * 提高减少异常开销。
	 */
	@Override
	public synchronized Throwable fillInStackTrace() {
		return this;
	}
	public static void main(String[] args) {
		try {
			throw new MyException("由于MyException重写了fillInStackTrace方法，那么它不会收集线程运行栈信息。");
		} catch (MyException e) {
			e.printStackTrace(); // 在控制台的打印结果为：demo.blog.java.exception.MyException: 由于MyException重写了fillInStackTrace方法，那么它不会收集线程运行栈信息。
		}
	}
}
```

在jdk7后Throwable对象中增加了一个新的构造器

protected Throwable(String message, Throwable cause,
                    boolean enableSuppression,
                    boolean writableStackTrace)   
第三个参数表示是否启用suppressedExceptions(try代码快中抛出异常，在finally中又抛出异常，导致try中的异常丢失) 。  

第四个参数表示是否填充异常栈，如果为false，异常在初始化的时候不会调用本地方法fillInStackTrace。   

在业务开发中，我们会使用很多异常来表示不同的业务限制，比如用户余额不足、用户权限不够、参数不合法，这样的异常是不需要填充栈信息的
