---
title: "整数反转"
date: 2020-06-22T16:07:02+08:00
draft: true
---

https://leetcode-cn.com/problems/reverse-integer/

答案：
```javascript
var reverse = function(x) {
  let result = 0
  while (x) {
    let last = x % 10

    if (result > 214748364 || (result === 214748364 && last > 7)) {
      return 0
    }

    if (result < -214748364 || (result == -214748364 && last < -8)) {
      return 0
    }

    result = result * 10 + last
    x = parseInt(x / 10)
  }
  return result
};
```

思路：
通过求余可以拿到尾数，如果不考虑2^32次方长度的溢出问题，那么直接一个循环就出来了
余数的问题也有两种方法可判断：
一种是判断最终反转转的数是否比2^32-1要大，以及比-2^32要小
另外一种是答案这种，正数部分2^31-1的值是2147483647，负数部分-2^32的值是-2147483648，所以除最后一位外，就是±214748364了，然后判断末尾是否为7和-8即可，符合这种情况的返回0就好