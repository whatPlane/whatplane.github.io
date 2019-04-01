---
layout:     post                    # 使用的布局（不需要改）
title:      heap_sort               # 标题 
subtitle:     #副标题
date:       2015-5-20              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 算法与数据结构
---
## 堆排序
- 时间复杂度是logn
- 我这里是利用数组实现
- 结构是完全二叉树 [你可能需要先了解树的相关内容](https://whatplane.github.io/2015/05/20/bst/)
- 主要操作是 插入 和 删除


> 在插入或者删除操作之后，我们必须保持该实现应有的性质: 1. 完全二叉树 2. 每个节点值都小于或等于它的子节点。这里需要利用**上浮操作**和**下沉操作**来保证 。
- 插入新元素时，先放置在树的末尾处，然后进行**上浮操作**。
- 删除节点（只能删除根节点）时，先用树末位的元素替补根节点，然后进行**下沉操作**。

**完全二叉树**是增加了限定条件的二叉树。假设一个二叉树的深度为n。为了满足完全二叉树的要求，该二叉树的前n-1层必须填满，第n层也必须按照从左到右的顺序被填满，比如下图:
![](https://gitee.com/whatplane/resource/raw/master/img/bd_20190214172428.png)

## 代码分析

```
//heap_sort.c
#include<stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include<time.h>

/*
堆排序
这里存储利用数组arr
arr[0]表示节点的个数

*/


static void swap(int *a,int *b);
static void insert_node(int come, int heap[]);//把come元素放置到树的最后位置 然后执行向上过滤
static int delete_node( int heap[]);//删除根节点 然后把树最后位置的元素作为替补 再执行向下过滤
static void percolate_up(int heap[]);
static void percolate_down(int heap[]);


void swap(int *a,int *b)
{
    int tmp = *a;
    *a = *b;
    *b = tmp;
}
void print_arr(int heap[])
{
    int count = heap[0];
    int i=1;
    printf("\n{");
    for(;i<=count;i++)
    {
        printf(" %d",heap[i]);
    }
    printf("}\n");
}

int isempty(int heap[])
{
    if(heap[0]==0)return 1;
    return 0;
}

void insert_node(int come, int heap[])
{
    //1 计算最后元素的位置
    //2 把come安置在该位置
    //3 执行向上过滤
    heap[0] = heap[0]+1; //0号索引位置记录节点个数
    int count = heap[0];
    heap[count]=come;
    percolate_up(heap);//开始上浮操作
}
//上浮操作
void percolate_up(int heap[])
{
    int parent_index = heap[0]/2;//获取最后一个元素的父节点位置
    int child_index = heap[0];//孩子节点的位置
    while( (parent_index!=0)
    && (heap[child_index]<heap[parent_index]) )
    {//如果孩子节点比父节点小 那么交换 也就是认为是在往上移动
        swap(heap+child_index,heap+parent_index);
        child_index = parent_index;
        parent_index = child_index/2;
    }
}
int delete_node(int heap[])
{
    int element = heap[1];//保存要删除的元素
    int count = heap[0];//当前元素的总个数
    heap[1] = heap[count];//把最后一个替换到根节点
    heap[0] = count-1;//总的元素个数递减
    percolate_down(heap);//开始下沉操作
    return element;
}
//下沉操作
void percolate_down(int heap[])
{
    //parent 如果比孩子节点中最小的要大 那么交换
    //循环比较的条件是 有孩子 且 比孩子大
    //区分左孩子 右孩子
    int count = heap[0];
    int parent_index = 1;
    int lchild_index;
    int rchild_index;
    int min_index;
    int a=0;
    while( parent_index <= count)
    {
        lchild_index = 2*parent_index;
        rchild_index = 2*parent_index+1;
        if( (lchild_index<=count)
        &&(rchild_index<=count) )
        {//左右孩子都存在
            min_index = heap[lchild_index]<heap[rchild_index]?lchild_index:rchild_index;
        }
        else if(lchild_index>count)
        {//没有孩子
            break;
        }
        else if((lchild_index<=count)
        &&(rchild_index>count))
        {//只有左孩子
            min_index = lchild_index;
        }
        if(heap[parent_index]>heap[min_index])
        {//比孩子节点中较小的孩子大 那么交换 也就是认为在往下移动
            swap(heap+parent_index,heap+min_index);
            parent_index = min_index;
        }
        else
        {
            break;
        }
        
    }
}
int main()
{
    int heap[100] = {0};
    int i;
    insert_node(8,heap);
    insert_node(12,heap);
    insert_node(14,heap);
    insert_node(2,heap);
    insert_node(9,heap);
    print_arr(heap);
    for(i=0;i<3;i++)
    {
        printf("%d \n",delete_node(heap));
    }
    
    return 0;
}


```