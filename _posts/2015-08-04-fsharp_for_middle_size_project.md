---
published: true
layout: post
title: FSharp for middle size projects.
tags : [csharp, fsharp, monad, js]
---

## {{page.title}}

<p class="meta">08 April 2015 &#8211; Karelia</p>

Fsharp and real world project.

I spent some time with Fsharp on a full time job and have some thoughts to share. We creates some ml oriented stuff and have code in c#, js, fsharp. So have a lot of interoperability between languages. Unfortunately I don't remember some real world cases and examples for some points. 
DISCLAIMER: This is just my humble opinion and I do not pretend to have the only truth. 

1. Type providers

Initialy we started to use them intensively, but after some time we removed thir usages from our codebase. First problem is scripts. When you created script with a type provider and copies it to some server first of all it shows you "The .NET SDK 4.0 or 4.5 tools could not be found". Also it increases a number of dependencies and is not very easy even if you using a Chocolately  and so on. Type safety of type providers sometimes become a hell when for examle you are woking with json type provider and forces you to create a lot of boilerplate code. For example to proccess json value which can be null, skipped, empty or has value. Also json type provider forces you to use F# Data: JSON Parse module but what if I prefere to use other json parser?

Try to use only when you really need them.

2. Tuples and other data structures with unnamed fields.

Initialy we started to use them actively. For easier reading we actively used type abbreviations for every position and different type abbreviations for. But type abbreviation are erased at compile time and this way is too error prone. For example when you inserts additional field into tuple or union type you are forced to fix a lot of lies where you are trying to deconstruct values. Also it is very easy to do a mistake and fail at runtime when your json serializer can't deserialize message and throws an exception. 

3. Inlining

Seems to be coll but can throw exception at runtime when used form csharp.
Use only internally inside Fsharp code and mainly for perfomance issues. 

4. Pattern matching and union types

Incorrectly used it move you code and brain into the "switch case hell" which you could remember from C like languages. Currently we partially rewrites code which uses union types to simple OOP classes and interfaces. Try to avoid union types creation but if you really need them for tasks like AST representation when you shoure that your items are const number of representations. Also use only name fields for union types. And before writing just think about visitor pattern again :). NEVER USE union types in your rest/json or other types of external api. I did it once and never do that again. 

5. Immutability

Is a defenitely way to go. It can reduce a lot of bugs and simplefies your code a lot. You dont need to think about cloning problems.
Try to use them actively but do not try to skip mutability at all.

6. Monads and computation expressions

Monads is very hard to understand and they do not compose. So try to avoid creation of your small monads and use predefined ones of from libs. Develop your own only when you really need it.

7. Automatic generation of Equals, GetHashCode and so on.

Outstanding feature of the language. But current version have some performance issues will be fixed in next versions of fsharp by different compilation providers.

8. Collections

Inconsistent api of different collections (fixed in F# 4) causes dramatic time loss in developing. Missing of some really useful methods like OrderByDescending (fixed in f# 4) can show you a lot of interesting stuff during debugging of production server :). No built in generic performant solution for collections. Seq module is too slow. Currently you can use perfect Fsharp streams or some kind of Reducers library. Hope some of them will be integrated into next versions of csharp and fsharp like clojure do. 

9. Fsharp idiomatic libs

Unfortunately after some use we decided to swith to more well tested production ready libs. They are not so idiomatic but easier to work form both fsharp and csharp.

10. Type inferrence.

After some time almost zero problems but you nee to understand limitations like F# does not allow type constraints involving two different type parameters.

11. OOP support

Currently we moves more and more code to OOP style from FP style. OOP feels just better and easier for maintain when your project starts growing.

Hope these notes will be helpfull.




