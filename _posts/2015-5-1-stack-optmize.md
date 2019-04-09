---
layout:     post                    # 使用的布局（不需要改）
title:      stack-optmize               # 标题 
subtitle:     #副标题
date:       2015-5-22              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 数据结构和算法基础
---
## 现在讨论另外一种stack的实现
- 前面讨论了stack的一种实现 [回顾stack的实现](https://whatplane.github.io/2015/05/22/stack/)  我们可以知道，这种实现中，head节点右边的第一个节点实际上是**栈底节点**。
- 这次我们的实现稍有不同。也就是我们把**top位置**定义在head节点的**右边第一个位置**。

下图是他们区别
![](https://gitee.com/whatplane/resource/raw/master/img/wx_20190214164221.jpg)

## 实现代码与分析 

```
//STACK.c
#include <stdio.h>
#include <stdlib.h>

#if 0
1 这里是一个栈的简单链表实现
2 这里的头结点也就是head 也是一个节点 分配了空间 只是没有有效元素数据
3 top节点是离head最近的节点 [注意跟之前的实现的主要差别]
4 所以打印链表 实际上是按照top位置开始的 

#endif

typedef int Element;

typedef struct mynode {
    Element element;
    struct mynode* next;
}node;

typedef node* STACK;

static STACK init_stack(void);
static void clear_stack(STACK);
static void destroy_stack(STACK);

static Element top(STACK);
static void push(STACK, Element);
static Element pop(STACK);
static node* create_node(Element );
static int is_null(STACK);
static int is_exist(STACK);

static void print_stack(STACK);

STACK init_stack(void)
{
    node* head = (node*)malloc(sizeof(node));
    head->element = 0;
    head->next = NULL;

    return head;

}

void clear_stack(STACK stack)
{
    node* head = stack;
    if(!is_exist(head))return;
    if(is_null(head))return;
    
    node* p = head->next;
    node* pnext ;
    while(p)
    {
        pnext = p->next;
        free(p);
        p = pnext;
    }
    head->next=NULL;

}
void destroy_stack(STACK stack)
{
    node* head = stack;
    if(!is_exist(head))return;
    clear_stack(head);
    free(head);
}

int is_exist(STACK stack)
{
    if(stack==NULL)
    {
        printf("no stack !!!\n");
        return 0;
    }
    return 1;
}
int is_null(STACK stack)
{
    node* head = stack;
    if(head->next==NULL)return 1;
    return 0;
}
node* create_node(Element e)
{
    node* p = (node*)malloc(sizeof(node));
    p->element = e;
    p->next = NULL;
    return p;
}

//调用前请先判断栈是否为空
Element top(STACK stack)
{
    node* head = stack;
    return head->next->element;
}

void push(STACK stack, Element e)
{
    node* head = stack;
    node* p = create_node(e);
    node* pnext = head->next;
    p->next = pnext;
    head->next = p;
}
//调用前请先判断栈是否为空 
//注意只有一个节点的情况
Element pop(STACK stack)
{
    node* head = stack;
    node* top = head->next;
    node* topnext = top->next;
    Element e = top->element;
    head->next = topnext;
    free(top);
    printf("pop:%d \n",e);
    return e;
}

void print_stack(STACK stack)
{
    node* head = stack;
    if(!is_exist(head))return;
    if(is_null(head))return;
    node* p = head->next;
    while(p)
    {
        printf("element: %d\n",p->element);
        p = p->next;
    }

}
int main()
{
    STACK stack = init_stack();
    push(stack,1);
    push(stack,3);
    push(stack,5);
    push(stack,7);

    printf("pirnt1 \n");
    print_stack(stack);

    push(stack,9);
    push(stack,11);
    //pop前要判断是否为空
    pop(stack);
    pop(stack);
    printf("pirnt2 \n");
    print_stack(stack);

    destroy_stack(stack);
    return 0;
}
```