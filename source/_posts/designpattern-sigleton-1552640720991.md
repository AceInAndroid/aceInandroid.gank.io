---
title: 设计模式（二）——单例模式
tags:
  - 设计模式
  - 单例模式
categories:
  - 设计模式
date: 2019-03-15 17:05:20
---

# 设计模式（二）——单例模式

本文主要介绍单例设计模式。包括单例的概念、用途、实现方式、如何防止被序列化破坏等。

## 概念
单例模式（Singleton Pattern）是 Java 中最简单的设计模式之一。这种类型的设计模式属于创建型模式。在 GOF 书中给出的定义为：保证一个类仅有一个实例，并提供一个访问它的全局访问点。

单例模式一般体现在类声明中，单例的类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。

## 用途
单例模式有以下两个优点：

**在内存里只有一个实例，减少了内存的开销，尤其是频繁的创建和销毁实例（比如网站首页页面缓存）。**

避免对资源的多重占用（比如写文件操作）。

有时候，我们在选择使用单例模式的时候，不仅仅考虑到其带来的优点，还有可能是有些场景就必须要单例。比如类似”一个党只能有一个主席”的情况。

## 实现方式
我们知道，一个类的对象的产生是由类构造函数来完成的。如果一个类对外提供了`public`的构造方法，那么外界就可以任意创建该类的对象。所以，如果想限制对象的产生，一个办法就是将构造函数变为私有的(至少是受保护的)，使外面的类不能通过引用来产生对象。同时为了保证类的可用性，就必须提供一个自己的对象以及访问这个对象的静态方法。

我们知道，一个类的对象的产生是由类构造函数来完成的。如果一个类对外提供了`public`的构造方法，那么外界就可以任意创建该类的对象。所以，如果想限制对象的产生，一个办法就是将构造函数变为私有的(至少是受保护的)，使外面的类不能通过引用来产生对象。同时为了保证类的可用性，就必须提供一个自己的对象以及访问这个对象的静态方法。


![](https://ws1.sinaimg.cn/large/007lnl1egy1g13ixshwuyj309106gt90.jpg)

### 饿汉式
下面是一个简单的单例的实现：


```java
//code 1
public class Singleton {
    //在类内部实例化一个实例
    private static Singleton instance = new Singleton();
    //私有的构造函数,外部无法访问
    private Singleton() {
    }
    //对外提供获取实例的静态方法
    public static Singleton getInstance() {
        return instance;
    }
}
```

使用以下代码测试：


```JAVA
//code2
public class SingletonClient {

    public static void main(String[] args) {
        SimpleSingleton simpleSingleton1 = SimpleSingleton.getInstance();
        SimpleSingleton simpleSingleton2 = SimpleSingleton.getInstance();
        System.out.println(simpleSingleton1==simpleSingleton2);
    }
}
```
输出结果：

> true


code 1就是一个简单的单例的实现，这种实现方式我们称之为饿汉式。所谓饿汉。这是个比较形象的比喻。对于一个饿汉来说，他希望他想要用到这个实例的时候就能够立即拿到，而不需要任何等待时间。所以，通过static的静态初始化方式，在该类第一次被加载的时候，就有一个SimpleSingleton的实例被创建出来了。这样就保证在第一次想要使用该对象时，他已经被初始化好了。

同时，由于该实例在类被加载的时候就创建出来了，所以也避免了线程安全问题。
### 还有一种饿汉模式的变种：

```JAVA
//code 3
public class Singleton2 {
    //在类内部定义
    private static Singleton2 instance;
    static {
        //实例化该实例
        instance = new Singleton2();
    }
    //私有的构造函数,外部无法访问
    private Singleton2() {
    }
    //对外提供获取实例的静态方法
    public static Singleton2 getInstance() {
        return instance;
    }
}
```

code 3和code 1其实是一样的，都是在类被加载的时候实例化一个对象。

**饿汉式单例，在类被加载的时候对象就会实例化。这也许会造成不必要的消耗，因为有可能这个实例根本就不会被用到。而且，如果这个类被多次加载的话也会造成多次实例化。其实解决这个问题的方式有很多，下面提供两种解决方式，第一种是使用静态内部类的形式。第二种是使用懒汉式。**

### 静态内部类式
先来看通过静态内部类的方式解决上面的问题：
```java
//code 4
public class StaticInnerClassSingleton {
    //在静态内部类中初始化实例对象
    private static class SingletonHolder {
        private static final StaticInnerClassSingleton INSTANCE = new StaticInnerClassSingleton();
    }
    //私有的构造方法
    private StaticInnerClassSingleton() {
    }
    //对外提供获取实例的静态方法
    public static final StaticInnerClassSingleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}
```
这种方式同样利用了`classloder`的机制来保证初始化`instance`时只有一个线程，它跟饿汉式不同的是（很细微的差别）：饿汉式是只要`Singleton`类被装载了，那么`instance`就会被实例化（没有达到lazy loading效果），而这种方式是Singleton类被装载了，`instance`不一定被初始化。因为`SingletonHolder`类没有被主动使用，只有显示通过调用`getInstance`方法时，才会显示装载`SingletonHolder`类，从而实例化`instance`。想象一下，如果实例化`instance`很消耗资源，我想让他延迟加载，另外一方面，我不希望在`Singleton`类加载时就实例化，因为我不能确保`Singleton`类还可能在其他的地方被主动使用从而被加载，那么这个时候实例化`instance`显然是不合适的。这个时候，这种方式相比饿汉式更加合理。

### 懒汉式
下面看另外一种在该对象真正被使用的时候才会实例化的单例模式——懒汉模式。
```java
//code 5
public class Singleton {
    //定义实例
    private static Singleton instance;
    //私有构造方法
    private Singleton(){}
    //对外提供获取实例的静态方法
    public static Singleton getInstance() {
        //在对象被使用的时候才实例化
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```
上面这种单例叫做懒汉式单例。懒汉，就是不会提前把实例创建出来，将类对自己的实例化延迟到第一次被引用的时候。`getInstance`方法的作用是希望该对象在第一次被使用的时候被new出来。

有没有发现，其实code 5这种懒汉式单例其实还存在一个问题，那就是线程安全问题。在多线程情况下，有可能两个线程同时进入if语句中，这样，在两个线程都从if中退出的时候就创建了两个不一样的对象。（这里就不详细讲解了，不理解的请恶补多线程知识）。

线程安全的懒汉式
针对线程不安全的懒汉式的单例，其实解决方式很简单，就是给创建对象的步骤加锁：
```java
//code 6
public class SynchronizedSingleton {
    //定义实例
    private static SynchronizedSingleton instance;
    //私有构造方法
    private SynchronizedSingleton(){}
    //对外提供获取实例的静态方法,对该方法加锁
    public static synchronized SynchronizedSingleton getInstance() {
        //在对象被使用的时候才实例化
        if (instance == null) {
            instance = new SynchronizedSingleton();
        }
        return instance;
    }
}
```
这种写法能够在多线程中很好的工作，而且看起来它也具备很好的延迟加载，但是，遗憾的是，他效率很低，因为99%情况下不需要同步。（因为上面的`synchronized`的加锁范围是整个方法，该方法的所有操作都是同步进行的，但是对于非第一次创建对象的情况，也就是没有进入if语句中的情况，根本不需要同步操作，可以直接返回`instance`。）

### 双重校验锁
针对上面code 6存在的问题，相信对并发编程了解的同学都知道如何解决。其实上面的代码存在的问题主要是锁的范围太大了。只要缩小锁的范围就可以了。那么如何缩小锁的范围呢？相比于同步方法，同步代码块的加锁范围更小。code 6可以改造成：
```java
//code 7
public class Singleton {

    private static Singleton singleton;

    private Singleton() {
    }

    public static Singleton getSingleton() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```
code 7是对于code 6的一种改进写法，通过使用同步代码块的方式减小了锁的范围。这样可以大大提高效率。（对于已经存在`singleton`的情况，无须同步，直接`return`）。

但是，事情这的有这么容易吗？上面的代码看上去好像是没有任何问题。实现了惰性初始化，解决了同步问题，还减小了锁的范围，提高了效率。但是，该代码还存在隐患。隐患的原因主要和Java内存模型（JMM）有关。考虑下面的事件序列：

> 线程A发现变量没有被初始化, 然后它获取锁并开始变量的初始化。

> 由于某些编程语言的语义，编译器生成的代码允许在线程A执行完变量的初始化之前，更新变量并将其指向部分初始化的对象。

> 线程B发现共享变量已经被初始化，并返回变量。由于线程B确信变量已被初始化，它没有获取锁。如果在A完成初始化之前共享变量对B可见（这是由于A没有完成初始化或者因为一些初始化的值还没有穿过B使用的内存(缓存一致性)），程序很可能会崩溃。

（上面的例子不太能理解的同学，请恶补JAVA内存模型相关知识）

在[J2SE 1.4](https://zh.wikipedia.org/wiki/Java_SE)或更早的版本中使用双重检查锁有潜在的危险，有时会正常工作（区分正确实现和有小问题的实现是很困难的。取决于编译器，线程的调度和其他并发系统活动，不正确的实现双重检查锁导致的异常结果可能会间歇性出现。重现异常是十分困难的。） 在[J2SE 5.0](https://zh.wikipedia.org/wiki/Java_SE)中，这一问题被修正了。[volatile](https://zh.wikipedia.org/wiki/Volatile%E5%8F%98%E9%87%8F)关键字保证多个线程可以正确处理单件实例

所以，针对code 7 ，可以有code 8 和code 9两种替代方案：

### 使用volatile的双重校验锁
```java
//code 8
public class VolatileSingleton {
    private static volatile VolatileSingleton singleton;

    private VolatileSingleton() {
    }

    public static VolatileSingleton getSingleton() {
        if (singleton == null) {
            synchronized (VolatileSingleton.class) {
                if (singleton == null) {
                    singleton = new VolatileSingleton();
                }
            }
        }
        return singleton;
    }
}
```
上面这种双重校验锁的方式用的比较广泛，他解决了前面提到的所有问题。但是，即使是这种看上去完美无缺的方式也可能存在问题，那就是遇到序列化的时候。详细内容后文介绍。

### 使用final的双重校验锁
```java
//code 9
class FinalWrapper<T> {
    public final T value;

    public FinalWrapper(T value) {
        this.value = value;
    }
}

public class FinalSingleton {
    private FinalWrapper<FinalSingleton> helperWrapper = null;

    public FinalSingleton getHelper() {
        FinalWrapper<FinalSingleton> wrapper = helperWrapper;

        if (wrapper == null) {
            synchronized (this) {
                if (helperWrapper == null) {
                    helperWrapper = new FinalWrapper<FinalSingleton>(new FinalSingleton());
                }
                wrapper = helperWrapper;
            }
        }
        return wrapper.value;
    }
}
```
### 枚举式
在1.5之前，实现单例一般只有以上几种办法，在1.5之后，还有另外一种实现单例的方式，那就是使用枚举：
```java
// code 10
public enum  Singleton {

    INSTANCE;
    Singleton() {
    }
}
```
**这种方式是Effective Java作者Josh Bloch 提倡的方式，它不仅能避免多线程同步问题，而且还能防止反序列化重新创建新的对象（下面会介绍），可谓是很坚强的壁垒啊，** 不过，个人认为由于1.5中才加入enum特性，用这种方式写不免让人感觉生疏，在实际工作中，我也很少看见有人这么写过，但是不代表他不好。

### 单例与序列化

但是，单例模式真的能够实现实例的唯一性吗？

答案是否定的，很多人都知道使用反射可以破坏单例模式，除了反射以外，使用序列化与反序列化也同样会破坏单例。

#### 序列化对单例的破坏


首先来写一个单例的类：

code 11


```JAVA
/**
 * 使用双重校验锁方式实现单例
 */
public class Singleton implements Serializable{
    private volatile static Singleton singleton;
    private Singleton (){}
    public static Singleton getSingleton() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```


接下来是一个测试类：
code 12
```java
public class SerializableDemo1 {
    //为了便于理解，忽略关闭流操作及删除文件操作。真正编码时千万不要忘记
    //Exception直接抛出
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        //Write Obj to file
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("tempFile"));
        oos.writeObject(Singleton.getSingleton());
        //Read Obj from file
        File file = new File("tempFile");
        ObjectInputStream ois =  new ObjectInputStream(new FileInputStream(file));
        Singleton newInstance = (Singleton) ois.readObject();
        //判断是否是同一个对象
        System.out.println(newInstance == Singleton.getSingleton());
    }
}
//false
```
输出结构为false，说明：

> 通过对Singleton的序列化与反序列化得到的对象是一个新的对象，这就破坏了Singleton的单例性。

这里，在介绍如何解决这个问题之前，我们先来深入分析一下，为什么会这样？在反序列化的过程中到底发生了什么。


对象的序列化过程通过`ObjectOutputStream`和`ObjectInputputStream`来实现的，那么带着刚刚的问题，分析一下`ObjectInputputStream` 的`readObject` 方法执行情况到底是怎样的。

为了节省篇幅，这里给出ObjectInputStream的readObject的调用栈：
图侵删

![](https://ws1.sinaimg.cn/large/007lnl1egy1g13kimgeg1j30l30g8tao.jpg)

这里看一下重点代码，readOrdinaryObject方法的代码片段： code 13

```java
private Object readOrdinaryObject(boolean unshared)
        throws IOException
    {
        //此处省略部分代码

        Object obj;
        //这里创建的这个obj对象，就是本方法要返回的对象
        //也可以暂时理解为是ObjectInputStream的readObject返回的对象。
        //isInstantiable：如果一个serializable/externalizable的类可以在运行时被实例化，
        //那么该方法就返回true。
        //desc.newInstance：该方法通过反射的方式调用无参构造方法新建一个对象。
        try {
            obj = desc.isInstantiable() ? desc.newInstance() : null; 
        } catch (Exception ex) {
            throw (IOException) new InvalidClassException(
                desc.forClass().getName(),
                "unable to create instance").initCause(ex);
        }

        //此处省略部分代码

        if (obj != null &&
            handles.lookupException(passHandle) == null &&
            desc.hasReadResolveMethod())
        {
            Object rep = desc.invokeReadResolve(obj);
            if (unshared && rep.getClass().isArray()) {
                rep = cloneArray(rep);
            }
            if (rep != obj) {
                handles.setObject(passHandle, obj = rep);
            }
        }

        return obj;
```
所以。到目前为止，也就可以解释，为什么序列化可以破坏单例了？

> 序列化会通过反射调用无参数的构造方法创建一个新的对象

#### 如何防止序列化/反序列化破坏单例模式。
防止序列化破坏单例模式
先给出解决方案，然后再具体分析原理：

只要在Singleton类中定义`readResolve`就可以解决该问题：

```java
public class Singleton implements Serializable{
    private volatile static Singleton singleton;
    private Singleton (){}
    public static Singleton getSingleton() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }

    private Object readResolve() {
        return singleton;
    }
}
```
还是运行以下测试类：

```java
public class SerializableDemo1 {
    //为了便于理解，忽略关闭流操作及删除文件操作。真正编码时千万不要忘记
    //Exception直接抛出
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        //Write Obj to file
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("tempFile"));
        oos.writeObject(Singleton.getSingleton());
        //Read Obj from file
        File file = new File("tempFile");
        ObjectInputStream ois =  new ObjectInputStream(new FileInputStream(file));
        Singleton newInstance = (Singleton) ois.readObject();
        //判断是否是同一个对象
        System.out.println(newInstance == Singleton.getSingleton());
    }
}
//true
```
本次输出结果为true。具体原理，我们回过头继续分析code 13中的第二段代码:


code 13.2
```java
private Object readOrdinaryObject(boolean unshared)
        throws IOException
    {
        //此处省略部分代码

        //这里创建的这个obj对象，就是本方法要返回的对象
        //desc.newInstance：该方法通过反射的方式调用无参构造方法新建一个对像
            obj = desc.isInstantiable() ? desc.newInstance() : null; 
   ......
        //此处省略部分代码

        if (obj != null &&
            handles.lookupException(passHandle) == null &&
            desc.hasReadResolveMethod())
        {
            Object rep = desc.invokeReadResolve(obj);
            if (unshared && rep.getClass().isArray()) {
                rep = cloneArray(rep);
            }
            if (rep != obj) {
                handles.setObject(passHandle, obj = rep);
            }
        }

        return obj;
```


hasReadResolveMethod:如果实现了serializable 或者 externalizable接口的类中包含readResolve则返回true

invokeReadResolve:通过反射的方式调用要被反序列化的类的readResolve方法。

所以，原理也就清楚了，主要在Singleton中定义readResolve方法，并在该方法中指定要返回的对象的生成策略，就可以防止单例被破坏。


## 总结
本文中介绍了几种实现单例的方法，主要包括饿汉、懒汉、使用静态内部类、双重校验锁、枚举等。还介绍了如何防止序列化破坏类的单例性。

从单例的实现中，我们可以发现，一个简单的单例模式就能涉及到这么多知识。在不断完善的过程中可以了解并运用到更多的知识。所谓学无止境。