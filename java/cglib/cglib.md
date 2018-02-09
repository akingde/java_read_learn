##### AbstractClassGenerator
CGLIB 核心类，这个抽象类作为CGLIB中代码生成调度员角色，做了缓存，定制ClassLoader，命名。
它定义了代理类生成的过程

##### Enhancer
Enhancer 继承 AbstractClassGenerator
从Enhancer中的crate方法系列开始这场旅行。  
以下三个方法用classOnly参数来控制返回的是Class对象，还是代理对象本身。所以我们可以知道在调用createHelper方法的时候这两个对象是要生成的。
```JAVA
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
```JAVA
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

AbstractClassGenerator.create(Object)方法:
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
        // 这一步就是nextInstance在省略的
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
protected final Function<K, KK> keyMapper;
```
Function类是一个函数接口（Functional Interface）：
```JAVA
public interface Function<K, V> {
    V apply(K key);
}
```



先打开ClassLoaderData的代码，它是AbstractClassGenerator的内部类：

```JAVA
protected static class ClassLoaderData {
       private final Set<String> reservedClassNames = new HashSet<String>();

       /**
        * {@link AbstractClassGenerator} here holds "cache key" (e.g. {@link net.sf.cglib.proxy.Enhancer}
        * configuration), and the value is the generated class plus some additional values
        * (see {@link #unwrapCachedValue(Object)}.
        * <p>The generated classes can be reused as long as their classloader is reachable.</p>
        * <p>Note: the only way to access a class is to find it through generatedClasses cache, thus
        * the key should not expire as long as the class itself is alive (its classloader is alive).</p>
        */
       private final LoadingCache<AbstractClassGenerator, Object, Object> generatedClasses;

       /**
        * Note: ClassLoaderData object is stored as a value of {@code WeakHashMap<ClassLoader, ...>} thus
        * this classLoader reference should be weak otherwise it would make classLoader strongly reachable
        * and alive forever.
        * Reference queue is not required since the cleanup is handled by {@link WeakHashMap}.
        */
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
           Function<AbstractClassGenerator, Object> load =
                   new Function<AbstractClassGenerator, Object>() {
                       public Object apply(AbstractClassGenerator gen) {
                           Class klass = gen.generate(ClassLoaderData.this);
                           return gen.wrapCachedClass(klass);
                       }
                   };
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

ClassLoaderData 构造方法中
