---
layout:     post                    # 使用的布局（不需要改）
title:      选择排序               # 标题 
subtitle:     #副标题
date:       2015-5-1              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 算法与数据结构
---
# 选择排序
- 时间复杂度是n的2次方
- 排序完成后结果是 array[i] <= array[i+1]


```
//select_sort.c  
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
//每一轮都是找出剩余集合中的最小值 找出本应该属于i位置的一个最小值需要比较 n-i 次
//注意跟冒泡排序不同 冒泡排序在剩余的集合中找到最小值时 如果发现没有交换发生 那么可以提前结束排序
//而选择排序每轮查找必须遍历所有剩下的元素
void fun(int * arr,int n)
{
    int i; //找出该位置上应该放的值
    int j; //每一轮初始化时表示本轮的目的是填充j位置 之后j不断递增 表示不断查找
    int minpos;
    for(i=0; i<n; i++)
    {
        minpos = i;//初始化本轮最小值的索引位置
        for(j=i+1; j<n; j++)//只用找出j位置之后的集合中最小值
        {
            if(arr[j]<arr[minpos]) 
            {
              minpos = j;
            }
        }
        swap(arr+i,arr+minpos);//把i位置放置正确的元素
    }
}

int main()  
{  
    int a[] = {22,33,4,2,24,78};
    //int a[] = {22,33,44,55,15,11};
    printArr(a,6);
    fun(a,6);
    printArr(a,6);
 
} 
```



