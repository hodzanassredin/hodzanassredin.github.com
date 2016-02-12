--- 
published: true 
layout: post 
title: DDD with DSL monad in fsharp 
tags : [fsharp, dsl, comonoid] 
--- 
 
 
## {{page.title}} 
 
 
 
 
<p class="meta">12 February 2016 &#8211; Karelia</p> 
 
 
#What is DDD? 
Lets check a description from Wikipedia. 
> Domain-driven design (DDD) is an approach to software development for  
> complex needs by connecting the implementation to an evolving model. The 
> premise of domain-driven design is the following: 
 
 
> - placing the project's primary focus on the core domain and domain logic 
> - basing complex designs on a model of the domain 
> - initiating a creative collaboration between technical and domain experts to iteratively refine a conceptual model that addresses particular domain problems 
 
 
DDD was described by Eric Evans in [Domain-Driven Design: Tackling Complexity in the Heart of Software](http://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215) 
If we dig deeper then we will find a term Ubiquitous Language.  
Short description from [Martin Fowler](http://martinfowler.com/bliki/UbiquitousLanguage.html) 
 
 
> Ubiquitous Language is the term Eric Evans uses in Domain Driven Design for  
> the practice of building up a common, rigorous language between developers  
> and users. This language should be based on the Domain Model used in the 
> software - hence the need for it to be rigorous, since software doesn't cope 
> well with ambiguity. 
 
 
So in short it is all about language between domain experts and developers.  
Usually it means that in our code we have to abstract our code from infrastructure details and describe everything in terms of a business model. 
Main pattern in OOP languages is to use Interface as abstraction and encapsulate infrastructure details in a class which implements an interface. All other classes depends only from other interfaces, so we can easy swap implementation. Main patterns for composition are [Dependency injection](https://en.wikipedia.org/wiki/Dependency_injection) and [Inversion of control](https://en.wikipedia.org/wiki/Inversion_of_control).  
 
 
There are a lot of tutorials how to do DDD in c#, but what about f#? 
Actually we can use the same way in fsharp, but this way has some drawbacks and we probably could find better way to do that. 
I found several excellent articles about how to do ddd in fsharp.  
For example [A different look at service design](http://bartoszsypytkowski.com/c-a-different-look-at-service-design/) and [Domain Driven Design with the F# type System](http://fsharpforfunandprofit.com/ddd/). They describe how to model domain in terms of functions and types. But we will try to go deeper. 
 
#DSL 
So ddd is about languages. In programming languages we could build our domain language as external or internal dsl. Internal dsls are better because we could use all existing tools of a host language. Internal dsls could be shallow or deep. Shallow embedding means that we describe our domain in types and have fixed interpretation of the data. It is not very useful, because for testing we want to use a fake interpreter and in production use an optimized interpreter. 
So we want to build deep internal dsl. How? We could represent all operations as abstract types and compose abstract computations with a computation builder. But we also want to abstract not only from implementation but also from function effects. DDD with interfaces has this problem. For example dependency without effect. 
{% highlight fsharp %} 
type MyInterface = 
   abstract member Add: int -> int -> int 
type Customer(dependency:MyInterface) =
    member this.AddOneTo x = dependency.Add 1 x |> printfn "%d" 
{% endhighlight %} 

Now we want to add numbers asynchronously. And we have to change everything. 

{% highlight fsharp %} 
type MyInterface =
   abstract member Add: int -> int -> Async<int> 
type Customer(dependency:MyInterface) =
    member this.AddOneAndPrint x = async{ 
            let! r = dependency.Add 1 x 
            printfn "%d" r 
            return () 
          } 
{% endhighlight %} 
Lets start. We need to represent our abstract operations as data types. But  how? In short our command will look like a request to a server(interpreter). 
Request is a type for representation of a command tag and input params 
{% highlight fsharp %} 
type Request<'i> = 'i 
type Dsl = Add of Request<int * int> 
{% endhighlight %} 
But we also need a way to compose commands together. So interpreter needs a way to execute a command and invoke some function which will take command result and return a new command to proceed or some kind of stop signal like return something command. Lets do that. 
{% highlight fsharp %} 
type Request<'i,'o,'k> = 'i * ('o -> 'k) 
{% endhighlight %} 
And for composition we need some bind function which will compose some request with a function which will take output of a command and produce a new one. Internally it will use bind function of continuation type 'k. 

{% highlight fsharp %} 
let bindRequest bind f (s,k) = s, fun v -> bind(k v,f) 
{% endhighlight %} 

This is the main building block of our dsl language. Now we could start to define our domain. 
{% highlight fsharp %} 
type Id = int 
type Entity<'e> = Entity of Id * 'e  
[<Measure>] type money 
type User = {name : string; email : string; balance : int<money>} 
type Product = { name : string; quantity : int; price : int<money>} 
type Email = {body:string; subject : string} 
{% endhighlight %} 
Database operations is a good way to represent dsls composition. So we have for example one language for our problem domain and one as an sub dsl for database ops(Something like DDD's repository pattern). Usually it is not the best way to do things but good enough for our example.  
{% highlight fsharp %} 
type DbOps<'e, 'k> =
      | Select of Request<unit,Entity<'e> seq, 'k> 
      | Get of Request<Id,Entity<'e> option,'k> 
      | Delete of Request<Entity<'e>,unit,'k> 
      | Update of Request<Entity<'e>, unit,'k> 
      | Insert of Request<'e, Id,'k> 
let bindDb v f bind =
            match v with 
            | Select(r) -> Select(bindRequest bind f r) 
            | Get(r) -> Get(bindRequest bind f r) 
            | Delete(r) -> Delete(bindRequest bind f r) 
            | Update(r) -> Update(bindRequest bind f r) 
            | Insert(r) -> Insert(bindRequest bind f r) 
{% endhighlight %} 

As you could see databse dsl in bind uses bindRequest function and additional bind function of parent dsl. Lets finally describe our main dsl. 
{% highlight fsharp %} 
type Dsl<'r> = 
  | UsersTable of DbOps<User, Dsl<'r>> 
  | ProductsTable of DbOps<Product, Dsl<'r>> 
  | SendEmail of Request<User * Email, unit, Dsl<'r>> 
  | Log of Request<string, unit, Dsl<'r>> 
  | Download of Request<string, string, Dsl<'r>> 
  | Pure of 'r  
type DslBuilder() =
    member x.Bind(v:Dsl<'a>,f:'a->Dsl<'b>) =
        match v with 
            | UsersTable(dbOp) -> UsersTable(bindDb dbOp f x.Bind) 
            | ProductsTable(dbOp) -> ProductsTable(bindDb dbOp f x.Bind) 
            | SendEmail(r) -> SendEmail(bindRequest x.Bind f r) 
            | Log(r) -> Log(bindRequest x.Bind f r) 
            | Download(r) -> Download(bindRequest x.Bind f r) 
            | Pure(v) -> f(v) 
    member x.Return v = Pure(v) 
    member x.ReturnFrom v = v
let dsl = DslBuilder() 
{% endhighlight %} 

So as you can see it is just a boilerplate code which probably could be generated by a type provider for you. Now we need to lift our type constructors into our dsl monad. 
 
{% highlight fsharp %} 
let lift ctor i = ctor(i, fun s -> Pure(s))
let usersTable op = lift (UsersTable << op) 
let productsTable op = lift (ProductsTable << op) 
let sendEmail = lift SendEmail 
let log = lift Log 
let download = lift Download 
{% endhighlight %} 
Now we can write programs by using lifted funcs in dsl builder. But before that lest create a fake interpreter. 
{% highlight fsharp %} 
type Db = { 
    users : Map<Id,User> 
    products : Map<Id,Product> 
} 
let runDb (table:Map<_,_>) dbOp  =
    match dbOp with 
          | Select(_,k) -> table
                            |> Seq.map (fun kv ->  Entity(kv.Key, kv.Value))
                            |> k, table 
          | Get(id,k) -> if table.ContainsKey id 
                         then k (Some(Entity(id, table.[id]))), table 
                         else k None, table 
          | Delete(Entity(id,_),k) -> k (), table.Remove id 
          | Update(Entity(id,v),k) -> k (), table
                                              |> Map.remove id
                                              |> Map.add id v 
          | Insert(v,k) -> let id = table |> Seq.map (fun kv -> kv.Key)
                                          |> Seq.max 
                           k id, Map.add (id + 1) v table
open System.Net 
open System.Text 
open System
let rec run db ast = async{ 
    match ast with 
          | ProductsTable(dbOp) -> let r, t = runDb db.products dbOp
                                   return! run {db with products = t} r 
          | UsersTable(dbOp) -> let r, t = runDb db.users dbOp 
                                return! run {db with users = t} r 
          | SendEmail((u,e), k) -> printfn "Sending email to: %s\nsubject:%s\nbody:%s" u.name e.subject e.body 
                                   return! k() |> run db 
          | Log(r,k) -> printfn "Log(%A):%s" (System.DateTime.Now) r 
                        return! k() |> run db 
          | Download(r,k) -> let client = new WebClient() 
                             client.Encoding <- Encoding.GetEncoding("utf-8") 
                             let! html = client.AsyncDownloadString(new Uri(r)) 
                             return! k(html) |> run db 
          | Pure(v) -> return v, db
    } 
{% endhighlight %} 

As you can see our abstract download function has no effects because all effects exist only in interpreter. In our case download has async effect.  
So everything is ready for programming. 

{% highlight fsharp %} 
let optZip a b = 
    match a,b with 
        | Some(a),Some(b) -> Some(a,b) 
        | _ -> None
let transferProductToUser p u count  =  
    if p.quantity < count  
    then Choice2Of2(sprintf "no enough quantity product '%s' quantity %d requested %d" p.name p.quantity count) 
    elif u.balance < p.price * count 
    then Choice2Of2(sprintf "no enough money product '%s' user balance %d requested sum %d" p.name u.balance (p.quantity * p.price)) 
    else let p = {p with quantity = p.quantity - count} 
         let u = {u with balance = u.balance - p.price * count} 
         Choice1Of2(p,u)
let getAndLog tname table id = 
    dsl{ 
        let! p = table Get id 
        match p with  
            | Some(_) -> return p 
            | None -> do! log <| sprintf "unable to find %d in %s" id tname 
                      return None 
    } 
let buyProduct pid uid count = dsl{ 
    let! p = getAndLog "products" productsTable pid 
    let! u = getAndLog "users" usersTable uid 
    match optZip u p with 
        | Some(Entity(idu,u), Entity(idp,p)) -> 
                    do! log("found product and user")  
                    let res = transferProductToUser p u count 
                    match res with 
                        | Choice1Of2(p,u) -> do! sendEmail (u, { body = "you bought a product";
                                                                 subject = sprintf "success bought %s quantity %d" p.name count}) 
                                             do! productsTable Update (Entity(idp,p)) 
                                             do! usersTable Update (Entity(idu,u)) 
                                             return true 
                        | Choice2Of2(error) -> do! log(error) 
                                               return false 
        | None -> return false 
}  
let program quantity = dsl{ 
    let! isOK = buyProduct 1 1 quantity 
    if isOK  
    then let! html = download "http://google.com" 
         return "ok " + html.Substring(0,100)   
    else return "not ok" 
}  
let db = { 
    users = [1 , {name = "hodza";  
                  email ="email@mail.com";  
                  balance = 100<money>}] |> Map.ofList 
    products = [1, {name="Fsharp fun and drugs";  
                    quantity = 10;  
                    price = 1<money>}] |> Map.ofList 
} 
{% endhighlight %} 

lets run our abstract program with fake interpreter. 

{% highlight fsharp %} 
printfn "run interpret" 
printfn "before %A" db 
run db (program 9)  |> Async.RunSynchronously |> printfn "result quantity 9 %A" 
run db (program 21) |> Async.RunSynchronously |> printfn "result quantity 21 %A" 
{% endhighlight %} 

and the result is 

{% highlight fsharp %} 
//run interpret 
//before {users = map [(1, {name = "hodza"; 
//                   email = "email@mail.com"; 
//                   balance = 100;})]; 
// products = map [(1, {name = "Fsharp fun and drugs"; 
//                      quantity = 10; 
//                      price = 1;})];} 
//Log(11.02.2016 18:21:36):found product and user 
//Sending email to: hodza 
//subject:success bought Fsharp fun and drugs quantity 9 
//body:you bought a product 
//result quantity 9 ("ok &lt;!doctype html&gt; &lt;html itemscope="" itemtype="http://schema.org/WebPage" lang="ru"&gt; &lt;head&gt;&lt;meta content", 
// {users = map [(1, {name = "hodza"; 
//                    email = "email@mail.com"; 
//                    balance = 91;})]; 
//  products = map [(1, {name = "Fsharp fun and drugs"; 
//                       quantity = 1; 
//                       price = 1;})];}) 
// =============================================================== 
//Log(11.02.2016 18:21:36):found product and user 
//Log(11.02.2016 18:21:36):no enough quantity product 'Fsharp fun and drugs' quantity 10 requested 21 
//result quantity 21 ("not ok", {users = map [(1, {name = "hodza"; 
//                              email = "email@mail.com"; 
//                              balance = 100;})]; 
//            products = map [(1, {name = "Fsharp fun and drugs"; 
//                                 quantity = 10; 
//                                 price = 1;})];}) 
{% endhighlight %} 
 
Code [here](https://gist.github.com/hodzanassredin/8183dd8ded6d47d55de0) 
 
We used an fake ad hoc interpreter in this post, but we could describe interpreter as a data structure(comonad). It allows us to abstract and compose interpreters. More advanced version with comonadic interpreter [here](https://gist.github.com/hodzanassredin/1ae9fad4316bdb502fc9) 
 
#Result 
Now we can write abstract programs which are easy to test with fake interpreters and we abstract our code from execution effects. No more code rewrite when we decided to use async/lazy version of some method. 
 
#Recommended reading: 
1. [Cofun with cofree comonads](http://dlaing.org/cofun/) 
 
 
