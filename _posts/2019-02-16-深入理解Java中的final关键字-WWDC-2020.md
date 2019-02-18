---
layout:     post
title:      深入理解Java中的final关键字
subtitle:   final关键字的作用及含义
date:       2019-02-18
author:     HM
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - final
---

## - 前言

<font color="#000000" size="4" face="宋体">Java中的final关键字非常重要，它可以修饰于类、方法以及变量。一说到final关键字，相信大家都会立刻想起一些基本的作用，那么我们先稍微用寥寥数行来回顾一下。</font>

## - final

><font color="#FF3030" size="4" face="宋体">关键词：final</font>

### - final关键字的含义？

<font color="#000000" size="3" face="宋体">final在Java中是一个保留的关键字，可以声明成员变量、方法、类以及本地变量。一旦你将引用声明作final，你将不能改变这个引用了，编译器会检查代码，如果你试图将变量再次初始化的话，编译器会报编译错误。</font>

### - final修饰变量
<font color="#000000" size="3" face="宋体">final修饰在成员变量或者局部变量上，那么我们可以称这个变量是final变量。final变量经常和static关键字一起使用，作为常量。下面是final变量的例子：</font>


    public static final int i = 0;
    i = 1;//invalid compilation error

<font color="#000000" size="3" face="宋体">当已经用final关键字修饰了变量，如果我们将被final修饰的变量重新赋值，编译器就会报出如图：<font color="#FF3030" size="4" face="黑体">cannot assign a value to final variable.</font>(不能给final变量赋值)。</font>

### - final修饰方法

<font color="#000000" size="3" face="宋体">被final所修饰的方法将无法被子类重写</font>

**<font color="#000000" size="3" face="宋体">“使用final方法的原因有两个。第一个原因是把方法锁定，以防任何继承类修改它的含义；第二个原因是效率。在早期的Java实现版本中，会将final方法转为内嵌调用。但是如果方法过于庞大，可能看不到内嵌调用带来的任何性能提升。在最近的Java版本中，不需要使用final方法进行这些优化了。” -- 摘自《Java编程思想》</font>**

<font color="#000000" size="3" face="宋体">当一个方法的功能已经足够完整了，子类中不需要改变的话，可以用final修饰此方法。final修饰的方法比普通方法要快，因为在编译的时候已经静态绑定了，不需要在运行时再动态绑定。下面是final方法的例子：</font>

    class Person{
    public final String getName(){
            return "personal loan";
        }
    }
    
    class loan extends Person{
        @Override
            public final Stirng getName(){
                    return "cheap personal loan";//compilation error: overridden method is final
            }
    }

### - final修饰类

<font color="#000000" size="3" face="宋体">使用final来修饰的类叫作final类。final类通常功能是完整的，<strong>它们不能被继承，也就没有子类</strong> 。<font color="#FF3030" size="3" face="宋体">并且final类的所有方法都会被隐式的修饰成final</font>。Java中有许多类是final的，譬如String, Interger以及其他包装类。</font>

### - final关键字的好处

<font color="#000000" size="3" face="宋体">下面总结了一些使用final关键字的好处</font>

- <font color="#000000" size="3" face="宋体">final关键字提高了性能。JVM和Java应用都会缓存final变量。</font>
- <font color="#000000" size="3" face="宋体">final变量可以安全的在多线程环境下进行共享，而不需要额外的同步开销。</font>
- <font color="#000000" size="3" face="宋体">使用final关键字，JVM会对方法、变量及类进行优化。</font>

## - 结语

<font color="#000000" size="3" face="宋体">上述内容仅供参考，不足之处望多多指教。</font>


