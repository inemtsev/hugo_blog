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
{{< gist inemtsev c0910f101437ddfceaddf58bf4e116ea "main.go" >}}

<iframe src="https://giphy.com/embed/3o6MbmRKD8vNq7eSvC" width="480" height="366" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/season-10-the-simpsons-10x22-3o6MbmRKD8vNq7eSvC">via GIPHY</a></p>