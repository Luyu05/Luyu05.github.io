---
title: Top-K
date: 2018-09-26 18:58:23
tags: Algorithm
categories: Algorithm
---
Outline
讨论了几种解决top-k问题的方法
<!--more-->
前言：
之前觉着一旦掌握了某些通性通法就不用去单独刷什么算法题，殊不知很多最优秀的解题思路单凭自己闭门造车是很难相出来的，所以还是要去多刷题（和高三一样），刷题-思考-答案-消化-思考-提升
题目：
int[] arr = new int[] {5,3,7,1,8,2,9,4,7,2,6,6}，从中找出top5的数字
### <font color=red size=5>排序解决</font>
利用排序算法，比如快排，对arr进行排序，之后找到前5的数字。
这种方法是最先想到的方法，时间复杂度为O(N*logN)
伪代码:
```java
quickSort(arr);
return arr[0,4];
```
时间复杂度O(NlogN)
### <font color=red size=5>初级排序解决</font>
再次审题，我们只需要top5的数字，无须对所有数组中的元素进行排序，所以可以使用初级排序算法，比如冒泡排序、选择排序
伪代码：
```java
for(int i=0;i<4;i++){
    findMaxAndSwap(arr);
}
return arr[0,4];
```
时间复杂度O(k*N)
### <font color=red size=5>堆排序解决</font>
数组堆解决这个问题，先令arr的前5个元素堆有序，解析来遍历后面的元素，如果大于堆中的最小值（根节点）进行swap，之后再使堆变得有序
伪代码
```java
heap[4] = make_heap(arr[0, 4]);
for(i=5 to n){
        if(arr[i] > arr[0]) {
            swap(arr,0,i);
            adjust_heap(heep[4]);
        } 
}
return heap[4];
```
时间复杂度O(N*logK)
### <font color=red size=5>随机选择解决</font>
说白了，还是利用快排解决这个问题
伪代码
```java
int RS(arr, low, high, k){
  if(low== high) return arr[low];
  i= partition(arr, low, high);
  temp= i-low; //数组前半部分元素个数
  if(temp>=k)
      return RS(arr, low, i-1, k); //求前半部分第k大
  else
      return RS(arr, i+1, high, k-i); //求后半部分第k-i大
}
```
时间复杂度O(N)

题外话：
<code>分治法</code>，每个分支“都要”递归，例如：快速排序，O(n*lg(n))
<code>减治法</code>，“只要”递归一个分支，例如：二分查找O(lg(n))，随机选择O(n)
### <font color=red size=5>bitMap解决</font>
空间换时间
5,3,7,1,8,2,9,4,7,2,6,6
```java
int max = findMax(arr);
int[] res = new int[max+1];
for(int i = 0;i<arr.length;i++) {
    res[arr[i]]++;
}
int cur = 0;
for(int j=res.length-1;j>0;j--) {
    for(int i= 0;i<res[j];i++) {
        arr[cur++] = j-1;
        if(cur >= 3) break;
    }
    if(cur >= 3) break;
}
return arr[k];
```
