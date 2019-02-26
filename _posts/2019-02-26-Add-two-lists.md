---
layout: post
title:  "2. Add Two Lists"
categories: leetcode
tags:  leetcode
author: wenzhilee77
---

## Add Two Numbers

You are given two non-empty linked lists representing two non-negative integers. The digits are stored in reverse order and each of their nodes contain a single digit. Add the two numbers and return it as a linked list.

You may assume the two numbers do not contain any leading zero, except the number 0 itself.


## Example

```
Input: (2 -> 4 -> 3) + (5 -> 6 -> 4)
Output: 7 -> 0 -> 8
Explanation: 342 + 465 = 807.
```


## Solution

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution 
{
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) 
    {
        ListNode head = null, tail = null;
        
        int carry = 0;
        
        while(l1 != null && l2 != null)
        {
            if(tail == null)
                head = tail = new ListNode((l1.val + l2.val + carry)%10);
            else 
            {
                tail.next = new ListNode((l1.val + l2.val + carry)%10);
                tail = tail.next;
            }
            carry = (l1.val + l2.val + carry)/10;
            l1 = l1.next;
            l2 = l2.next;
        }
        
        while(l1 != null)
        {
            tail.next = new ListNode((l1.val + carry)%10);
            tail = tail.next;
            carry = (l1.val + carry)/10;
            l1 = l1.next;
        }
        
        while(l2 != null)
        {
            tail.next = new ListNode((l2.val + carry)%10);
            tail = tail.next;
            carry = (l2.val + carry)/10;
            l2 = l2.next;
        }
        
        if(carry == 1)
            tail.next = new ListNode(1);
        
        return head;
    }
}
```


## Tips

* l1 = l1.next; l2 = l2.next;

* carry = (l1.val + l2.val + carry)/10;

* if(carry == 1) tail.next = new ListNode(1);

* if(tail == null) head = tail = new ListNode((l1.val + l2.val + carry)%10);

