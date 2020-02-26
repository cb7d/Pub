---
title: "Sort"
date: 2020-02-24T22:51:07+08:00
showDate: true
draft: false
tags: ["blog","story","arithmetic"]
---

# 基本排序算法

正所谓基础不牢，地动山摇，排序算法基本上是我们上课的时候接触的第一种算法，这里写个小小的总结，希望能帮助大家回忆起来，然后我们可以接着往下一起进行我们的算法之旅。



- [冒泡排序](#冒泡排序)
- [插入排序](#插入排序)
- [希尔排序](#希尔排序)
- [选择排序](#选择排序)
- [归并排序](#归并排序)
- [快速排序](#快速排序)
- [堆排序](#堆排序)
- 基数排序(todo)



## 冒泡排序

时间复杂度：O(n^2)

空间复杂度：O(1)

是否稳定：是

特点：简单粗暴，小学生一样能想出来😄

实现步骤

- 比较相邻元素，如果前者比后者大就交换两者
- 对每一对相邻元素作同样的操作，使得单次操作最后一位为最大
- 未排序序列去掉最后一个，重复上述操作

```cpp
void BubbleSort(int arr[], long size) {
    for (int i = 0; i < size; i++) {
        bool flag = true;
        for (int j = 0; j < size - i - 1; j++) {
            if (arr[j] > arr[j+1]) {
                swap(arr[j], arr[j+1]);
                flag = false;
            }
        }
        if (true == flag) return;
    }
}
```

## 插入排序

时间复杂度：O(n^2)

空间复杂度：O(1)

是否稳定：是

特点：简单直观，通过构建有序序列，对于未排序数据，在已排序序列中从后向前扫描找到位置并插入

实现步骤

- 将待排序序列的第一个元素看作有序序列，第二个到最后一个看作无序序列
- 从头到尾扫描未排序序列，将扫描到的每个元素插入有序序列的适当位置（相等的元素则插入在该元素后面）

```cpp
void InsertSort(int arr[], long size) {

    for (int i = 1; i < size; i++) {
        int tmp = arr[i];
        int j = i - 1;
        while (j >= 0) {

            if (arr[j] > tmp) {
                arr[j+1] = arr[j];
            }else {
                break;
            }
            j--;
        }
        arr[j+1] = tmp;
    }
}
```

## 希尔排序

时间复杂度：根据步长序列的不同而不同 O(n^2)  O(n log2n)

空间复杂度：O(*n*)

是否稳定：否

特点：希尔排序也叫做递减增量排序算法，是对插入排序的一种更高效的改进版本，但是是非稳定排序算法

实现步骤

希尔排序基于一下亮点特性提出改进方法

- 插入排序对几乎已经排好序的数据操作时效率高，可以达到线性排序的效果

- 但是插入排序一般来说效率不高，每次只能移动一位

希尔排序的基本思想

先将整个待排序的记录序列分割为若干子序列分别进行直接插入排序，待整个序列中的记录「基本有序」时，再对全体记录进行依此直接插入排序

```cpp
void ShellSort(int arr[], long size) {
    int i, j, gap;

    for (gap = size/2; gap > 0; gap /=2) {
        for (i = gap; i < size; i++) {
            for (j = i - gap; j >= 0 && arr[j] > arr[j+gap]; j -= gap) {
                swap(arr[j], arr[j+gap]);
            }
        }
    }
}
```

## 选择排序

时间复杂度：O(n^2)

空间复杂度：O(1)

是否稳定：否

特点：无

实现步骤：

- 在未排序序列中找到最大元素，存放到序列最末尾

- 从剩余序列中找到最大元素，放入已排序序列头部

- 重复

```cpp
void SelectSort(int arr[], long size) {
    for (long i = size - 1; i >= 0; i--) {
        long maxNum = arr[i];
        long rank = i;
        for (long j = 0; j < i; j++) {
            if (arr[j] > maxNum) {
                maxNum = arr[j];
                rank = j;
            }
        }
        swap(arr[rank], arr[i]);
    }
}
```

## 归并排序

时间复杂度：O(n log n)

空间复杂度：O(n)

是否稳定：是

特点：由冯诺伊曼提出，分治思想

```cpp
// 合并数组，排好序，然后再拷贝到原来的数组
void MergeArray(int arr[], int start, int end, int mid, int temp[]) {
    int i = start;
    int j = mid + 1;
    int k = 0;
    while (i <= mid && j <= end) {
        if (arr[i] < arr[j]) {
            temp[k++] = arr[i++];
        } else {
            temp[k++] = arr[j++];
        }
    }
    while (i <= mid) {
        temp[k++] = arr[i++];
    }
    while (j <= end) {
        temp[k++] = arr[j++];
    }
    for (int i = 0; i < k; i++) {
        arr[start + i] = temp[i];
    }
}

// 归并排序，将数组前半部分后半部分分成最小块，然后再合并
void MergeSort(int arr[], int start, int end, int temp[]) {
    if (start < end) {
        int mid = (start + end)/2;
        MergeSort(arr, start, mid, temp);
        MergeSort(arr,mid + 1, end, temp);
        MergeArray(arr, start, end, mid, temp);
    }
}

// 在这里创建临时数组，节约内存开销，因为以后的temp都是在递归时使用的
void MergeSort(int arr[], int size) {
    int start = 0;
    int end = size - 1;
    int *temp = new int[size];
    MergeSort(arr, start, end, temp);
}
```

## 快速排序

时间复杂度：O(n log n)

空间复杂度：O(n)

是否稳定：否

特点：通常快速排序会快于平均复杂度，因为他的内部循环可以在大部分的架构上很有效率的被实现出来

实现步骤

平均状况下，排序n个体项目需要 O(n log n) 次比较，在最坏情况下需要 O(n2) 次比较

快速排序使用分治法策略将一个串分为两个子串

算法步骤

- 丛序列中挑选一个元素，称为基数

- 重新排列序列，所有比基数小的都排序到基数前面，所有比基数大的都排序到基数后面（分区操作）

- 递归的把基数左侧序列和基数右侧序列进行排序

递归的最底部情形是子序列的长度为0或1

```cpp
void QuickSort(int arr[], int start, int end) {

    if (start >= end) return;

    int left = start;
    int right = end - 1;
    int base = arr[end];
    while (true) {

        while(arr[left] < base) ++left;
        while(arr[right] >= base) --right;
        if (left >= right) break;
        swap(arr[left], arr[right]);
    }
    swap(arr[left], arr[end]);
    QuickSort(arr, start, left - 1);
    QuickSort(arr, left + 1, end);
}
```

## 堆排序

时间复杂度：O(n log n) 其中建堆复杂度为O(n)

空间复杂度：O(1)

是否稳定：否

特点：利用堆这一数据结构进行排序

堆是一种特殊的数据结构，即完全二叉树。堆分为大根堆和小根堆

大根堆为根节点的值大于两个子节点的值

小根堆为根节点的值小于两个子节点的值

同时根节点的两个子树也分别为一个堆

通俗来讲，二叉树在按层序遍历时遇到的第一个NULL指针即作为结尾的二叉树就可以称为完全二叉树



根节点下标为0，若父节点相应数组下标为i，则其左孩子相应数组下标为2*i+1

右孩子为2*i+2



注意：如果我们现在处理第i个序号的节点的数，那么他的父节点序号就是(i-1)/2

他的孩子节点就是2*i+1 与2*i+2。

这个容易想，右子节点如果直接除以2，得到i+1，但是2*i+2-1和2*i+1-1除以2可以得到。

根节点从0开始，则n个节点的堆，最后一个叶子节点的下标为n-1，

其父节点是一个非叶子节点，下标为((n-1)-1)/2 = (n-2)/2 = n/2 - 1

- 首先用前n个元素的无序序列构建大顶堆；构建时从最后一个非叶子节点n/2-1的位置开始检查节点与其孩子节点是否满足大顶堆的要求，不满足则需要调整该元素与其孩子节点元素的位置，如果有调整，则调整过的孩子节点（子树）也要递归调用调整子树中的元素值位置，保证子树也是大顶堆。然后按照层次变得顺序依此往前，从右向左，从下到上调整所有非叶节点的值，最后根节点的值就是最大值。

- 得到大顶堆后将根节点与数组待排序部分的最后一个元素交换位置，即将最大元素“沉”到数组末端。

- 交换过后待排序数组长度减1，再对新长度的待排序数组重复上述过程，直到整个数组排序完成。如果我们要数组整体递增有序，则每次构建的是大顶堆，如果我们要数组整体递减有序，则每次构建的是最小顶堆

```cpp
// 将i节点为根的堆中小的数依此上移，n表示堆中的数据个数
void maxHeapIfy(int arr[], int i, int n) {

    int l = 2 * i + 1; // i 的左孩子
    int r = 2 * i + 2; // i 的右孩子

    int largest = i; // 先设置父节点和子节点三个节点中最大值的位置为父节点下标

    if (l < n && arr[l] > arr[largest]) {
        largest = l;
    }

    if (r < n && arr[r] > arr[largest]) {
        largest = r;
    }

    if (largest != i) {
        swap(arr[i], arr[largest]);
        // 递归调用，保证子树也是最大堆
        maxHeapIfy(arr, largest, n);
    }
}

// 构建堆代码实现
void buildMaxHeap(int arr[], int size) {

    // 从最后一个非叶子节点（n/2-1）开始自底向上构建
    for (int i = size / 2 - 1; i >= 0; i--) // 从 （n/2-1）调用一次maxHeapIfy就可以得到最大堆
        maxHeapIfy(arr, i, size);
}

void HeapSort(int arr[], int size) {
    buildMaxHeap(arr, size);
    for (int i = size - 1; i > 0; i--) {
        
        // 将根节点（最大值）与数组待排序部分的最后一个元素交换，这样最终得到的是递增序列
        swap(arr[0], arr[i]);
        // 将待排序数组长度减1，只要对换到根节点的元素进行排序，将它下沉就好了
        maxHeapIfy(arr, 0, i);
    }
}
```

