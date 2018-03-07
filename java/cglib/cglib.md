##### 概念
在java的世界里，基于jvm实现的语言最终要进入jvm编译的流程都需要把上层高级语言所表达的内容自行编译成字节码文件，而cglib是一个操作字节码生成自定义类的库，它底层调用的是ASM库来操作字节码的。示意图：  
<img src="https://raw.githubusercontent.com/dchack/java_read_learn/master/view/java-class.jpg" width="65%" height="65%">  

这里主要以使用cglib入口为起点进入它源代码，详细查看内部的实现机制。

##### AbstractClassGenerator
CGLIB 核心类，这个抽象类作为CGLIB中代码生成调度员角色，包含缓存操作，定制ClassLoader，命名策略（NamingPolicy），代码生成策略（GeneratorStrategy）。

##### Enhancer
Enhancer 继承 AbstractClassGenerator
从Enhancer中的crate方法系列开始这场旅行。  
以下三个方法用classOnly参数来控制返回的是Class对象，还是代理对象本身。所以我们可以知道在调用createHelper方法的时候这两个对象是要生成的。

```java
/**
  *
  * 入口方法，产生一个代理对象
  * @return a new instance
  */
 public Object create() {
     classOnly = false;
     argumentTypes = null;
     return createHelper();
 }

 public Object create(Class[] argumentTypes, Object[] arguments) {
     classOnly = false;
     if (argumentTypes == null || arguments == null || argumentTypes.length != arguments.length) {
         throw new IllegalArgumentException("Arguments must be non-null and of equal length");
     }
     this.argumentTypes = argumentTypes;
     this.arguments = arguments;
     return createHelper();
 }

 public Class createClass() {
     classOnly = true;
     return (Class)createHelper();
 }
```
静态的crate方法：  

```java
  public static Object create(Class type, Callback callback) {
      Enhancer e = new Enhancer();
      e.setSuperclass(type);
      e.setCallback(callback);
      return e.create();
  }

  public static Object create(Class superclass, Class interfaces[], Callback callback) {
      Enhancer e = new Enhancer();
      e.setSuperclass(superclass);
      e.setInterfaces(interfaces);
      e.setCallback(callback);
      return e.create();
  }

  public static Object create(Class superclass, Class[] interfaces, CallbackFilter filter, Callback[] callbacks) {
      Enhancer e = new Enhancer();
      e.setSuperclass(superclass);
      e.setInterfaces(interfaces);
      e.setCallbackFilter(filter);
      e.setCallbacks(callbacks);
      return e.create();
  }
```
都是new 一个Enhancer然后进行操作，最后都调用create()方法，提供给外部调用者不同的调用方式，我们可以看到至多需要以下几个元素：
* Class superclass
* Class[] interfaces
* CallbackFilter filter
* Callback[] callbacks

那么最后这些方法在设置好变量后，都会调用到createHelper()方法：

```JAVA
private Object createHelper() {
      // 校验
      preValidate();
      // 生成key
      Object key = KEY_FACTORY.newInstance((superclass != null) ? superclass.getName() : null,
              ReflectUtils.getNames(interfaces),
              filter == ALL_ZERO ? null : new WeakCacheKey<CallbackFilter>(filter),
              callbackTypes,
              useFactory,
              interceptDuringConstruction,
              serialVersionUID);
      this.currentKey = key;
      // 调用AbstractClassGenerator.create(Object)方法
      Object result = super.create(key);
      return result;
  }
```

AbstractClassGenerator.create(Object)方法，这是一个核心模版方法:
```JAVA
protected Object create(Object key) {
    try {
        ClassLoader loader = getClassLoader();
        Map<ClassLoader, ClassLoaderData> cache = CACHE;
        ClassLoaderData data = cache.get(loader);
        if (data == null) {
            synchronized (AbstractClassGenerator.class) {
                cache = CACHE;
                data = cache.get(loader);
                if (data == null) {
                    Map<ClassLoader, ClassLoaderData> newCache = new WeakHashMap<ClassLoader, ClassLoaderData>(cache);
                    data = new ClassLoaderData(loader);
                    newCache.put(loader, data);
                    CACHE = newCache;
                }
            }
        }
        this.key = key;
        // 这里产生class对象 背后有做缓存功能
        Object obj = data.get(this, getUseCache());
        if (obj instanceof Class) {
            // 模版方法 用class对象产生代理对象
            return firstInstance((Class) obj);
        }
        //模版方法
        return nextInstance(obj);
    } catch (RuntimeException e) {
        throw e;
    } catch (Error e) {
        throw e;
    } catch (Exception e) {
        throw new CodeGenerationException(e);
    }
}
abstract protected Object firstInstance(Class type) throws Exception;
abstract protected Object nextInstance(Object instance) throws Exception;
```
我们看到firstInstance 和 nextInstance模版方法给子类实现，
而在进入执行子类模版方法前，代理类的class字节码必然已经组装好了。
```JAVA
Object obj = data.get(this, getUseCache());
```
这行获取class实例或者EnhancerFactoryData，返回的类型也是判断执行firstInstance 或 nextInstance的依据。这里就是组装类字节码，缓存类信息等操作的入口。后续展开

先看一下看Enhancer的两个实例化入口的实现：
```JAVA
protected Object firstInstance(Class type) throws Exception {
   if (classOnly) {
       return type;
   } else {
       return createUsingReflection(type);
   }
}
protected Object nextInstance(Object instance) {
     EnhancerFactoryData data = (EnhancerFactoryData) instance;

     if (classOnly) {
         return data.generatedClass;
     }

     Class[] argumentTypes = this.argumentTypes;
     Object[] arguments = this.arguments;
     if (argumentTypes == null) {
         argumentTypes = Constants.EMPTY_CLASS_ARRAY;
         arguments = null;
     }
     return data.newInstance(argumentTypes, arguments, callbacks);
 }
```
classOnly字段控制返回类型，从代码实现上来看nextInstance是firstInstance的升级版，在firstInstance上的原文注释上我们也可以印证，实际自定义代理类在创建时是不会调用到firstInstance而是调用nextInstance，而nextInstance中是做类一个缓存功能。   
而我们可以查看上面连个方法要实例化出对象最终都会调用到ReflectUtils.newInstance(final Constructor cstruct, final Object[] args)方法。查看方法我们发现是用cstruct.newInstance(args)这行代码来实现的：
```JAVA
public static Object newInstance(final Constructor cstruct, final Object[] args) {

       boolean flag = cstruct.isAccessible();
       try {
           if (!flag) {
               cstruct.setAccessible(true);
           }
           // 最终调用 产生代理对象
           Object result = cstruct.newInstance(args);
           return result;
       } catch (InstantiationException e) {
           throw new CodeGenerationException(e);
       } catch (IllegalAccessException e) {
           throw new CodeGenerationException(e);
       } catch (InvocationTargetException e) {
           throw new CodeGenerationException(e.getTargetException());
       } finally {
           if (!flag) {
               cstruct.setAccessible(flag);
           }
       }

   }
```
传入nextInstance方法的是EnhancerFactoryData实例。 EnhancerFactoryData中保存了以下这些内容：

```JAVA
public final Class generatedClass;
private final Method setThreadCallbacks;
private final Class[] primaryConstructorArgTypes;
private final Constructor primaryConstructor;
```
到这里我们就明白了nextInstance是firstInstance升级版的意义，就是把Constructor缓存起来在每次要实例化时不需要像firstInstance那样调用下面的代码去遍历出Constructor：
```JAVA
public static Constructor getConstructor(Class type, Class[] parameterTypes) {
    try {
        // 这一步就是nextInstance在省略的内容
        Constructor constructor = type.getDeclaredConstructor(parameterTypes);
        constructor.setAccessible(true);
        return constructor;
    } catch (NoSuchMethodException e) {
        throw new CodeGenerationException(e);
    }
}
```
前面提到Object obj = data.get(this, getUseCache());返回组装好字节码的代理类或包装类EnhancerFactoryData，用于实例化代理类。  
调用到AbstractClassGenerator中的内部类ClassLoaderData的方法get：
```JAVA
public Object get(AbstractClassGenerator gen, boolean useCache) {
    if (!useCache) {
      return gen.generate(ClassLoaderData.this);
    } else {
      Object cachedValue = generatedClasses.get(gen);
      return gen.unwrapCachedValue(cachedValue);
    }
}
```
从上面的代码有不使用缓存的逻辑就直接调用AbstractClassGenerator.generate(ClassLoaderData data)方法。看名字就知道这个是核心方法了，而用走缓存分支肯定也是要调这个方法，只是多了一份缓存的逻辑。  
generatedClasses 是一个LoadingCache类，这个LoadingCache是用于存储的设计类：
```JAVA
// 实际存储map
protected final ConcurrentMap<KK, Object> map;
protected final Function<K, V> loader;
// 名字是map 其实是封装了获得kk的算法 获取是调用apply方法
protected final Function<K, KK> keyMapper;
```
Function类是一个函数接口（Functional Interface）：
```JAVA
public interface Function<K, V> {
    V apply(K key);
}
```
可以理解为带某个自定义算法的类，可以传送给其他类使用。  
LoadingCache类中的核心方法get(K key)（其中key后续进行详细分析），最终会调用到createEntry，：
```JAVA
public V get(K key) {
    // 包装key的算法
    final KK cacheKey = keyMapper.apply(key);
    Object v = map.get(cacheKey);
    // 如果是FutureTask 则说明还在创建中，如果不是FutureTask，则说明已经创建好可直接返回
    if (v != null && !(v instanceof FutureTask)) {
        return (V) v;
    }
    return createEntry(key, cacheKey, v);
}
protected V createEntry(final K key, KK cacheKey, Object v) {
     FutureTask<V> task;
     // 标记是一个新建的流程
     boolean creator = false;
     // v有值说明是已经找到在执行的FutureTask
     if (v != null) {
         // Another thread is already loading an instance
         task = (FutureTask<V>) v;
     } else {
         //新建一个FutureTask
         task = new FutureTask<V>(new Callable<V>() {
             public V call() throws Exception {
                 // task执行内容
                 return loader.apply(key);
             }
         });
         // putIfAbsent判断是否已经有值
         Object prevTask = map.putIfAbsent(cacheKey, task);
         // 三种情况
         // 1，没值 则是新放的task 就启动这个task
         // 2，有值 是FutureTask 说明有线程在我执行putIfAbsent之前已经捷足先登了 那就把自己新建的task抛弃掉
         // 3，有值 不是FutureTask 说明已经有task已经执行完成并放入了result 那就直接返回这个resutl即可
         if (prevTask == null) {
             // creator does the load
             creator = true;
             task.run();
         } else if (prevTask instanceof FutureTask) {
             task = (FutureTask<V>) prevTask;
         } else {
             return (V) prevTask;
         }
     }

     V result;
     try {
          // task执行完毕返回值
         result = task.get();
     } catch (InterruptedException e) {
         throw new IllegalStateException("Interrupted while loading cache item", e);
     } catch (ExecutionException e) {
         Throwable cause = e.getCause();
         if (cause instanceof RuntimeException) {
             throw ((RuntimeException) cause);
         }
         throw new IllegalStateException("Unable to load cache item", cause);
     }
     if (creator) {
         // 放缓存
         map.put(cacheKey, result);
     }
     return result;
 }
}
```
以上代码详细解读后发现是这样设计的：
1，用ConcurrentMap存储，先放的value是FutureTask，执行完成后value放执行结果，并保证在FutureTask放入之后，再不能进行替换操作，无论是否执行完毕。
2，利用FutureTask异步获取执行结果的能力把编织字节码的过程异步化，新的线程获取同一个代理类时，因为保证在放入map后的task只执行一次，也就没有并发情况是多个相同代理类的编织消耗了。
下面画了示意图：  
<img src="https://raw.githubusercontent.com/dchack/java_read_learn/master/view/cglib1-1.jpg" width="50%" height="50%">  

这个设计的场景应该是比较常见的，产生一个对象比较消耗，这时候自然会想到把它缓存起来，一般的写法就向下面的代码：  
先组装这个对象，然后放入缓存，放入的时候判断是否已存在。但是这种写法在高并发时一波线程全部同时到达第一步代码，然后都去执行消耗的代码，然后进入第二步的时候就要不断替换，虽然最后的结果可能是正确的，不过会有无谓的浪费。现在再看一下cglib的实现就可以学到了。
```JAVA
Object object = create();//1
map.putIfAbsent(key, obj);//2
```

这里详细再展开下，因为这也是非常值得学习的地方，我们想如果我们并不想用异步的方式呢？以下是一个网上解决方案，很有意思：
```JAVA
public class concurrentMapTest {

    // 记录自旋状态的轻量级类，只封装了一个volatile状态
    public static class SpinStatus{
        volatile boolean released;
    }

    // 辅助并发控制的Map，用来找出每个key对应的第一个成功进入的线程
    private ConcurrentMap<String, SpinStatus> raceUtil = new ConcurrentHashMap<String, SpinStatus>();

    private ConcurrentMap<String, Object> map = new ConcurrentHashMap<String, Object>();

    public Object test(String key){
        Object value = map.get(key);
        // 第一次
        if(value == null){
            // 需要为并发的线程new一个自旋状态，只有第一个成功执行putIfAbsent方法的线程设置的SpinStatus会被共享
            SpinStatus spinStatus = new SpinStatus();
            SpinStatus oldSpinStatus = raceUtil.putIfAbsent(key, spinStatus);
            //只有第一个执行成功的线程拿到的oldSpinStatus是null，其他线程拿到的oldSpinStatus是第一个线程设置的，可以在所有线程中共享
            if(oldSpinStatus == null){
                value = create();
                // 放入共享的并发Map，后续线程执行get()方法后可以直接拿到非null的引用返回
                map.put(key, value);
                // 释放其他自旋的线程,注意，对第一个成功执行的线程使用的是spinStatus的引用
                spinStatus.released = true;
            }else{
                // 其他线程在oldSpinStatus引用所指向的共享自旋状态上自旋，等待被释放
                while(!oldSpinStatus.released){};
            }

            // 再次获取一下，这时候是肯定不为空
            value = map.get(key);
        }
        return value;
    }

    /**
     * 假装耗时代码
     * @return
     */
    public static String create(){
        return "1";
    }
}

```

新建task的代码就是组装代理类的代码，但是这个return loader.apply(key);里的load是调用方传入的，我们看下调用方的代码：
先打开ClassLoaderData的代码，它是AbstractClassGenerator的内部类：

```JAVA
protected static class ClassLoaderData {
       private final Set<String> reservedClassNames = new HashSet<String>();
       private final LoadingCache<AbstractClassGenerator, Object, Object> generatedClasses;
       private final WeakReference<ClassLoader> classLoader;

       private final Predicate uniqueNamePredicate = new Predicate() {
           public boolean evaluate(Object name) {
               return reservedClassNames.contains(name);
           }
       };

       private static final Function<AbstractClassGenerator, Object> GET_KEY = new Function<AbstractClassGenerator, Object>() {
           public Object apply(AbstractClassGenerator gen) {
               return gen.key;
           }
       };

       public ClassLoaderData(ClassLoader classLoader) {
           if (classLoader == null) {
               throw new IllegalArgumentException("classLoader == null is not yet supported");
           }
           this.classLoader = new WeakReference<ClassLoader>(classLoader);
           // 组装load
           Function<AbstractClassGenerator, Object> load =
                   new Function<AbstractClassGenerator, Object>() {
                       public Object apply(AbstractClassGenerator gen) {
                           Class klass = gen.generate(ClassLoaderData.this);
                           return gen.wrapCachedClass(klass);
                       }
                   };
          // 组装LoadingCache代码
           generatedClasses = new LoadingCache<AbstractClassGenerator, Object, Object>(GET_KEY, load);
       }

       public ClassLoader getClassLoader() {
           return classLoader.get();
       }

       public void reserveName(String name) {
           reservedClassNames.add(name);
       }

       public Predicate getUniqueNamePredicate() {
           return uniqueNamePredicate;
       }

       public Object get(AbstractClassGenerator gen, boolean useCache) {
           if (!useCache) {
             return gen.generate(ClassLoaderData.this);
           } else {
             Object cachedValue = generatedClasses.get(gen);
             return gen.unwrapCachedValue(cachedValue);
           }
       }
   }

```
直接取出loader.apply(key);的代码：
```java
public Object apply(AbstractClassGenerator gen) {
    Class klass = gen.generate(ClassLoaderData.this);
    return gen.wrapCachedClass(klass);
}
```
首先调用的是
```java
protected Class generate(ClassLoaderData data) {
    Class gen;
    Object save = CURRENT.get();
    CURRENT.set(this);
    try {
        ClassLoader classLoader = data.getClassLoader();
        if (classLoader == null) {
            throw new IllegalStateException("ClassLoader is null while trying to define class " +
                    getClassName() + ". It seems that the loader has been expired from a weak reference somehow. " +
                    "Please file an issue at cglib's issue tracker.");
        }
        synchronized (classLoader) {
          String name = generateClassName(data.getUniqueNamePredicate());              
          data.reserveName(name);
          this.setClassName(name);
        }
        if (attemptLoad) {
            try {
                gen = classLoader.loadClass(getClassName());
                return gen;
            } catch (ClassNotFoundException e) {
                // ignore
            }
        }
        // 调用strategy的generate
        byte[] b = strategy.generate(this);
        String className = ClassNameReader.getClassName(new ClassReader(b));
        ProtectionDomain protectionDomain = getProtectionDomain();
        synchronized (classLoader) { // just in case
            if (protectionDomain == null) {
                gen = ReflectUtils.defineClass(className, b, classLoader);
            } else {
                gen = ReflectUtils.defineClass(className, b, classLoader, protectionDomain);
            }
        }
        return gen;
    } catch (RuntimeException e) {
        throw e;
    } catch (Error e) {
        throw e;
    } catch (Exception e) {
        throw new CodeGenerationException(e);
    } finally {
        CURRENT.set(save);
    }
}

```
调用strategy的generate，这里就是代码生成策略预留的口子，我们可以通过AbstractClassGenerator.setStrategy(GeneratorStrategy strategy)来设置自定义的strategy，默认是DefaultGeneratorStrategy：
```java
public byte[] generate(ClassGenerator cg) throws Exception {
      // DebuggingClassWriter中DEBUG_LOCATION_PROPERTY可以设置代理类class文件的路径
       DebuggingClassWriter cw = getClassVisitor();
       transform(cg).generateClass(cw);
       return transform(cw.toByteArray());
   }
```
最后还是调用到net.sf.cglib.proxy.Enhancer#generateClass 而这个方法是ClassGenerator接口定义要实现的方法。
因为在Enhancer中已经保存了编织代理类的全部信息，编织过程的入口由自己来实现。这里就不再继续深入研究字节码编织的过程，因为这需要理解asm的api和class文件格式已经jvm编译规范。后续的学习过程中将补全这部分内容。  
那么至此基本写完类以cglib产生代理类的主流程。

#### 反编译例子
以下是一个例子附加了代理类的反编译代码：
被代理类：
```java
public class SampleClass {
    public String test(String input) {
        return "Hello world!";
    }

    public void big(String i){
        System.out.println("1111");
    }


    public int test1(String input) {
        return 1;
    }
}
```
操作类：
```java
public void testMethodInterceptor() throws Exception {
   Enhancer enhancer = new Enhancer();
   enhancer.setSuperclass(SampleClass.class);
   enhancer.setCallback(new MethodInterceptor() {
       @Override
       public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy)
               throws Throwable {
           if(method.getDeclaringClass() != Object.class && method.getReturnType() == String.class) {
               return "Hello cglib!";
           } else {
               return proxy.invokeSuper(obj, args);
           }
       }
   });
   SampleClass proxy = (SampleClass) enhancer.create();
}
```
我们可以通过下面代码的设置将cglib产生的代理类生成到自定义的路径上去，方便自己反编译：
```java
System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "/gitwork/");
```
反编译后代码，我们可以直接看到它继承了SampleClass类，然后在test方法实现的地方做了处理：
```java
package com.hope.learn.third.cglib;

import com.hope.learn.third.cglib.SampleClass;
import java.lang.reflect.Method;
import net.sf.cglib.core.ReflectUtils;
import net.sf.cglib.core.Signature;
import net.sf.cglib.proxy.Callback;
import net.sf.cglib.proxy.Factory;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

public class SampleClass$$EnhancerByCGLIB$$a2b2935d extends SampleClass implements Factory {

   private boolean CGLIB$BOUND;
   public static Object CGLIB$FACTORY_DATA;
   private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
   private static final Callback[] CGLIB$STATIC_CALLBACKS;
   private MethodInterceptor CGLIB$CALLBACK_0;
   private static Object CGLIB$CALLBACK_FILTER;
   private static final Method CGLIB$test$0$Method;
   private static final MethodProxy CGLIB$test$0$Proxy;
   private static final Object[] CGLIB$emptyArgs;
   private static final Method CGLIB$big$1$Method;
   private static final MethodProxy CGLIB$big$1$Proxy;
   private static final Method CGLIB$test1$2$Method;
   private static final MethodProxy CGLIB$test1$2$Proxy;
   private static final Method CGLIB$equals$3$Method;
   private static final MethodProxy CGLIB$equals$3$Proxy;
   private static final Method CGLIB$toString$4$Method;
   private static final MethodProxy CGLIB$toString$4$Proxy;
   private static final Method CGLIB$hashCode$5$Method;
   private static final MethodProxy CGLIB$hashCode$5$Proxy;
   private static final Method CGLIB$clone$6$Method;
   private static final MethodProxy CGLIB$clone$6$Proxy;


   static void CGLIB$STATICHOOK1() {
      CGLIB$THREAD_CALLBACKS = new ThreadLocal();
      CGLIB$emptyArgs = new Object[0];
      Class var0 = Class.forName("com.hope.learn.third.cglib.SampleClass$$EnhancerByCGLIB$$a2b2935d");
      Class var1;
      Method[] var10000 = ReflectUtils.findMethods(new String[]{"test", "(Ljava/lang/String;)Ljava/lang/String;", "big", "(Ljava/lang/String;)V", "test1", "(Ljava/lang/String;)I"}, (var1 = Class.forName("com.hope.learn.third.cglib.SampleClass")).getDeclaredMethods());
      CGLIB$test$0$Method = var10000[0];
      CGLIB$test$0$Proxy = MethodProxy.create(var1, var0, "(Ljava/lang/String;)Ljava/lang/String;", "test", "CGLIB$test$0");
      CGLIB$big$1$Method = var10000[1];
      CGLIB$big$1$Proxy = MethodProxy.create(var1, var0, "(Ljava/lang/String;)V", "big", "CGLIB$big$1");
      CGLIB$test1$2$Method = var10000[2];
      CGLIB$test1$2$Proxy = MethodProxy.create(var1, var0, "(Ljava/lang/String;)I", "test1", "CGLIB$test1$2");
      var10000 = ReflectUtils.findMethods(new String[]{"equals", "(Ljava/lang/Object;)Z", "toString", "()Ljava/lang/String;", "hashCode", "()I", "clone", "()Ljava/lang/Object;"}, (var1 = Class.forName("java.lang.Object")).getDeclaredMethods());
      CGLIB$equals$3$Method = var10000[0];
      CGLIB$equals$3$Proxy = MethodProxy.create(var1, var0, "(Ljava/lang/Object;)Z", "equals", "CGLIB$equals$3");
      CGLIB$toString$4$Method = var10000[1];
      CGLIB$toString$4$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/String;", "toString", "CGLIB$toString$4");
      CGLIB$hashCode$5$Method = var10000[2];
      CGLIB$hashCode$5$Proxy = MethodProxy.create(var1, var0, "()I", "hashCode", "CGLIB$hashCode$5");
      CGLIB$clone$6$Method = var10000[3];
      CGLIB$clone$6$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/Object;", "clone", "CGLIB$clone$6");
   }

   final String CGLIB$test$0(String var1) {
      return super.test(var1);
   }

   public final String test(String var1) {
      MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
      if(this.CGLIB$CALLBACK_0 == null) {
         CGLIB$BIND_CALLBACKS(this);
         var10000 = this.CGLIB$CALLBACK_0;
      }

      return var10000 != null?(String)var10000.intercept(this, CGLIB$test$0$Method, new Object[]{var1}, CGLIB$test$0$Proxy):super.test(var1);
   }

   final void CGLIB$big$1(String var1) {
      super.big(var1);
   }

   public final void big(String var1) {
      MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
      if(this.CGLIB$CALLBACK_0 == null) {
         CGLIB$BIND_CALLBACKS(this);
         var10000 = this.CGLIB$CALLBACK_0;
      }

      if(var10000 != null) {
         var10000.intercept(this, CGLIB$big$1$Method, new Object[]{var1}, CGLIB$big$1$Proxy);
      } else {
         super.big(var1);
      }
   }

   final int CGLIB$test1$2(String var1) {
      return super.test1(var1);
   }

   public final int test1(String var1) {
      MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
      if(this.CGLIB$CALLBACK_0 == null) {
         CGLIB$BIND_CALLBACKS(this);
         var10000 = this.CGLIB$CALLBACK_0;
      }

      if(var10000 != null) {
         Object var2 = var10000.intercept(this, CGLIB$test1$2$Method, new Object[]{var1}, CGLIB$test1$2$Proxy);
         return var2 == null?0:((Number)var2).intValue();
      } else {
         return super.test1(var1);
      }
   }

   final boolean CGLIB$equals$3(Object var1) {
      return super.equals(var1);
   }

   public final boolean equals(Object var1) {
      MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
      if(this.CGLIB$CALLBACK_0 == null) {
         CGLIB$BIND_CALLBACKS(this);
         var10000 = this.CGLIB$CALLBACK_0;
      }

      if(var10000 != null) {
         Object var2 = var10000.intercept(this, CGLIB$equals$3$Method, new Object[]{var1}, CGLIB$equals$3$Proxy);
         return var2 == null?false:((Boolean)var2).booleanValue();
      } else {
         return super.equals(var1);
      }
   }

   final String CGLIB$toString$4() {
      return super.toString();
   }

   public final String toString() {
      MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
      if(this.CGLIB$CALLBACK_0 == null) {
         CGLIB$BIND_CALLBACKS(this);
         var10000 = this.CGLIB$CALLBACK_0;
      }

      return var10000 != null?(String)var10000.intercept(this, CGLIB$toString$4$Method, CGLIB$emptyArgs, CGLIB$toString$4$Proxy):super.toString();
   }

   final int CGLIB$hashCode$5() {
      return super.hashCode();
   }

   public final int hashCode() {
      MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
      if(this.CGLIB$CALLBACK_0 == null) {
         CGLIB$BIND_CALLBACKS(this);
         var10000 = this.CGLIB$CALLBACK_0;
      }

      if(var10000 != null) {
         Object var1 = var10000.intercept(this, CGLIB$hashCode$5$Method, CGLIB$emptyArgs, CGLIB$hashCode$5$Proxy);
         return var1 == null?0:((Number)var1).intValue();
      } else {
         return super.hashCode();
      }
   }

   final Object CGLIB$clone$6() throws CloneNotSupportedException {
      return super.clone();
   }

   protected final Object clone() throws CloneNotSupportedException {
      MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
      if(this.CGLIB$CALLBACK_0 == null) {
         CGLIB$BIND_CALLBACKS(this);
         var10000 = this.CGLIB$CALLBACK_0;
      }

      return var10000 != null?var10000.intercept(this, CGLIB$clone$6$Method, CGLIB$emptyArgs, CGLIB$clone$6$Proxy):super.clone();
   }

   public static MethodProxy CGLIB$findMethodProxy(Signature var0) {
      String var10000 = var0.toString();
      switch(var10000.hashCode()) {
      case -1315810049:
         if(var10000.equals("big(Ljava/lang/String;)V")) {
            return CGLIB$big$1$Proxy;
         }
         break;
      case -508378822:
         if(var10000.equals("clone()Ljava/lang/Object;")) {
            return CGLIB$clone$6$Proxy;
         }
         break;
      case -178329709:
         if(var10000.equals("test(Ljava/lang/String;)Ljava/lang/String;")) {
            return CGLIB$test$0$Proxy;
         }
         break;
      case 992023923:
         if(var10000.equals("test1(Ljava/lang/String;)I")) {
            return CGLIB$test1$2$Proxy;
         }
         break;
      case 1826985398:
         if(var10000.equals("equals(Ljava/lang/Object;)Z")) {
            return CGLIB$equals$3$Proxy;
         }
         break;
      case 1913648695:
         if(var10000.equals("toString()Ljava/lang/String;")) {
            return CGLIB$toString$4$Proxy;
         }
         break;
      case 1984935277:
         if(var10000.equals("hashCode()I")) {
            return CGLIB$hashCode$5$Proxy;
         }
      }

      return null;
   }

   public SampleClass$$EnhancerByCGLIB$$a2b2935d() {
      CGLIB$BIND_CALLBACKS(this);
   }

   public static void CGLIB$SET_THREAD_CALLBACKS(Callback[] var0) {
      CGLIB$THREAD_CALLBACKS.set(var0);
   }

   public static void CGLIB$SET_STATIC_CALLBACKS(Callback[] var0) {
      CGLIB$STATIC_CALLBACKS = var0;
   }

   private static final void CGLIB$BIND_CALLBACKS(Object var0) {
      SampleClass$$EnhancerByCGLIB$$a2b2935d var1 = (SampleClass$$EnhancerByCGLIB$$a2b2935d)var0;
      if(!var1.CGLIB$BOUND) {
         var1.CGLIB$BOUND = true;
         Object var10000 = CGLIB$THREAD_CALLBACKS.get();
         if(var10000 == null) {
            var10000 = CGLIB$STATIC_CALLBACKS;
            if(CGLIB$STATIC_CALLBACKS == null) {
               return;
            }
         }

         var1.CGLIB$CALLBACK_0 = (MethodInterceptor)((Callback[])var10000)[0];
      }

   }

   public Object newInstance(Callback[] var1) {
      CGLIB$SET_THREAD_CALLBACKS(var1);
      SampleClass$$EnhancerByCGLIB$$a2b2935d var10000 = new SampleClass$$EnhancerByCGLIB$$a2b2935d();
      CGLIB$SET_THREAD_CALLBACKS((Callback[])null);
      return var10000;
   }

   public Object newInstance(Callback var1) {
      CGLIB$SET_THREAD_CALLBACKS(new Callback[]{var1});
      SampleClass$$EnhancerByCGLIB$$a2b2935d var10000 = new SampleClass$$EnhancerByCGLIB$$a2b2935d();
      CGLIB$SET_THREAD_CALLBACKS((Callback[])null);
      return var10000;
   }

   public Object newInstance(Class[] var1, Object[] var2, Callback[] var3) {
      CGLIB$SET_THREAD_CALLBACKS(var3);
      SampleClass$$EnhancerByCGLIB$$a2b2935d var10000 = new SampleClass$$EnhancerByCGLIB$$a2b2935d;
      switch(var1.length) {
      case 0:
         var10000.<init>();
         CGLIB$SET_THREAD_CALLBACKS((Callback[])null);
         return var10000;
      default:
         throw new IllegalArgumentException("Constructor not found");
      }
   }

   public Callback getCallback(int var1) {
      CGLIB$BIND_CALLBACKS(this);
      MethodInterceptor var10000;
      switch(var1) {
      case 0:
         var10000 = this.CGLIB$CALLBACK_0;
         break;
      default:
         var10000 = null;
      }

      return var10000;
   }

   public void setCallback(int var1, Callback var2) {
      switch(var1) {
      case 0:
         this.CGLIB$CALLBACK_0 = (MethodInterceptor)var2;
      default:
      }
   }

   public Callback[] getCallbacks() {
      CGLIB$BIND_CALLBACKS(this);
      return new Callback[]{this.CGLIB$CALLBACK_0};
   }

   public void setCallbacks(Callback[] var1) {
      this.CGLIB$CALLBACK_0 = (MethodInterceptor)var1[0];
   }

   static {
      CGLIB$STATICHOOK1();
   }
}
```
