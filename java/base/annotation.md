#### java 世界中Annotation
描述数据的数据  
xml也是做这件事，引入Annotation是因为xml数量增加，可以分散在不同位置，维护对开发编程了负担。但是，事实上引入xml这种描述数据的工具就是为了分离代码和配置。  
所以，如何权衡使用Annotation和xml，也成为了开发需要思考的问题。  
1，如果是描述常量，配置参数的情况下，使用xml是较好的选择  
2，如果配置会被做一些功能类的事情的，需要和特定的代码结合使用的，使用Annotation。
大多数优秀的框架都是结合两者配合使用的，发挥两者的优势。  

#### 分类
1 标准 Annotation  
Override, Deprecated, SuppressWarnings  
标准 Annotation 是指 Java 自带的几个 Annotation，上面三个分别表示重写函数，函数已经被禁止使用，忽略某项 Warning  

2 元 Annotation  
@Retention, @Target, @Inherited, @Documented，元 Annotation 是指用来定义 Annotation 的 Annotation，在后面 Annotation 自定义部分会详细介绍含义  

3 自定义 Annotation  
自定义 Annotation 表示自己根据需要定义的 Annotation，定义时需要用到上面的元 Annotation  

#### 详细
既然元 Annotation 是描述Annotation的 那么标准还是自定义Annotation都是被它描述的。在源码里我们可以看Override是怎么被描述的：

```JAVA
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```
从代码我们看到@Target(ElementType.METHOD)表示这个Annotation用语注解方法的。
@Retention(RetentionPolicy.SOURCE) 则表示解析Annotation的时间点是编译的时候
被描述的注解在什么范围内有效。


RetentionPolicy.RUNTIME
在.java .class文件时存在,加载到内存后还存在了
使用反射机制获取注解内容：
````JAVA
method.getAnnotation(AnnotationName.class);
method.getAnnotations();
method.isAnnotationPresent(AnnotationName.class);
````

RetentionPolicy.CLASS
在.java .class文件时存在，加载到内存后就不存在了
jdk提供里一个apt机制（https://docs.oracle.com/javase/6/docs/technotes/guides/apt/GettingStarted.html#AnnotationProcessor）


RetentionPolicy.SOURCE
在.java时存在 .class文件时就不存在了
所以这个是给java编译器使用的  

总结一下：  
注解作为数据的描述者，可以在java代码从源码到运行的各个阶段被解析出内容，然后依据内容做想做的事。
