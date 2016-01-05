---
published: true
layout: post
title: Book review Beginning Haskell A Project-Based Approach.
tags : [Haskell, book review]
---

## {{page.title}}

<p class="meta">05 January 2016 &#8211; Karelia</p>

Several years ago I tried to read different books about Haskell and had never
finished. For example when I started "[Real World Haskell](http://book.realworldhaskell.org/)"
everything was ok until [chapter about monads](http://book.realworldhaskell.org/read/monads.html).
It was just to hard for me to understand monads when I even had troubles with understanding of (.) and ($) operators.
But now when I spent more than two years with Fsharp I decided to give it another try.
But RWH now seems to be a little bit outdated and I checked several books in stores
and finally bought [Beginning Haskell A Project-Based Approach](http://www.apress.com/9781430262503)
by Alejandro Serrano Mena. Yesterday I finally finished it and I'm proud of that.))

#Book summary
Every book has three types of content: boring, interesting, hard. Mainly theses types are subjective because every reader has different skills and knowledge and the book, at the same time, focuses on a wide audience.
And it is always hard to keep a balance between simplicity and complexity. So I prefer books where every chapter is orthogonal to each other so I can skip parts which I understand and concentrate on interesting and hard parts.
But book usually is not just a collection of articles, but has some context and main theme to connect different concepts and describes how to use them together. And now I can conclude that this book probably not perfect, but very good in terms of that. Before reading I had knowledge about monads, monoids, type classes, monad transformers, but I never had a whole picture. And this book finally connects everything in my head. The book is modern so it descries latest changes in GHC and mention [Stack](http://docs.haskellstack.org/en/stable/README.html). It is also practical, main theme is how to build online store in Haskell. But do not expect something like [Agile Web Development with Rails 4](https://pragprog.com/book/rails4/agile-web-development-with-rails-4). It is always funny when people from Haskell, Golang, Fsharp(Suave.IO ;-)) ... are trying to sell their minimal web frameworks as a "You can do everything".
All of them are too far away from Django and Rails. I recommend not to fight with giants, but to sell them for more appropriate tasks. For example Golang's http is for speed, Suave.IO is for minimalistic web services/front ends with very complex logic or for extremely complex SPAs when used with WebSharper or when you need to keep common code base for frontend Xamarin app and backend api server. So the book is very practical, it even describes K-means for clients clustering. The book describes a lot, but unfortunately some things are mentioned, but not described(free monads) or not mentioned at all(comonads). So if you are like me or just a fp beginner I highly recommend this book.

#Chapters summary

## Chapter 1
- Boring: Installation. What is FP. Why Haskell?

## Chapter 2
- Boring: Basic types, pattern matching.

## Chapter 3
- Boring: Lists, parametric polymorphism, partial application, currying, modules, folds.
- Interesting: Haskell origami, equational reasoning.

## Chapter 4
- Boring: packaging, containers(map, set, trees), ad hoc polymorphism(type classes), binary trees.
- Interesting: graph, monoid, monoidal cache, functor, foldable

## Chapter 5
- Interesting: thunk, profiling, strict fields.
- Hard: lazy model of evaluation

## Chapter 6
- Boring: monad, monad laws, monads(state, reader, writer)
- Interesting: k-means clustering, lenses, monads(RWS, ST)

This chapter is where hard stuff starts. It was also interesting to find that [Update computation expression](http://tomasp.net/blog/2014/update-monads/) also exists as RWS monad in Haskell.

## Chapter 7
- Boring: monads(list), monad transformers.
- Interesting: monads(MonadPlus, Logic), Association Rule Support(Apriori algorithm), liftm vs 'op', monadic classes.

## Chapter 8
- Interesting: monad(Par,STM), stm queues, cloud haskell.

Cloud Haskell is a super combo of mbrace with erlang.

## Chapter 9
- Boring: IO
- Interesting: errors and exceptions MonadError, lazy IO(Conduit)

## Chapter 10
- Boring: text, parsers(autoparsec), JSON.
- Interesting:Applicative, Traversable

I thought that python 2 strings is a mess. No. Haskell is a clear winner: 5 types for string representation !!!

## Chapter 11
- Boring: database(Persistent and Esqueletto).

It is always funny for dot net dev(linq, type providers) to read about how cool is XXX strongly typed lib for db access.
This part is a good example of how Haskell sucks when comparing with more modern c# and f#. All that dirty template workarounds.

## Chapter 12
- Boring: web(scotty, fay)

## Chapter 13
- Boring: embedded dsls(shallow, deep).
- Interesting and fucking hard: depended types(Idris), type families, Functional Dependencies, type promotion(singletons).
- Hard:GADT

I know GADT and phantom types, but description in that part really sucks. Really hard to understand what is going on.

## Chapter 14
- Interesting: attribute grammars(UUAGC) and how it relates with monads(Reader,Writer) and monoids, origami programming(cata, ana, d-algebra, recursion-schemes)

## Chapter 15
- Boring: documentation, testing(HUnit, SmallCheck, QuickCheck).
- Interesting: formal verification(idris)

## Chapter 16
- Interesting: patterns, relation with oop, free monad(just an example without description).

## Appendix 1
Links, summary.

## Appendix 2
[Spoiler do not click if you are going to read the book ;-)](https://github.com/DanBurton/tardis)]

#P.S.
I still doesn't know Haskell well and have just a small understanding of monads, but now I can read Haskell code.
Will I use Haskell? Definitely no. I've already have golang for speed, python for ready to use libs, ml and web programming, fsharp for business logic and web/mobile UI. So I don't see a place for Haskell in my life. But now I have ability to bring some patterns from Haskell to my Fhsarp code and also know some interesting stuff to learn like attribute grammars. What's next? I've almost finished to learn existing mainstream languages. This year I'm going to refresh my knowledge of clojure/racket and will try to use them intensively. Because SICP is still the best. I will write a blog post about that next January. Will be interesting to finally compare clojure and fsharp. Who is number one?

#Further reading
- [Origami programming](http://www.cs.ox.ac.uk/people/jeremy.gibbons/publications/origami.pdf)
- [Functional Programming with Bananas, Lenses, Envelopes and Barbed Wire](http://eprints.eemcs.utwente.nl/7281/01/db-utwente-40501F46.pdf)
