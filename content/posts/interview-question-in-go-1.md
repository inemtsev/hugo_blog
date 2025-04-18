---
title: "Go Interview Question #1 - Find all the prime numbers less than or equal to 'n'"
date: 2020-05-06T13:10:43+07:00
draft: false
tags: ["interview","golang","number theory"]
sitemap: 
    priority: 0.5
---

As a means of keeping my logic sharp and helping others prepare for **Go** interviews I decided to release a solved interview question every week or so. 

In this post, let's write a function in Go that generates us all the primes at or below a specific integer.

To even attempt this question, we need to know the exact definition of a **prime number**. According to [Wolfram](https://mathworld.wolfram.com/PrimeNumber.html), *A prime number is a positive integer p>1 that has no positive integer divisors other than 1 and p itself*. In short, a number that can only be divided by itself and 1 and is greater than 1. 

1. **Basic and dirty approach:** We can make a loop that iterates **i** from **2** to **n** inclusively and inside this loop we iterate **j** from **n** down to **2** checking whether **i** can be devided by anything less than itself. This approach is easy to come up with, I just followed the basic definition of a prime. Unfortunately, while fast to code, this approach is quite slow to run. We are looking at complexity of O(n^2) here.

2. **Very efficient approach:** Staring at the above solution for a few minutes, we can see where the inefficiency lies. We seem to be checking divisibility of many unnecessary numbers. We can use the [Fundamental Theorem of Arithmetic](https://en.wikipedia.org/wiki/Fundamental_theorem_of_arithmetic) to significantly increase the efficiency of our solution, which states: *every integer greater than 1 either is a prime number itself or can be represented as the product of prime numbers and that, moreover, this representation is unique*. Wait a second, if every non prime number has to be divisible by primes, we can apply this to our solution.

We will still have an outer loop that iterates **i** from **2** to **n**. But, the *inner loop* does not need to divide by all numbers less than itself, we can simply divide by all the primes we have found so far and that would be enough. Let's code that in Go :)

```go
package main

import "fmt"

func main() {
        maxPrime := 10000
        fmt.Printf("Printing primes less than %v", maxPrime)
        fmt.Println()
        primes := getPrimes(maxPrime)
        fmt.Println(primes)
}

func getPrimes(maxNum int) []int {
        primes := make([]int, 0, maxNum)        // create a slice big enough for all potential prime numbers

        for i := 2; i <= maxNum; i++ {  // go through all the numbers less than the input
                isPrime := true
                for _, p := range primes {      // check the number against all the primes we found so far
                        if i % p == 0 {
                                isPrime = false // if it fails the test, no need to check further
                                break
                        }
                }

                if isPrime == true {
                        primes = append(primes, i)      // if it passes the test, we add it to our list of primes
                }
        }

        return primes
}
```

<iframe src="https://giphy.com/embed/3o7qE8cQUUCsrRQ1ry" width="480" height="480" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/rockthisboat-rockthisboat-premiere-season2-3o7qE8cQUUCsrRQ1ry">via GIPHY</a></p>