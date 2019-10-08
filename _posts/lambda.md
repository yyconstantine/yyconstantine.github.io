---
layout:     post
title:      Lambda学习
subtitle:   Lambda学习
date:       2019-10-08
author:     yyconstantine
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Java基础
---

### Lambda学习

> Lambda表达式为Java添加了缺失的函数式编程特点，使我们能将函数当作一等公民看待。尽管不完全正确，我们很快就会见识到Lambda与闭包的不同之处，但是又无限地接近闭包。在支持一类函数的语言中，Lambda表达式的类型将是函数。但是在Java中，Lambda表达式是对象，他们必须依附于一类特别的对象类型——函数式接口。

---

#### 表达式简介

Lambda表达式通常使用```(argument) -> (body)```语法书写，如👇

```java
(arg1, arg2...) -> {body}
(type1 arg1, type2 arg2...) -> {body}
```

一些Lambda表达式例子👇

```java
(int a,int b) -> {return a + b}
() -> System.out.println("Hello World");
(String s) -> {System.out.println(s);}
() -> 42
() -> {return 3.1415}
```

---

#### 表达式结构

- 一个Lambda表达式可以有零个或多个参数
- 参数的类型既可以明确声明，也可以根据上下文来推断。例如：```(int a)```与```(a)```效果相同
- 所有参数需包含在圆括号内，参数之间用逗号相隔。例如：```(a,b)```或```(int a, int b)```或```(String a, int b, float c)```
- 空圆括号代表参数集为空。例如```() -> 42```
- 当只有一个参数，且其类型可推导时，圆括号可省略，如```a -> return a*a```
- Lambda表达式的主体可能包含零条或多条语句
- 如果Lambda表达式的主体只有一条语句，花括号可省略。匿名函数的返回类型与该主体表达式一致
- 如果Lambda表达式的主体包含一条以上语句，则表达式必须包含在花括号中（形成代码块）。匿名函数的返回类型与代码块的返回类型一致，若没有返回则为空

---

#### 函数式接口

在Java中，Marker（标记）类型的接口是一种没有方法或属性声明的接口。简单地说，Marker接口是空接口。相似地，函数式接口是只包含一个抽象方法声明的接口。

```java.lang.Runnable```就是一种函数式接口，在Runnable接口中之声明了一个方法```void run()```，相似地，ActionListener接口也是一种函数式接口，我们使用匿名内部类来实例化函数式接口的对象，有了Lambda表达式，这一方式可以得到简化。

每个Lambda表达式都能隐式地赋值给函数式接口，例如，我们可以通过Lambda表达式创建Runnable接口的引用👇

```java
Runnable r = () -> System.out.println("Hello World");
```

当不指明函数式接口时，编译器会自动解释这种转化👇

```java
new Thread(
    () -> System.out.println("Hello World");
).start();
```

因此，在上面的代码中，编译器会自动推断：根据线程类的构造函数签名```public Thread(Runnable r) {}```，将该Lambda表达式赋给Runnable接口。

其他Lambda表达式及其函数式接口：

```java
Consumer<Integer> c = (int x) -> {System.out.println(x)};
BiConsumer<Integer, String> b = (Integer x, String y) -> System.out.println(x + " : " + y);
Predicater<String> p = (String s) -> {s == null};
```

```@FunctionalInterface```是Java8新加入的一种接口，用于指明该接口类型声明是根据Java与语言规范定义的函数式接口。Java8还声明了一些Lambda表达式可以使用的函数式接口，当你注释的接口不是有效的函数式接口时，可以使用```@FunctionalInterface```解决编译层面的错误。

以下是一种自定义的函数式接口👇

```java
@FunctionalInterface
public interface WorkerInterface {
    void doSomeWork();
}
```

定义好函数式接口后，我们就可以使用了👇

```java
public class WorkerInterfaceTest {
    public static void execute(WorkerInterface worker) {
        worker.doSomeWork();
    }
    
    public static void main(String[] args) {
        // old method in java 7
        execute(new WorkerInterface() {
            @Override
            public void doSomeWork() {
                System.out.println("Work in Java 7");
            }
        });
        
        // new method in java 8
        execute( () -> {System.out.println("Work in Java 8")});
    }
}
```

---

#### 应用

- 线程通过Lambda初始化

- PS：Lambda可以视为一个匿名内部类对象，我们创建的Lambda表达式实质是创建一个对象，所以只要我们创建了接口对应的对象，然后将对应传递给指定的方法，即可完成方法的实际调用

  ```java
  // old method
  new Thread(new Runnable() {
      @Override
      public void run() {
          System.out.println("Hello From Java 7");
      }
  });
  
  // new method
  new Thread( () -> System.out.println("Hello From Java 8"); ).start();
  ```

- Swing的事件处理

  ```java
  // old method
  button.addActionListener(new ActionListener() {
      @Override
      public void actionPerformed(ActionEvent e) {
          System.out.println("Using old method click the button");
      }
  });
  
  // new method
  button.addActionListener( (e) -> System.out.println("Using new method click the button"););
  ```

- 打印给定数组中的所有元素

  ```java
  // old method
  List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7);
  for (Integer n : list) {
      System.out.println(n);
  }
  
  // new method
  List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7);
  list.forEach(n -> System.out.println(n));
  
  // another method
  list.forEach(System.out::println);
  ```

- 打印list中每个元素的平方

  ```java
  // old method
  List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7);
  for (Integer n : list) {
      int x = n * n;
      System.out.println(x);
  }
  
  // new method
  List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7);
  list.stream().map((x) -> x*x).forEach(System.out::println);
  ```

---

#### Lambda与匿名内部类

- 对于匿名内部类，关键字```this```解读为匿名类
- 对于Lambda表达式，关键词```this```解读为Lambda的外部类，且Java在编译代码时将Lambda表达式转化为类内的私有函数，它实用Java 7中的```invokedynamic```指令动态绑定该方法

---

> 本文摘录自 >> [http://blog.oneapm.com/apm-tech/226.html](http://blog.oneapm.com/apm-tech/226.html)