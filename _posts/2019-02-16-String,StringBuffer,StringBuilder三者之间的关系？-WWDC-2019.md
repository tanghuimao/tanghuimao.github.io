---
layout:     post
title:      String,StringBuffer,StringBuilder三者之间的关系？
subtitle:   String,StringBuffer,StringBuilder底层是如何实现的呢？
date:       2019-02-16
author:     HM
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - String
    - StringBuffer
    - StringBuilder
---

## - 前言

<font color="#000000" size="3" face="宋体">对于很多初学Java的同学来说String,StringBuffer,StringBuilder三者之间的关系和底层的实现原理，相对来说不是那么容易理解。
那么这篇文章可能会对你有所帮助。</font>

## - String,StringBuffer,StringBuilder

><font color="#FF3030" size="3" face="宋体">关键词：String,StringBuffer,StringBuilder</font>

### - String,StringBuffer,StringBuilder关系？ 

<font color="#000000" size="3" face="宋体">String,StringBuffer,StringBuilder他们三者都是由final所修饰的类（可以查看其源码），通常由final修饰的类是不能被继承，如果对final关键字的特点和含义不明白，可以查看<a href="https://tanghuimao.github.io/2019/02/18/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Java%E4%B8%AD%E7%9A%84final%E5%85%B3%E9%94%AE%E5%AD%97-WWDC-2020/">《深入理解Java中的final关键字》</a>进行了解。</font>

<font color="#000000" size="3" face="宋体"><br/>String,StringBuffer，StringBuilder都是字符序列。</font>

### - String,StringBuffer,StringBuilder为什么都是字符序列？ 
<font color="#000000" size="3" face="宋体">String类他本身底层实现的原理是由<strong>一个final修饰的char类型数组</strong>实现，因此String类为不可变字符序列。</font>

<font color="#000000" size="3" face="宋体">以下为String类源码一部分：</font>

    public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
        /** The value is used for character storage. */
        private final char value[];
        /** Cache the hash code for the string */
        private int hash; // Default to 0
        /** use serialVersionUID from JDK 1.0.2 for interoperability */
        private static final long serialVersionUID = -6849794470754667710L;

<font color="#000000" size="3" face="宋体">StringBuffer和StringBuilder则是继承其同一父类AbstractStringBuilder类，AbstractStringBuilder类底层则是<strong>没有final修饰char类型的数组</strong>实现，因此StringBuffer和StringBuilder为可变字符序列。</font>

<font color="#000000" size="3" face="宋体">以下为StringBuffer和StringBuilder和AbstractStringBuilder的部分源码：</font>
<font color="#000000" size="3" face="宋体"><strong><br/>AbstractStringBuilder</strong>：</font>

    abstract class AbstractStringBuilder implements Appendable, CharSequence {
        /**
        * The value is used for character storage.
        */
        char[] value;

        /**
        * The count is the number of characters used.
        */
        int count;

<font color="#000000" size="3" face="宋体"><strong>StringBuffer</strong>：</font>

    public final class StringBuffer extends AbstractStringBuilder implements java.io.Serializable, CharSequence{
        /**
            * A cache of the last value returned by toString. Cleared
            * whenever the StringBuffer is modified.
            */
        private transient char[] toStringCache;

        /** use serialVersionUID from JDK 1.0.2 for interoperability */
        static final long serialVersionUID = 3388685877147921107L;

<font color="#000000" size="3" face="宋体"><strong>StringBuilder</strong>：</font>

    public final class StringBuilder extends AbstractStringBuilder implements java.io.Serializable, CharSequence{
        /** use serialVersionUID for interoperability */
        static final long serialVersionUID = 4383685877147921099L;

        /**
            * Constructs a string builder with no characters in it and an
            * initial capacity of 16 characters.
            */
        public StringBuilder() {
            super(16);
        }

### - String与StringBuffer区别？StringBuffer与StringBuilder区别？

- <font color="#000000" size="3" face="宋体">string为不可变的字符序列。</font>
- <font color="#000000" size="3" face="宋体">stringbuffer（线程安全），stringbuffer类的所有方法都是由synchronized修饰，因此是线程安全的。</font>
- <font color="#000000" size="3" face="宋体">stringbuilder（非线程安全），效率相对于stringbuffer更高。<font color="#FF3030" size="3" face="宋体">小提示：如果是在方法内部可以直接使用stringbuilder，不会有线程安全的问题。</font>

### - String是不可变的字符序列，怎么理解？

<font color="#000000" size="3" face="宋体">当我们在调用string的concat方法时、首先获取添加字符的长度，将原先的char数组进行复制成新数组（char数组长度+字符长度）由于value被final修饰指向不能改变，返回新的string对象回去。之前的string对象则变成垃圾对象，等待GC。</font>

    public String concat(String str) {
        int otherLen = str.length();
        if (otherLen == 0) {
            return this;
        }
        int len = value.length;
        char buf[] = Arrays.copyOf(value, len + otherLen);
        str.getChars(buf, len);
        return new String(buf, true);
    }

### - StringBuffer和StringBuilder是可变的字符序列，怎么理解？

<font color="#000000" size="3" face="宋体">当我们在调用StringBuffer和StringBuilder的<strong>append</strong>方法时、是直接调用其父类AbstractStringBuilder类的<strong>append</strong>方法，先获取添加字符的长度，将原先的char数组进行复制成新数组（char数组长度+字符长度）将value的指向新数组，返回当前对象。</font>

    public AbstractStringBuilder append(String str) {
        if (str == null)
            return appendNull();
        int len = str.length();
        ensureCapacityInternal(count + len);
        str.getChars(0, len, value, count);
        count += len;
        return this;
    }

    // Documentation in subclasses because of synchro difference
    public AbstractStringBuilder append(StringBuffer sb) {
        if (sb == null)
            return appendNull();
        int len = sb.length();
        ensureCapacityInternal(count + len);
        sb.getChars(0, len, value, count);
        count += len;
        return this;
    }

    /**
        * @since 1.8
        */
    AbstractStringBuilder append(AbstractStringBuilder asb) {
        if (asb == null)
            return appendNull();
        int len = asb.length();
        ensureCapacityInternal(count + len);
        asb.getChars(0, len, value, count);
        count += len;
        return this;
    }


## - 结语

<font color="#000000" size="3" face="宋体">上述内容仅供参考，不足之处望多多指教。</font>


 

