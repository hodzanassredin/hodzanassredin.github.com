--- 
published: true 
layout: post 
title: Image processing with comonads and phantom types 
tags : [fsharp, comonads] 
--- 
 
 
## {{page.title}} 
 
 
 
 
<p class="meta">28 March 2016 &#8211; Karelia</p> 
 
 
# What is Comonad? 
Comonad is just a dual of a monad. Super simple ;-).
We could represent monad as a type with a pair of functions

{% highlight fsharp %} 
type Id<'a> = Id of 'a
let return' v = Id(v)
let bind f (Id(v) : Id<'a>) : Id<'b> = f v 
{% endhighlight %} 

We need return function to convert any value to a monad and bind function to apply a function
which returns monad to a monad. Or in different words we have functions like 'a -> M<'b> and
want to compose them with a >> operator. To do that we need to use this pattern
(bind f) >> (bind f2) ...
And to convert any function 'a -> 'b to a function like 'a -> Monad<'b> we have to use f >> return'
so finally we could compose functions

{% highlight fsharp %} 
let fn a b = a + b
let monadicFn v = Id(1 + v)
let composition = bind (fn 1 >> return') >> bind  monadicFn
{% endhighlight %} 

And comonad is inverse type with a pair of function extract and extend

{% highlight fsharp %} 
let extract (Id(v)) = v
let extend f (v : Id<'a>) : Id<'b> = Id(f v) 
{% endhighlight %} 

And composition looks like this

{% highlight fsharp %} 
let comonadicFn (Id(v)) = 1 + v
let compositionC = extend (extract >> fn 1) >> extend comonadicFn
{% endhighlight %} 

# When we need comonads?
We need them when we have to use some additional context in our functions.
For example, we could use additional execution context like we do in a reader monad.

{% highlight fsharp %} 
type Context = { availableMemory : int}
type Runtime<'a> = Runtime of Context * 'a
let extract (Runtime(ctx, v)) = v
let extend f (Runtime(ctx, v)) = Runtime(ctx, f (Runtime(ctx, v)))
let addR (Runtime(ctx, v)) = if ctx.availableMemory > 100 then v + 1 else v
{% endhighlight %} 

And we could represent any non-empty collection with attached index as a comonad.
Let’s implement a tiny image processing library as an example.
Image(grayscale) will be represented as an 2d array of ints with i and j indexes. 
But array could be empty, how could we fix that? We could simply return default value when 
there is nothing to extract.
We need two array helper functions:

{% highlight fsharp %} 
let arraySize arr = Array2D.length1 arr, Array2D.length2 arr
let safeGet i' j' (arr:_[,])  = try arr.[i',j'] with | ex -> Unchecked.defaultof<'a>
{% endhighlight %} 

Some code for image loading.

{% highlight fsharp %} 
#r "System.Drawing.dll"
module Bitmap =
    open System
    open System.IO
    open System.Drawing
    open System.Drawing.Imaging

    let toArray2d (image:Bitmap) =
        let f i j = image.GetPixel(i,j) 
        let arr = Array2D.init image.Width image.Height f
        arr

    let fromArray arr =
        let w, h = arraySize arr
        let res = new Bitmap (w,h)
        Array2D.iteri (fun i j c -> res.SetPixel(i,j,c)) arr
        res

    let load path : Bitmap = downcast Image.FromFile(path, true)
    let save path (image : Bitmap) = image.Save path
    
    let toGrayScale (c:Color) = (int(c.R) + int(c.G) + int(c.B)) / 3
    let fromGrayScale s = 
        let s = if s < 0 then 0 
                elif s > 255 then 255
                else s
        Color.FromArgb(s,s,s)
{% endhighlight %} 

And a module for our comonad (array with indexes).
In a module we have to define our comonad type, 
extract function which returns a value from the array using current position,
extend function which works like a standard array's mapi function.
Also we have to define get function which uses additional, relative to a current position, indexes.
 
{% highlight fsharp %} 
module CArray2D =
    type CArray2D<'a> = CA2 of 'a[,] * int * int
    let create arr = CA2(arr, 0, 0)
    let get i' j' (CA2(a, i, j):CArray2D<'a>)  = safeGet (i+ i') (j + j') a
    let (?) c (i':int,j':int) = get i' j' c 
    let extract c = get 0 0 c

    let extend (f: CArray2D<'a> -> 'b) (CA2(a, i, j) : CArray2D<'a>) : CArray2D<'b> = 
        let w, h = arraySize a
        let f i j =  f(CA2(a,i,j)) 
        let r = Array2D.init w h f
        CA2(r,i,j)

    let run (f: CArray2D<'a> -> CArray2D<'b>) arr = 
        let (CA2(arr,_,_)) = f (CA2(arr,0,0)) 
        arr

    let zip (CA2(a, i, j)) (CA2(b, _, _)) =
        let w, h = arraySize a
        let f i j =  (safeGet i j a, safeGet i j b)
        let r = Array2D.init w h f
        CA2(r,i,j)
{% endhighlight %} 

Now it is time for image filters. Usually a filter represented as a function which takes our comonadic value
and uses function "get" and/or "extract" to calculate output value of current position based on values 
from current position and probably additional positions near of current position(neighbors).

{% highlight fsharp %} 
let laplace2d a = 
          a ? (-1,0) 
        + a ? (0,1) 
        + a ? (0,-1) 
        + a ? (1,0) 
        - 4 * a ? (0,0)

let gauss2D a = (a ? (-1, 0) + a ? (1, 0) + a ? (0, -1) + a ? (0, 1) + 2 * a ? (0, 0)) / 6
{% endhighlight %} 

Now we could build additional filter by composing existing ones.

{% highlight fsharp %} 
let minus x y = CArray2D.extract x - CArray2D.extract y

let contours x = 
    let y = CArray2D.extend ImageProcessing.gauss2D x
    let w = CArray2D.extend (fun y' -> let z = CArray2D.extend ImageProcessing.gauss2D y'
                                       in minus y' z) y
    ImageProcessing.laplace2d w

let gaussLaplace = CArray2D.extend ImageProcessing.gauss2D >> ImageProcessing.laplace2d
{% endhighlight %} 

And now we could run our code using collection of test images.

{% highlight fsharp %} 
open ImageProcessing
open CArray2D

let applyTransform (ipath, f, fname) =
    Bitmap.load ipath 
    |> Bitmap.toArray2d 
    |> Array2D.map toGrayScale
    |> run f
    |> Array2D.map fromGrayScale
    |> Bitmap.fromArray
    |> Bitmap.save (sprintf "%s.out.%s.%s" ipath fname (ipath.Split('.').[1])) 

let tests = [extend extract, "extract";
             extend gauss2D, "gauss2D";
             extend laplace2d, "laplace2d";
             extend gaussLaplace, "gaussLaplace";
             extend contours, "contours"]

let fname = sprintf "D:\\img\\%s" 
let files = ["test.bmp";
             "laplacian1.jpg";
             "Lena.png";
             "fce2.bmp";
             "tahaa.jpg";] |> List.map fname

for file in files do
    for testf, fname in tests do
        time (sprintf "%s - %s" file fname) applyTransform (file, testf, fname)
{% endhighlight %} 

Let’s execute it. Oh no it takes too long to execute some filters. What is the problem?
After investigation we could find that complexity of extract function is O(1) 
and complexity of extend function is O(N). So in case of composition "extend extract >> extend extract"
final complexity is O(N) and this is ok.
But in case of "extend (extend extract)" complexity is O(N^2). So if we want to compose 
our functions we have to remove this complexity growth or prevent extending of extended functions.
First way could be done by implementing array as a lazy array.

{% highlight fsharp %} 
module LazyArray2D = 
    type LArray2D<'a> = LArray2D of (int -> int -> 'a)  * int * int
    let empty x y v = LArray2D((fun _ _ -> v), x, y)
    let get (LArray2D(f, x, y)) i j  = f i j
    let size (LArray2D(f, x, y)) = x,y
    let init x y f = LArray2D(f, x, y)
    let map f' (LArray2D(f, x, y)) = 
        let f' = fun i j -> f' (f i j)
        LArray2D(f', x, y)

    let mapi f' (LArray2D(f, x, y)) = 
        let f' = fun i j -> f' i j (f i j)
        LArray2D(f', x, y)

    let iteri f' (LArray2D(f, x, y)) = 
        for i in 0..(x-1) do
            for j in 0..(y-1) do
                f' i j (f i j)
                
type CArray2D<'a> = CA2 of LArray2D<'a> * int * int
let fmap f (CA2(a, i,j)) = CA2(map f a, i, j)
let extract (CA2(a, i, j)) = get a i j
let extend f (CA2(a, i, j)) =
    let f = fun i j _ -> f (CA2(a,i,j)) 
    let es' = LazyArray2D.mapi f a 
    in CA2(es',i,j)
{% endhighlight %} 

And yes this way works, there is no fast complexity growth, but in terms of raw performance it is far from ideal.
Because there is a lot of duplicate calls to underlying filters. So ideal solution is to use arrays 
but prevent, somehow, incorrect composition. And this can be done with phantom types. 
# What is Phantom type?
It is just an additional type variable in generic type, which is used only in type declarations.
Usually it is used to add some compile time checks. So let’s add phantom type to our comonad.

{% highlight fsharp %} 
type CArray2D<'a,'p> = CA2 of 'a[,] * int * int
{% endhighlight %} 

A type variable 'p is a place where we will put our marker types

{% highlight fsharp %} 
type Composed () = class end
type Raw () = class end
{% endhighlight %} 

Raw type will be used to mark comonad with complexity O(1) and Composed marker will be applied to
comonads returned by extend function with complexity O(N) and extend function as an input will accept 
only composed comonad and argument function will be restricted to a funcs which accept only raw comonad. 

{% highlight fsharp %} 
let get i' j' (CA2(a, i, j):CArray2D<'a, Raw>)  = safeGet (i+ i') (j + j') a
let (?) c (i':int,j':int) = get i' j' c 
let extract c = get 0 0 c

let extend (f: CArray2D<'a, Raw> -> 'b) (CA2(a, i, j) : CArray2D<'a, Composed>) : CArray2D<'b, Composed> = 
    let w, h = arraySize a
    let f i j =  f(CA2(a,i,j)) 
    let r = Array2D.init w h f
    CA2(r,i,j)

let run (f: CArray2D<'a, Composed> -> CArray2D<'b, Composed>) arr = 
    let (CA2(arr,_,_)) = f (CA2(arr,0,0)) 
    arr
{% endhighlight %} 

Now every line in our code where we used incorrect composition will be marked as an error.
For example, gaussLaplace will be marked with error: the type Composed doesn't match the type Raw. 
Let’s fix it.
{% highlight fsharp %} 
let contours x = 
        let y = extend gauss2D x
        let z = extend gauss2D y
        let yz = zip y z
        let w = extend (extract >> fun (a,b) -> a + b) yz
        extend laplace2d w

let gaussLaplace = extend gauss2D >> extend  laplace2d
{% endhighlight %} 

Now it works as expected.
For example, on my machine contours filter:
First version takes a lot of time (x minutes) to execute.
Lazy version takes about 16s. 
Final version takes about 242ms. 

[Final code](https://gist.github.com/hodzanassredin/017f0c6d4435067c5fc9) 
[Lazy array version](https://gist.github.com/hodzanassredin/1cf3914b67c2f68dc26c)
 
# Recommended reading: 
1. [The codo-notation package](https://hackage.haskell.org/package/codo-notation) 
2. [Coeffects: The next big programming challenge](http://tomasp.net/blog/2014/why-coeffects-matter/)
3. [Comonads in Haskell](http://www.slideshare.net/davidoverton/comonad) 
