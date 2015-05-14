---
published: true
layout: post
title: Distributed computing done right? Maximizing workers throughput.
tags : [azure, fsharp, distributed, csp, patterns]
---

## {{page.title}}


<p class="meta">14 May 2015 &#8211; Karelia</p>

Let’s continue our investigation about how to create real world distributed apps. But before we continue, we should find a well-established dictionary for our problem domain.
#Patterns
Distributed computing have a long history and there are a lot of patterns which works better than others. And just to read and understand other posts and articles about distributed computing we have to know pattern names and terminology. Distributed patterns have a lot in common with patterns of concurrency, parallelism and asynchronous methods. Distribution just adds additional level of complexity related with placement, connection speed, time synchronization and so on. Distributed patterns tries to solve problems caused by physical world in your code.
There are some links where you could find patterns description.

1. [Cloud Design Patterns: Prescriptive Architecture Guidance for Cloud Applications](https://msdn.microsoft.com/en-us/library/dn568099.aspx) 
2. [Patterns and Best Practices for Enterprise Integration](http://www.enterpriseintegrationpatterns.com/)
3. [Design Patterns for Decomposition and Coordination on Multicore Architectures](https://msdn.microsoft.com/en-us/library/ff963553.aspx)
4. [REACTIVE DESIGN PATTERNS](http://www.typesafe.com/resources/e-book/reactive-design-patterns)
5. [Seven Concurrency Models in Seven Weeks: When Threads Unravel](https://pragprog.com/book/pb7con/seven-concurrency-models-in-seven-weeks)

I think you can easily find more, but these resources were very helpful for me.
Now we can name some patterns naively implemented in [previous post](http://hodzanassredin.github.io/2015/04/18/fsharp_and_azure.html):
Pipes and Filters, Routing Slip...
#Problem
Now let’s describe a problem which is usual when you are developing a worker role. For example we have a task to process a lot of images, apply some filter to them. We could develop a worker role which will grab image ids from an input queue, load, and process and save them in parallel. We can also setup auto scaling to add or remove additional role instances based on the input queue length. 
Probably our worker is cpu bound, but also it could be io bound if image storage will be not so fast. Let’s represent a problem as a simple function. 

{% highlight fsharp %}
let numImages = 2000
let size = 512
let numPixels = size * size
let processImageRepeats = 20
let mutable io_bound = true

let transformImage (pixels, imageNum) =
    for i in 1 .. processImageRepeats do 
        pixels |> Array.map (fun b -> b + 1uy) |> ignore
    pixels |> Array.map (fun b -> b + 1uy)

let processImageSync i =
    use inStream =  File.OpenRead(sprintf "Image%d.tmp" i)
    if io_bound then Thread.Sleep(200)
    let pixels = Array.zeroCreate numPixels
    let nPixels = inStream.Read(pixels, 0, numPixels);
    let pixels' = if not io_bound || i % 2 = 0 
                        then transformImage(pixels, i) 
                        else pixels
    use outStream =  File.OpenWrite(sprintf "Image%d.done" i)
    outStream.Write(pixels', 0, numPixels)

{% endhighlight %}

We wrote simple function which uses blocking io to perform the task. Also we added "io_bound" variable to emulate (very naively) io or cpu boud work. And finally our worker will look like this

{% highlight fsharp %}
//emulating input queue
let ImageIdsFormQueueSeq = Enumerable.Range(1, numImages)
let processImagesSync () =
        for i in ImageIdsFormQueueSeq do
            processImageSync(i)
{% endhighlight %}

But this sequential lets parallelize it.

{% highlight fsharp %}
let parallelSync () =
        Parallel.ForEach(ImageIdsFormQueueSeq, processImageSync) |> ignore
{% endhighlight %}

But this function uses sync io and it blocks threads from the thread pool. We will rewrite it with async io.
{% highlight fsharp %}
let processImageAsync i = async {
       use inStream = File.OpenRead(sprintf "Image%d.tmp" i)
       if io_bound then do! Async.Sleep(200)
       let! pixels = inStream.AsyncRead(numPixels)
       let  pixels' = if not io_bound || i % 2 = 0 
                        then transformImage(pixels, i) 
                        else pixels
       use outStream = File.OpenWrite(sprintf "Image%d.done" i)
       do! outStream.AsyncWrite(pixels')
}
//fsharp version
let processImagesAsync() =
    let tasks = [for i in ImageIdsFormQueueSeq -> processImageAsync(i)]
    Async.RunSynchronously (Async.Parallel tasks) |> ignore
//csharp like version
let processImagesAsyncTasks() =
    let tasks = ImageIdsFormQueueSeq
                    .Select(fun i -> Async.StartAsTask(processImageAsync(i)))
    Task.WhenAll(tasks).Wait()
{% endhighlight %}
Let’s add some time measurement and check which version is faster.
{% highlight fsharp %}
let time f name =
    let stopWatch = System.Diagnostics.Stopwatch.StartNew()
    printfn "%s started" name;
    f()
    stopWatch.Stop()
time processImagesAsync "processImagesAsync"
time processImagesAsyncTasks "processImagesAsyncTasks"
time parallelSync "parallelSync"
{% endhighlight %}
On my machine with numImages = 2000, parallel asynchronous and parallel sync versions for cpu bound tasks take the same amount of time, but for io bound tasks sync version is 2 times slower than async. Sequential sync version is 10 times slower than parallel sync version. But do you see a problem here with async version? It has unbounded concurrency level. In short it maps all ids into tasks and starts them in parallel. This kind of behavior will stress our data source until eventually it fails and requests starts to error. Let’s limit concurrency level with a semaphore. We can use TPL schedulers for that purpose also but semaphore is easier to understand.
{% highlight fsharp %}
let processImagesAsyncSlim limit =
    let throttler = new SemaphoreSlim(limit);
    let countdownEvent = new CountdownEvent(numImages)
    let awaitTask (t: Task) = t |> Async.AwaitIAsyncResult |> Async.Ignore
    let wrap wrkfl = async{
        try 
            do! wrkfl
        finally
            throttler.Release() |> ignore
            countdownEvent.Signal() |> ignore
    }
    let tasks = async{
            for i in 1 .. numImages do 
                do!awaitTask (throttler.WaitAsync())
                Async.Start(processImageAsync(i)|> wrap)
            countdownEvent.Wait()
            }
                            
    Async.RunSynchronously (tasks) |> ignore
{% endhighlight %}
Now we could respect concurrency level allowed by a data source and probably find a best level by starting it several times with different limits. Unfortunately we still have a problem. We have different types of concurrency in our pipeline which could request different levels of concurrency. For example in our case reading from disk could be done with 100 concurrent requests, image processing is cpu bound and usually it has level which equals to count of available processors. So we need to split our pipeline into smaller parts and set them different levels of concurrency. But we need to connect them somehow and synchronize their speeds.
#Consumer producer pattern
In short the pattern describes two processes, the producer and the consumer, who share a common, fixed-size buffer used as a queue.
Consumer could write into the queue when it is not full, in other case it will block. Producer could read from the queue when it is not empty and in other case it will block. The queue is buffered to increase overall speed and increase throughput in case when there are multiple consumers and producers. When queue is safe to use by multiple consumers and producers then it could be easy to change level of concurrency for different parts of pipeline in runtime. We will use BlockingQueueAgent described by Thomas Patricheck [here](http://tomasp.net/blog/parallel-extra-blockingagent.aspx/) next samples will be almost the same as in this [post](http://tomasp.net/blog/parallel-extra-image-pipeline.aspx/), but with some additions. So you could go and read these awesome posts and after that return back. But I'll describe them here in short just to keep all info in one place. 
First task is to split our pipeline to separate tasks.  
{% highlight fsharp %}
//ids source could be for eample azure storage queue
let ids = new BlockingQueueAgent<_>(numImages)
let idGen = async {
    for i in 1.. numImages do
        do! ids.AsyncAdd(i) 
}
let loadedImages = new BlockingQueueAgent<_>(Environment.ProcessorCount * 4)
let proccessed_images = new BlockingQueueAgent<_>(100)
let finished = new BlockingQueueAgent<_>(numImages)
{% endhighlight %}
Ok we have queues and it looks really similar to code which you could see in Golang and Clojure async. Async bounded channels as a connection between async sequential processes. And yes we use the patterns of interaction in concurrent systems that initially was described by C. A. R. Hoare and well known as [Communicating Sequential Processes (CSP)](http://en.wikipedia.org/wiki/Communicating_sequential_processes). But we need an additional item to be a fully CSP compatible it is a "choice" operator. In short it allows us to compose several get add operations on different channels (queues). For example we could try to read form two channels and continue execution with result which arrives first. If one queue wins (first unblocks process) then second should not be affected (no values should be got or added to the second queue). Our blocking queue is built on top of a mailbox processor so all operations are not atomic so we need to use "Compensating Transaction Pattern" to undo effect of a get operation. For simplicity we will support only get operations in our switch implementation.
{% highlight fsharp %}
// based on http://www.fssnip.net/6D
/// Takes several asynchronous workflows and returns 
/// the result of the first workflow that successfully completes
type Either<'a,'b> =
    | Left of 'a
    | Right of 'b  
let choice wrkfl1 wrkfl2 rollback1 rollback2= 
    Async.FromContinuations(fun (cont, _, _) ->
      let completed = ref false
      let lockObj = new obj()
      let synchronized f = lock lockObj f

      /// called when a result is available - the function uses locks
      /// to make sure that it calls the continuation only once
      let completeOnce res = 
        let run =
          synchronized(fun () ->
            if completed.Value then false
            else completed := true; true)
        if run then cont res //first wins continuation
        else match res with //second compensated
                |Left(res) -> 
                    Async.Start(rollback1 res)
                |Right(res) -> 
                    Async.Start(rollback2 res)

      /// Workflow that will be started for each argument - run the 
      /// operation, cancel pending workflows and then return result
      let runWorkflow workflow = async {
        let! res = workflow
        completeOnce res }

      // Start all workflows using cancellation token
      Async.Start(runWorkflow wrkfl1) 
      Async.Start(runWorkflow wrkfl2) )

let switch (q1:BlockingQueueAgent<_>) (q2:BlockingQueueAgent<_>) =
    let ret1 v = async{ 
        do! q1.AsyncAdd(v)
        }
    let get1 = async{ let! n = q1.AsyncGet()
                      return Left(n)}
    let ret2 v = async{ 
        do! q2.AsyncAdd(v)
        }
    let get2 = async{ let! n = q2.AsyncGet()
                      return Right(n)}
    choice get1 get2 ret1 ret2
{% endhighlight %}
Very naive but simple and works. Now we could express our wrapper for workers which adds possibility to be stopped.
{% highlight fsharp %}
let stoppableWorker (inp:BlockingQueueAgent<_>) 
                    (stop:BlockingQueueAgent<_>) 
                    f = 
async {
    let running = ref true
    while !running do 
        let! msg = Csp.switch inp stop
        match msg with
            | Csp.Left(value) ->
                do! f(value)
            | Csp.Right(_) -> 
            	running := false
}
{% endhighlight %}
It tries to get from both a stop channel and an input channel. If it receives stop signal then it stops, in other case it invokes worker function. And finally workers are built on top of stoppable worker.
{% highlight fsharp %}
et loadImages stopChannel= stoppableWorker ids stopChannel (fun i -> 
async {
    use inStream = File.OpenRead(sprintf "Image%d.tmp" i)
    if io_bound then do! Async.Sleep(200)
    let! pixels = inStream.AsyncRead(numPixels)
    do! loaded_images.AsyncAdd((i,pixels)) 
})
let proccess_images stopChannel = stoppableWorker loaded_images stopChannel 
    (fun (i, pixels) -> 
        async {
            let  pixels' = if not io_bound || i % 2 = 0 
                                then transformImage(pixels, i) 
                                else pixels
            do! proccessed_images.AsyncAdd((i, pixels')) })

let save_images stopChannel= stoppableWorker proccessed_images stopChannel 
(fun (i, pixels) -> 
    async{
        use outStream = File.OpenWrite(sprintf "Image%d.done" i)
        do! outStream.AsyncWrite(pixels) 
        do! finished.AsyncAdd(()) 
    })

let wait_finish ch = async{
    let proccessed = ref 0 
    while !proccessed < numImages do 
        let! _ = ch |> Channel.AsyncGet
        proccessed := !proccessed + 1
        printfn "proccessed %d" !proccessed
}
{% endhighlight %}
It is time for load balancer, it will start specified number of workers and will check lengths of input and output queues and increase or reduce number of workers at runtime.
{% highlight fsharp %}
let loadBalancer (cts:CancellationTokenSource) 
				 name 
				 worker 
				 (inch: BlockingQueueAgent<_>)  
				 (outch: BlockingQueueAgent<_>) 
				 min 
				 max = async {
    let stopper = new BlockingQueueAgent<_>(1)
    for _ in 1..min do
        Async.Start(worker stopper)
    let workers = ref min
    while true do
        if !workers > min && inch.Count = 0 then 
            do! stopper.AsyncAdd(())
            workers := !workers - 1
        elif !workers < max && (inch.Count = inch.MaxCount || outch.Count = 0) 
            then Async.Start(worker stopper)
                 workers := !workers + 1
        printfn "%s inq %d outq %d workers %d" 
                name 
                inch.Count 
                outch.Count 
                !workers
        do! Async.Sleep(100)
}
{% endhighlight %}
To start our pipeline we should wrap our stoppable workers into load balancer and start result workflows.
{% highlight fsharp %}
let cts = new CancellationTokenSource()
Async.Start(idGen,cts.Token)
Async.Start(loadBalancer cts 
                         "load_images" 
                         loadImages 
                         ids 
                         loaded_images 
                         1 
                         100, cts.Token)
Async.Start(loadBalancer cts 
                         "proccess_images" 
                         proccess_images 
                         loaded_images 
                         proccessed_images 
                         1 
                         System.Environment.ProcessorCount, cts.Token)
Async.Start(loadBalancer cts 
                         "save_images" 
                         save_images 
                         proccessed_images 
                         finished 
                         1 
                         100, cts.Token)
try Async.RunSynchronously(wait_finish finished, 
                            cancellationToken = cts.Token)
with :? System.OperationCanceledException -> ()
cts.Cancel()
{% endhighlight %}
If you run this code then you will see something like this.
	load_images inq 1 outq 0 workers 2proccess_images inq 0 outq 0 workers 2
	save_images inq 0 outq 0 workers 2

	load_images inq 1998 outq 0 workers 3
	save_images inq 0 outq 0 workers 1
	proccess_images inq 0 outq 0 workers 1
	proccess_images inq 0 outq 0 workers 2
	save_images inq 0 outq 0 workers 2
	load_images inq 1997 outq 0 workers 4
	load_images inq 1993 outq 0 workers 5
	save_images inq 0 outq 0 workers 1
	proccess_images inq 0 outq 0 workers 1
	load_images inq 1991 outq 0 workers 6
	proccess_images inq 0 outq 0 workers 2
	save_images inq 0 outq 0 workers 2
	load_images inq 1986 outq 0 workers 7
	proccess_images inq 0 outq 0 workers 1save_images inq 0 outq 0 workers 1

	load_images inq 1984 outq save_images inq 0 outq 0 workers 2
	0 workers 8
	proccess_images inq 0 outq 0 workers 2
	load_images inq 1977 outq 0 workers 9
	proccess_images inq 0 outq 0 workers 1
	save_images inq 0 outq 0 workers 1
	save_images inq 0 outq 0 workers 2
	proccess_images inq 4 outq 0 workers 2
	load_images inq 1971 outq 4 workers 9
It spend some time before it will find the best count of workers for every task. For better performance you could change starting number of workers from minimal to maximum count. Also we could easily change rules for load balancer and use performance metrics of used resource and so on. It is not production quality code but could be rewrote and after that you will have possibility to read and use [Golang patterns](http://blog.golang.org/advanced-go-concurrency-patterns) in your fsharp code.
Unfortunately I have no enough time for this blog so actors will arrive only in the next part. And as usual commants and corrections are welcome.
