---
layout:     post                    # 使用的布局（不需要改）
title:      insert_sort               # 标题 
subtitle:     #副标题
date:       2015-5-1              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 数据结构和算法基础
---
## 插入排序
- 时间复杂度是n的2次方
- 排序完成后结果是 array[i] <= array[i+1]


## 代码实现与分析

```
//insert_sort.c  
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

//初始时左边的集合只有一个元素arr[0]
//之后不断的把右边的元素插入到左边的集合中来
//本质上左边的集合已经是有序集合 新准备加入的元素只要发现比自己小的就可以暂停比较了
void fun(int * arr,int n)
{
    int i ;//初始化时表示[开始比较的起始位置] 之后不断递减 表示向左边集合查询
    int j ;//表示把该位置的值插入到左边的集合中 最坏的情况下是 这个值要放到最左边 即要跟左边所有值比较
    for (j=1; j<n; j++)
    {
        for(i=j; i>0; i--)//确定j应该放置在左边集合中的位置
        {
            if(arr[i]<arr[i-1])//等于的情况也没有交换的必要
            {
                swap(arr+i,arr+i-1);
            }
            else
            {
                break;//j的位置已经放置好了
            }
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



