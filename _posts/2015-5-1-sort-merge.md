---
layout:     post                    # 使用的布局（不需要改）
title:      merge_sort               # 标题 
subtitle:     #副标题
date:       2013-5-10              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Data Structures
---
## 归并排序
- 平均时间复杂度是n*logn 
- 排序完成后结果是 array[i] <= array[i+1]
- 本质是递归算法


## 归并排序递归算法注意两个点
1. 怎么合并已经排序的A B两个子集合
  - 分别从A B集合中取出一个元素 比较
  - 保存较小的元素到临时表tmp中 然后再从该元素之前所在的序列中取出一个元素来比较
  - 一直循环比较直到其中一个序列没有元素可以比较 则退出循环
  - 然后把还有多余元素没有参加比较的直接添加到tmp后面
2. 子集合不需要再递归调用的条件是什么
  - 只有一个元素则直接返回



```
//merge_sort.c  
#include<stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include<time.h>

#define random(x) (rand()%x)
static int RAND_RANGE = 100;
static void swap(int *a,int *b);
static void printArr(int *arr,int n);
static void fun(int * arr,int n);
static int* create_arr(int n);
static int destroy_arr(int* p);

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


int* create_arr(int n)
{
    int i = n;
    int *p = (int*)malloc(sizeof(int)*n);
    memset(p,0,sizeof(int)*n);
    for(i=0; i<n; i++)
    {
        p[i] = random(RAND_RANGE);//随机数范围
    }
    return p;
}

int destroy_arr(int* p)
{
    free(p);
}

void fun(int *arr ,int n)
{
    if(1==n) {return;}
    int left = 0;
    int right = n-1;
    int middle = (left+right)/2;
    int i=0;//左边集合的起始位置
    int j=middle+1;//右边集合的起始位置
    int k=0;
    //middle位置的元素 算在左边的集合中 这里划分很重要 
    fun(arr+left,middle-left+1);
    fun(arr+middle+1,right-middle);

    int *tmp = (int*)malloc(sizeof(int)*n);
    while( (i<=middle) &&(j<=right) )
    {
        if(arr[i]<arr[j])
        {
            tmp[k++] = arr[i++];
        }
        else
        {
            tmp[k++] = arr[j++];
        }
    }
    if(i <= middle)
    {//说明 i组还有多余元素 
        for(;i<=middle;i++)
        {
            tmp[k++] = arr[i];
        }
    }
    if(j<=right)
    {//说明 j组还有多余元素 
        for(;j<=right;j++)
        {
            tmp[k++] = arr[j];
        }
    }
    /*if(i > middle) 这段代码的意思是 谁导致跳出循环的 那么说明另外一组还有多余的元素需要添加到tmp
    {问题是 这里修改了 j 会影响后面的判断
        for(;j<=right;j++)
        {
            tmp[k++] = arr[j];
        }
    }
    if(j>right)
    {
        for(;i<middle;i++)
        {
            tmp[k++] = arr[i];
        }
    }*/
    memcpy(arr,tmp,sizeof(int)*n);
    free(tmp);
    tmp=NULL;
}

int main()  
{  
    srand(time(NULL));
    int* arr;
    int size = 21;
    
    arr = create_arr(size);
    
    printArr(arr,size);
    fun(arr,size);
    printArr(arr,size);

    destroy_arr(arr);
 
    return 0;
} 
```