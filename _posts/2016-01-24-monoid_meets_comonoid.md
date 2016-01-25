---
published: true
layout: post
title: (Co)Monoids for composable resource management in fsharp
tags : [fsharp, monoid, comonoid]
---

## {{page.title}}


<p class="meta">24 January 2016 &#8211; Karelia</p>

This is my attempt to describe monoids and comonoids and how to use them. Hope you'll enjoy it.
You should understand what is monoid, in other case go and read [Understanding monoids](http://fsharpforfunandprofit.com/series/understanding-monoids.html) before reading this post. Also you need to understand what is dual. Info about duals could be find [here](http://codebetter.com/matthewpodwysocki/2009/11/03/introduction-to-the-reactive-framework-part-ii/). 

I spend a lot of time trying to understand monads, but never look at monoid because it feels so simple. Actually in pure languages monoid is really simple, just a pair of funcs () -> 'a and ('a * 'a) -> 'a. Second one is some join operation and is ok, but the first one has unit as an input argument. As we remember unit type usually tell us that function can have side effects. But in language like Haskell there are no side effects. So implementation for that function is usually trivial. And if we look at monoid's dual comonoid then we will see a pair of funcs: 'a -> () and 'a -> ('a * 'a). Actually in haskell every type has comonoid implementation for free: fun x -> () and fun x -> x,x. First has a name destroy and the second one is clone or pair. All articles about comonoids usually say nothing. And I didn’t find anyone for fsharp. So I decided to write my own. Because in fsharp we have side effects everywhere and monoids and comonoids could be used to compose them.  

#New language
Lets imagine that we are in the body of Don Syme and have a task to invent a brand new language codename Fsharp vNeXXXt. So lets start.
We want to be cool guys, so we have to use functional programming. Our main building block will be a function. Inventing a language is a task like an universe creation. And in our universe we have nothing. This universe is a place where programs written in our language will live.  Our universe is inside of other universe "The Interpreter". So we can communicate to this external world somehow. Currently we have no types and we have a name for nothing, it is unit and could be writen as (). Any program written in our language should start from nothing and finish with nothing. It will be a function () -> (). And in our language without types, every function has the same signature () -> (). But they are not the same, because they actually forces our interpreter to do something. But we need a way to compose our program from small building blocks. We could connect output of one func to input of other func. Lets introduce a function composition func/operator(>>) and a pair. Composition func will take a pair of funcs (()->(),()->()) and return a single one. But lets add some fsharp like syntactic sugar (>>)(f1,f2) = f1 >> f2. So a program in our language will be something like that: fn1 >> fn2 >> fn3. But for correctness we also need an associativity rule: 
 (>>)(fn1,(>>) (fn2,fn3))) = (>>)((>>)(fn1,fn2), fn3)).
So far so good.
But to do something useful we need types. But we have no types and doesn't know how to create them. So lets invent a constructor funcs. They have a signature of () -> 'a where 'a is a placeholder for created type, for example () -> int will create a number. We need to have several such functions for every possible int number. It is ok for now. But how can we compose our functions? We can use >> to () -> () and () -> 'a. But we cant compose them in different order. We don't know how to destroy objects(yet). But could we magically convert any ()->() into () -> 'a. Yes we can, but for every type 'a in our system, we need a special func () -> 'a' which returns some kind of Empty representation of a type. Lets name that func as mempty. So now we could compose () -> () >> mempty and it will give us a new func which will looks like  constructor. () -> 'a. Now we need to find a way to compose two funcs ()->'a. Hm, only way to do that is to merge two results into a single one. So our constructors composition func, will look like that (() -> 'a,() -> 'a) -> (() -> 'a) and internally it will use some func lets name it as mappend which will take two inputs and will return one ('a,'a) -> 'a and this func also has to be associative. Interesting, but this func takes two inputs and gives us only one output. But we need a way to destroy one of inputs and return other (or destroy both of them and create a new one) or in other case our universe will be full of unused objects. Oh no, our language starts to grow too fast. It is pity, but we forced to add a new construct. Destructor: 'a -> (). The point of destructors is to destroy an instance of a type. Now we have everything to express basic mappend func. How to compose destructors with both ()->() and constructors ()->’a? Constructor and destructor can be composed with function composition in both ways and 'a -> () and () -> () also could be composed and form a new destructor, but ()->() and 'a -> () cannot. What could we do? We could compose () -> () with mempty and after that compose empty constructor ()->'a with our destructor 'a -> (). But how could we compose destructors? We have to introduce a destructors compose func ('a -> (),'a -> ()) -> ('a -> ()). Inside this func we have to somehow clone input value and pass it into two destructors and return nothing. So internally we have to use split func 'a -> ('a,'a) defined for our type 'a. It is also have to be associative. Ok lets add ... Wake up Neo. Wake up. Oh no our language was just a dream, and we are not Don Syme and we have to go to work and write some big enterprise code in fsharp. But lets think about that tiny language. It has interesting properties we could compose destructors and constructors and also every created resource will be released when program finishes. Could we use something like that in our fsharp code. Definitely yes.  So any type which has a pair of functions mempty and mappend is a monoid. And every type which has a destructor and split functions is a comonoid. 

Lets play with some simple type to warm up our brain. For example string. Lets define required functions for it.

{% highlight fsharp %}
let mzero () = ""
let mappend (s:string) (s2:string) = s + s2
let pair (s:string) = s,s
let destroy (s:string) = ()
{% endhighlight %}

It was trivial and there are no side effects. Lets create compose functions and some additional constructors and destructors with side effects.

{% highlight fsharp %}
//constructor composition
let ctorCompose f1 f2 () = mappend (f1()) (f2())
let (|>>) a b = ctorCompose a b
let toCtor f = f >> mzero

//constructor with side effects
let ctor1 () = printfn "Please enter string" 
               System.Console.ReadLine()
//pure constructor
let ctor2 () = "aaa"
let printSmth () = printfn "smth" 
//constructor with side effects composed from side effect func and empty ctor
let ctor3 = toCtor printSmth
let comboCtor = ctor1 |>> ctor2 |>> ctor3
//composition of destructors
let dectorCompose f1 f2 a = let a,a2 = pair a
                            f1(a)
                            f2(a2)
                            ()

let toDestructor (fn: unit -> unit) = destroy >> fn
let (>>>) a b = dectorCompose a b

let destructor = printfn "%s"
let destructor2 (s:string) = ()
let createlogFileDestructor path =
    fun (s:string) -> use wr = new StreamWriter(path, true)
                      wr.WriteLine(s)

let logFileStringDestructor = createlogFileDestructor 
									<| Path.GetTempFileName()
let composedDestructor = destructor 
							>>> destructor2
							>>> logFileStringDestructor 
							>>> (toDestructor printSmth)

let res = comboCtor()
printfn "result of combo constructor %s" res
composedDestructor res
{% endhighlight %}

Yes it is funny. I have never thought that printf "%s" is a string destructor. Unfortunately this code is not really interesting. But could we use it for something more real. For example for composable resource releasing? Lets define simple type which will hold destroy function(for example Dispose method invocation) and implement comonoid and monoid funcs.

{% highlight fsharp %}
open System.IO
type Resource = Dispose of (unit -> unit) 

let mzero = Dispose(id)
let mappend (Dispose(a)) (Dispose(b)) = Dispose(a >> b)
let destroy (Dispose(r)) = r()

let pair (Dispose(d)) =  
    let lockObj = new obj()
    let synchronized f () = lock lockObj f
    let f = ref false
    let destructor () = 
        if !f then d()
        else f := true

    let destructor = synchronized destructor
    Dispose(destructor),Dispose(destructor)    
{% endhighlight %}

Now when we created some object we will return dispose func with it. 

{% highlight fsharp %}
let createWriter (path:string) =
                  printfn "Created writer %s" path
                  let w = new StreamWriter(path, true)
                  w, Dispose(fun () -> printfn "disposed writer %s" path
                                       w.Dispose())
    
let createReader (path:string) =
              printfn "Created reader %s" path
              let r = new StreamReader(path, true)
              r, Dispose(fun () -> printfn "disposed reader %s" path
                                   r.Dispose())
{% endhighlight %}

Now lets think how could we compose functions which uses that type. We have constructors () -> Dispose<'r'>, destructors Dispose<'r'> -> (), pure funcs 'a -> 'b and consumer funcs Dispose<'r'> -> 'res. Seems that we have to find some general representation for that funcs, but how to find it. Hm seems that funcs really similar to funcs in reader,writer and state monads. So actually we can use something like that and create common representation and computation builder for that. If you want to refresh memories about these monads then you need to read [Stateful computations in F# with update monads](http://tomasp.net/blog/2014/update-monads/) 

{% highlight fsharp %}
type Managed<'r> = Resource -> 'r * Resource
type ResourceBuilder() = 
      member inline x.Return(v) : Managed<'r> = fun r -> destroy r
                                                         v, mzero
      member inline x.ReturnFrom(v) : Managed<'T> = v

      member inline x.Bind(rm1:Managed<'T>, f:'T -> Managed<'T2>) : Managed<'T2> =  
            fun r -> let r,r2 = pair r
                     let v, r = rm1 r
                     f v (mappend r r2)


let resource = ResourceBuilder()
et destroySnd t = destroy <| snd t
                  fst t

let run (tr:Managed<'r>) = mzero |> tr |> destroySnd
{% endhighlight %}
The idea is quite simple, we just need to follow several rules.

1. dispose resource object before exit.
2. merge result of function invocation resource object with current resource object
3. before passing resource as an argument create its clone

Now we could move previously described constructors into our builder and use our new builder.

{% highlight fsharp %}
let liftCtor c a: Managed<_>  = fun r -> destroy r
                                         c a

let createWriterM = liftCtor createWriter
let createReaderM = liftCtor createReader

let subProgram () = resource{
    let! r = createReaderM "D:\\install.ini"
    printfn "sub program: reading line"
    let line = r.ReadLine()
    return line
}

let subProgram2 () = resource{
    printfn "creating reader in subProgram2"
    let r = createReaderM "D:\\install.ini"
    printfn "creating unused writer in subProgram2"
    let! w = createWriterM <| Path.GetTempFileName()
    return! r
}

let asyncWriteLine (line:string) (w:StreamWriter)  = async{
    printfn "async sleeping"
    do! Async.Sleep(1000)
    printfn "async writing line"
    w.WriteLine(line)
}

let startAsync a r = 
        let task = async{
                        do! a
                        destroy r
                        return ()
                    } |> Async.StartChild
        task, mzero

let program = resource{
    printfn "executing sub program"
    let! line = subProgram()
    printfn "finished executing sub program result is: %s" line
    printfn "creating writer"
    let! w = createWriterM <| Path.GetTempFileName()
    printfn "executing subProgram2"
    let! r = subProgram2()
    printfn "finished executing sub program2"
    printfn "executing async write line"
    //captures both reader and writer
    let! a = startAsync (asyncWriteLine "async line" w)
    printfn "finished executing async write line"
    printfn "reading line"
    let line = r.ReadLine()
    printfn "writing line"
    w.WriteLine(line)
    return a, line
}

let a, line = run program 
printfn "result line %A" line
Async.RunSynchronously a
Async.RunSynchronously <| Async.Sleep 2000
{% endhighlight %}
if you execute it, you will see that resources are disposed in expected way and we didn't use "use" keyword. Now we are able to pass objects into async computations and run them in parallel. 

That it. Hope you found something interesting for you. 

Code is [here](https://gist.github.com/hodzanassredin/1bc1521bb49b215413e2)

#Recommended reading:
1. [What does a nontrivial comonoid look like?](http://stackoverflow.com/questions/23855070/what-does-a-nontrivial-comonoid-look-like)
2. [rust-comonoid](https://github.com/srijs/rust-comonoid)

