---
layout:     post                    # 使用的布局（不需要改）
title:      希尔排序               # 标题 
subtitle:     #副标题
date:       2015-5-10              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 算法与数据结构
---
# 希尔排序
- 平均时间复杂度是n的1.3次方 
- 排序完成后结果是 array[i] <= array[i+1]
- 是对插入排序的改进 因为每次跳跃幅度更大 导致交换减少




```
//shell_sort.c  
#include<stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include<time.h>

#define random(x) (rand()%x)
static int RAND_RANGE = 100;

static void swap(int *a,int *b);
static void printArr(int *arr,int n);
static void fun(int * arr,int n);
static void fun_optimize(int * arr,int n);
static int* create_arr(int n);
static int destroy_arr(int* p);
static void checkSwap(int* arr,int n,int j,int step);


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

//arr是数组
//n是数组长度
//j表示当前正要插入左边集合的元素位置 [这个j的位置不是arr中的位置 而是以间隔划分arr形成的组所在的位置]
//step是间隔
void checkSwap(int* arr,int n,int j,int step)
{
    int k;
    for(k=0;k<step;k++)
    {
        if((j*step+k<n)
        &&(arr[j*step+k]<arr[(j-1)*step+k]))
        {
            swap(arr+j*step+k,arr+(j-1)*step+k);
            printf("swap");
            printArr(arr,n);
            sleep(1);
        }
    }
   
}
```

1. 希尔排序的特点是 实现大的跳跃交换
2. 通过划分不同间隔的组来实现
3. 比如当前间隔是4 总共有12个元素 那么可以分为3个A B C组，
    - 抽取每个组中的第一个元素组成一个新的序列 对新序列使用插入排序
    - 抽取每个组中剩下元素的第一个元素组成一个新的序列，对新序列继续使用插入排序
    ..
    - 不断的抽取A B C 三个组的元素组成新的序列，直到没有元素可以抽取为止 此时代表间隔4的情况处理完毕 进一步缩小间隔为原来的一半 。这里也就是4/2 也就是间隔是2 总共元素12个元素 那么可以分为 6个组 ...j

> 注意：这里说的组 和新序列的 关系 新序列是取出每个组中的第一个元素形成的


```
void fun(int * arr,int n)
{
    int step = n;
    int i;
    int j;
    int k;
    step = step/2;//step表示一个组有多少元素 也就是当前间隔
    int numofarr;//当前划分的组个数 即按照间隔分配 所有元素可以划分成几组
    while(step>0)
    {//处理当前间隔下的交换跳跃
        numofarr = n/step;
        for(i=1;i<numofarr;i++)//每一个组认为是一个元素 就可以类比插入排序了 
        {
            j = i;
            for(;j>0;j--)
            {
                //checkSwap(arr,n,j,step);
                for (k = 0; k < step; k++)//相邻元素交换step次 因为这个元素实际上是一个组
                {
                    if ((j * step + k < n) 
                    && (arr[j * step + k] < arr[(j - 1) * step + k]))
                    {
                        swap(arr + j * step + k, arr + (j - 1) * step + k);
                    }
                }
                //
            }
        }
        printf("按照间隔[%d]跳跃后 \n",step);
        printArr(arr,n);
        step = step/2;
    }
    

}
```


这是稍微优化后的方法，这个方法的工作过程是：
比如 先设置一个间隔3 总共12个元素 那么会形成4个组 3个序列 A B C 默认第一个组的元素分别是 A B C序列的第一个元素
接下来处理步骤是

- 处理A序列的第二个元素
- 处理B序列的第二个元素
- 处理C序列的第二个元素

- 处理A序列的第三个元素
- 处理B序列的第三个元素
- 处理C序列的第三个元素 

这个过程有点类似cpu时间分片，注意参考下图:
> 图-1
![](https://gitee.com/whatplane/resource/raw/master/img/wx_20190212235035.png)



```
void fun_optimize(int * arr,int n)
{
    int step = n/2;//step表示一个组有多少元素 也就是当前间隔
    int i;
    int j;
    while(step>0)
    {
        for (i=step; i<n; i++)//这里的i表示不同组的元素
        {
            j = i;
            for(;j>0;j=j-step)//这里是对某一个组里面的元素进行处理 以便找到正确位置
            { 
                if( (j-step>=0)
                &&(arr[j]<arr[j-step]) )
                {
                    swap(arr+j,arr+j-step);
                }
                else
                {
                    break;
                }

            }  
           
        }
        step = step/2;
    }
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

int main()  
{  
    srand(time(NULL));
    int* arr;
    int size = 21;
    
    arr = create_arr(size);
    
    printArr(arr,size);
	//实现方法一
    //fun(arr,size);
	//实现方法二
    fun_optimize(arr,size);
    printArr(arr,size);

    destroy_arr(arr);
 
    return 0;
} 
```



