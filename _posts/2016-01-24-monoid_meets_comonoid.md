---
published: false
layout: post
title: Monoid meets comonoid
tags : [fsharp, monoid, comonoid]
---

## {{page.title}}


<p class="meta">24 January 2016 &#8211; Karelia</p>

This is my attempt to describe monoids and comonoids. Hope you'll enjoy it.

#New language
Lets imagine that we are in the body of Don Syme and have a task to invent a brand new langauge codename Fsharp vNeXXXt. We lets start.
We want to be cool guys so we have to use functional programming. So our main building block will be a function. Inventing a language is a task like universe creation. So in our universe we have nothing. This universe is a place where programs writen in our language will live.  Our universe is inside of other universe "The Interpreter". So we can communicate to this external world somehow. Currently we have no types and we have a name for nothing it is a unit and could be writen as (). Any program writen in our language should start from nothing and finish with nothing. So actually it will be a function () -> (). And in our language without types every function has the same signature () -> (). But they are not the same becouse they actualle forces our interpreter do something. But we need a way to compose our program from small building blocks. So we could connect output of one func to input of other func. Also we have the ability to asociate a nema with a func. Lets introduce a function composition func/operator(>>) and a pair. Composition func will take a pair of funcs (()->(),()->()) and return a single one. But lets add some fsharp like syntactig sugar (>>)(f1,f2) = f1 >> f2. So a program in our language will be something like that: fn1 >> fn2 >> fn3. But for correctness we also need an associativity rule: 
 (>>)(fn1,(>>) (fn2,fn3))) = (>>)((>>)(fn1,fn2), fn3)).
So far so good.
But to do something useful we ned types. But we have no types and doesn't know how to create them. So lets invent a constructor funcs. They have a signature of () -> 'a where 'a is a placeholder for created type for example () -> int will create a number. We need to have several such functions for every possible int number. It is ok for now. But how can we compose our functions? We can use >> to () -> () and () -> 'a. But we cant compose them in different order. We don't know how to destroy objects(yet). But could we magically convert any ()->() into () -> 'a. Yes we can but for every type 'a in our system we need a special func () -> 'a' which returns some kind of Empty representation of a type. Lets name that func as mempty. So now we could compose () -> () >> mempty and it will give us a new func which will looks like  constructor. () -> 'a. Unforunately not everyone type has Empty instance and we cant compose it in our programs. Now we need to find a way to compose two funcs ()->'a. Hm only way to do that is to merge two results into a single one. So our constructors composition func, will look like that (() -> 'a,() -> 'a) -> (() -> 'a) and internally it will use some func lets name it as mappend which will take two inputs and will return one ('a,'a) -> 'a and,this func also has to be associative. Interesting but this func takes two inputs and gives us only one output. But we need a way to destroy one of inputs ond return other (or destroy both of them and create a new one) or in other case our universe will be full of unused objects. Oh no, our language starts to grow too fast. It is pity, but we forced to add a new construct. Destructor: 'a -> (). The point of destructors is to destroy an instance of a type. Now we have everything to express basic mappend funcs. How to compose destructors with both ()->() and constructors ()->'a. Constructor and destructor can be composed with function composition in both ways and 'a -> () and () -> () also could be composed and form a new destructor, but ()->() and 'a -> () cannot. What could we do? We could compose () -> () with mempty and after that compose empty constructor ()->'a with our destructor 'a -> (). But how could we compose destructors? We have to introduce a destructors compose func ('a -> (),'a -> ()) -> 'a -> (). Inside this func we have to soomehow clone input value and pass it into two destructors and return nothing. So internally we have to use split func 'a -> ('a,'a) defined for our type 'a. It is also have to be associative. Ok lets add ... Wake up Neo. Wake up. Oh no our language was just a dream, you we are not Don Syme and we have to go to work and write some big enterprise code in fsharp. But lets think about that tiny language. It has interesting properties we could compose destructors and constructors and also every created resource will be realeased when program finishes. Could we use something like that in our fsharp code. Defenitely yes.  So any type which has a pair of funcs mempty and mappend is a monoid. And every type which has a destructor and split funcs is a comonoid. int free seq file samples.

{% highlight fsharp %}
{% endhighlight %}


#Recommended reading:
1. [An Introduction and Developer’s Guide to Cloud Computing with MBrace](http://www.m-brace.net/mbrace-manual.pdf)
2. [Design patterns/best practice for building Actor-based system](http://stackoverflow.com/questions/3931994/design-patterns-best-practice-for-building-actor-based-system)

