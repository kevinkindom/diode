---
title: "两数相加"
date: 2020-06-22T15:54:04+08:00
draft: true
---

https://leetcode-cn.com/problems/add-two-numbers/

答案：
```javascript
function ListNode(val) {
  this.val = val;
  this.next = null;
}

/**
 * @param {ListNode} l1
 * @param {ListNode} l2
 * @return {ListNode}
 */
var addTwoNumbers = function(l1, l2) {
  let result = new ListNode(0)
  let current = result
  let carry = 0

  while (l1 || l2) {
    let a = l1 ? l1.val : 0
    let b = l2 ? l2.val : 0

    let sum = a + b + carry
    carry = sum >= 10 ? 1 : 0
    sum = sum % 10

    current.next = new ListNode(sum)

    current = current.next
    l1 && (l1 = l1.next)
    l2 && (l2 = l2.next)
  }

  if (carry === 1) {
    current.next = new ListNode(carry)
  }

  return result.next
};
```

思路：
两个链表长度可能都不一样，但由于是倒序加的，所以不用做位数对齐，只要链表有val就可以加，如果没有val可以用0表示，同时记录一个carry，用于做进位
最后要的结果是一个新的链表，所以开头我们就可以新建出这个链表
循环的时候因为要不断的往链表后面加节点，所以需要新建个变量current指向当前操作的节点
添加完成后，根节点固定是为0的，所以最终返回的是result.next