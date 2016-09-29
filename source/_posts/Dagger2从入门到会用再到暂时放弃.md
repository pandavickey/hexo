---
title: Dagger2从入门到会用再到暂时放弃
date: 2016-09-29 15:06:07
tags:
---
Dagger2是一种依赖注入框架，目前由Google维护。说到依赖注入，标准定义是目标类中所依赖的其他的类的初始化过程，不是通过手动编码的方式创建，而是通过技术手段可以把其他的类的已经初始化好的实例自动注入到目标类中。说简单就是一次构建，到处注入。目前只能理解这个定义，具体的应用场景坑还没踩够，还不能意会。

这两天也看了不少Dagger2的Blog,大部分都翻译自[tasting-dagger-2-on-android](http://fernandocejas.com/2015/04/11/tasting-dagger-2-on-android/) 这篇文章再修修改改，参考的工程主要是[Android-CleanArchitecture](https://github.com/android10/Android-CleanArchitecture) 关于Clean框架用Dagger实现以及[官方](https://github.com/google/dagger)咖啡机的故事。主要实现的代码总结起来是以下几行代码：
```
public class Hello {
    @Inject Hello() {  //Inject注解构造函数
    }
    public void say() {
        System.out.println(“hello world”);
    }
}
```
```
@Component  //Component组件绑定宿主
public interface HelloComponent {
    void inject(HelloDagger dagger);
}
```
```
public class HelloDagger {
    @Inject
    static Hello hello;//声明注入依赖

    public static void main(String[] args) {
        DaggerHelloComponent.create().inject(new HelloDagger());//Dagger默认生成DaggerHelloComponent类，调用inject方法注入对象
        hello.say();  //对象已经初始化了
    }
}
```
这是最简单的一个Dagger2的实例，当然Hello类是无参的构造函数，如果有参构造的话还需要引入module和provides两个注解，代码如下：
```
public class Hello {
    private String hello;
    @Inject Hello(String hello) {  //有参构造函数
        this.hello = hello;
    }
    public void say() {
        System.out.println(hello);
    }
}
```
```
@Module
public class HelloModule {
    String hello = "hello";
    public HelloModule(String hello) {
        this.hello = hello;
    }
    @Provides Hello getHello(){  //Provides注解返回依赖的对象
        return new Hello(hello);
    }
}
```
```
@Component(modules = HelloModule.class)  //绑定依赖对象和宿主
public interface HelloComponent {
    void inject(HelloDagger dagger);
}
```
```
public class HelloDagger {
    @Inject
    static Hello hello;
    public static void main(String[] args) {
        DaggerHelloComponent.builder()
                .helloModule(new HelloModule("hello world"))  //创建module对象
                .build()
                .inject(new HelloDagger());//注入
        hello.say();
    }
}
```
比较常用的就是@Inject、@Module、@Provides以及@Component注解，inject在宿主中声明依赖对象，module和provides配合初始化依赖对象，component连接彼此，完成注入过程。整个过程就是一个A a = new A()的过程，只是彼此分离，还是一个解耦的目的。

在构建代码的过程中我们可能还感受不到依赖注入的好处，当程序在膨胀，业务不断累积的过程中，我们发现我们很难遵守面向对象六大原则之依赖倒置原则。如果我们使用了Dagger，当依赖对象发生改变时，我们的Inject和component模块都不需要改变，只需要改变module模块，而这一模块相对独立，变动起来真的随心所欲。

Dagger2终归还是一个模块解耦的利器，坑还没踩够，所以体会不到真正的好处，只能暂时放弃了。就像Fernando Cejas说的，Dagger2的好处主要有三点：
- Since dependencies can be injected and configured externally we can reuse those components.
- When injecting abstractions as collaborators, we can just change the implementation of any object without having to make a lot of changes in our codebase, since that object instantiation resides in one place isolated and decoupled.
- Dependencies can be injected into a component: it is possible to inject mock implementations of these dependencies which makes testing easier.

最后大神镇楼
![Fernando Cejas.png](/img/Fernando Cejas.png)