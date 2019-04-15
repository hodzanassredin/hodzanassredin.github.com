--- 
published: true 
layout: post 
title: Implementing fixed point arithmetic in fsharp.
tags : [fixed-point, fsharp, forth] 
--- 
 
 
## {{page.title}} 
 
 
 
 
<p class="meta">15 April 2019 &#8211; Karelia</p> 
 

I've just finished reading of extremely interesting book "Starting FORTH" by Leo Brodie and found an implementation details for fixed-point arithmetic(FixedPA). Actually there are some libs for doing FixedPA on dot net and at the same time 
I found several blog posts related to dot net with different implementations in c# but some of them were incorrect.
So, this is a short description how FixedPA was implemented in old days in some Forth systems.

# Do we have FixedPA types in dot net's System?
No, we don't. We have only floating-point types: float, double and decimal.
Some people think that Decimal is FixedPA but actually it is floating point type but with 10 as base.
We are going to build our FFA lib using Int16 as a main data type. But you could trivially change it to use other types like Int32.

# The magic of scale function 

At the heart of our library we will use magic scale function.

{% highlight fsharp %} 
open System

let scale (n3: int16) (n2: int16) (n1: int16) : int16 = (int n1) * (int n2) / (int n3) |> int16
{% endhighlight %} 

AS you could see we multiply two Int16 and put the result into Int32 to prevent overflow. 
After that we divide Int32 by Int16 and return result as Int16.
So how could we use it?

{% highlight fsharp %} 
let percent = scale 100s//% calculation

let percent20 = percent 20s//20 % calculation

let r = percent20 50s//result 10s

//without scale function
let test = 2000s * 34s / 100s//result is incorrect 24s because of overflow 

//with scale function
let testFixed = percent 34s 2000s//correct result 680s 

//implementation of rounded percent
let roundedPercent percent value = (scale 10s value percent + 5s) / 10s

let testRounded = roundedPercent 32s 227s//result 73s

let testFloating = 2.0f /3.0f * 171.0f//expected 113.9999 but 114 
let testFixed = scale 3s 2s 171s//114

let test2Floating = 7.105/12.250 * 150.0//87.0
let test2Fixed = scale 12250s 7105s 150s //87s 

// we could even create real(irrational) numbers like Pi, E and so on
let pi = scale 113s 355s 
let e = scale 10546s 28667s

let testPi = pi 1000s//result 3141s

let square (r:double) = System.Math.PI * (r ** 2.)

let squareFixed x=  pi ( x * x)

let testSquareFloating = square 10.//314.1592654
let testSquareFixed = squareFixed 10s//314s
{% endhighlight %} 

It is very useful because it allows to avoid overflows during multiplication.
Now it is time to FixedPA. We are going to map fractions and floating numbers (Double) into our representation of fixed point numbers (Int16).

We have possibility to store values from -32768s to 32767s in Int16.
And for example, we are going to use only floating-point numbers from -2. to 2.
So, we have 1. mapped to 16384 and 1.9999 mapped to 32767s and -2. mapped to -32768s.
And finally, we need to define two operation multiplication and division.
So basically to map a value into fixed world we need to multiply by 16384(or just do binary left shift 14).

{% highlight fsharp %} 
let _1 = 16384s
let inline (./) (x:int16) (y:int16) = scale y _1 x 
let inline (.*) (x:int16) (y:int16) = scale _1 y x 
{% endhighlight %} 
Actually, we've done. That's all. We need first operation ./ to convert any fraction into our representation.
Second one is for multiplication.
We also need some helpers to move between floating and fixed worlds. 

{% highlight fsharp %} 
let toDouble (x: int16) = x .* 10000s |> double |> fun x -> x / 10000.
let toFixed (x: double) : int16 = x * 10000. |> int16 |> fun x -> x ./ 10000s
{% endhighlight %} 

And some tests

{% highlight fsharp %} 
let oneDivOneIsOne = 1s ./ 1s  = _1//true

let oneDivTwoIsHalf = 1s ./ 2s = (_1 / 2s)//true

let testFractionsFloating =  7./34.+23./99.//0.4382055853

let testFractionsFixed =  7s ./ 34s + 23s ./ 99s |> toDouble//0.4381

let sqrtFixed (n:int16) = 
    let rec si (root: int16)= 
        let newRoot = (n ./ root + root) / 2s
        if root = newRoot  then root else si newRoot 
    si (1s./10s) 

let sqrt = toFixed >> sqrtFixed >> toDouble 

let testSqrtFloating = Math.Sqrt(0.01)//0.1
let testSqrtFixed = sqrt 0.01//0.0997
{% endhighlight %} 

Hope you have enjoyed simplicity and elegance of this solution.

[The Philosophy of Fixed Point](https://www.forth.com/starting-forth/5-fixed-point-arithmetic/)

[Full code](https://gist.github.com/hodzanassredin/5f60c093905aa7a78dfa38899d3c076f)


