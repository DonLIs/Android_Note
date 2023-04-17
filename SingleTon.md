# Android开发--线程安全的单例模式

## Java

### 饿汉模式

```
public class SingleTon {

    private static SingleTon instance = new SingleTon();

    private SingleTon() {
    }

    public static SingleTon getInstance() {
        return instance;
    }

}
```

### 懒汉模式

```
public class SingleTon {

    private static SingleTon instance;

    private SingleTon() {
    }

    public synchronized static SingleTon getInstance() {
        if (instance == null) {
            instance = new SingleTon();
        }
        return instance;
    }
}
```

### 双重校验锁模式

```
public class SingleTon {

  private volatile static SingleTon instance;
  
  private SingleTon(){}
  
  public static SingleTon getInstance(){
    if(instance == null){
      synchronized(SingleTon.class){
        if(instance == null){ 
          instance = new SingleTon();
        } 
      }  
    }
    return instance;
  }
  
}
```

### 静态内部类模式

```
public class SingleTon {

  private SingleTon(){}
 
  private static class SingleTonBuilder{
     private static SingleTon instance = new SingleTon();
  }
 
  public static SingleTon getInstance(){
     return SingleTonBuilder.instance;
  }
  
}
```

### CAS

```
public class SingleTon {

    private static final AtomicReference<SingleTon> INSTANCE = new AtomicReference<SingleTon>();

    private SingleTon() {
    }

    public static final SingleTon getInstance() {
        while (true) {
            SingleTon instance = INSTANCE.get();
            if (null == instance) {
                INSTANCE.compareAndSet(null, new SingleTon());
            }
            return INSTANCE.get();
        }
    }
}
```

### 枚举

```
public enum SingleTon {
    //定义一个枚举代表单例
    INSTANCE;
}
```


## Kotlin

### 饿汉模式

```
object Singleton
```

### 懒汉模式

```
class Singleton private constructor() {
 
    companion object {
        private var instance: Singleton? = null
            get() {
                if (field == null) field = Singleton()
                return field
            }
 
        @Synchronized
        fun instance(): Singleton {
            return instance!!
        }
    }
}
```

### 双重校验锁模式

```
class Singleton private constructor() {
    companion object {
        val instance by lazy { Singleton() }
    }
}
```

### 静态内部类模式

```
class Singleton private constructor() {
    companion object {
        val instance = SingletonHolder.holder
    }
 
    private object SingletonHolder {
        val holder = Singleton()
    }
}
```

### 枚举

```
enum class Singleton {
    INSTANCE;
}
```
