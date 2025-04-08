---
title: "Go Interview Question #3 - Write a function that checks if brackets are balanced"
date: 2020-08-23T12:13:00+07:00
draft: false
tags: ["interview","golang","stack"]
sitemap: 
    priority: 0.5
---

Another very common interview question, is to check if brackets are balanced in a string. If you have never done any stack questions before, this can be tricky. But, once you see how a stack can be used, it becomes very straightforward. 

Let's look at what balanced brackets look like: "(())()", "((()))", "()"...

Unbalanced brackets: "(((", "())", "(()"...

A **stack** is like a can of Pringles, **the last chip that was put into it, is the first to come out.**

{{< figure src="pringles-can.png" alt="Yummy yummy!" loading="lazy" >}}

Since Golang does not implement stack structure out of the box, we can use a **slice** to simulate its behavior. To simulate the the push() function, we will simply use built-in [append()](https://godoc.org/builtin#append). For pop() we will have to use `s = s[:len(s)-1]`.

Now, the strategy should be more clear. If we see an opening bracket, we add it to the "stack". If we see a closing one, we try to pop it off the "stack". If at any point we cannot pop the chip off the stack, we know the brackets are not balanced. if there are any brackets left over in the stack after the loop is finished, the brackets are also not balanced. The resulting code: 

```go
func IsBalanced(text string) bool {
        isBalanced := true
        s := make([]rune, 0, len(text))

        for _, c := range text {
                if c == '(' {
                        s = append(s, c)
                } else if c == ')' {
                        if len(s) == 0 {
                                isBalanced = false
                                break
                        }
                        s = s[:len(s)-1]
                }
        }

        if len(s) != 0 {
                isBalanced = false
        }

        return isBalanced
}
```

There is a more complicated variation of this problem, where the brackers can be any of the following: {([])}
At first, this sounds much more complicated, but it really isn't. The only additional thing we need to do when we are trying to pop the last value, is check whether the last bracket in our "stack" is a match for our popping bracket. The easiest way is to use a map, resulting in: 

```go
func IsBalanced2(text string) bool {
        isBalanced := true
        bMap := map[rune]rune{
                '(': ')',
                '[': ']',
                '{': '}',
        }
        s := make([]rune, 0, len(text))

        for _, c := range text {
                if _, ok := bMap[c]; ok == true {
                        s = append(s, c)
                } else if len(s) == 0 {
                        isBalanced = false
                        break
                } else if bMap[s[len(s)-1]] != c {
                        isBalanced = false
                        break
                } else {
                        s = s[:len(s)-1]
                }
        }

        if len(s) != 0 {
                isBalanced = false
        }

        return isBalanced
}
```

