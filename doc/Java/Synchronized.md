# 一文读懂 Synchronized
在Java中线程问题的主要诱因：
- 存在共享数据
- 存在多条线程共同操动作这些数据

 synchronized 关键字的引入就是为了解决这样的问题，其解决问题的根本方法是**同一时刻有且仅有一个线程在操作共享数据，其他线程必须等到该线程处理完数据后再对共享数据进行操作。**

 通过这个描述可以说 **synchronized 是一把互斥锁**，互斥锁主要有两个特性互斥性和可见性。互斥性是指同一时间只允许一个线程持有某个对象锁；可见性是指必须确保在锁被释放前，对共享变量所做的修改，对随后获得该锁的另一个线程是可见的。

同时，需要注意 **synchronized 锁的不是代码，锁的是对象。** 那么 synchronized 就可以分为两种情况使用，**对象锁和类锁**。因为在Java中类其实对应了一个class对象，其本质还是对象锁。

### 获取对象锁的两种用法
1. 同步代码块
```java
synchronized(this){
    // do somethings;
}
synchronized(object){
    // do somethings;
}
```

2. 同步非静态方法
```java
public synchronized void sayHello(){
    System.out.print("hello");
}
```

### 获取类锁的两种用法
1. 同步代码块
```java
synchronized(Demo.class)){
    // do somethings;
}
```
2. 同步静态方法
```java
public synchronized static void sayHello(){
    System.out.print("hello");
}
```
### 各种锁干扰情况
1. 有线程访问对线果断同步代码块时，另外的线程可以访问该对象的非同步代码块。
2. 若锁住的是同一个对象，一个线程访问对象的同步代码块时，另一个线程访问对象的同步代码块会被阻塞。
3. 若锁住的是同一个对象，一个线程访问对象的同步方法时，另一个线程访问对线的同步方法会被阻塞。
4. 若锁住的是同一个对象，一个线程访问对象的同步代码块时，另一个线程访问对象的同步方法会被阻塞，反之亦然。
5. 同一个类的不同对象的对象锁互不干扰。
6. 类锁由于也是一种特殊的对象锁，因此表现和上述1，2，3，4一致。但是由于一个类只有一个class对象，所以同一个类的不同对象使用类所将会是同步的。
7. 类所和对象锁互不干扰。