---
title: leetcode_linklist2
date: 2018-06-03 21:40:41
tags: leetcode_linklist
categories: leetcode
---
### leetcode——单链表2
#### partition list
这个是快速排序中一个很重要的步骤，即比该数大于等于的放其前小的放后。在链表中，基本思路:扫描链表，比给定数字小的不处理，比给定数字大的取下并插入到末尾 <!--more-->
```c
/*
Given a linked list and a value x, partition it such that all nodes less than x come before nodes greater than or equal to x.

You should preserve the original relative order of the nodes in each of the two partitions.

Example:

Input: head = 1->4->3->2->5->2, x = 3
Output: 1->2->2->4->3->5*/

/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     struct ListNode *next;
 * };
 */
struct ListNode* partition(struct ListNode* head, int x) {
    /**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     struct ListNode *next;
 * };
 */

    if(head==NULL)
				return NULL;
    if(head->next ==NULL)
              return head;
        struct ListNode *list=(struct ListNode*)malloc(sizeof(struct  ListNode));
        list->next=head;
		struct ListNode*head1=list;
		struct ListNode*cur = head1->next;
		struct ListNode*tail=head1;
		int lenoflist1=0;
		while(tail->next!=NULL)
		{
				tail=tail->next;
				lenoflist1++;
		}
       struct ListNode* tmp=tail;
		while(lenoflist1>=1)
		{ 
            lenoflist1--;
            if(tail==cur)
                continue;
          if(cur->val>=x)
		  {
				  head1->next=cur->next;
				  cur->next=NULL;
		          tail->next=cur;
				  tail=tail->next;//not consid at first
				  cur=head1->next;

		  } 
		  else{
				  head1=head1->next;
				  cur=cur->next;
		  }
		 
		 //  printf("%d : ",cur->num);
		}
    //if(head1->next==NULL)head1->next=tmp;
         head=list->next;
   list->next=NULL;
    free(list);
		printf("\n");
    return head;
    
}
```
//此题目最终被accepted
//应该在函数中添加头节点而不是依靠传入的链表带头节点,一开始没有考虑到传入的链表不带链表头。。，没有考虑到一个元素的情况，这个方案最终accept


#### 链表中的子链表反转，考察头插法
+ 头插法在链表的反转，倒序，常被用到  

```c 
/* reverse a linklist from m to n
 * 1->2->4->5->null and m,n(ex:2,4)
 * return 1->5->4->2->null
 */

#include<stdlib.h>
#include<stdio.h>
typedef struct LinkList
{
	int num;
	struct LinkList  *next;
}Linklist;
int reverselinklist(Linklist *list1,int m,int n,Linklist **result)
{
		if(list1==NULL)
				return -1;
		int i=0;
		Linklist *head1,*head2,*cur,*tmp,*pre;
		cur=list1;
		head1=cur;
		pre=head1;
		if(m==1)
		{
				cur=cur->next;
				for(i=m;i<n;i++)
				{
				  pre->next=cur->next;
				  cur->next=head1;
				  head1=cur;
				  cur=pre->next;
				}
			    *result=head1;
				return 1;
		}

		for(i=1;i<m;i++)//node1->node2->node3---1->2->3--ex:m=2,list1->1
		{
		  head1=cur;
          cur=cur->next;
		}
		head2=cur;
		cur=cur->next;
		pre=head2;
		for(i=m;i<n;i++)
		{
		   pre->next=cur->next;
		   cur->next=head1->next;
		   head1->next=cur;
		   cur=pre->next;
		}
      return 3;
}

int main ()
{
		Linklist *list1=(Linklist*)malloc(sizeof(Linklist));
		Linklist *result11=list1;
		int i;
		list1->num=4;
		printf("4 ");
       for(i=1; i<9;i++)
	  {
						     
			list1->next = (Linklist *)malloc(sizeof(Linklist));
			list1 = list1->next;
		    list1->num = i*2;
			printf("%d ",i*2);
	  }
	   printf("\n");
	   Linklist *rr=(Linklist *)malloc(sizeof(Linklist));
	   Linklist **resull=&rr;
	   int rere=reverselinklist(result11,1,9,resull);
	   if(rere==3) {
	    for(i=0;result11!=NULL ;i++,result11=result11->next)
				       printf("%d ",result11->num); 
	  	free(list1);
	   }
	  else
	  { 
	    for(i=0;*resull!=NULL ;i++,*resull= (*resull)->next)
				       printf("%d ",(*resull)->num); 
	  	free(*resull);
	  }
		return 0;
}```
//此解法未经过leetcode检验，不过应该问题不大

#### 有序链表移除重复元素
+ 考察的是遍历链表删除链表中的元素，注意边缘条件控制如头部和尾部

```c
/**

 Given a sorted linked list, delete all duplicates such that each element appear only once.

Example 1:

Input: 1->1->2
Output: 1->2

Example 2:

Input: 1->1->2->3->3
Output: 1->2->3

 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     struct ListNode *next;
 * };
 */
struct ListNode* deleteDuplicates(struct ListNode* head) {
    if(head==NULL)return NULL;
    if(head->next==NULL)return head;
    struct ListNode *first=head;
    struct ListNode  *second=head->next;
    struct ListNode *tmp=second;
    while(second!=NULL) 
    {
        tmp=second;
        if(first->val==second->val)
        {
            second=second->next;
            
        }
        else
        {
            first->next=second;
            first=first->next;
            second=second->next;
        }
    }
    if(first->val==tmp->val)
        first->next=NULL;
    return head;
}```
//此方案最后被accepted

#### 删除有序链表中的有重复的node
+ 和上一道题目类似:

```c
/**

 Given a sorted linked list, delete all duplicates such that each element appear only once.

Example 1:

Input: 1->1->2
Output: 1->2

Example 2:

Input: 1->1->2->3->3
Output: 1->2->3

 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     struct ListNode *next;
 * };
 */
#include<stdio.h>
#include<stdlib.h>
struct ListNode {
      int val;
      struct ListNode *next;
  };
struct ListNode* deleteDuplicates(struct ListNode* head) {
    if(head==NULL)return NULL;
    if(head->next==NULL)return head;
    struct ListNode *headnode=(struct ListNode*)malloc(sizeof(struct ListNode));
    headnode->next=head;
    
    struct ListNode *first=head;
    struct ListNode  *second=head->next;
    int numsame=0;
    struct ListNode *tmphead=headnode;
    while(second!=NULL) 
    {
        if(first->val==second->val)
        {
               numsame++;
               first=first->next;
               second=second->next;
               
        }
        else if(first->val !=second->val && numsame==0)
        {
            tmphead->next=first;
            tmphead=tmphead->next;
            first=first->next;
            second=second->next;
        }
        else 
        {
            first=first->next;
            second=second->next;
            numsame=0;
        }
    }
    if(numsame==0)tmphead->next=first;
    else tmphead->next =NULL;
    head=headnode->next;
    headnode->next=NULL;
    free(headnode);   
    return head;
}


int main()
{
   struct ListNode *list1=(struct ListNode *)malloc(sizeof(struct ListNode));
   struct ListNode *tmp=list1;
   struct ListNode *freelist1=list1;
   int i=0;
   for(i=1;i<9;i++)
   {
		 list1->val=20-i;
		 list1->next=(struct ListNode *)malloc(sizeof(struct ListNode));
		 list1=list1->next;
		 printf("%d ",20-i);
   }
   list1->val=12;
   tmp=deleteDuplicates(tmp);
   printf("\n");
   for(i=1;tmp!=NULL;i++)
   {
		   printf("%d  ",tmp->val);
		   tmp=tmp->next;
   }
   free(freelist1);
   return 0;
}


这道题做的比较顺利，一次提交就=通过了
注意：加头节点，由于解题的思路是两个指针的遍历，在第二个指针为空时停止，就会有尾巴为几种情况时的处理：如当尾巴为：4,4,4;4,5,5 这两种情况时，去掉尾巴，否则4,5,6.4,4,5时尾巴保留；见代码```
