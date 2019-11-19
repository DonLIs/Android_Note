# Android开发--工厂方法模式

本文例子来自于《Android源码设计模式解析与实践》书中第五章--应用最广泛的模式--工厂方法模式<br>

```
//抽象类
public abstract class Phone{

  public abstract void email(String address);

  public abstract void playGame(String game);
  
  public abstract void call(String number);
  
  public abstract String getPhone();
  
}
```
```
//实现类A
public class Iphone extends Phone{

  @Override
  public abstract void email(String address){
  
  }

  @Override
  public abstract void playGame(String game){
    
  }
  
  @Override
  public abstract void call(String number){
    
  }
  
  @Override
  public abstract String getPhone(){
    return "Iphone";
  }

}

//实现类B
public class HUAWEI extends Phone{

  @Override
  public abstract void email(String address){
  
  }

  @Override
  public abstract void playGame(String game){
    
  }
  
  @Override
  public abstract void call(String number){
    
  }
  
  @Override
  public abstract String getPhone(){
    return "HUAWEI";
  }

}

//实现类C
public class XIAOMI extends Phone{

  @Override
  public abstract void email(String address){
  
  }

  @Override
  public abstract void playGame(String game){
    
  }
  
  @Override
  public abstract void call(String number){
    
  }
  
  @Override
  public abstract String getPhone(){
    return "XIAOMI";
  }

}
```
```
//工厂类
public class PhoneFactory{

  public static <T extends Phone> T createPhone(Class<T> class){
  
    Phone phone = null;
    
    try{
      phone = (Phone) Class.forName(class.getName()).newInstance();
    }catch(Exception e){
      e.printStackTrace();
    }
    
    return (T) phone;
  }

}
```
