# Android开发--解决依赖包冲突方法

1、查看各个依赖包中的情况<br>

```
//在Terminal终端中输入
gradlew -q app:dependencies
```
  
2、在项目的build.gradle文件里<br>

```
subprojects{
  project.configurations.all {
    resolutionStrategy.eachDependency { details ->
        if(details.requested.group == 'androidx.fragment') {//androidx.fragment是指定的包名
            details.useVersion "1.1.0-alpha02" //指定的版本号
        }
    }
}
```
  
