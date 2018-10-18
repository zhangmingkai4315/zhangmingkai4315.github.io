---
id: 142
title: 算法基础
date: 2018-09-03T20:39:45+00:00
author: mingkai
layout: post
guid: http://mikezhang.cc/?p=142
permalink: /2018/09/basic-algorithm/
categories:
  - algorithm
tags:
  - 排序算法
  - 查询算法
  - 笔记
format: aside
---
算法实际上处理问题的一种方式或者一组指令，并且可以以代码的方式程呈现出来，应用在实际需求中。这些算法一般会用于改善解决问题的速度，或者解决某一类问题，比如使用二分法查找可以在有序列表中极大的减少查询次数，GPS查找利用图算法进行最短路径查找，动态规划算法可以用来编写国际跳棋AI的等等

#### 1&#46; 二分法查找

假如一个电话本（按照字母顺序排列），查找一个姓张的人的联系方式，我们可以直接翻到最后几页查找，而不是从姓名的拼音为A的开始查找。二分法的思想基本上就是，先取中间的值，比对大小，这样可以排除一半的选项，然后从剩下的一半的数据中重复此操作，直到找到需要的数据。

二分法查找的次数约为数据元素的logN次数，比如要在100个数据中查找到符合的数字，需要约为7步之内即可查询到

<pre><code class="math">\log_{2}100&lt;7
</code></pre>

使用二分法查找必须为有序列表，否则无法准确的定位数据。以下为python版本的二分法查找算法,我们可以首先比对是否超出了最大值和最小值，否则查询仍旧会循环多次。

<pre><code class="python">def binary_search(list,item):
  low = 0 
  high = len(list)-1
  print("Begin search {0} in list".format(item))
  if item &gt; list[high] or item &lt;list[0]:
    return None
  while low&lt;=high:
    mid = (low+high)/2
    guess = list[mid]
    print("Curren scope is [{0},{1}]".format(low,high))
    if guess == item:
      return mid
    if guess &gt; item:
      high = mid-1
    else:
      low = mid +1
  return None
</code></pre>

逐个数据的简单比对，查询时间一般为线性增长也就是O(N)时间，而二分法显然不是线性的，他的查询时间为O(logN)时间。

O(n)复杂度代表了查询所需要的最糟糕的情况， 可能有时候查询一次即可，但是最糟糕的时候需要遍历所有的n个项目才能找到。

其他的复杂度的例子

  1. O(logn)线性查找，简单查找
  2. O(logn)对数查找 二分法查找
  3. O(n*logn)快速排序算法（较快）
  4. O(log(n*n)) 选择排序算法（较慢）
  5. O(logn！)旅行商问题的解决办法（非常慢）

以下就是算法复杂度的曲线图，注意其中的速度并非执行时间，指的操作的次数，但是基本上次数与时间是成正比的：

![曲线图](https://cdn-images-1.medium.com/max/1600/1*wt2zSdhsA4koehmPsAO8gQ.jpeg)

其中旅行商问题的复杂度为O(n!)，解决的类似于路径选择的难题，比如前往5个城市，怎么走旅行的路径最短，通过计算5个城市的路径组合约为5！, 也就是我们需要比较这些组合哪些是最短的路径，如果城市数量太多，可能会超出预期的时间，因此需要考虑一些近似的算法解决这个问题。

#### 2&#46; 数组和链表

数组和链表的区别在于实际的存储和访问方式，对于数组来说，在内存中是连续的存储空间，如果增加元素时候，原有空间没有合适的大小够分配则转移原有的数据到新的地址空间上，复制代价较大。

链表则使用地址指针的方式存储数据，每一个数据项都包含指针指向下一个数据，因此添加元素的时候很简单，只需要将新的数据存储位置告诉当前最后的元素即可，无需任何重复的复制。

链表的访问，当访问链表中第100个元素的时候，需要从头开始依次遍历所有99个元素后才能找到100个元素的位置。而数组在访问的时候则无需此操作，直接通过乘法即可获取到内容（所有内容均是连续存储）

![](http://www.stoimen.com/blog/wp-content/uploads/2012/07/Array-Linked-List.png)

> 链表不支持随机访问的。数组的随机读取元素速度更快一些。

对于常见的插入操作，链表的复杂度为O(1),而数组的复杂度为O(n); 对于读取的操作，则链表的复杂度为O(n),而数组的复杂度为O(1);

当需要在中间插入的时候，链表只需要修改中间元素的链接地址即可，而数组则需要顺序移动所有后面的元素位置，当元素所在空间不足时候，还要重新移动所有元素到新的分配的内存地址位置。

#### 3&#46; 选择排序

选择排序的方式比较简单，假设有列表元素[7,4,5,9,8,2,1]，为无序的列表，需要通过排序获得从小到大有序的列表，执行的方式如下图所示

![](http://4.bp.blogspot.com/-YSYtQJc_IuE/ULCnycV8PFI/AAAAAAAAACo/B5NeEwMnCWY/s1600/selection-sort.JPG)

每次排序都需要获取到最小的值并进行数值的交换，因此对于一个n个项目的元素每次都需要检查1-n个元素，平均检查的数量为n/2个，而总共需要进行n轮，因此复杂度为O(n&#42;n/2)去除常数后为O(n&#42;n)

选择排序的python代码实现如下：

<pre><code class="python">def find_smallest(arr):
  smallest = arr[0]
  smallest_index = 0
  for i in range(1,len(arr)):
    if arr[i] &lt; smallest:
      smallest = arr[i]
      smallest_index = i
  return smallest_index

def selection_sort(arr):
  new_arr = []
  for i in range(len(arr)):
    smallest = find_smallest(arr[i:])
    temp= arr[smallest+i]
    arr[smallest+i] = arr[i]
    arr[i] = temp
  return arr

</code></pre>

#### 4&#46; 递归算法

> 如果使用循环则程序执行效率可能更高，但是使用递归则使得程序执行更清楚。

递归调用总是包含有基线条件和递归条件， 基线条件代表何时终止程序，而递归条件则代表何时继续递归操作，比如下面的代码中,i>0为递归条件，i<=0则为基线条件。

<pre><code class="python">def count(i):
  if i&gt;0:
    print("current i = {}".format(i))
    count(i-1)
  else:
    return 0
</code></pre>

递归调用的时候函数调用栈将依次将count函数以及当前的参数压入栈中，栈中的元素不断增多，但是因为函数都没有返回，所以不会有任何从栈中被移除。直到参数为0的元素传入count函数中时候，第一次count(0)函数返回，从栈顶移除该函数调用，其他的函数count(1),count(2)&#8230;count(100)才依次的从栈顶移除。下面是一个阶乘的计算n!的递归调用

![](https://i.stack.imgur.com/PK6Ht.png)

当调用另一个函数的时候，当前函数暂停执行并处于未完成状态，函数的所有参数变量都还在内存中，当调用函数结束时候才能恢复当前函数的继续执行。当我们执行的递归数量超出一定限制的时候，程序不会一直向栈中加入新的元素：

**RuntimeError**: maximum recursion depth exceeded while getting the str of an object

同时需要考虑的是，递归操作将函数和参数压入栈是相对于循环更消耗资源的，需要存储大量函数调用的信息，所以选择递归的时候一定要衡量利弊。

#### 5&#46;快速排序

分而治之,递归式解决问题的办法，一种解决问题的思路。编写递归的时候，基线条件通常是空或者只包含一个元素。以下为递归的进行累加操作，通过递归每次将任务分为更小的单元进行处理，基线设置为最终只有一个元素

<pre><code class="python">def recursive_sum(arr):
  if len(arr) == 1:
    return arr[0]
  return arr[0]+recursive_sum(arr[1:])

def main():
  arr = range(100)
  print(recursive_sum(arr))
</code></pre>

快速排序是一种常见的排序算法，比选择排序更快一些。C语言中qsort实现就是通过快排实现的。

快速排序的思想也是分治法的运用，通过分而治之将数组分为两个部分，再分别对于分开的数组进行排序，最后做数组的合并。

![](https://www.geeksforgeeks.org/wp-content/uploads/gq/2014/01/QuickSort2.png)

快速排序每次通过选定一个基准值，按照基准值比较的方式，分为左边数组和右边数组，所有左边数组的都会比基准值小，所有右边数组都会比基准值大。通过分割逐次获得排序的数组，最后进行合并，如上图所示

<pre><code class="python">&lt;br />def quick_sort(arr):
  if len(arr)&lt;2:
    return arr
  pivot = arr[0]
  left = [i for i in arr[1:] if i &lt;= pivot]
  right = [i for i in arr[1:] if i &gt; pivot]
  return quick_sort(left)+[pivot]+quick_sort(right)
def main():
  arr = [100,2,3,32,3,32,332,43,224,22,322]
  print(quick_sort(arr))

</code></pre>

快速排序的复杂度为O(nlogn)，这比选择排序要快了很多，与合并排序的复杂度比较相似，只是对于快速排序，如果每次选择第一个元素，恰好数组又是有序的递增数组，则每次选择的左边都为空数组，因此导致此时的快速排序执行时间接近n*n的大小。

**快速排序对于链表结构的数据排序并不理想**，因为随机访问数组的代价比较大（通过链表遍历），而数组结构则不存在该问题，随机访问成本较低。 对于链表，可以考虑使用合并排序，执行merge操作和排序的成本也比较低。

对于快速排序每次选择都是随机，则层数应该是logn的大小，每次调用都是n次比较，所以最佳也是平均情况下复杂度为O(nlogn)