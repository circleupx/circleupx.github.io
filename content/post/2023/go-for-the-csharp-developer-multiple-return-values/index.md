---
title: Go for the C# Developer - Multiple Return Values
tags: [Go]
author: "Yunier"
date: "2023-08-14"
description: Getting used to Go.
series: [Learning Go]
---

I've been spending the last few weeks learning [Go](https://go.dev/learn/) by reading [Learning Go](https://a.co/d/dlJyukR) by [Jon Bodner](https://www.amazon.com/stores/Jon-Bodner/author/B08SWGN5NN), so far I've been enjoying learning about Go, though there is still one thing I keep tripping over, in Go, you can return one or more values, for me Go is the first language that I have worked with that does that, in every other language I had to introduce a custom discriminating union to achieve what Go does natively. 

Take the following Go code.

```Go
func twoSum(nums []int, target int) []int {
    s := make(map[int]int)

    for idx, num := range nums {
        if pos, ok := s[target-num]; ok {
            return []int{pos, idx}
        }
        s[num] = idx
    }
    return []int{}
}
```

Most of it can be understood even by those that have never worked with Go, the part that I was having a rough time understanding was the if statement below.

```Go
if pos, ok := s[target-num]; ok {
    return []int{pos, idx}
}
```

I wasn't understanding how the variable "ok" could evaluate to true, I've worked so much with C# that my brain naturally tried to read the code as if it were C#, and in C# like many other objection-oriented languages you can only return one value. Looking at the official docs I found the following note under [Index Expression](https://go.dev/ref/spec#Index_expressions).

> An index expression on a map of type map[K]V used in an assignment statement or initialization of the special form yields an additional untyped boolean value. The value of ok is true if the key x is present in the map, and false otherwise.

In other words, in the example above if target minus sum yields a value, that value is assigned to the variable pos, and the untyped boolean value is assigned to the variable "ok", then the variable "ok" is evaluated.