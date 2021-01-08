---
published: true
layout: post
title: Implementing arithmetic in fsharp.
tags : [math, fsharp]
---
 
 
## {{page.title}}
 
 
 
 
<p class="meta">08 January 2020 &#8211; Karelia</p>
 
A lot of time without new blog posts. Fortunately found a time to write about an interesting thing for me.
Currently, I'm trying to refresh my math skills and reading some math books.
It is always interesting to start from the base things and try to rethink them from the other point of view. So, I started to think about zero and sum and other basic things in terms of programming. And this blog post is an experiment to write about math and replace formulas with programs.

# The numbers

Let’s start with the number. Initially, we have only nothing(zero)
and to count something we need to have something nonzero. We are like Hegel now :-).
And we need a way to combine things together and express any natural number even infinity.

We could use the idea of lazy lists but without values.

{% highlight fsharp %}
type Number = Zero | Succ of Lazy<Number>
{% endhighlight %}

Additionally, we need two additional helper functions. You will see later why we need them.

{% highlight fsharp %}
let swap f a b = f b a

let toOption = function
    | Zero -> None
    | r -> Some(r)

{% endhighlight %}

Now we need to express basic functions to work with our numbers like increment, decrement, printing, and translating into int numbers.
But actually, we could use fold and unfold to express them. The fold function will iterate a number and create an aggregate.
Unfold function will generate a number from some state.

{% highlight fsharp %}
let rec fold (f:('State -> 'State)) (state: 'State) (n: Number) : 'State =
    match n with
    | Zero -> state
    | Succ(n) -> fold f (f state) n.Value

let rec unfold (f:('State -> 'State option)) (state:'State) : Number =
    match (f state) with
    | None -> Zero
    | Some(state) -> Succ(lazy(unfold f state))

//take some int and generate a number
let number = unfold (fun n -> if n = 0 then None else Some(n-1))
//toake some number and return int
let eval = fold (fun n ->
                    //printfn "evaluation %d" n
                    n + 1) 0


let evalOpt o = match o with None -> -1 | Some(o) -> eval o
//check some function and print result
let check f n1 n2 opName =
    f (number n1) (number n2) |> eval |> printfn "%d%s%d=%d" n1 opName n2
//check some function which returns optional result and print int for some number or -1 for Nothing
let checkOpt f n1 n2 opName =
    f (number n1) (number n2) |> evalOpt |> printfn "%d%s%d=%d" n1 opName n2

{% endhighlight %}

# Arithmetic

Now we are ready to start with arithmetic. But even a simple decrement from zero is not supported in our numbers.
We need to return an optional answer in case of unsupported operation. Probably in future
we will add support for negative numbers, maybe with zippers or something like that.

{% highlight fsharp %}
//increment number
let inc a = Succ(lazy(a))

//decrement number.
let dec = function
| Zero -> None
| Succ(n) -> Some(n.Value)

{% endhighlight %}

We are ready for addition and subtraction.
One interesting thing to note I expected to see unfold in subtraction, but it is fold probably because of unsupported negative numbers. We can see sum as an increment in a for loop.
And minus to find the argument of the sum function with a given result and argument.
And we don't need two functions because addition is commutative. So "swap sum" function is equal to sum.


{% highlight fsharp %}
let sum a b = fold inc a b

let minus a b = fold (Option.bind dec) (Some(a)) b

check sum 1 3 "+"

checkOpt minus 3 1 "-"
checkOpt minus 3 3 "-"
checkOpt minus 3 5 "-"

{% endhighlight %}

Now we are ready for more advanced functions and they look interesting, we can see some patterns. Unfortunately, we do not support rational numbers so we implement division as a function that will return how many times (rounded to integer) numerator contains denominator.

{% highlight fsharp %}
let mul a b = fold (sum a) Zero b //foreach b add a to acc, acc is zero initially

let div a b = unfold (swap minus b) a //while we can minus b from acc inc res , acc initially is a

let pow a b = fold (mul a) (inc Zero) b

let log a b = unfold ((swap div b >> toOption) ) a

check mul 3 2 "*"
check mul 3 5 "*"
check mul 3 0 "*"

check div 4 2 "/"
check div 3 2 "/"
check div 2 2 "/"
check div 1 2 "/"

check pow 1 2 " pow "
check pow 2 2 " pow "
check pow 2 3 " pow "
check pow 2 4 " pow "

check log 4 2 " log "
check log 8 2 " log "
check log 16 2 " log "

{% endhighlight %}

It was fun to see that arithmetic looks so interesting in a functional manner.

The next steps will be to support all real numbers (including irrational) and root function.

One interesting thing is about div by zero.
If we delete any number (even zero) by zero, we will have an infinite sequence as a result.
It is a great start to check different kinds of infinity and to add infinity detector into fold unfold functions.
For example, we can keep the whole history of a state and try to find a current state in the state history.
And extend our number with Infinity case. We will be able to express rational numbers.
Unfortunately, it will not work for irrational numbers but who knows about other ways to check for infinity. Probably we can express them as a continued fraction.


{% highlight fsharp %}
let inf1 = div (number 3) Zero
let inf2 = div Zero Zero
{% endhighlight %}

Math is fun.

[AlgorithmicAdventures](https://www.amazon.com/Algorithmic-Adventures-Knowledge-Juraj-Hromkovi%C4%8D/dp/3540859853)

[Full code](https://gist.github.com/hodzanassredin/e65c5d13956cb8d5533e1b76fcb2ccfa)





