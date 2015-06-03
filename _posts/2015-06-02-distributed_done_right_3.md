---
published: true
layout: post
title: Distributed computing done right? Actors.
tags : [fsharp, distributed, actor, protocols]
---

## {{page.title}}


<p class="meta">02 June 2015 &#8211; Karelia</p>

In a previous [post](http://hodzanassredin.github.io/2015/05/14/distributed_done_right_2.html) we found a great way to compose our concurrent processes with csp. Our implementation was not perfect and if you want to use this style of communication in production then you have to check [Hopac library](https://github.com/Hopac/Hopac). But there is another way Actors.
We will discuss mainly actor implementations, but not theories, because I've no PhD in CS. There are a lot of materials in the web about actors and I don't want to write another one (like I did with [monads](http://hodzanassredin.github.io/2014/06/21/yet-another-monad-guide.html)) so it is a boring link post.
#Description
Actors can be described as a process with an input unbounded queue.
Actor can react to messages from its input queue and change its state. Actor system guaranties that only one instance of actor is working at the same time (if actor is statefull). So there is no concurrent access to actor's state. Also actors can create other actors and so on. 

#Why actors?
There is a well-known way to distribute and parallelize work in a real world: async servers. What is an async server? It is a process which can accept request from a client and return response after some time [request–reply pattern](http://en.wikipedia.org/wiki/Request%E2%80%93response). It will not block requester as a CSP process, but will try to handle as much requests as possible. If one instance is not enough, we should increase instances count and put a load balancer in front of them. In case when there is no enough resources to handle requests right now, then all messages will be stored in the input queue and processed later. if we have some real world limitation of our input queue size(limited by server's memory) then we could throttle messages and probably return an error to a client or back our queue by a real unbounded queue for example Azure Storage queue. Clients also can use fire and forget pattern in that case server don't have to send a response at al.
Actors uses the same way to do their work, so it is easy to understand them. Also if you have a class then it could be trivially implemented as an actor, because all classes follows request/response pattern. So it is easy for programmers to use them.

#Why not multiple channels like in csp?
Because of guarded choice operator which is not easy to implement and has some problems:

1. Starvation. The use of synchronous channels can cause starvation when a process attempts to get messages from multiple channels in a guarded choice command. For example you can take values only from one channel and not from other.
2. Livelock. The use of synchronous channels can cause a process to be caught in livelock when it attempts to get messages from multiple channels in a guarded choice command.
3. Efficiency. The use of synchronous channels can require a large number of communications in order to get messages from multiple channels in a guarded choice command.

#Why not a synchronous (bounded channel)?
Because it is harder to implement bounded channel. Main idea that we start from the simplest construct and add additional features on top of it. It is the same as tcp vs udp. Everyone is using tcp protocol, but after working with a protocol for remote controlled cars, it is clear for me, that it is a bad idea to use tcp(when you are limited in resources). It adds a lot of handshakes and not needed guaranties. You just can't remove handshakes. There a lot of questions on stackoverflow  “Why my super cool tcp based IoT protocol eats money from sim cards like a hungry shark?” After some time you will start to investigate udp and will found that it is a perfect fit. It easier to understand, easier to add handshaking on top of it, easier to work with. As we saw in a previous post BlockingQueueAgent adds possibility to use an actor as a bounded queue.

#Is it Agent?
No actors are not Agents. For example agents in Clojure. In Agents the behavior is defined outside and is pushed to the Agent, and in Actors the behavior is defined inside the Actor.
Also "agent" is used (as name) in  ConcurrentConstraintProgramming and ReactiveDemandProgramming and has completely different meaning, check [this](http://c2.com/cgi/wiki?ActorVsAgent)

#Implementations
There are several actor implementations for fsharp.

1. ##MailboxProcessor 
Well documented and widely used.
Let’s write a simple Logging actor. 
{% highlight fsharp %}
type Logger() =
    let logger = 
    	MailboxProcessor<string>.Start(
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

This simple actor are wrapped into a class and can be used as a regular object. More info in [blog posts](http://blogs.msdn.com/b/dsyme/archive/2010/02/15/async-and-parallel-design-patterns-in-f-part-3-agents.aspx) form  Don Syme's WebLog.
Mailbox processor has no built in ability to be distributed.

2. #[FSharp.CloudAgent](http://isaacabraham.github.io/FSharp.CloudAgent/) 

It uses Azure Service Bus as a transport. More info [Distributing the F# Mailbox Processor](https://cockneycoder.wordpress.com/2014/12/04/distributing-the-f-mailbox-processor/)

{% highlight fsharp %}
open FSharp.CloudAgent
open FSharp.CloudAgent.Messaging
open FSharp.CloudAgent.Connections

// Standard Azure Service Bus connection string
let serviceBusConnection = ServiceBusConnection "servicebusconnectionstringgoeshere"

// A DTO
type Person = { Name : string; Age : int }

// A function which creates an Agent on demand.
let createASimpleAgent agentId =
    MailboxProcessor.Start(fun inbox ->
        async {
            while true do
                let! message = inbox.Receive()
                printfn "%s is %d years old." message.Name message.Age
        })

// Create a worker cloud connection to the Service Bus Queue "myMessageQueue"
let cloudConnection = WorkerCloudConnection(serviceBusConnection, Queue "myMessageQueue")

// Start listening! A local pool of agents will be created that will receive messages.
// Service bus messages will be automatically deserialised into the required message type.
ConnectionFactory.StartListening(cloudConnection, createASimpleAgent >> BasicCloudAgent)
{% endhighlight %}

3. #[Orleans](https://github.com/dotnet/orleans).

Orleans is an actor framework from Microsoft. Its main ideas are virtual actors and actor representation as an OOP class.
##Virtual actors
> 1. Perpetual existence: actors are purely logical
> entities that always exist, virtually. An actor cannot be
> explicitly created or destroyed and its virtual existence is
> unaffected by the failure of a server that executes it.
> Since actors always exist, they are always addressable.
> 2. Automatic instantiation: Orleans’ runtime
> automatically creates in-memory instances of an actor
> called activations. At any point in time an actor may
> have zero or more activations. An actor will not be
> instantiated if there are no requests pending for it. When
> a new request is sent to an actor that is currently not
> instantiated, the Orleans runtime automatically creates
> an activation by picking a server, instantiating on that
> server the .NET object that implements the actor, and
> invoking its ActivateAsync method for initialization. If
> the server where an actor currently is instantiated fails,
> the runtime will automatically re-instantiate it on a new
> server on its next invocation. This means that Orleans
> has no need for supervision trees as in Erlang [3] and
> Akka [2], where the application is responsible for recreating
> a failed actor. An unused actor’s in-memory 
> 3 instance is automatically reclaimed as part of runtime
> resource management. When doing so Orleans invokes
> the DeactivateAsync method, which gives the actor an
> opportunity to perform a cleanup operation.
> 3. Location transparency: an actor may be
> instantiated in different locations at different times, and
> sometimes might not have a physical location at all. An
> application interacting with an actor or running within an
> actor does not know the actor’s physical location. This is
> similar to virtual memory, where a given logical memory
> page may be mapped to a variety of physical addresses
> over time, and may at times be “paged out” and not
> mapped to any physical address. Just as an operating
> system pages in a memory page from disk automatically,
> the Orleans runtime automatically instantiates a noninstantiated
> actor upon a new request
> 4. Automatic scale out: Currently, Orleans supports
> two activation modes for actor types: single activation
> mode (default), in which only one simultaneous
> activation of an actor is allowed, and stateless worker
> mode, in which many independent activations of an actor
> are created automatically by Orleans on-demand (up to a
> limit) to increase throughput. “Independent” implies that
> there is no state reconciliation between different
> activations of the same actor. Therefore, the stateless
> worker mode is appropriate for actors with immutable or
> no state, such as an actor that acts as a read-only cache

So in short it has main aim: simplify distributed programming for developers without any knowledge of distributed programming and messaging patterns. So they are created an abstraction on top of actors. Imho it is something like Asp.Net WebForms which emulates statefull controls and pages on top of stateless protocol. Web forms are simple on start, but is a total mess in a difficult scenarios. Ajax UpdatePanel is a monster, it is still trying to destroy my projects in my nightmares. 
Orleans uses code generation for proxy classes’ creation and uses custom task scheduler. Both of them have some pros and cons.
For example currently it is not an easy task to use fshrp Async with custom task scheduler. More info [here](https://github.com/dotnet/orleans/issues/38) and [here](http://stackoverflow.com/questions/24813359/translating-async-await-c-sharp-code-to-f-with-respect-to-the-scheduler). There is a project [Orleankka](https://github.com/yevhen/Orleankka)which tries to address some problems and do orleans programming more akka like. 
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
As you can see, there is a "task" computation builder instead of "async", we have to use it to prevent problems with Orleans’s custom task scheduler (deadlocking).    
You can find more documentation [here](http://dotnet.github.io/orleans/). Orleankka introduction is [here](https://medium.com/@AntyaDev/introduction-to-orleankka-5962d83c5a27)

4. #[Akka.net](http://getakka.net/)

This is a port of a well-known Akka framework. So a lot of documentation and usages in production. Current version of Akka.net is suitable for production use. This implementation is not as abstract as Orleans and gives us less guaranties and more control. Integration with fsharp implemented as "actor" computation expression. Let’s check hello world in akka.net.
{% highlight fsharp %}
let aref =
    spawn system "my-actor"
        (fun mailbox ->
            let rec loop() = actor {
                let! message = mailbox.Receive()
                printfn "%A" message
                return! loop()
            }
            loop())
{% endhighlight %}
I like this code a lot. It uses the same pattern as Fsharp's Mailbox Processor so it is trivial to move your code to akka.net and use akka's benefits. Why "actor" computation expression? Because it allows you to do a remote deployment and do stuff like hot code swap. Internally it uses F# quotations. More details about remote deployments you can find in [Akka.NET Remote Deployment With F#](http://bartoszsypytkowski.com/blog/2014/12/14/fsharp-akka-remote-deploy/). Yes this way to do remote deployment is limited, but they are working on a more interesting Mbrace like deployments [Akka.FSharp.HotLoad](https://github.com/akkadotnet/akka.net/issues/542)

There are some comparisons of akka, erlang vs orleans. It is worth reading.
[A look at Microsoft Orleans through Erlang-tinted glasses](http://theburningmonk.com/2014/12/a-look-at-microsoft-orleans-through-erlang-tinted-glasses/)
[Orleans and Akka Actors: A Comparison(Roland Kuhn)](https://github.com/akka/akka-meta/blob/master/ComparisonWithOrleans.md)
[Orleans, Distributed Virtual Actors for Programming and Scalability Comparison](http://christophermeiklejohn.com/papers/2015/05/03/orleans.html)

5. #[Thespian](http://nessos.github.io/Thespian/)

This is an internal project of Nessos company, it is a part of [MBrace](http://www.m-brace.net/) stack. MBrace is a king of distributed computations and it is a huge win for fsharp community to have it. There are some other extremely useful tools from Nessos [FsPickler](http://nessos.github.io/FsPickler/), [Vagabond](http://nessos.github.io/Vagabond/),[Streams](https://github.com/nessos/Streams)...

There is a quote from github about current project's state.
> We created this library a couple of years ago to power mbrace,
> when actor frameworks weren't that widely available. We released 
> this now as part of our effort to open source the entire mbrace
> stack. So I would say that it is stable but offering it for
> standalone use is not one of our priorities for the moment 
> (lacking documentation etc.)

{% highlight fsharp %}
open Nessos.Thespian

type Msg = Msg of IReplyChannel<int> * int

let behavior state (Msg (rc,v)) = async {
    printfn "Received %d" v
    rc.Reply <| Value state
    return (state + v)
}

let actor =
    behavior
    |> Behavior.stateful 0 
    |> Actor.bind
    |> Actor.start

let post v = actor.Ref <!= fun ch -> Msg(ch, v)

post 42
{% endhighlight %}

5. #[Cricket](http://fsprojects.github.io/Cricket/)

quote form [Introducing Cricket (formerly FSharp.Actor)](http://www.colinbull.net/2014/11/06/Introducing-Cricket/)
> Cricket, formerly FSharp.Actor, is yet another actor framework. 
> Built entirely in F#, Cricket is a lightweight alternative to Akka 
> et. al. To this end it is not as feature rich as these out of 
> the box, but all of the core requirements like location transpancy,
> remoting, supervisors, metrics and tracing. Other things 
> like failure detection and clustering are in the pipeline 
> it is just a question of time.

{% highlight fsharp %}
let greeter = 
    actor {
        name "greeter"
        body (
            let rec loop() = messageHandler {
                let! msg = Message.receive() //Wait for a message

                match msg with
                | Hello ->  printfn "Hello" //Handle Hello leg
                | HelloWorld -> printfn "Hello World" //Handle HelloWorld leg
                | Name name -> printfn "Hello, %s" name //Handle Name leg

                return! loop() //Recursively loop

            }
            loop())
    } |> Actor.spawn
{% endhighlight %}

5. #[Ractor.CLR](https://github.com/buybackoff/Ractor.CLR)

It is not an actor framework, but very close to actors, it uses process-oriented programming paradigm. In short it is very close to orlean's Virtual Actors. in Ractor actors are virtual and exist in Redis per se as lists of messages, while a number of ephemeral workers (actors' "incarnations") take messages from Redis, process them and post results back.
{% highlight fsharp %}
#r "Ractor.dll"
#r "Ractor.Persistence.dll"

open System
open System.Text
open Ractor

let fredis = new Ractor("localhost")

let computation (input:string) : Async<unit> =
    async {
        Console.WriteLine("Hello, " + input)
    }

let greeter = fredis.CreateActor("greeter", computation)
// type annotations are required
let sameGreeter  = Ractor.GetActor<string, unit>("greeter")
greeter.Post("Greeter 1")
greeter.Post("Greeter 2")
greeter.Post("Greeter 3")
greeter.Post("Greeter 4")
greeter.Post("Greeter 5")

sameGreeter.Post("Greeter via instance from Ractor.GetActor")

// this will fail if computation returns not Async<unit>
"greeter" <-- "Greeter via operator"

()
{% endhighlight %}

#What to choose
Extremely hard question, but I hope now it is more clear for you, how to choose one or another. I prefer to use mailbox processors(MB) and akka.net. You can start from fsx with MB and after that, move your code into a project (using [ProjectScaffold](https://github.com/fsprojects/ProjectScaffold)) and add remoting capabilities by converting(it is simple) your MB actors into akka.net actors.
#No Silver Bullet
Actors and csp are great tools to simplify concurrent programming. They limits shared state, so you don't have to worry about Visibility and Ordering and you don't have to use memory barriers and think about caches and processor’s registers. If you forgot about these kind of problems, so it is time to refresh your memory by reading a [chapter](http://www.albahari.com/threading/part4.aspx) form [Threading in C#](http://www.albahari.com/threading/). Strongly recommend to read, if you want to be a low level concurrency ninja.  
You don't even have to use locks. Also actors are solves additional problems of scalability, transparency and inconsistency. That's great. But can you relax and write stuff without thinking? Unfortunately no. Deadlocks, Starvation, Live-locks and Race Conditions is still here, we will check them in future blog posts and will check how to use other abstractions to prevent them.

Next part will be about Protocols, but it is summer time and I have almost zero feedback to my previous posts, so it will be eventually soon. ;)

#Recommended reading:
1. [An Introduction and Developer’s Guide to Cloud Computing with MBrace](http://www.m-brace.net/mbrace-manual.pdf)
2. [Design patterns/best practice for building Actor-based system](http://stackoverflow.com/questions/3931994/design-patterns-best-practice-for-building-actor-based-system)
