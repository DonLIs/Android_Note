# Android开发--线程安全的单例模式

*方案一：* 双重加锁模式

```
public class SingleTon{

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

*方案二：* 内部静态类模式

```
public class SingleTon{

  private SingleTon(){}
 
  private static class SingleTonBuilder{
     private static SingleTon instance = new SingleTon();
  }
 
  public static SingleTon getInstance(){
     return SingleTonBuilder.instance;
  }
  
}
```

