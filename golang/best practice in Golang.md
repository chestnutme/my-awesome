---
title: Best Practice  in Golang
tags: go
---


## Abstract
Here record some best practices when I learn Golang and using it in coding and projects. It derived mostly from others' blog, lecture, open source code, etc, and then I absorb them as my own experience.



## Content
- [Abstract](#abstract)
- [Content](#content)
	- [1. golangs internal](#1-golangs-internal)
	- [2. concurrency in golang](#2-concurrency-in-golang)
		- [2.1 Go Concurrency patterns](#21-go-concurrency-patterns)
		- [2.2 Concurrency is not parallelism](#22-concurrency-is-not-parallelism)
		- [2.3 Don't communicate by sharing memory, share memory by communicating.](#23-dont-communicate-by-sharing-memory-share-memory-by-communicating)
	- [3. network progamming](#3-network-progamming)
	- [4. go basic](#4-go-basic)
		- [4.1 The Go Blog: Strings, bytes, runes and characters in Go](#41-the-go-blog-strings-bytes-runes-and-characters-in-go)
	- [n. miscellaneou](#n-miscellaneou)


### 1. golangs internal
* [golang memory allocator](https://juejin.im/post/59f2e19f5188253d6816d504)



### 2. concurrency in golang

#### 2.1 Go Concurrency patterns

> Concurrency is the key to designing high performance network services. Go's concurrency primitives (goroutines and channels) provide a simple and efficient means of expressing concurrent execution. 

refer:

* [The Go Blog: Go videos from Google I/O 2012](https://blog.golang.org/go-videos-from-google-io-2012), Rob show **generator** pattern by implementing a **google search**

* [The Go Blog: Advanced Go Concurrency Patterns](https://blog.golang.org/advanced-go-concurrency-patterns)

#### 2.2 Concurrency is not parallelism

> In programming, concurrency is the composition of independently executing processes, while parallelism is the simultaneous execution of (possibly related) computations. Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once.
-Rob Pike

refer:

[The Go Blog: Concurrency is not parallelism](https://blog.golang.org/concurrency-is-not-parallelism), in this talk, Rob implementes a simple **load balancer**

#### 2.3 Don't communicate by sharing memory, share memory by communicating.

> Go's approach to concurrency differs from the traditional use of threads and shared memory. Philosophically, it can be summarized: 
**Don't communicate by sharing memory; share memory by communicating.**

refer:

* [Codewalk: Share Memory By Communicating](https://golang.org/doc/codewalk/sharemem/)

* [A tour of go:web_crawler](https://github.com/filipegoncalves/go-samples/blob/master/web_crawler/web_crawler.go)

* communicate by sharing memory
```go
package main

import (
	"fmt"
	"sync"
)

type visited struct {
	mu        sync.Mutex
	isVisited map[string]bool
}

func (v visited) isExisted(url string) bool {
	v.mu.Lock()
	defer v.mu.Unlock()
	_, ok := v.isVisited[url]
	return ok
}

func (v visited) setVisited(url string) {
	v.mu.Lock()
	defer v.mu.Unlock()
	v.isVisited[url] = true
}

var v visited

type Fetcher interface {
	// Fetch returns the body of URL and
	// a slice of URLs found on that page.
	Fetch(url string) (body string, urls []string, err error)
}

// Crawl uses fetcher to recursively crawl
// pages starting with url, to a maximum of depth.
func Crawl(url string, depth int, fetcher Fetcher, ch chan string) {
	// TODO: Fetch URLs in parallel.
	// TODO: Don't fetch the same URL twice.
	// This implementation doesn't do either:
	
	defer close(ch)
	
	if depth <= 0 {
		return
	}

	if v.isExisted(url) {
		ch <- fmt.Sprintf("%s has been crawled", url)
		return
	}

	v.setVisited(url)

	
	body, urls, err := fetcher.Fetch(url)
	if err != nil {
		ch <- fmt.Sprintln(err)
		return
	}

	ch <- fmt.Sprintf("found: %s %q\n", url, body)

	chans := make([]chan string, len(urls))
	for i, u := range urls {
		chans[i] = make(chan string, 128)
		go Crawl(u, depth-1, fetcher, chans[i])
	}

	for i := range chans {
		for s := range chans[i] {
			ch <- s
		}
	}

	return
}

func main() {
	v = visited{isVisited: make(map[string]bool)}
	ch := make(chan string, 128)
	go Crawl("http://golang.org/", 4, fetcher, ch)
	for s := range ch {
		fmt.Println(s)
	}
}

// fakeFetcher is Fetcher that returns canned results.
type fakeFetcher map[string]*fakeResult

type fakeResult struct {
	body string
	urls []string
}

func (f fakeFetcher) Fetch(url string) (string, []string, error) {
	if res, ok := f[url]; ok {
		return res.body, res.urls, nil
	}
	return "", nil, fmt.Errorf("not found: %s", url)
}

// fetcher is a populated fakeFetcher.
var fetcher = fakeFetcher{
	"http://golang.org/": &fakeResult{
		"The Go Programming Language",
		[]string{
			"http://golang.org/pkg/",
			"http://golang.org/cmd/",
		},
	},
	"http://golang.org/pkg/": &fakeResult{
		"Packages",
		[]string{
			"http://golang.org/",
			"http://golang.org/cmd/",
			"http://golang.org/pkg/fmt/",
			"http://golang.org/pkg/os/",
		},
	},
	"http://golang.org/pkg/fmt/": &fakeResult{
		"Package fmt",
		[]string{
			"http://golang.org/",
			"http://golang.org/pkg/",
		},
	},
	"http://golang.org/pkg/os/": &fakeResult{
		"Package os",
		[]string{
			"http://golang.org/",
			"http://golang.org/pkg/",
		},
	},
}
```

* share memory by communicating
```go
/* Taken from the Go Tour (Exercise: Web Crawler)
 *
 * In this exercise you'll use Go's concurrency features to parallelize a web crawler.
 *
 * Modify the Crawl function to fetch URLs in parallel without fetching the same URL twice.
 *
 */

package main

import (
	"fmt"
)

type Fetcher interface {
	Fetch(url string) (body string, urls []string, err error)
}

func Crawl(url string, depth int, fetcher Fetcher, ch chan string, visited_ch chan map[string]bool) {

	defer close(ch)

	if depth <= 0 {
		return
	}

	/* This implements an atomic "test and set" for the visited map. It atomically fetches the
         * visited status for the URL and sets it.
         *
         * This is cleverly achieved by using a buffered channel with unitary capacity where worker
         * threads consume the map when they want to read and mutate it, and write it back to the
         * channel once they're done.
         *
         * Note that the channel must be buffered with a capacity of 1, otherwise we would deadlock
         * because unbuffered channels block readers and writers until the other end is ready.
         *
         * This is Go's philosophy of concurrency:
         * Don't communicate by sharing memory, share memory by communicating
         *
         * How brilliant is that?
         */
	visited := <- visited_ch
	_, found := visited[url]
	visited[url] = true
	visited_ch <- visited

	if found {
		return
	}

	body, urls, err := fetcher.Fetch(url)

	if err != nil {
		ch <- fmt.Sprintln(err)
		return
	}

	ch <- fmt.Sprintf("found: %s %q\n", url, body)

	chans := make([]chan string, len(urls))
	for i, u := range urls {
		chans[i] = make(chan string, 128)
		go Crawl(u, depth-1, fetcher, chans[i], visited_ch)
	}

	/* This is how we implement synchronization and wait for other threads to finish.
         *
         * Each Crawl() thread is assigned its own channel to write results to. Each thread closes
         * its channel once it's done, that is, after writing its own results into the channel and
         * the results of the goroutines it spawned. Thus, results flow from a set of channels up
         * the "channel tree" until they reach the main, primary channel.
         *
         * This clever mechanism allows goroutines to wait for other spawned routines to terminate
         * before returning and closing their own channel.
         *
         * The channels are buffered (with a capacity of 128) because otherwise there is not much
         * parallelism, since each thread could only make progress after the parent thread fetched
         * the last result written.
         *
         * Synchronization is implicitly achieved with the channels, because each thread defers
         * closing the channel, which is wonderful.
         */

	for i := range chans {
		for s := range chans[i] {
			ch <- s
		}
	}

	return
}

func main() {
	ch := make(chan string, 128)

	visited_ch := make(chan map[string]bool, 1)
	visited_ch <- make(map[string]bool)

	go Crawl("http://golang.org/", 4, fetcher, ch, visited_ch)

	for s := range ch {
		fmt.Print(s)
	}
}

// fakeFetcher is a Fetcher that returns canned results.
type fakeFetcher map[string]*fakeResult

type fakeResult struct {
	body string
	urls []string
}

func (f fakeFetcher) Fetch(url string) (string, []string, error) {
	if res, ok := f[url]; ok {
		return res.body, res.urls, nil
	}
	return "", nil, fmt.Errorf("not found: %s", url)
}

// fetcher is a populated fakeFetcher.
var fetcher = fakeFetcher{
	"http://golang.org/": &fakeResult{
		"The Go Programming Language",
		[]string{
			"http://golang.org/pkg/",
			"http://golang.org/cmd/",
		},
	},
	"http://golang.org/pkg/": &fakeResult{
		"Packages",
		[]string{
			"http://golang.org/",
			"http://golang.org/cmd/",
			"http://golang.org/pkg/fmt/",
			"http://golang.org/pkg/os/",
		},
	},
	"http://golang.org/pkg/fmt/": &fakeResult{
		"Package fmt",
		[]string{
			"http://golang.org/",
			"http://golang.org/pkg/",
		},
	},
	"http://golang.org/pkg/os/": &fakeResult{
		"Package os",
		[]string{
			"http://golang.org/",
			"http://golang.org/pkg/",
		},
	},
}
```

### 3. network progamming


### 4. go basic

#### 4.1 [The Go Blog: Strings, bytes, runes and characters in Go](https://blog.golang.org/strings)

> use strings well requires understanding not only how they work, but also the difference between a byte, a character, and a rune, the difference between Unicode and UTF-8, the difference between a string and a string literal, and other even more subtle distinctions.

### n. miscellaneou

1. [go-faq](https://pdos.csail.mit.edu/6.824/papers/go-faq.txt)

> frequent asked problem when first encouter Golang, eg:var definition where type is on the right, goroutine, no class..

2. [effective go](https://golang.org/doc/effective_go.html)
