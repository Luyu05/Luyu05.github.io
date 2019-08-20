---
title: 排序算法
date: 2018-08-06 10:01:50
tags: Algorithm
categories: Algorithm
---
Outline
1.回顾几种排序算法
<!-- more -->
先来两个利用递归的排序算法
共同点：
入参相同都有lo和hi
都是利用递归
递归终止的条件相同都是hi<=lo return
不同点：
快排核心操作位于两个递归前（快嘛而且入参和排序算法相同）
归并核心操作位于两个递归后（入参多了一个mid）
### <font color=red size=5>快速排序</font>
```java
private void sort(int[] a, int lo, int hi) {
    if(hi<=lo) return;
    int j = partition(a, lo, hi);
    sort(a,lo,j-1);
    sort(a,j+1,hi);
}
//目标：让a[lo]位于它该位于的地方(j)；比a[lo]小的都位于其左侧，大的位于其右侧
private int partition(int[] a, int lo, int hi) {
    int v = a[lo];
    int i = lo, j = hi+1;
    while(true) {
        //这里=很关键
        while(a[++i] <= v) if(i == hi) break;
        while(a[--j] > v) if(j == lo) break;
        if(i >= j) break;
        swap(a, i ,j);
    }
    swap(a,lo,j);
    return j;
}
```
关于partiton
1）遇到合适的元素要停在当前位置，所以要提前++和--，不然swap时候索引又发生改变了，所以初始的i和j要在真正需要遍历的元素范围之外
2）外层循环跳出的条件显然要在swap和while之间
3）最后的swap要和j去交换，此时需要交换一个小于arr[low]的元素
<code>算法分析</code>
时间复杂度O(NlogN)
空间复杂度O(1)
当a本身就有序的时候，算法的时间复杂度会退化到O(N2)
将int[] a = {1,2,3,4,5,6,7,8}输入到算法中发现算法退化到sort(a,0,7)->sort(a,1,7)->sort(a,2,7)...sort(a,7,7)
### <font color=red size=5>归并排序</font>
```java
private void sort(int[] a, int lo, int hi) {
    if (hi <= lo) return;
    int mid = lo + (hi - lo) / 2;
    sort(a, lo, mid);
    sort(a, mid + 1, hi);
    //这里的入参很关键 要有lo 和 hi
    merge(a, lo, mid, hi);
}
private void merge(int[] a, int lo, int mid, int hi) {
    int i = lo, j = mid + 1;
    int[] aux = new int[a.length];
    //这里
    for (int k = lo; k <= hi; k++) {
        aux[k] = a[k];
    }
    for (int k = lo; k <= hi; k++) {
        if (i > mid) a[k] = aux[j++];
        else if (j > hi) a[k] = aux[i++];
        else if (aux[i] < aux[j]) a[k] = aux[i++];
        else a[k] = aux[j++];
    }
}
```
<code>算法分析</code>
时间复杂度O(NlogN)
空间复杂度O(N)
### <font color=red size=5>冒泡排序</font>
```java
void bubbleSort(int[] arr, int n) {
    for (int i = 0; i < n - 1; i++) {
        boolean isOrdered = true;
        for (int j = 0; j < n - 1 - i; j++) {
            if (arr[j] > arr[j + 1]) {
                int tem = arr[j + 1];
                arr[j + 1] = arr[j];
                arr[j] = tem;
                isOrdered = false;
            }
        }
        if (isOrdered) break;
    }
}
```
<code>算法分析</code>
时间复杂度O(N2)
所谓冒泡是说每次从底部向上遍历，每层循环会将最大的元素置为数组末端而且关键的一点是每次只能和周围的元素进行比较
当数组本身就有序 的时候，这种写法的冒泡排序复杂度为O(N)
### <font color=red size=5>选择排序</font>
```java
private void sort(int[]a) { 
    for(int i=0;i<a.length-1;i++){
        for(int j=i+1;j<a.length;j++){
            if(a[j]<a[i]) swap(a,i,j);
        }
    }
}
```
```java
private void sort(int[]a){
    //多少趟数
    for(int i = 0;i < a.length-1;i++) {
        //没趟多少次
        //和冒泡相比每次是和最x元素比较而不是和周围比较
        for(int j = 0;j<a.length-1-i;j++){
            if(a[j]>a[a.length-1-i]){
                swap(a,j,a.length-1-i);
            }
        }
    }
}
```
<code>算法分析</code>
时间复杂度O(N2)
算法思路：每次遍历不再是和周围元素比较，而是和指定位置的元素比较，和冒泡排序相比数据移动次数明显减少
优化方案不必每次小于a[i]都进行交换操作，只需存储最小值的索引，内层循环结束再去交换
```java
static void selectSort(int[] arr, int n) {
    for (int i = 0; i < n - 1; i++) {
        int curMin = Integer.MAX_VALUE;
        int curMinIndex = -1;
        for (int j = n - 1; j >= i; j--) {
            if (arr[j] < curMin) {
                curMin = arr[j];
                curMinIndex = j;
            }
        }
        arr[curMinIndex] = arr[i];
        arr[i] = curMin;
    }
}
```

### <font color=red size=5>插入排序</font>
```java
private void sort(int[]a) {
    //牌的位置，默认已经有一张牌所以从1开始
    for(int i=1;i<a.length;i++){
        //对当前牌进行交换
        for(int j=i;j>=1 && a[j]<a[j-1];j--){
            swap(a,j,j-1);
        }
    } 
}
```
<code>算法分析</code>
时间复杂度O(N2)
保证i左侧都是有序的，接下来将i放到合适的位置
适合处理有序或者部分有序的数
优化思路，内层循环不使用swap操作，采用错位赋值
```java
static void insertSort(int[] arr, int n) {
    for (int i = 1; i < n; i++) {
        int value = arr[i];
        for (int j = i - 1; j >= 0; j--) {
            if (value < arr[j]) {
                arr[j + 1] = arr[j];
            } else {
                arr[j + 1] = value;
                break;
            }
        }
    }
}
```
### <font color=red size=5>希尔排序</font>
```java
private void sort(int[] a) {
    int h = 1;
    int N = a.length;
    while (h < N / 3) h = 3 * h + 1;
    while (h >= 1) {
        for (int i = h; i < N; i++) {
            for (int j = i; j >= h && a[j] < a[j - h]; j -= h) {
                swap(a, j, j - h);
            }
        }
        h /= 3;
    }
}
```
<code>算法分析</code>
和普通的插入排序相比，在进行h有序排序过程中，元素移动的位置更大，无需一个个的交换
虽然最终当h=1的时候仍然是普通的插入排序，但这个时候需要需要移动的次数就不会太多了
比如最小的元素位于数组末端，如果采用普通的插入排序需要交换N-1次，而希尔排序则不一样，当h很大的时候可以将位于数组末端的最小元素直接直接换到数组前端
特点：编写简单，不需要额外空间并且性能也不太差
### <font color=red size=5>堆排序</font>
优雅的堆排序，想起来当初面试的时候被现在的老大问堆排序问到冷场，time flies~
<code>说起堆排序，先提纲挈领一发，就是利用优先队列实现的一种排序方法</code>
所谓优先队列其实就两个方法，一是插入元素，二是删除最大(小)元素
```java
public interface PriorityQueue{
    public void insert(int a);
    public int delMax();
}
```
```java
private void sort(int[] a){
    PriorityQueue queue = new MyPriorityQueue();
    for(int i : a){
        queue.insert(a);
    }
    for(int i = a.length-1;i>=0;i--){
        a[i] = queue.delMax();
    }
}
```
接下来问题就简单了，如何实现MypriorityQueue?这个时候就引出了"堆"
这里的堆和我们认识中的stack不同，而是利用数组实现的一种数据结构
<code>二叉堆是一组能够用堆有序的完全二叉树排序的元素，并在数组中按照层级存储（不使用数组的第一个位置）</code>
堆有一个有趣的特性，位置k的结点的父结点位置为k/2向下取整；位置k的结点的两个子结点分别是2k和2k+1,注意堆的父节点是同时大于（小于）两个子节点的
<img src='stack.png'>
<code>如何利用堆实现优先队列？</code>
```java
public class MyPriorityQueue implements PriorityQueue {
    private int[] arr;
    private int N = 0;
    public MyPriorityQueue(int size) {
        arr = new int[size+1];
    }
    public void insert(int a){
        arr[++N] = a;
        swim(N);
    }
    public int delMax(){
        int max = arr[1];
        swap(arr,1,N--);
        sink(1);
        return max;
    }
    private void swim(int k){
        while(k>1 && arr[k/2]<arr[k]){
            swap(arr,k/2,k);
            k/=2;
        }
    }
    private void sink(int k){
        while(2*k<=N){
            int j=2*k;
            if(j<N && arr[j] < arr[j+1]) j++;
            if(arr[k]>=arr[j]) break;
            swap(arr,k,j);
            k=j;
        }
    }
}
```
我们已经通过实现优先队列的api完成了基于堆的排序，但是让人烦躁的是每次都需要堆无序数组逐一插入到优先队列中，<code>能否将一个无序数组直接变成堆有序的数组？</code>
```java
private void sort(int[] a){
    int N = a.length;
    for(int k = N/2;k>=1;k--){
        sink(k);
    }
}
```
由此联想到，直接将数组进行原地排序
1.将数组置为堆有序
2.堆有序的数组通过交换和下沉操作完成排序
Ps需要将第一位空出来
```java
private void sort(int[]a){
    int N = a.length;
    //堆有序
    for(int k=N/2;k>=1;k--){
        sink(a,k,N);
    }
    //最大元素和最小元素交换位置
    //下沉最小元素，调整堆的结构
    while(N>1){
        swap(a,1,N--);
        sink(a,1,N);
    }
}
private void sink(int[] arr,int k, int size){
    while(2*k<=size){
        int j=2*k;
        if(j<size && arr[j] < arr[j+1]) j++;
        if(arr[k]>=arr[j]) break;
        swap(arr,k,j);
        k=j;
    }
}
```
无须空出第一位的写法
```java
static void sink(int[] a, int k, int N) {
    while (2 * k + 1 <= N) {
        int j = 2 * k + 1;
        if (j+1 <= N && a[j] < a[j + 1]) j++;
        if (a[k] >= a[j]) break;
        TemTest.swap(a, k, j);
        k = j;
    }
}

static void sort(int[] a) {
    int N = a.length-1;
    int k = N/2;
    while (k >= 0) {
        sink(a, k, N);
        k--;
    }
    while (N >= 1) {
        TemTest.swap(a,0,N--);
        sink(a,0,N);
    }
}
```
时间复杂度O(NlogN)，空间复杂度O(1)