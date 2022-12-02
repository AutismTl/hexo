---
title: Java注解-Annotation
date: 2017-02-06
tags: Java
---
-------

## 使用
```java
@Override
public void onCreate(...){...}

@GET("api/getuser/{username}")
User getUser(@Path("username") String username);

@BindView(R.id.btn_getuser)
Button getUser;
....
```

## 概念以及作用
1. 概念:能够添加到Java源代码的语法元数据。类，方法，变量，参数，包都可以被注解，可以用来将信息元数据与程序元素进行关联。
2. 作用:
- 标记，告诉编译器一些信息。
- 编译时动态处理，如动态生成代码
- 运行时动态处理，如得到注解信息
<!--more-->

## 原理
Annotation其实是一种接口。通过Java的反射机制相关的API来访问Annotation信息。相关类（框架或工具中的类即使用注解的类）根据这些信息来决定如何使用该程序元素或改变它们的行为。
 
Annoation和程序代码的隔离性：Annotation是不会影响程序代码的执行，无论Annotation怎么变化，代码都始终如一地执行。

忽略性：Java语言解释器在工作时会忽略这些annotation，因此在JVM 中这些Annotation是“不起作用”的，只能通过配套的工具才能对这些Annontaion类型的信息进行访问和处理。


## 分类
1.标准Annotation:Java自带的几个Annotation
- Override 重写方法
- Deprecated 不鼓励使用（有更好方式，存在风险，不再维护）
- SuppressWarnings 忽略某项Warning

2.元Annotation:元注解的作用就是负责注解其他注解
- @Rerention 保留时间(即:被描述的注解在什么范围内有效)，可选SOURCE(源码时) CLASS(编译时) RUNTIME(运行时）默认CLASS，SOURCE 大都为Mark Annotation（下面会说）这类Annotation大都用来校验，比如Override
- @Target 用于描述注解的使用范围(即：被描述的注解可以用在什么地方)。取值(ElementType)有：
　　　　1.CONSTRUCTOR:用于描述构造器
　　　　2.FIELD:用于描述域即类成员变量
　　　　3.LOCAL_VARIABLE:用于描述局部变量
　　　　4.METHOD:用于描述方法
　　　　5.PACKAGE:用于描述包
　　　　6.PARAMETER:用于描述参数
　　　　7.TYPE:用于描述类、接口(包括注解类型) 或enum声明
- @Inherited 是一个标记注解，@Inherited阐述了某个被标注的类型是被继承的。如果一个使用了@Inherited修饰的annotation类型被用于一个class，则这个annotation将被用于该class的子类。
- @Documented 也是一个标记注解,代表是否会保存到JavaDoc中

3.自定义Annotation:自己根据需要定义的Annotation,定义时需要用上面的元Annotation

## 自定义Annotation
1.定义
```java
@Ducumented
@Retention（RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Inherited
public  @Interface MethodInfo{

    String title() default "tl";
    String date();
    String content();
}
```
- 通过@interface定义
- 所有方法没有方法体 没有参数和修饰符 默认public
- 方法返回值只有基本类型，String，Class，annotation，enumeration或者它们的一维数组
- 一个属性都没有表示该 Annotation 为 Mark Annotation
- 可以用default表示默认值
- 只有一个属性时，最好定义为value，因为使用时可以省略"value="

2.使用
```java
public class Test{
   @MethodInfo(title = "123",
               date = "20160912",
               content = "456745645625")
    public Notice getNotice(）{
         return new Notice();
    }
}
```
## Annotation解析
1.运行时Annotation解析（@Retention为RUNTIME)
可以用以下方法：
- method/field/class.getAnnotations(); 得到该Target某个Annotation的信息
- method/field/class.getAnnotation(AnnotationName.class); 表示得到该Target所有Annotation
- method/field/class.isAnnotationPresent(AnnotationName.class); 表示该Target是否被某个Annotation修饰

实例:
```java
try {
            Class cls=Class.forName("com.tl.aidl.Test");
            for(Method method:cls.getMethods()){
                MethodInfo methodInfo=method.getAnnotation(MethodInfo.class);
                if(methodInfo!=null){
                    Log.i(TAG,method.getName());
                    Log.i(TAG,methodInfo.title());
                    Log.i(TAG,methodInfo.date());
                    Log.i(TAG,methodInfo.content());
                }
            }
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }

```

2.编译时Annotation解析（@Retention为CLASS）
这部分编译器会自动解析，不过我们需要定义一个类继承自AbstractProcessor并重写process方法。编译器就会在编译时自动查找所有继承自AbstractProcessor的类，然后调用它们的process方法处理。

实例:
```java
//表示要处理的Annotation名称
@SupportedAnnotationTypes({ "com.tl.java.MethodInfo" })
public class MethodInfoProcessor extends AbstractProcessor {
   // annotations-待处理的Annotations
   // env-运行环境
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) {
        HashMap<String, String> map = new HashMap<String, String>();
        for (TypeElement te : annotations) {
            for (Element element : env.getElementsAnnotatedWith(te)) {
                MethodInfo methodInfo = element.getAnnotation(MethodInfo.class);
                map.put(element.getEnclosingElement().toString(), methodInfo.author());
            }
        }
        
        //表示这组annotation是否被这个Processor接受，如果接受后续的processor不会再对这个Annotations进行处理
        return false;
    }
}
```