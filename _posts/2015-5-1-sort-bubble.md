---
layout:     post                    # 使用的布局（不需要改）
title:      bubble_sort               # 标题 
subtitle:     #副标题
date:       2015-5-1              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 算法与数据结构
---
## 冒泡排序
- 时间复杂度是n的2次方
- 排序完成后结果是 array[i] <= array[i+1]
- 每一趟都是从右至左不断扫描，判断相邻元素是否需要交换
- 每一趟都的结果是安置一个元素到左边


## 代码实现与分析

```
//bubble_sort.c  
#include<stdio.h>


static void swap(int *a,int *b);
static void printArr(int *arr,int n);
static void fun(int * arr,int n);

void swap(int *a,int *b)
{
    int tmp = *a;
    *a = *b;
    *b = tmp;
}

void printArr(int *arr,int n)
{
    printf("\n{");
    int i;
    for(i=0; i<n;i++)
    {
        printf(" %d",arr[i]);
    }
    printf("}\n");
}

//最左边的表示最小值
//每一轮都是找出剩余集合中的最小值 如果这一轮没有任何交换 说明整个排序已经完成
void fun(int * arr,int n)
{
    int i; //找出该位置上应该放的值
    int j; 
    int hasSwap;//是否有交换行为
    for(i=0; i<n; i++)
    {
        hasSwap = 0;
        for(j=n-1; j>i; j--)//只用找出i位置之后的集合中最小值
        {
            if(arr[j]<arr[j-1]) 
            {
                swap(arr+j,arr+j-1);
                hasSwap = 1;//记录交换
            }
        }
        if(0==hasSwap)//没有交换记录
        {
            printf("第[%d]轮就已经排序好了\n",i);
            break; //说明已经排序好了 可以提前收工
        }
    }
}

int main()  
{  
    //int a[] = {22,33,4,2,24,78};
    int a[] = {22,33,44,55,15,11};
    printArr(a,6);
    fun(a,6);
    printArr(a,6);
 
} 
```



