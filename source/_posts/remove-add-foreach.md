---
title: foreach循环中为什么不要进行remove/add操作
date: 2018-05-17 21:16:31
tags: Java 进阶
categories: Java 进阶
---
先来看一段代码，摘自阿里巴巴的java开发手册
``` java
List<String> a = new ArrayList<String>();
  a.add("1");
  a.add("2");
  for (String temp : a) {
      if("1".equals(temp)){
          a.remove(temp);
 } 
 }
```
<!-- more -->
此时执行代码，没有问题，但是需要注意，循环此时只执行了一次。具体过程后面去分析。
再来看一段会出问题的代码
``` java
List<String> a = new ArrayList<String>();
 a.add("1");
 a.add("2");
 for (String temp : a) {
     if("2".equals(temp)){
         a.remove(temp);
} 
}
```
输出为:<font color=red>Exception in thread "main" java.util.ConcurrentModificationException</font>
是不是很奇怪？接下来将class文件，反编译下，结果如下
``` java
 List a = new ArrayList();
  a.add("1");
  a.add("2");
  Iterator i$ = a.iterator();
  do
  {
      if(!i$.hasNext())
          break;
      String temp = (String)i$.next();
     if("1".equals(temp))
         a.remove(temp);
 } while(true);
```
几个需要注意的点：
1. foreach遍历集合，实际上内部使用的是iterator。
2. 代码先判断是否hasNext，然后再去调用next，这两个函数是引起问题的关键。
3. 这里的remove还是list的remove方法。  
 
<font color=red>先去观察下list.remove()方法中的核心方法fastRemove()方法。</font>
```java
private void fastRemove(int index) {
         modCount++;
         int numMoved = size - index - 1;
         if (numMoved > 0)
             System.arraycopy(elementData, index+1, elementData, index,
                              numMoved);
         elementData[--size] = null; // clear to let GC do its work
     }
```
注意下,modCount++,此处先不表，下文再说这个参数。<br><font color=red>顺路观察下list.add()方法</font>
```java
public boolean add(E e) {
         ensureCapacityInternal(size + 1);  // Increments modCount!!
         elementData[size++] = e;
         return true;
     }
```
注意第二行的注释，说明这个方法也会使modCount++
<font color=red>再去观察下，iterator()方法</font>
```java
public Iterator<E> iterator() {
         return new Itr();
  }
```
```java
private class Itr implements Iterator<E> {
          int cursor;       // index of next element to return
          int lastRet = -1; // index of last element returned; -1 if no such
          int expectedModCount = modCount;
  
          public boolean hasNext() {
              return cursor != size;
          }
  
         @SuppressWarnings("unchecked")
         public E next() {
             checkForComodification();//万恶之源
             int i = cursor;
             if (i >= size)
                 throw new NoSuchElementException();
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
 
         final void checkForComodification() {
             if (modCount != expectedModCount)
                 throw new ConcurrentModificationException();
         }
     }
```
几个需要注意的点：
1. 在iterator初始化的时候（也就是for循环开始处），expectedModCount = modCount，猜测是和当时list内部的元素数量有关系(已证实)。
2. 当cursor != size的时候，hasNext返回true
3. next()函数的第一行，checkForComodification()这个函数就是报错的原因 这个函数就是万恶之源
4. 第39行，mod != expectedModCount 就会抛出ConcurrentModificationException()
---
接下来分析文章开头的第一个例子，为啥不会报错？
第一个例子执行完第一次循环后，mod = 3 expectedModCount =2 cursor = 1 size = 1  所以程序在执行hasNext()的时候会返回false，所以程序不会报错。
第二个例子执行完第二次循环后,mod = 3 expectdModCount = 2 cursor = 2 size = 1 此时cursor != size 程序认定还有元素，继续执行循环，调用next方法但是此时mod != expectedModCount 所以此时会报错。
道理我们都懂了，再看一个例子
```java
public static void main(String[] args) throws Exception {
        List<String> a = new ArrayList<String>();
        a.add("1");
        a.add("2");
        for (String temp : a) {
            System.out.println(temp);
            if("2".equals(temp)){
                a.add("3");
                a.remove("2");
            } 
        }
}
 ```
 此时输出为：
1
2
显然，程序并没有执行第三次循环，第二次循环结束，cursor再一次等于size，程序退出循环。
<font color=bule>与remove类似，将文章开头的代码中remove替换为add，我们会发现无论是第一个例子还是第二个例子，都会抛出ConcurrentModificationException错误。</font>
原因同上，代码略。

---
手册上推荐的代码如下
```java
Iterator<String> it = a.iterator(); while(it.hasNext()){
 String temp = it.next(); if(删除元素的条件){
         it.remove();
        }
 }
```
此时remove是iterator的remove，我们看一下它的源码：
```java
public void remove() {
              if (lastRet < 0)
                  throw new IllegalStateException();
              checkForComodification();
  
              try {
                  ArrayList.this.remove(lastRet);
                  cursor = lastRet;   //index of last element returned;-1 if no such
                  lastRet = -1;
                 expectedModCount = modCount;
             } catch (IndexOutOfBoundsException ex) {
                 throw new ConcurrentModificationException();
             }
         }
```
注意第10行，第8行，所以此时程序不会有之前的问题。
但是手册上推荐的方法，在多线程环境还是有可能出现问题，一个线程执行上面的代码，一个线程遍历迭代器中的元素，同样会抛出CocurrentModificationException。
如果要并发操作，需要对iterator对象加锁。

---
平时遍历list，然后删除某个元素的时候，如果仅仅删除第一个且删除之后调用break  //代表着此时不会再去执行iterator.next方法 也就不会触发万恶之源
而如果要删除所有的某元素，则会报错，谨记！
Ps再来看一个佐证
```java
public static void main(String[] args) {
            ArrayList<Integer> list = new ArrayList<>();
            list.add(1);
            list.add(2);
            list.add(3);
            for(int i : list){
                System.out.println(i);
                if(i == 2){
                    list.remove((Object)2);
                }
            }

        }
```
此时只会输出
1
2
当把remove对象改为3时候，再次报错。
