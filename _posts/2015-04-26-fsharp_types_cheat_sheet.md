---
published: true
layout: post
title: Fsharp types as a composition of smaller ones.
tags : [azure, fsharp, monad, ditributed]
---

## {{page.title}}

<p class="meta">26 April 2015 &#8211; Karelia</p>

#Description
Just a small cheat sheet, it describes some f# types as a composition of smaller ones. Sometimes it is easier to understand a difference between some types in behaviour and semantics by looking at their type’s signatures. However, types could be very complex, so we want to exam types without infrastructure details and probably as a composition of smaller ones. For example with this cheat sheet, it is easy to understand a difference between some very similar types like Observable<'T> and NessosStream<'T> and see that NessosStream is just  an impure reducer.
{% highlight fsharp %}
open System
//duals
type Map<'T,'T2> = 'T -> 'T2
type MapBack<'T,'T2> = 'T2 -> 'T

type Pull<'T> = Map<unit,'T> //input side effect
type Push<'T> = MapBack<unit,'T> //output side effect 
type Do = Map<unit,unit>
type Lazy<'T> = Pull<'T>
type Or<'T1,'T2> = Left of 'T1 | Right of 'T2
type And<'T1,'T2> = 'T1 * 'T2
type Cont<'T,'T2> = Map<Map<'T,'T2>, 'T2>
type Maybe<'T> = Or<'T, unit>
//different semantics of maybe
type Finite<'T> = Maybe<'T>
type Cancelable<'T> = Maybe<'T>
type Throwable<'T> = Or<'T,Exception>
type Unsubscribe = Do

type InfiniteIterator<'T> = Pull<'T>
type Iterator<'T> = InfiniteIterator<Finite<'T>>
type Iteratable<'T> = Pull<Iterator<'T>>
type Observer<'T> = Push<Throwable<Finite<'T>>>
type Observable<'T> = Map<Observer<'T>,Unsubscribe>
type IEvent<'T> = Push<Push<'T>>
type Event<'T> = And<IEvent<'T>,Push<'T>>
type Id<'T> = Map<'T,'T>
type Reader<'TState, 'T> = Map<'TState,'T>
type Writer<'TState, 'T> = And<'TState,'T>
type State<'TState, 'T>  = Map<'TState, And<'TState,'T>>
type Update<'TState,'TUpdate,'T> = Map<'TState, And<'TUpdate,'T>>

type Curry<'T,'T2,'T3> = Map<Map<And<'T,'T2>,'T3>,Map<'T, Map<'T2,'T3>>>
type Async<'T, 'T2> = Cont<Cancelable<Throwable<'T>>, 'T2>
type Recursion<'T> = Recursion of And<'T,Lazy<Recursion<'T>>>
type FiniteRecursion<'T> = FiniteRecursion of Finite<Lazy<And<'T,FiniteRecursion<'T>>>>
type AsyncSeq<'T> = AsyncSeq of Async<Finite<And<'T, AsyncSeq<'T>>>>
type List<'T> = List of Finite<And<'T, List<'T>>>
type LazyList<'T> = LazyList of Finite<Lazy<And<'T, LazyList<'T>>>>
type Reducer<'TACC,'T> = Map<And<'TACC,'T>, 'TACC>
type Reducible<'TACC,'T> = Map<And<Reducer<'TACC,'T>, 'TACC>, 'TACC>
type Transducer<'TACC,'T> = Map<Reducer<'TACC,'T>,Reducer<'TACC,'T>>
type Composition<'T> = Reducer<'T,'T>
type Composable<'T> = Reducible<'T,'T>
type Dictionary<'TKey, 'TValue> = And<Map<'TKey,'TValue>,Push<And<'TKey,'TValue>>>
type Array<'T> = Dictionary<int,'T>

type NessosStream<'T> = Reducible<unit,'T>

type ContReader<'TENV, 'T, 'TResult> = Reader<'TENV, Cont<'T, 'TResult>>

type MailBox<'T> = Pull<'T>
type Actor<'T> = Map<MailBox<'T>, Async<unit>> 
{% endhighlight %}
