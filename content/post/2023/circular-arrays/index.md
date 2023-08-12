---
title: Circular Arrays
tags: [Arrays, Modulo]
author: "Yunier"
date: "2023-08-11"
description: "Using Modulo operator to create circular arrays."
---

Most developers are familiar with the [modulo opertor](https://en.wikipedia.org/wiki/Modulo), it is often presented in an example that determines if a number is odd or even as shown in the code below taken from [Testing whether a value is odd or even](https://stackoverflow.com/questions/6211613/testing-whether-a-value-is-odd-or-even).

```JavaScript
function isEven(n) {
   return n % 2 == 0;
}

function isOdd(n) {
   return Math.abs(n % 2) == 1;
}
```

Determining if a number is odd or even is just one use case for the modulo operator. Another use case that I learn a while back is that you can use the modulo operator to set a range, a boundary if you will, allowing you to rotate the array, this is because fundamentally A mod B is between 0 and B - 1 or another way to think about it, **0 <= A < B**.

For example, imagine an array of length 4 and an indexer that is incremented as we loop through the array.

```JavaScript
0 % 4 = 0
1 % 4 = 1
2 % 4 = 2
3 % 4 = 3
4 % 4 = 0 // Return back to zero.
5 % 4 = 1
6 & 4 = 2
7 & 4 = 3
8 & 4 = 0 // Again, returns to zero.
```

As you can see from the example above, the modulo operator created a boundary between 0 and 3, this technique is known as a [circular array](https://www.quora.com/What-is-a-circular-array-and-how-does-it-work). You can use a circular array if you are implementing [pagination](https://www.seoptimer.com/blog/what-is-pagination/) and need to return to the first page when all the pages have been visited.