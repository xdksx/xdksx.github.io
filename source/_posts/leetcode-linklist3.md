---
title: leetcode_linklist3
date: 2018-06-07 23:33:03
tags: leetcode_linklist
categories: leetcode
---
### leetcode_linklist3
continue..
#### 反转链表
Given a linked list, rotate the list to the right by k places, where k is non-negative.
<!--more-->
Example 1:

Input: 1->2->3->4->5->NULL, k = 2
Output: 4->5->1->2->3->NULL
Explanation:
rotate 1 steps to the right: 5->1->2->3->4->NULL
rotate 2 steps to the right: 4->5->1->2->3->NULL

Example 2:

Input: 0->1->2->NULL, k = 4
Output: 2->0->1->NULL
Explanation:
rotate 1 steps to the right: 2->0->1->NULL
rotate 2 steps to the right: 1->2->0->NULL
rotate 3 steps to the right: 0->1->2->NULL
rotate 4 steps to the right: 2->0->1->NULL
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     struct ListNode *next;
 * };
 */
 
```c
struct ListNode* rotateRight(struct ListNode* head, int k) {
    if(head==NULL)
        return NULL;
    if(head->next==NULL)
        return head;
    struct ListNode *headnode=(struct ListNode*)malloc(sizeof(struct ListNode));
    headnode->next=head;
    struct ListNode *sumhead=head;
    //compute len of list
    int sum=0;
    while(sumhead->next!=NULL)
    {
        sum++;
        sumhead=sumhead->next;
    }
    sum++;
    //compare k and len,or just compute the rota num
    int num=k%sum;
    int i=0;
    struct ListNode *dealheadf=head;
    for(i=1;i<sum-num;i++)
    {
        dealheadf=dealheadf->next;
    }
    
    sumhead->next=headnode->next;
    headnode->next=dealheadf->next;
    dealheadf->next=NULL;
   
    head=headnode->next;
    headnode->next=NULL;
    free(headnode);
    return  head;
}
```





#### 移除倒数第n个元素
Given a linked list, remove the n-th node from the end of list and return its head.

Example:

Given linked list: 1->2->3->4->5, and n = 2.

After removing the second node from the end, the linked list becomes 1->2->3->5.

Note:

Given n will always be valid.

Follow up:

Could you do this in one pass?

/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     struct ListNode *next;
 * };
 */
 
```c
struct ListNode* removeNthFromEnd(struct ListNode* head, int n) {
     if(head==NULL)//这个几乎没道题都要注意
         return head;
     if(head->next==NULL&&n>=1)
         return NULL;
     
     int sum=0;
     struct ListNode *sumhead=head;
     while(sumhead!=NULL)
     {
         sum++;
         sumhead=sumhead->next;
     }
     int remove=sum-n;
     struct ListNode *removenode=head;
     while(remove>1)
     {
         removenode=removenode->next;
         remove--;
     }
     struct ListNode *rmnode;
    if(remove==1)
    {
     struct ListNode *rmnode;
     rmnode=removenode->next;
     removenode->next=removenode->next->next;
     rmnode->next=NULL;
     free(rmnode);
    }
    else//删除头
    {
        rmnode=removenode;
        head=head->next;
        rmnode->next=NULL;
        free(rmnode);
        
    }
     return head;
    
}
```
//一开始未考虑到删除头的情况，所以加了else 部分

#### 交换元素
Given a linked list, swap every two adjacent nodes and return its head.

Example:

Given 1->2->3->4, you should return the list as 2->1->4->3.

Note:

    Your algorithm should use only constant extra space.
    You may not modify the values in the list's nodes, only nodes itself may be changed.




/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     struct ListNode *next;
 * };
 */
 
```c
struct ListNode* swapPairs(struct ListNode* head) {
    if(head==NULL)
        return head;
    if(head->next==NULL)
        return head;
    struct ListNode *headNode=(struct ListNode*)malloc(sizeof(struct ListNode));
    struct ListNode *second=head->next;
    struct ListNode *first=head;
    struct ListNode *curhead=headNode;
    while(1)
    {
        if(first==NULL)break;
        second=first->next;
        if(second!=NULL)
       {
          first->next=second->next==NULL?NULL:second->next;
          second->next=first;
          curhead->next=second;
          curhead=first;
          first=first->next==NULL?NULL:first->next;
        }
        else break;
    }
    head=headNode->next;
    headNode->next=NULL;
    free(headNode);
    return head;
     
}
```

#### k组反转

Given a linked list, reverse the nodes of a linked list k at a time and return its modified list.

k is a positive integer and is less than or equal to the length of the linked list. If the number of nodes is not a multiple of k then left-out nodes in the end should remain as it is.

Example:

Given this linked list: 1->2->3->4->5

For k = 2, you should return: 2->1->4->3->5

For k = 3, you should return: 3->2->1->4->5

Note:

    Only constant extra memory is allowed.
    You may not alter the values in the list's nodes, only nodes itself may be changed.
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     struct ListNode *next;
 * };
 */
/*
这道题目原想在遍历一次就解决，可是还是想不出好的方法，得遍历两次，一次计算长度，一次正真的处理；
在处理的过程使用头插法。注意边界条件，但因为有数量控制好些*/
```c
struct ListNode* reverseKGroup(struct ListNode* head, int k) {
    if(head==NULL)
        return head;
    struct ListNode* headnode=(struct ListNode*)malloc(sizeof(struct ListNode));
    headnode->next=head;
    int len=0;
    struct ListNode  *lenhead=head,*curhead=headnode,*cur=head,*tmp=NULL;
    while(lenhead!=NULL)//计算长度
    {
        len++;
        lenhead=lenhead->next;
    }
    int numofreverse=len/k;//要reverse几次
    int i=0;
    int j=0;
    for(i=1;i<=numofreverse;i++)
    {
        j=k;
        while(j>1)//每一次reverse k次，头插法
        {
            tmp=curhead->next;
            curhead->next=cur->next;
            cur->next=cur->next->next;
            curhead->next->next=tmp;
            j--;
        }
        curhead=cur;
        cur=cur->next;
    }
    head=headnode->next;
    headnode->next=NULL;
    free(headnode);
    return head;   
}
```
#### 总结
后面还有几道题，不贴了，这里简述下：  
+ 检查是否链表中存在循环
+ 检查链表中是否存在循环并找到循环的起点
+ 深度复制链表，链表中的每个节点存在一个指向任意节点的指针
+ 设计一个LRU cache,即（最近使用的）
+ 。。。。。

+ 链表的套路：
  + 常使用头插法进行反转操作
  + 使用两个指针解决循环问题，指针相遇测试是否存在循环，指针遍历减少时间
  + 使用哈希，以空间换时间
  + 加头节点，简化逻辑
+ 使用链表注意
  + 检查空和是否只有一个节点
  + 释放空间，和放置取空指针，可以通过次数控制和判空