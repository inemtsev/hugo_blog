---
title: "Make Multiple Api Calls At The Same Time With GoRoutines"
date: 2019-06-03T21:16:17+07:00
draft: false
imgPad: 14
tags: ["golang","api","concurrency"]
sitemap:
    priority: 0.5
---

Golang is efficient, very efficient. Much of this efficiency is attributed to its unique abstractions when dealing with concurrency. Java for example, maps its threads to OS threads, while Go uses its own goroutines scheduler to further abstract its lightweight goroutines from OS threads. In short, Golang is very frugal with how it uses OS threads; if a goroutine becomes blocked, Go's scheduler will switch in another goroutine in its place to keep the thread busy as much as possible. Since each CPU core handles a limited number of threads (and spawning new threads is expensive), keeping these threads fed with work is a great thing indeed. 

So, how do we use Golang to make multiple http calls concurrently? If you have used C# or modern JavaScript you may have used **async/await** to make multiple api calls. Golang isn't quite as easy, but it is all in the name of efficiency. Go always has at least one goroutine running, which takes care of running main(). We can spawn new routines with the keyword **go** before the function call. If you worked with Java/C# asynchronous calls, then *goroutines* may remind you of the idea of *context*. Go Scheduler allows the developer to make thousands of these lightweight goroutines and manages the CPU time spent on each one for us. Everytime a function prefixed with **go** is executed, a new goroutine is created to run that function, the **main** goroutine continues on its way immidiately after spawning a new goroutine, until it hits a blocking operator (similar to an await in C# or Js). 


Let's start with a simple console app that make calls to a few GitHub profiles and checks whether the connection was successful or not. At first, there are no goroutines here and all the calls are made consecutively; booo not very efficient. 

```go
package main

import "fmt"
import "net/http"

func main() {
	links := []string{
		"https://github.com/fabpot",
		"https://github.com/andrew",
		"https://github.com/taylorotwell",
		"https://github.com/egoist",
		"https://github.com/HugoGiraudel",
	}

	checkUrls(links)
}

func checkUrls(urls []string) {
	for _, link := range urls {
		checkUrl(link)
	}
}

func checkUrl(url string) {
	_, err := http.Get(url)

	if err != nil {
		fmt.Println("We could not reach:", url)
	} else {
		fmt.Println("Success reaching the website:", url)
	}
}
```

First, we need to add something called **channel**. Since Golang functions running in their own goroutines are just simple functions, we need a way through which inner goroutines can tell their result to the outer goroutine; this is done using channels. We initialize them simply by: **c := make(chan string)**
We are able to send the resulting value(s) to our channel using the **<-** arrow, and we assign the value from the channel using this arrow as well. 

Second, we need to add a tracker of sorts, to keep track of how many values we should be expecting to come out of this channel. This can be done using the type sync.WaitGroup.

Implementing these two ideas, results in the following: 

```go
import (
	"fmt"
	"net/http"
	"sync"
)

func main() {
	links := []string{
		"https://github.com/fabpot",
		"https://github.com/andrew",
		"https://github.com/taylorotwell",
		"https://github.com/egoist",
		"https://github.com/HugoGiraudel",
	}

	checkUrls(links)
}

func checkUrls(urls []string) {
	c := make(chan string)
	var wg sync.WaitGroup

	for _, link := range urls {
		wg.Add(1)   // This tells the waitgroup, that there is now 1 pending operation here
		go checkUrl(link, c, &wg)
	}

    // this function literal (also called 'anonymous function' or 'lambda expression' in other languages)
    // is useful because 'go' needs to prefix a function and we can save some space by not declaring a whole new function for this
	go func() {
		wg.Wait()	// this blocks the goroutine until WaitGroup counter is zero
		close(c)    // Channels need to be closed, otherwise the below loop will go on forever
	}()    // This calls itself

    // this shorthand loop is syntactic sugar for an endless loop that just waits for results to come in through the 'c' channel
	for msg := range c {
		fmt.Println(msg)
	}
}

func checkUrl(url string, c chan string, wg *sync.WaitGroup) {
	defer (*wg).Done()
	_, err := http.Get(url)

	if err != nil {
		c <- "We could not reach:" + url    // pump the result into the channel
	} else {
		c <- "Success reaching the website:" + url    // pump the result into the channel
	}
}
```

