---
title: "Go Interview Question #2 - Write a function to check if a string is a palindrome"
date: 2020-05-16T15:52:15+07:00
draft: false
tags: ["interview","golang","array"]
sitemap: 
    priority: 0.5
---

This question is very common on interviews, and while fairly straightforward, plenty developers mess it up on the interview. 

What is a **palindrome**? [Webster dictionary](https://www.merriam-webster.com/dictionary/palindrome) defines it as a word, sentence or number that reads the same forwards and backwards. Some examples are: dad, 1881, abba.

There are a few approaches to solving this problem, we can reverse the array of characters and compare the resulting string for example. The more efficient way of doing this is to compare the characters from the beginning and from the end one-by-one. This way we can quit comparison as soon as we find a non-matching character. 

Let's convert this to Go code :)
```go
package main

import "fmt"

func main() {
        fmt.Println(checkPalindrome("abba"))
        fmt.Println(checkPalindrome("Superman"))
        fmt.Println(checkPalindrome("racecar"))
        fmt.Println(checkPalindrome("madam"))
}

func checkPalindrome(testString string) bool {
        isPalindrome := true
        sLength := len(testString)

        for i := 0; i < sLength/2; i++ {
                currCharFwd := testString[i]    // character to compare going forward
                currCharBwd := testString[sLength-1-i]  // character to compare going from the end

                if currCharFwd != currCharBwd {
                        isPalindrome = false    // we found a non-match, it's not a palindrome
                        break   // no need to compare further
                }
        }

        return isPalindrome
}
```

<iframe src="https://giphy.com/embed/3o6MbmRKD8vNq7eSvC" width="480" height="366" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/season-10-the-simpsons-10x22-3o6MbmRKD8vNq7eSvC">via GIPHY</a></p>