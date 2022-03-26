---
title: leetcode_linklist1
date: 2018-06-03 21:37:57
tags: leetcode_linklist
categories: leetcode
---
### leetcode——单链表

#### 两数相加：
+ 这道题是经一年多没刷题目之后，重新开始的第一道题，本来想着只是本地刷刷，后面开始就到leetcode提交了，毕竟写的程序还是得经过检验
+ 所以这道题就纯粹的解答，未经检验，只是本地跑通，并且加上了自己的一些看法，比如对该题目该思路可能可以应用在哪些地方，这样算法才有了真正的意义，个人十分不赞同一些在公司呆久了的人说数据结构算法没什么用的人2333
+ 废话不多说：  好久没写，第一道就别吐槽了，慢慢来
+ 题目描述：  <!--more-->
You are given two non-empty linked lists representing two non-negative integers. The digits are stored in reverse order and each of their nodes contain a single digit. Add the two numbers and return it as a linked list.  
You may assume the two numbers do not contain any leading zero, except the number 0 itself.
Example
Input: (2 -> 4 -> 3) + (5 -> 6 -> 4)
Output: 7 -> 0 -> 8
Explanation: 342 + 465 = 807.
注意两个数字位数可能不同，所以需要一些特殊情况要处理
```c
#include<stdio.h>
#include<stdlib.h>```

```c
struct ListNode {
		struct ListNode *next;
		int num;
} *linklist,listnode;

//这是自己加的扩展，把输入的两个大数字符串转换为链表
int changtolist(struct ListNode *list1,char num1[],struct ListNode *list2,char  num2[])
{//应该在接口内计算长度好些
   int i=0;
   int lennum1=0;
   while(num1[lennum1]!='\0'){lennum1++;}
   int lennum2=0;
   while(num2[lennum2]!='\0'){lennum2++;}
//分配空间加字母转数字，无头节点
   list1->num=num1[lennum1-1]-48;
   for(i=lennum1-2;i>=0;i--)
   {
		list1->next=(struct ListNode*)malloc(sizeof(struct ListNode));
		list1=list1->next;
        list1->num=num1[i]-48;
   }
   list1->next=NULL;
   list2->num=num2[lennum2-1]-48;
   for(i=lennum2-2;i>=0;i--)
   {
		list2->next=(struct ListNode*)malloc(sizeof(struct ListNode));
		list2=list2->next;
		list2->num=num2[i]-48;
   }
   list2->next=NULL;
   return 0;
}
//两个大数相加，不用头节点的方式，麻烦一些
int Add_two_num(struct ListNode *list1,struct ListNode *list2)
{
	 int adding = 0;
     if(list1 == NULL || list2 == NULL)
			 return -1;
/*	 while(list1!=NULL && list2!=NULL)
	 {
			 list1->num = (list1->num+list2->num+adding)%10;
             adding = (list1->num + list2->num+adding)/10;
			 list1 = list1->next;
			 list2 = list2->next;
	}
	 if(list1==NULL && list2!=NULL)
	*/		 
	  struct ListNode *xx=list1;
      int n=0;
	  int sum=0;
	  sum = list1->num+list2->num+adding;
	  list1->num = sum%10;
      adding = sum/10;
	do {//常规情况，两个同长度部分
			 list1 = list1->next;
			 list2 = list2->next;
			 sum= list1->num+list2->num+adding;
			 list1->num = sum %10;
			// printf("%d ",list1->num);
             adding = sum/10;
    }while(list1->next!=NULL && list2->next!=NULL);
	if(list1->next ==NULL&& list2->next==NULL &&adding!=0)//串1短于串2
	{
			list1->next = (struct ListNode *)malloc(sizeof(struct ListNode));
			list1->next->num= adding;
			printf("show:%d\n",adding);
	}
	if(list1->next==NULL && list2->next !=NULL)
	{

			list1->next = list2->next;
			while(list2->next!=NULL&&adding !=0)
			{
              list2=list2->next;
			  sum= list2->num+adding;
              list2->num=sum%10;
			  adding = sum/10;
			}
			if(adding>0)
			{
					list2->next=(struct ListNode *)malloc(sizeof(struct ListNode));
		            list2->next->num=adding;
			}
	}  
    ...//串1长于 串2
    .....
  	return 0;

}
//需要思考，当进位到最后时，可能1,2都结束了，但是产生高位，此时要new　listnode 完成
//或者剩下２，和进位，则考虑２加进位
int main ()
{
  struct ListNode *list1,*list2;
  list1 = (struct ListNode*)malloc(sizeof(struct ListNode));
  list2 = (struct ListNode*)malloc(sizeof(struct ListNode));
  list1->num=3;
  list2->num=5;
  struct ListNode *tmplist1=list1;
  struct ListNode *tmplist2=list2;
  int i;
  for(i=1; i<9;i++)
  {
    
	list1->next = (struct ListNode*)malloc(sizeof(struct ListNode));
	list2->next = (struct ListNode*)malloc(sizeof(struct ListNode));
	list1 = list1->next;
	list2 = list2->next;
    list1->num = 2;
	list2->num= 8;
  }
  list2->next = (struct ListNode*)malloc(sizeof(struct ListNode));
  list2->next->num=9; 
  struct ListNode *result=tmplist1;
  struct ListNode *freelist1 = tmplist1;
  struct ListNode *freelist2 = tmplist2;
  for (i=0;i<9;i++,tmplist1=tmplist1->next)
		  printf("%d ",tmplist1->num);
  printf("\n");

  for (i=0;i<10;i++,tmplist2=tmplist2->next)
		  printf("%d ",tmplist2->num);
  printf("\n");
  Add_two_num(freelist1,freelist2);
  for(i=0;result!=NULL ;i++,result=result->next)
       printf("%d ",result->num); 
  free(freelist1);
  free(freelist2);


  printf("\n");
//-------------------------------
  struct ListNode *list11,*list22;
  list11 = (struct ListNode*)malloc(sizeof(struct ListNode));
  list22 = (struct ListNode*)malloc(sizeof(struct ListNode));


  char num1[100],num2[100];
  gets(num1);
  gets(num2);
  changtolist(list11,num1,list22,num2);
  struct ListNode *result11 = list11;
  Add_two_num(list11,list22);
  for(i=0;result11!=NULL ;i++,result11=result11->next)
       printf("%d ",result11->num); 
  free(list11);
  free(list22);
  return 0;
}```
//思考，扩大这道题的应用，在大数相加的时候，增加用户输入接口:"
//5263565656554+5656537677834546
//由char读入，int/char相加，
-------极其丑的程序，以后不能这么搞，留个纪念
想贴下两年多前（大三吧，那会刷这道题通过的程序--看来最喜欢的还是c）
```c
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */

 class Solution {
 public:
	 ListNode* addTwoNumbers(ListNode* l1, ListNode* l2) {
		 ListNode *sum, *l4;
		 sum = new ListNode(0); //新节点
		 l4 = sum;
		 int sum_single, en = 0;//en表示进位的标志
		 while (l1 != NULL&&l2 != NULL)
		 {
			 
				 sum->next = new ListNode(0);//这里有个问题变为sum=NULL就行
				 sum = sum->next;
			// signal = 0;
			 sum_single = l1->val + l2->val + en;
			 if (sum_single<10)
			 {
				 sum->val = sum_single;
				 en = 0;
			 }
			 else if (sum_single >= 10)
			 {
				 sum->val = sum_single - 10;
				 en = 1;
			 }
			 l1 = l1->next;
			 l2 = l2->next;
		
		 }
		 while (l1 != NULL&&l2 == NULL)
		 { 
			 sum->next = new ListNode(0);
			 sum = sum->next;
			 sum_single = en + l1->val;
			 if (sum_single >= 10)
			 {
				 sum->val = sum_single - 10;
				 en = 1;
			 }
			 else
			 {
				 sum->val = sum_single;
				 en = 0;
			 }
			 l1 = l1->next;
			

		 }
		 while (l2 != NULL&&l1 == NULL)
		 { 
			 sum->next = new ListNode(0);
			 sum = sum->next;
			 sum_single = en + l2->val;
			 if (sum_single >= 10)
			 {
				 sum->val = sum_single - 10;
				 en = 1;
			 }
			 else
			 {
				 sum->val = sum_single;
				 en = 0;
			 }
			 l2 = l2->next;
		 }
		 if (l1 == NULL&&l2 == NULL&&en == 1)
			 sum->next = new ListNode(en);
		//if(l1==NULL&&l2==NULL&&en==0)sum = NULL;
	/*	 while (l3)
		 {
			 cout << l3->val;
			 l3 = l3->next;
		 }*/
		 return l4->next;
	 }
 };
/*
int main()
{
    Solution sou;
    ListNode *l1,*l2,*l3;
    l1=new ListNode(3);
    l1->next=new ListNode(7);
    l1->next->next=new ListNode(5);
    
     l2=new ListNode(7);
    l2->next=new ListNode(7);
    l2->next->next=new ListNode(5);
    l3=sou.addTwoNumbers(l1,l2);
    cout<<l3->val<<l3->next->val<<l3->next->next->val;
    return 0;
}



*/```