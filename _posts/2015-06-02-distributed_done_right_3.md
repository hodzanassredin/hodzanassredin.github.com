---
published: false
layout: post
title: Distributed computing done right? Actors and protocols.
tags : [fsharp, distributed, actor, protocols]
---

## {{page.title}}


<p class="meta">02 June 2015 &#8211; Karelia</p>

In a previous post we found a great way to compose our concurrent processes with csp. Our implementation was not perfect and if you want to use this style of communication in production then you have to check [Hopac library](https://github.com/Hopac/Hopac). But there is another way Actors.
We will discuss mainly actor implementations but not theories because I've no phd in cs. 
#Description
Actors can be described as a proccess with input unbounded queue.
Actor can react to messages from it's input queue and change his state. Actor system guaranties that only one instance of actor is working at the same time. So there is no concurrent access to actor's state.

#Why actors?
There is a well known way to distribute and parallelize work in a real world we are working with async servers. What is an async server? It is a proccess which can accept request from a client and return response after some time[request–reply pattern](http://en.wikipedia.org/wiki/Request%E2%80%93response). It will not block requester as an CSP proccess, but will try to handle as much requests as possible. If one instance is not enought, we should increase instances count and put a load ballancer in front of them. In case when there is no enought resources to handle requests right now then all messages will be stored in input queue and proccessed later. if we have some real world limitation of our input queue size(limited by server's memory) then we could throtle messages and probably return an error to a client or back our queue by a really unbounded queue for example Azure Storage queue. Clients also can use fire and forget pattern in that case server don't have to send a response.
Actors uses the same way to do their work. It is easy to understand how they work. Also if you have an class then it could be trivially implemented as actor becouse any class follows request/response pattern. So it is easy to programmers to adopt.
#Why not multiple channels like in csp?
Becouse of guarded choice operator which is not easy to implement and has some problems:

1. Starvation. The use of sychronous channels can cause starvation when a process attempts to get messages from multiple channels in a guarded choice command. For example you can take values only from one channel and not from other.
2. Livelock. The use of synchronous channels can cause a process to be caught in livelock when it attempts to get messages from multiple channels in a guarded choice command.
Efficiency. The use of synchronous channels can require a large number of communications in order to get messages from multiple channels in a guarded choice command.

#Why not a synchronouse(bounded channel)?
Because it is harder to implement bounded channel. Main idea that we start from the most simple construct and adds additional features on top of it. 
It is the same as tcp vs udp. Everyone is using tcp protocol but after working with remote conteolled cars protocol, it is clear for me that it just impossible to use tcp. It adds a lot of handshakes and usually not needed guaranties. It behaves like a leaky abstraction. When you can't remove that crazy handshaking. There a lot of questions in stackoverflow Why my super cool tcp based IoT protocol eats money from clients sim card like a hungry shark. After some time you are starting to investigate udp and found that it is a perfect fit. It easier to understand, easier to add hanshaking on top of it, easier to work with. As we saw in a previous post BlockingQueueAgent added posiiblity to use an actor as a bounded queue.

#Is it Agent?
No actors are not Agents. For example agents in clojure.  In Agents the behavior is defined outside and is pushed to the Agent, and in Actors the behavior is defined inside the Actor.
Also agents are used as units in  ConcurrentConstraintProgramming or ReactiveDemandProgramming and has completely different behaviour check [this](http://c2.com/cgi/wiki?ActorVsAgent)
#Implementations
There are 3 main production ready implementations for fsharp.
1. MailboxProccessor is well documented and widly used.
Also mailbox pattern is used by other implementations.
Lets define some simple Logging actor 
{% highlight fsharp %}
type Logger() =
    let logger = MailboxProcessor<string>.Start(
                    fun inbox ->
                            let rec loop () =
                                async {
                                        let! msg = inbox.Receive();
                                        printfn "%s: %s" (DateTime.Now.ToString()) msg
                                        return! loop ()
                                }
                            loop ())
    member x.Log msg = logger.Post(msg)
{% endhighlight %}
This simple actor are wrapped into a class and can be used as a regular object. Mailbox proccesor has no built in ability to be distributed. More info in a [blog posts](http://blogs.msdn.com/b/dsyme/archive/2010/02/15/async-and-parallel-design-patterns-in-f-part-3-agents.aspx) form  Don Syme's WebLog.

2. Orleans.
Orleans is an actors framework from Microsoft. It\s main idea is virtual actors and actors representation in an OOP style. So it has main aim: simplify distributed programming for developers without any knowledge of distributed programming and messaging patterns. So they are created an abstraction on top of actors. Imho it is something like Asp.Net WebForms which emulates statefull controls and pages on top of stateless protocol. Web forms are simple on start but is a total mess in a difficult scenarios. Ajax UpdatePanel is a monster, it sometimes trying to catch me in my nightmares. 
Orleans uses code generation for proxy classes creation and usese custom task scheduler. Both of them have some drawbacks.
traction when you use uncontrolled library which usese Task.Run which uses thread poll scheduler. More info [here](https://github.com/dotnet/orleans/issues/38). There is a project [Orleankka](https://github.com/yevhen/Orleankka)which tries to remove some problems and do orleans programming more functional. 
Simple Hello World actor in orleans
{% highlight fsharp %}
open System
open System.Reflection

open Orleankka
open Orleankka.FSharp
open Orleankka.Playground

type Message = 
   | Greet of string
   | Hi

type Greeter() = 
   inherit Actor<Message>()   

   override this.Receive message reply = task {
      match message with
      | Greet who -> printfn "Hello %s" who
      | Hi -> printfn "Hello from F#!"           
   }

[<EntryPoint>]
let main argv = 

    printfn "Running demo. Booting cluster might take some time ...\n"

    use system = ActorSystem.Configure()
                            .Playground()
                            .Register(Assembly.GetExecutingAssembly())
                            .Done()
                  
    let actor = system.ActorOf<Greeter>(Guid.NewGuid().ToString())

    let job() = task {
      do! actor <! Hi
      do! actor <! Greet "Yevhen"
      do! actor <! Greet "AntyaDev"
    }
    
    Task.run(job) |> ignore
    
    Console.ReadLine() |> ignore    
    0
{% endhighlight %}
As you can see you have to use "task" computation builder instead of "async" we have to use it to prevent problems with orlean's custom task scheduler. In fsharp Mailbox this problem doesn't exist because we are in a loop and only way to break it to   
You can find more documentation [here](http://dotnet.github.io/orleans/). Orleankka introduction is [here](https://medium.com/@AntyaDev/introduction-to-orleankka-5962d83c5a27)
[Translating async-await C# code to F# with respect to the scheduler](http://stackoverflow.com/questions/24813359/translating-async-await-c-sharp-code-to-f-with-respect-to-the-scheduler)
[A look at Microsoft Orleans through Erlang-tinted glasses](http://theburningmonk.com/2014/12/a-look-at-microsoft-orleans-through-erlang-tinted-glasses/)
[Orleans and Akka Actors: A Comparison(Roland Kuhn)](https://github.com/akka/akka-meta/blob/master/ComparisonWithOrleans.md)
3. Akka.net
This is a port of well known Akka fromework. So a lot of documentation and uses in production. Current version of Akka.net is suitable for production use. This implementation is not so abstract as Orleans and gives us less guaranties and more control. Intergation with fsharp implemented as "actor" computation expression. Lets check hello world in akka.net.
{% highlight fsharp %}
{% endhighlight %}
#No Silver Bullet


{% highlight fsharp %}
{% endhighlight %}
