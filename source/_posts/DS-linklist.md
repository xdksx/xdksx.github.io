---
title: DS_linklist
date: 2018-05-26 21:18:09
tags: 数据结构
categories: 数据结构和算法
---
## 数据结构之线性表：
### 有序表：数组：
### 单链表：
链表定义  <!-- more -->
{  
 * 数据成员:常见的基本类型或者对象类型均可  
 * 数据关系:在链表中的块的成员在连续的内存中，块与块之间内存位置不连续  
 * 指向块的指针：单链表只有一个next,双链表加上pre  

}  
基本运算：  
{
+ InitList(&L);:初始化链表，在c/c++中需要做分配内存，初始化链表头和其值的操作；  
+ DestroyList(&L); 在销毁时需要free内存  
+ Length(L);链表的长度是块的个数
+ GetElem(L,i,&e); 在链表中取元素需要一个一个遍历直到下标值，若为双链表会快些。所以这里取索引值对应的值和取链表的值，效率几乎一样；  
+ LocateElem(L,e,compare()); 和链表中的元素做对比
+ InsertElem(&L,i,e); 在链表中插入元素，复杂度O(1),在提供位置的时候，若不提供位置，还需要加搜索的时间
+ DeleteElem(&L,i,&e); 在链表中删除元素，复杂度O(1)
……  

}  
eg:  
```c
typedef struct LNode { 
       ElemType data；//数据域
       struct LNode  *next； //指针域
} LNode,  *LinkList;
LNode  *L;
LinkList  L;
L =  (LinkList) malloc( sizeof (LNode) );
或 L = new LNode;
L->data;
LNode  L;
L.date
```
{% asset_img linklist.png 链表示意图 %}    
### 链表的两种头部：
1. 没有头的链表：第一个块就开始存储数据  
2. 任何时候都有头的链表：  第一个块不存储数据，作为头节点，第二个块开始存数据  
3. 应用：在leetcode中的ReverseLinklist中，当反转的是从1开始的时候，若使用的是没有头的，则需要额外判断处理，否则可以统一处理；  

{% asset_img headnode.png 头节点示意图 %}  

###  链表的几个常见操作：　
+ 取第i个元素：  
```c
Status GetElem_L(LinkList L, int i, ElemType &e)
{//查找操作
    p = L->next;  
    j = 1;
    while( p && j < i){
          p = p->next; 
          ++j;
    }
    if (!p || j>i) return ERROR;
    e = p->data;
    return OK;
}
```
+ 插入元素：在第i个位置上插入    
{% asset_img insert.png 插入示意图 %}    
```c
Status ListInsert_L(LinkList &L, int i, ElemType e)
{
  p = L; j = 0;
     while (p && j < i-1) { p = p->next; ++j }
     if (!p || j>i-1)  return ERROR;
  s =  (LinkList) malloc( sizeof (LNode) );
     s->data = e;  
  s->next = p->next;  
  p->next = s;  
     return OK;
     }
 ```
+ 删除元素:删除第i个元素:  
{% asset_img delete.png 删除示意图 %}  
```c
 Status ListDelete_L(LinkList &L, int i, ElemType &e)
{
 p = L; j = 0;
     while (p->next && j < i-1) { p = p->next; ++j }
     if (!(p->next) || j>i-1)  return ERROR;
 q = p->next;
     e = q->data;  
 p->next = p->next->next;  //(p->next = q->next;)
 free(q);  
  return OK;
}
```

### 链表的建立：   
+ 头插法：  
{% asset_img headbuild.png 头插法示意图 %}  
```c
CreateList_L(LinkList &L, int n)
{
     L = (LinkList) malloc( sizeof (LNode) );
     L->next = NULL;
     for( i=n; i>0; --i){
         s = (LinkList) malloc( sizeof (LNode) );
         scanf( &s->data);
         s->next = L->next; ①
         L->next = s; ②
     }
}```
+ 尾插法：  
{% asset_img tailbuile.png 尾插法示意图 %}  
```c
CreateList_L(LinkList &L, int n)
{
     tail = L = (LinkList) malloc( sizeof (LNode) );
     L->next = NULL;
     for( i=n; i>0; --i){
         s = (LinkList) malloc( sizeof (LNode) );
         scanf( &s->data);
         tail->next = s; ①
         tail = s; ②
     }      }
```

###  链表的常见复杂操作：
+ 两个有序链表的合并：
```c
void MergeList_L(LinkList &La, LinkList &Lb, LinkList &Lc)
{
    pa = La->next; pb = Lb->next;
    Lc = pc = La;
    while( pa && pb ){
        if(pa->data <= pb->data){
             pc->next = pa; pc = pa; pa = pa->next;
       }
       else{
             pc->next = pb; pc = pb; pb= pb->next;
       }
    }
    pc->next = pa ? pa : pb;
    free( Lb );
} 
```

### 一些特殊的链表：
+ 单向循环链表：
+ 图示：  
{% asset_img sigrecyclelink.png 单向循环链表 %}
{% asset_img mergerecycle.png 合并 %}
+ 多重循环链表：
+ 双向链表：
```c
typedef struct DuLNode{
     ElemType               data;
     struct DuLNode    *prior;
     struct DuLNode    *next;
}DuLNode,  *DuLinkList;
```
双向循环链表：

### 探讨：
+ 链表作为最基础的数据结构，是对其他数据结构：如栈，树等的支撑，其他这些大部分是在链表的基础上建立的；  
+ 从较底层的层面：即深入到内存，链表实现了对零散内存的有效使用，内存在为链表分配空间的时候不用大段的连续空间；
+ 从较高层，即抽象的层面看，链表仅仅是一批数据的拓扑结构之一，就常见的栈，队列，树，图等等，你还可以想出其他可能有用没用的，如，立体的结构，方块，金字塔形(emm这个像堆），还有随意的，感觉想到了矩阵里面的压缩。   
 
### 应用： 
+ 链表的应用：如
+ 在文件中，对大文件的存储，采用类似链表的结构，
+ 大数相加，像两个100位的整数相加，使用链表，每个块为一位，就可以解决这个问题，数组也行
+ 倒排索引结构，在操作系统中用到，我的github上放着一个本科毕业设计做的简单搜索引擎，几乎全是自己搞的（解析网页那块自己c写了状态机，觉得效果不如beatifulsoup好，后面用它了),不过想想那会还不会很有意识的去想如果让它运行在大量用户下，各种使用条件（如输入条件下，会怎么样，年轻啊~)
+ 其他，当然是其他数据结构基于链表做的，多了去了
 
