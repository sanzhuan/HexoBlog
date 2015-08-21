title: "Linus:利用二级指针删除单向链表"
date: 2015-05-20 20:27:03
tags: [linux,链表]
---

Linus大婶在slashdot上回答一些编程爱好者的提问，其中一个人问他什么样的代码是他所喜好的，大婶表述了自己一些观点之后，举了一个指针的例子，解释了什么才是core low-level coding。Linus举了一个单向链表的例子，但给出的代码太短了，一般的人很难搞明白这两个代码后面的含义。已经有几位牛人[[1]](http://wordaligned.org/articles/two-star-programming)[[2]](http://www.oldlinux.org/oldlinux/viewthread.php?tid=14575&extra=page%3D1)[[3]](http://coolshell.cn/articles/8990.html#more-8990)对代码作了解释：

<!-- more --> 

一般教科书上的单链表删除的代码非常易懂，代码如下：

``` C++ 
  typedef struct node 
  {  
    struct node *next;
    .... 
  } node;

  typedef bool(* remove_fn)(node const* v);

  // Remove all nodes from the supplied list for which the 
  // supplied remove function returns true.
  // Returns the new head of the list.

  node * remove_if(node * head, remove_fn rm)
  {
    for(node * prev = NULL,* curr = head; curr != NULL;)
  	{        
      node *const next = curr->next;
     	if(rm(curr))
     	{
     		if(prev)                
     			prev->next=next;
     		else                
     			head =next;            
     		free(curr);
     	}
     	else            
     		prev = curr;        
     	curr =next;
    }
     return head;
  }
```

但是个人觉得他们对下面代码的解释不是很详细，有点费解。因此在这详细解释下面的代码。

``` C++
void remove_if(node ** head, remove_fn rm)
{
    for(node** curr = head; *curr; )
    {
        node * entry = *curr;
        if(rm(entry))
        {
          *curr = entry->next;
          free(entry);
        }
        else
           curr = &entry->next;
    }
}
```
+ 删除节点不是表头的情况：举个例子，假设有A B C三个节点，A是当前节点，A的下一节点是B，B的下一节点是C。假设不删除A，那么执行第12行后，curr的值是A的next指针变量的地址。由于curr是个二级指针，那么*curr的值就是A的next变量的值，即*curr指向B节点。假设第二遍循环要删除节点B，由于*curr就是A->next,那么第8行的作用就是讲节点A的next内容变为节点C的地址，然后就可以释放节点B。

+ 删除节点是表头的情况，输入参数中传入head的二级指针，在for循环里将其初始化curr，然后entry就是*head(*curr)，我们马上删除它，那么第8行就等效于`*head = (*head)->next`，就是删除表头的实现，是不是很巧妙？

