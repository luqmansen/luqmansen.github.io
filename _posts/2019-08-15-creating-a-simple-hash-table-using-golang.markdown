---
layout: post
title: Creating Simple Hash Table using Golang
date: '2019-08-15 02:58:04'
image: assets/images/Golang.png
categories: [programming,golang]
---

<figure class="kg-card kg-image-card kg-card-hascaption"><img src="https://res-5.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/315px-Hash_table_3_1_1_0_1_0_0_SP.svg.png" class="kg-image"><figcaption></figcaption></figure><!--kg-card-end: image-->

So, recently i getting serious on learning go language. I using some course that intended for someone who new to programming, so I mostly just fast forward the videos, until i got to material where i should implement hash table. This part was actually part of map data structure part.

I already heard a lot about hash table on my data structure course but never really implement this from scratch. Hash table is the underlying data structure for map data structure. Hash table allows map to get the key value with (O)1 time complexity.

Below i'll explain try code line by line, this blog post was actually for reinforce my learning. Soo let's start

~~~.language-go
res, err := http.Get("http://www.gutenberg.org/files/2701/old/moby10b.txt")
if err != nil{
	log.Fatal(err)
	}
~~~

first we get the data from the link, we use http library here. Get method demand a string parameter and return two 2 value, the response and an error. For the error, just make some error handling with an if conditional.

~~~.language-go
func Get(url string) (resp *Response, err error) {
	return DefaultClient.Get(url)
}
~~~

The return response is a pointer to type Response. so let's check the respon type itself

~~~.language-go
type Response struct {
	Status string // e.g. "200 OK"
	StatusCode int // e.g. 200
	Proto string // e.g. "HTTP/1.0"
	ProtoMajor int // e.g. 1
	ProtoMinor int // e.g. 0
	Header Header
	Body io.ReadCloser
	ContentLength int64
~~~

This is a struct type wich one of the field is the Response body, it is also a new defined &nbsp;type of io.ReadCloser, let's check what it is

~~~.language-go
type ReadCloser interface {
	Reader
	Closer
}
~~~

So, ReadCloser is actually a interface type that has 2 field, Reader and closer. Well, let's check what is Reader

~~~.language-go
    type Reader interface {
    	Read(p []byte) (n int, err error)
    }

~~~

Aaand the Reader itself is a interface which implement read method, accept byte slices as parameter, and return an integer and an error, wow, cool. Now let's go to next code

~~~.language-go
    scanner := bufio.NewScanner(res.Body)
    defer res.Body.Close()

~~~

Then we declare and assign a scanner. We give this one a NewScanner method from which accept a res.Body. let's check what NewScanner method is

~~~.language-go

    func NewScanner(r io.Reader) *Scanner {
    	return &Scanner{
    		r: r,
    		split: ScanLines,
    		maxTokenSize: MaxScanTokenSize,
    	}
    }

~~~

See, this method accept io.Reader. Remember what io.Reader is ? &nbsp;That was from Reader interface that implement Read method like i mentioned before. Cool right ? this is how we learn Go lang, diving into the source code itself. Okay keep going.

~~~.language-go

    scanner.Split(bufio.ScanWords)

~~~

this one, we implement split method to our scanner, because new scanner is returning a pointer to scanner type so this split method down here

~~~.language-go

    func (s *Scanner) Split(split SplitFunc) {
    	if s.scanCalled {
    		panic("Split called after Scan")
    	}
    	s.split = split
    }

~~~

is accepting a Scanner type as argument, yeah cool.

~~~.language-go

    buckets := make([][]string, 12)

~~~

Now lets make a bucket. This bucket a multi dimensional slice type is going to going to be stored our bucket that going to contain our hashed word. We going to limit the number of our bucket in 12 bucket.

~~~.language-go

    for i := 0; i<12; i++{
    		buckets = append(buckets, []string{})
    	}
    

~~~

Let's actually iterate through our bucket and append a string slice for each bucket. this bucket is the bucket that actually going to stored the word that already hashed.

~~~.language-go

    for scanner.Scan(){
    		word := scanner.Text()
    		n := HashBucket(word,12)
    		buckets[n] = append(buckets[n], word)
    	}

~~~

This one, we iterate using scanner.Scan() method. This method is going to iterate through our novel and always returning true as long as there is still word available, if it reached end, then it will return false.

The scanner.Text method is going to return a token, or to make it simpler, the last scanned word from the source. After that, we call our HashBucket function.

~~~.language-go

    func HashBucket(word string, buckets int) int {
    	var sum int
    	for _, v := range word{
    		sum += int(v)
    	}
    	return sum % buckets
    

~~~

This function is accepting a word, and a number of bucket needed. There is a iteration function through the range of the word. This will make each word integer (in ascii format) then sum it up. The return is going to be that sum modulus by the number of bucket, in this case, it's always 12.

~~~.language-go

    n := HashBucket(word,12)
    buckets[n] = append(buckets[n], word)

~~~

this statement is going to append each word to an specific bucket in which the number of bucket has been calculated from previous function

~~~.language-go

    for i:= 0; i<12; i++{
    		fmt.Println(i, " - ", len(buckets[i]))
    	}

~~~

At the end, just to make sure our bucket working, lets just print the number of word stored in each bucket. Here is the result

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="https://res-2.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/go-hash-table.png" class="kg-image"></figure><!--kg-card-end: image-->

Pretty cool right ? The highest magnitude was 32.000-ish and the smaller around 11.000-ish so the ratio was 3:1, not too bad. The has algorithm can be more optimized for more even distribution.

Here is the full code

<!--kg-card-begin: html--><script src="https://gist.github.com/luqmansen/b8337c23184ecc64624764990d4e099e.js"></script><!--kg-card-end: html-->

Well done kiddo, see you in next article, keep studying !

