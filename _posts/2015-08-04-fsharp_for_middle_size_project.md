---
published: true
layout: post
title: FSharp for middle size projects.
tags : [csharp, fsharp, monad, js]
---

## {{page.title}}

<p class="meta">08 April 2015 &#8211; Karelia</p>

I spent some time with Fsharp on a full time job and have some thoughts to share. We creates some ml oriented stuff and have code in c#, js, fsharp. So we have a lot of interoperability between languages. Unfortunately I don't remember some real world cases and examples for some points. 

#Type providers

Initialy we started to use them intensively, but after some time we removed all usages from our codebase. First problem is scripts. When you created a script with a type provider and copies it to some server, first of all it shows you "The .NET SDK 4.0 or 4.5 tools could not be found". Also it increases a number of dependencies and is not very easy even if you are using Chocolately  or so on. Type safety of type providers sometimes become a hell when for example you are working with json type provider and it forces you to create a lot of boilerplate code. For example to process json value which can be null, skipped, empty or has value. Also json type provider forces you to use F# Data: JSON Parse module but what if I prefer to use an other json parser? Db type providers seems to be similar but all have different possibilities and problems and differnt api. It is easier to use EF with blackjack and migrations. Provaiders seems to be perfect fit for external uncontrolled data sources, but unfortunately with some drawbacks. 
Try to use only when you really need them.

#Tuples and other data structures with unnamed fields.

Initially we started to use them actively. For easier reading we used type abbreviations for every position and different type abbreviations for diferent composite types. But type abbreviation are erased at compile time and this way is too error prone. For example when you inserts or removes a field in tuple or union type you are forced to fix a lot of lines where you are trying to construct or deconstruct values. Also it is very easy to do a mistake and fail at runtime when your json serializer can't (de)serialize message and throws an exception. 

#Inlining

Seems to be cool but can throw exception at runtime when used form csharp.
Use only internally inside Fsharp code and mainly for performance issues. 

#Pattern matching and union types

Incorrectly used it moves your code and brain into the "switch case hell" which you could remember from C like languages. Currently we partially rewrites code which uses union types to simple OOP classes and interfaces. Try to avoid union types creation, but if you really need them for tasks like AST representation then you need to be sure that your type has const number of representations. Also use only named fields for union types. And before writing, just think about visitor pattern again :). NEVER USE union types in your external api like rest/json. I did it once and never do that again. 

#Immutability

It is a definitely way to go. It can reduce a lot of bugs and simplifies your code a lot. You don’t need to think about cloning problems. Try to use them actively, but remember for some tasks mutability is a better fit.

#Monads and computation expressions

Monads is very hard to understand for novice devs and they do not compose. So try to avoid creation of your own monads and use predefined ones or from libs. Develop your own only when you really need it.

#Automatic generation of Equals, GetHashCode and so on.

Outstanding feature of the language. Current version have some performance issues, but it will be fixed in next versions of fsharp by different compilation providers.

#Collections

Inconsistent api for different collections (fixed in F# 4) causes dramatic time loss in developing. Missing of some really useful methods like OrderByDescending (fixed in f# 4) can show you a lot of interesting stuff during debugging on production server :). No built in generic performant solution for collections. Seq module is too slow. Currently you can use perfect Fsharp streams lib or some kind of Reducers library. Hope some of them will be integrated into next versions of csharp and fsharp like clojure do. 

#Fsharp idiomatic libs

Unfortunately after some use we decided to switch to better tested production ready libs. They are not so idiomatic, but easier to work form both fsharp and csharp.

#Type inference.

After some time almost zero problemsб, but you need to understand limitations like: F# does not allow type constraints involving two different type parameters.

#OOP support

Currently we move more and more code to OOP style from FP style. OOP feels just better and easier for maintain when your project starts growing. Usual fp project way: functions with positioned args and pattern matching -> functions with args in one record -> record with closures ->> old school OOP.  

Hope these notes will be helpful.
