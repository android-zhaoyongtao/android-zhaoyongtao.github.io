---
layout:     post
title:      ArrayList源码分析
subtitle:   基于jdk-1.8
date:       2018-09-14
author:     ZYT
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Android
    - 源码解析
---

# 类及继承关系


```
public class ArrayList<E> extends AbstractList<E> 
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

 implemnts RandomAccess代表了其拥有随机快速访问的能力,可以根据下标快速访问元素

    ArrayList底层是数组,会占据连续的内存空间(数组的长度>真正数据的长度)， 数组的缺点:空间效率不高。 

    当添加元素，超出现有数组容量时，便会进行扩容操作:

    ArrayList的增长因子1,扩容1/2

    
```
public void ensureCapacity(int minCapacity) {//设定初始容量的构造函数
        int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)        
           // any size if not default element table 
           ?0   
           // larger than default for default empty table. It's already        
           // supposed to be at default size.        
           : DEFAULT_CAPACITY;
        if (minCapacity > minExpand) {
            ensureExplicitCapacity(minCapacity);
        }}

    //以下三个方法实现了Array扩容的操作
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;//扩容必定导致此值++
        // overflow-conscious code
        if (minCapacity - elementData.length > 0)//目标长度>现有长途条件下
            grow(minCapacity);
    }

    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);//扩容容量为旧容量的1.5倍,增长因子为1/2
        if (newCapacity - minCapacity < 0)//如果之后容量还是不够用
            newCapacity = minCapacity;//扩容到目标容量
        if (newCapacity - MAX_ARRAY_SIZE > 0)//判断超限
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);//此方法最总会调用System.ArrayCopy()
    }

    public static <T> T[] copyOf(T[] original, int newLength) {
        return (T[]) copyOf(original, newLength, original.getClass());
    }

    //System类中的静态方法,这是一个native方法.
    //通过此方法实现复制出新数组,实现数组的扩容(其是对对象引用的复制-浅度复制)
    public static void arraycopy(Object src,
                             int srcPos,
                             Object dest,
                             int destPos,
                             int length)
```


# 成员变量

int modCount: 每次数组结构改变都会modCount++(增加导致扩容，或者删)

private int size; 集合数据的长度-list.size()获取的值

transient Object[] elementData;真正的存储东西的元素数组. 长度会>= size

    修饰词:transient，不参与序列化。 一般数组的容量会大于所存元素的个数，所以没必要全部参与序列化,浪费多余空间.在writeObject()中序列化真正存储数据的元素就可以了.

# 增,删,改,查

数组的结构改变标识为modCount的值改变了

`System.arraycopy()`是一个native的数组之间复制的操作,由集合中的grow(minCapacity) 方法控制

下面是ArrayList的增删改查方法里,☝两个关键数据的变化的情况

## 1.增---public boolean add(E e)

    `modCount++
    if (minCapacity - elementData.length > 0)`----如果新集合长度>现在数组长度 会`grow(minCapacity)`;

## 2.删---remove(Object o)

    `modCount++
    grow(minCapacity);`

## 3. 改---set(int index, E element)

    两个操作都没发生

## 4.查---contains(Object o) ,indexOf(Object o)

    两个操作都没发生

# 迭代器


```
private class Itr implements Iterator<E> {
    int cursor;       // index of next element to return
    int lastRet = -1; // index of last element returned; -1 if no such
    int expectedModCount = modCount;//将modCount的值记录下来
    public boolean hasNext() {
        return cursor != size;
    }
    @SuppressWarnings("unchecked")
    public E next() {
        checkForComodification();//每次都会先检查是否发生过结构改变(modCount值是否相同)
        int i = cursor;
        if (i >= size)            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i + 1;
        return (E) elementData[lastRet = i];
    }
    public void remove() {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();
        try {
            ArrayList.this.remove(lastRet);
            cursor = lastRet;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
    @Override
    @SuppressWarnings("unchecked")
    public void forEachRemaining(Consumer<? super E> consumer) {
        Objects.requireNonNull(consumer);
        final int size = ArrayList.this.size;
        int i = cursor;
        if (i >= size) {
            return;
        }
        final Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length) {
            throw new ConcurrentModificationException();
        }
        while (i != size && modCount == expectedModCount) {
            consumer.accept((E) elementData[i++]);
        }
        // update once at end of iteration to reduce heap write traffic
        cursor = i;
        lastRet = i - 1;
        checkForComodification();
    }
    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }}
```

# 总结

    增,删,改,查等每次都会先检查是否索引越界
    增删数组肯定会修改modCount的值
    System.arrayCopy()发生条件:a集合增加导致扩容,b删除元素。而集合的改查不会发生数组拷贝
    结合安卓加载数据使用场景, 列表展示常用，查是高频操作，增删较少，所以ArrayList最合适 



