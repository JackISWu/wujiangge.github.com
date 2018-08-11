---
title: Design Pattern - Singleton
date: 2018-08-11 18:11:08
tags: "design pattern"
---

单例模式是在一个应用(Application)中某个类(Class)有且仅有一个实例(Instance)存在。

相当于一个全局对象，方便对整个系统的行为进行协调，比如服务器的配置信息由一个单例对象保存。

单例模式又可细分为 懒汉方式(lazy-initialization)和饿汉方式，懒汉方式是只有需要的时候才会去创建，饿汉方式则是在Application初始化或者.class类文件加载阶段直接创建。

###饿汉模式（不存在线程不安全的问题）：
```
public class Singleton {
    private final static Singleton INSTANCE = new Singleton();
    // Private constructor suppresses 
    private Singleton() {}
    // default public constructor
    public final static Singleton getInstance() {
        return INSTANCE;
    }
}
```
`final static`的`INSTANCE`只有在Singleton的成员变量或者函数(非final static literal成员变量)第一次被使用的时候才会触发实例化。在`Singleton`类加载到JVM的时候`INSTANCE`被放到JVM方法区常量池中，其值`new Singleton()`则只是存了`Singleton`类以及constructor方法的符号引用，实例化被JVM推迟。

###一般的懒汉模式（非线程安全）：
```
public class Singleton {   
    private static Singleton INSTANCE = null;   
    private Singleton() {}   
     public static Singleton getInstance() {       
        if(INSTANCE == null){  INSTANCE = new Singleton();  }
    }
    return INSTANCE;   
}
```
试想在单核模式下当两个线程t1和t2一同访问`getInstance()`函数，t1先检查`INSTANCE`是否是`null`之后被挂起，t2检查`INSTANCE`是`null`之后创建`Singleton`的instance并挂起，t1被唤醒再次创建`Singleton`的instance，造成了数据的覆盖。
###一般的懒汉模式（线程安全）：
```
public class Singleton {   
    private static Singleton INSTANCE = null;   
    private Singleton() {}   
     public static synchronized Singleton getInstance() {       
        if(INSTANCE == null){  INSTANCE = new Singleton();  }
    }
    return INSTANCE;   
}
```
直接给`getInstance()`Class级别函数加互斥锁，每个访问该函数的线程都得先拿到该Class的monitor锁。但是这样过多的加锁解锁消耗资源不够efficiency。

##Double Check的懒汉模式（线程安全）：
```
public class Singleton {
    private static volatile Singleton INSTANCE = null;
    // Private constructor suppresses
    private Singleton() {}
    //thread safe and performance  promote
    public static  Singleton getInstance() {
        if(INSTANCE == null){
            synchronized(Singleton.class){
                if(INSTANCE == null){
                    INSTANCE = new Singleton();
                  }
              }
        }
        return INSTANCE;
    }
}
```
相比前一个实现，每次`getInstance`都先检查是否是`null`，如果不是`null`则不执行加锁解锁的操作，所以更高效。

`volatile`关键字的作用是使变量内存可见(visible to memory），防止代码重排序(确保happens-before原则)。

`synchronized`关键字则会为`Singleton.class`加monitor锁，每个调用该段代码的Thread必须先拿到`Singleton.class`的monitor锁才能执行该段代码，而同一时间只能有一个Thread持有该monitor锁。

##静态内部类的实现：
```
public final class Singleton{
    public static class SingletonHandler{
        private final static Singleton INSTANCE=new Singleton();
    }
    public final static Singleton getInstanc(){
        return SingletonHandler.INSTANCE;
    }
}
```
内部类`StingletonHandler`在编译之后会生成一个单独的`.class`文件：`Singleton$SingletonHandler.class`。`Singleton.class`被加载方法区之后保留的只是`SingletonHandler`和`SingletonHandler.INSTANCE`的qualified name(全限定名的符号引用symbolic reference)，只有在调用`getInstance`方法时JVM才会通过classloader将`Singleton$SingletonHandler.class`加载到方法区，在heap区创建`Singleton`的实例，并将符号引用(symbolic reference)转换为直接引用(direct reference)。
以上加载过程都是线程安全的，所以该实现高效&自动线程安全。

**使用场景**：当Singleton这个类有提供其他很多`static`的函数的话，通过`SingletonHandler`可以实现懒加载(仅在需要用到这个instance的时候才去加载内部类并创建`Singleton`的instance)。

##Enum实现：
```
public Enum Singleton{
    INSTANCE;
    //functions to add
}
```
Decompile之后的代码是:

```
public final class Singleton extends Enum
{
    private Singleton(String s, int i)
    {
        super(s, i);
    }

    public static Singleton[] values()
    {
        Singleton asingleton[];
        int i;
        Singleton asingleton1[];
        System.arraycopy(asingleton = ENUM$VALUES, 0, asingleton1 = new Singleton[i = asingleton.length], 0, i);
        return asingleton1;
    }

    public static Singleton valueOf(String s)
    {
        return (Singleton)Enum.valueOf(structure/proxy/Singleton, s);
    }

    public static final Singleton INSTANCE;
    private static final Singleton ENUM$VALUES[];

    static
    {
        INSTANCE = new Singleton("INSTANCE", 0);
        ENUM$VALUES = (new Singleton[] {
            INSTANCE
        });
    }
}
```
和饿汉模式很像，相比而言优点是写法更简单，而且自动提供了Serialisable序列化(其他单例实现的序列化则需手动implements Serialisable接口实现序列化)。

关于JVM类加载流程在这里先简单写一下：
1. loading -> 找.class文件(TYPE)并加载入JVM
2. linking -> 分三部分
2.1. verification -> 检查引入TYPE文件正确性
2.2. preparation -> 给class变量分配内存(在方法区)并赋值default value:(boolean:false, int:0, reference:null)
2.3. resolution -> 将symbolic reference转为direct reference (通常延后触发)
3. initialization -> 触发代码提供的赋值语句(通常延后触发)

欢迎批评指正，谢谢！

#Reference
https://zh.wikipedia.org/wiki/单例模式
https://stackoverflow.com/questions/70689/what-is-an-efficient-way-to-implement-a-singleton-pattern-in-java
https://stackoverflow.com/questions/16771373/singleton-via-enum-way-is-lazy-initialized
