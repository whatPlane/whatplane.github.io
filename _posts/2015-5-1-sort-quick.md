---
layout:     post                    # 使用的布局（不需要改）
title:      快速排序               # 标题 
subtitle:     #副标题
date:       2015-5-10              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 算法与数据结构
---
# 快速排序
- 平均时间复杂度是n*logn 最差复杂度是n的2次方
- 排序完成后结果是 array[i] <= array[i+1]


```
//quick_sort.c  
#include<stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include<time.h>

#define random(x) (rand()%x)
static int RAND_RANGE = 100;

static void swap(int *a,int *b);
static void printArr(int *arr,int n);
static void fun(int * arr,int n);
static void fun_2(int *arr ,int n);
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

```



快速排序递归 划分一个序列的另一种方法 ：左边是小于等于x 最右边开始是大于x
- b代表大于等于x的元素 big 
- s代表小于x的元素 small
- 从左边的位置1开始 从左往右 与x比较 如果某元素b大于x 那么准备交换到右边 但是如果右边的元素也是大于x的 
- 那么需要从右往左找到一个小于等于x的数s 与b交换  交换后 继续从左往右比较
 - 1 位置0的元素作为x 假设总用有 7 个元素
 - 2 位置1元素b如果比x大 那么从 最右边往左边找到一个小于等于x的元素s  然后s跟b交换
 - 3 继续从左往右
 - 4 位置2元素s如果比x小 那么继续从左往右
      ..
- 表示划分完成的条件是 i==j

可以抽象成这样的 s与s交换没有任何影响 b与b交换没有任何影响
[x bbbsssbbsbbsss]---->[sssssss x bbbbbb]



# 方法二：
```

void fun_2(int *arr ,int n)
{
    if(n<=1)return;//n可能等于0 因为划分的时候middle的某一侧可能没有元素
    int i =1;//表示从左往右的起点
    int j= n-1;//表示从右往左的起点
    int x = arr[0];
    int middle ;//该位置的左边都满足小于等于x
    for(i=1;i<n;i++)
    {
        if(arr[i]>x)
        {//从右往左找到可以交换的位置
            for(;j>i;j--)//查询的位置限制为i的右边部分
            {
                if(arr[j]<=x)
                {
                    swap(arr+i,arr+j);
                    break;
                }
            }
            if(i==j)
            {//判断是查找到了结束 还是划分完成了
                middle = i;
            }
        }
        else
        {//这里主要是判断middle位置
            if(i==j)
            {
                middle = i+1;
            }
        }
    }
    swap(arr+0,arr+middle-1);

    fun_2(arr+0,middle-1);
    fun_2(arr+middle,n-(middle-1)-1);
}

```


快速递归排序
-     1 首先划分成为两个集合
-     2 合并集合
    - 从一个集合中随便选择一个元素保证左边的元素都是小于等于他 右边的都是大于他
    -     1 先随机选择一个元素x
    -     2 x放置在0位置上 middle 用来记录分界线 初始化为 1 表示middle左边的都是小于等于x 
    -     3 从 1 位置开始跟x比较
    -     4 如果元素比x大 那么不做任何变化
    -     5 如果元素小于等于x 那么元素跟middle位置的元素交换 同时middle+1 
    -     6 开始下一个元素比较
    > 特殊的地方是：
    当 1位置元素跟x比较时 如果小于等于x 那么1 需要与middle位置交换 而初始化时middle就是1 
    接着如果 2位置的元素跟x比较时候 也是小于等于x 那么2位置元素同样与middle交换 此时middle是 2
    ...
    

# 方法一：

```
void fun(int *arr ,int n)
{
    if (n <= 1)
    {
        return;
    }
    int middle = 1;
    int x = arr[0];
    int i = 1;
    for (; i < n; i++)
    {
        if (arr[i] <= x)
        { //以middle为界 分配元素到两边 左边的小于等于x 右边的大于x
            swap(arr + i, arr + middle);
            middle++;
        }
    }
    swap(arr + 0, arr + middle - 1);
    //if (n <= 2){return;} //这里算是一点优化 虽然作用不是很大 但是感觉上让递归返回的条件变麻烦了 
    fun(arr + 0, middle - 1);
    fun(arr + middle, n - middle);
}

int main()  
{  
    srand(time(NULL));
    int* arr;
    int size = 7;
    
    arr = create_arr(size);
    
    printArr(arr,size);
    //fun(arr,size);
    fun_2(arr,size);
    printArr(arr,size);

    destroy_arr(arr);
 
    return 0;
} 
```