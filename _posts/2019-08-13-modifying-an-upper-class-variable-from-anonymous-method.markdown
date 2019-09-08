---
layout: post
title: Modifying an Outer Variable within an Anonymous Class on Retrofit
date: '2019-08-13 01:51:27'
image: assets/images/1_TB8U_zc34XG2qk1LeOBdPA.jpg
tags:
- programming
- java
- android
---

So, here's the thing, I want to be able to access a defined variable inside the method to be accessed by an anonymous class inside that method, something like this

~~~.language-java
    public class SomeClassName {
    
        public String SomeMethod() {
            
            String outerVariable = 0;
            
            public void insideMethod(){
                
                outerVariable = 1
                }   
        return outerVariable;
        }
    }            

~~~

yeaaa kinda like that, but normally is, I can't. Because if a method invoked a variable outside it's scope, after a runtime, the program might free that address so it probably could referencing to another address.

The only thing that possible is to declare that outer variable as final, but if did that, then I couldn't change its value, that's not what I want !

This use case was to get a string value from Retrofit onResponse code below

<!--kg-card-begin: html--><script src="https://gist.github.com/luqmansen/3ec94ac566be2a23c0de9ac21cceec8f.js"></script><!--kg-card-end: html-->

After browsing on internet quite long, I bumbed up to [this](https://stackoverflow.com/questions/44872115/how-can-i-return-value-from-onresponse-of-retrofit-v2/44881355)solution, you can check it out.

> Apparently, because the method onResponse is an asynchronous process, so it will finish most of times after the method getVideoID has returned.

This is the question after hours of debugging why is my code returning null. We cannot return anything from inside that method just like a normal method did. However, we can pass that value once it's done the process. How we did this ? Using our holy callback. Honestly, after following many of tutorial i just get the understanding one of the function of an interface and callback stuff.

So, the one of the option of passing the string value is through this callback, then, I need a interface.

<!--kg-card-begin: html--><script src="https://gist.github.com/luqmansen/8c3c398baa9117bbd04280c971d775da.js"></script><!--kg-card-end: html-->

This interface are the method that whoever wants to get the value from my callback, have to implement this interface. So for my use case, I pass the callback interface to my method parameters and call the onSuccess method in onResponse like this

<!--kg-card-begin: html--><script src="https://gist.github.com/luqmansen/9ffc9f5a5b8da945404399909db05a0d.js"></script><!--kg-card-end: html-->

Like this make us able to receiver the string "youtubeID" from the response asynchronously without having to worry about returning a value.

And, just call the method on the other classes that needs that, here's the example for my case

<!--kg-card-begin: html--><script src="https://gist.github.com/luqmansen/1cd2827428acd3ca9f97c13c929f369a.js"></script><!--kg-card-end: html-->

Because that class should have implement the interface that we create earlier, than we just needs to past "this" to the callback interface parameter.

And we're done, here is my full project to be clear or if you just courious

[https://github.com/luqmansen/movie-catalogue/](https://github.com/luqmansen/movie-catalogue/)

