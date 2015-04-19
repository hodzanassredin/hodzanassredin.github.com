---
published: false
layout: post
title: FSharp workflows and azure.
tags : [azure, fsharp, monad]
---

## {{page.title}}

<p class="meta">18 April 2015 &#8211; Karelia</p>

#Introduction
I decided to write a history and current state of our distributed azure project. We started from a simple azure worker and finished with a very complex system. I suppose it will be interesting for developers who is going to build some distributed application and hope this post will help them to avoid some mistakes. In addition, I am going to describe two possible solutions in FSharp for our current problems.

#History of an Azure project

##Start
We started form a single azure worker, it had only two queues one for input and other for output and it was single threaded. Many azure tutorials shows the way to build this kind of worker. We decided to use azure storage queue instead of Azure Service Bus queue. It is cheaper and has better SLA. For possible future changes we abstracted queue in simple C# interface:

{% highlight csharp %}
//queue interface
public interface IQueueWrapper<T> where T : BaseMessage
{
    void Close();
    void OnMessage(Func<T, Task> act);
    Task Publish(T message);
    Task Init(string connectionString, string queueName, TimeSpan messagesTimeToLive, CancellationToken token);
    Task<int?> Length();
}
//pull queue, we need to invoke that method periodically
public interface ISyncQueue
{
    Task<bool> Fetch(int count);
}
{% endhighlight %}

As you can see, main interface is push based but for example, storage queue is pull based so we need additional one to let worker know what it should do some additional work. We have implementations for both types of azure queues and for RabbitMQ. Woker uses it this way:
{% highlight csharp %}
//queue interface
Trace.TraceInformation(workerName + " Run.");
Requests.OnMessage(
async req =>
{
	//message handler
});
while (!CancellationToken.IsCancellationRequested)
{
    Trace.TraceInformation(workerName + " Heartbeat.");
    if (Requests is ISyncQueue)
    {
        var r = Requests as ISyncQueue;
        var recievedMessage = await r.Fetch(2);
        if (!recievedMessage)
        {
            await Task.Delay(1000, CancellationToken);
        }
    }
    else await Task.Delay(10000, CancellationToken);
}
{% endhighlight %}

I am not going to show client code but it almost the same. This simple abstraction works very well for us. All queues implementations on message fetching invokes subscribed action with fetched message and in case of no exception, deletes massage from queue. If an exception thrown then our queue wrapper do nothing and queue provider after some time will return this message to the queue and it can be handled second time by some worker. When something is going wrong with your worker, you can easy see this by checking your requests queue in visual studio's server explorer. In case of problems there will be some messages with non-zero fetching count. 

In addition, we hide queues behind a REST api facade implemented as azure web site. Therefore, our external clients know nothing about queues. First question was how to add possibility to use worker by multiple clients at the same time and give them identical bandwidth. Our abstraction gave us a way to solve it without changing worker code. For azure storage queue, we implemented QueueStorageMulti class.
{% highlight csharp %}
public class QueueStorageMulti<T> : IQueueWrapper<T>, ISyncQueue where T : BaseMessage
{
    ...
    string prefix;
    ...

    public void Close()
    {
        foreach (var q in _queues.Values)
        {
	        q.Close();
        }
    }

    public void OnMessage(Func<T, Task> act)
    {
        _act = act;
        foreach (var q in _queues.Values)
        {
            q.OnMessage(_act);
        }
    }

    public async Task Publish(T message)
    {
        foreach (var item in _queues)
        {
	        await item.Publish(message);
        }
    }

    public async Task Init(string connectionString, string queuePrefix, TimeSpan publishTimeToLive, System.Threading.CancellationToken token)
    {
        ...
        await Refresh();
    }

    public async Task<int?> Length()
    {
        var res = 0;
        foreach (var q in _queues)
        {
            var qlen = await q.Length();
	        res = res + (qlen.HasValue ? qlen.Value : 0);
        }
        return res;
    }
    IEnumerator<QueueStorage<T>> enumerator;
    public async Task<bool> Fetch(int count)
    {
        while(enumerator.MoveNext())
        {
            var q = enumerator.Current;
            if (await q.Fetch(count)) return true;
        }
        await Refresh();
        return false;
    }

    public async Task Refresh()
    {
        //refresh queus by prefix
    }
}
{% endhighlight %}
Now worker uses QueueStorageMulti instead of QueueStorage. Every time when it fetches message it uses next queue. There is a problem with refresh strategy, code above refreshes queues list only after when it finishes fetching from all existing queues. If we have many clients then new one should wait a little bit more. Now problem, which queue should be used by a worker to send result back to client? We can use some client id in message and use some convention about queues naming. If we have requests queue with worker1 prefix then request queue has name worker1_clientId and response queue has name responses_worker_clientid. However, we decided to use some base message class with response address property. Back address also has a type, in our case types are: redis, storage queue, email. Answer could be send not only to some queue, but also to other supported back address types. We should know about errors so there is Error and StackTrace fields. 
big messages, back address

{% highlight csharp %}
publi enum AddressType{
    StorageQueue,
    Redis,
    Smtp
}

public class Address{
    public string Value { get; set; }
    public AddressType Type { get; set; }
}

public abstract class BaseMessage {
    public string ReauestId { get; set; }
    public abstract string GetKey();
    public List<Address> ResponseRecievers { get; set; }

    public String Error { get; set; }
    public String StackTrace { get; set; }
}
{% endhighlight %}
New problem: different types of queues have different maximum limit of message size. We solved that problem by implementing Ref message type. If queue wrapper see that message is too big then it converts it to a ref message. Ref message stores message data in azure blob storage.
{% highlight csharp %}
 public class BlobRefMessage
{
     public BlobRefMessage()
     {
         BlobReference = Guid.Empty;
         Value = null;
     }
    public BlobRefMessage(Object obj)
    {
        BlobReference = Guid.Empty;
        Value = obj;
    }

    public async Task ToRef(BlobStorage storage)
    {
        if (BlobReference != Guid.Empty) return;
        BlobReference = Guid.NewGuid();
        string str = JsonConvert.SerializeObject(Value, Settings.json);
        await storage.SaveBlob(str, BlobReference.ToString("N"));
        Value = null;
    }

    public async Task FromRef(BlobStorage storage)
    {
        if (BlobReference == Guid.Empty) return;
        var str = await storage.LoadText(BlobReference.ToString("N"));
        Value = JsonConvert.DeserializeObject(str, Settings.json);
    }

    public async Task DeleteRef(BlobStorage storage)
    {
        if (BlobReference == Guid.Empty) return;
        await storage.DeleteBlob(BlobReference.ToString("N"));
    }

    public Guid BlobReference { get; set; }
    public Object Value { get; set; }
}
{% endhighlight %}
 A lot of work done and our simple worker can work without any issues. However, after some time you understand that your worker is not enough and you need to scale it. Scaling should be no problem because this is a Cloud. Right? No. It is extremely hard in azure. Yes there is some possibilities to do auto scaling which depends on some metric But we need to scale depending on queue length and our queue could be changed to other type unsupported by auto scaling. For example, there is no auto scaling support for azure storage queue. After some googling I found auto scaling application block. It was outdated and after several days without any success, I decided to write my own. Main idea is to use azure management api. You probably think it is simple as installing nuget package and writing several lines of code. NOOOOO. It is extremely undocumented and not easy to understand process. In two words you need to load security certificate by thumbprint and after that do some black magic with downloading xml and changing instance count in it and uploading it back. I will not write a code here because it is a theme for a different blog post. Now is time to load some file to all of our worker instances. We do it this way. Upload you file to a blob storage and during worker loading, download it to a local worker’s storage. You need to enable it in your worker’s configuration file and specify local storage size. You should not make a mistake. Because after deploy, future changes in local storage settings can broke you deployment and you will be forced to delete cloud service from azure and redeploy it. In addition, if you specify local storage size, which is less than size of your file, then you will get a cryptic soap error message during deployment.
Next step is to utilize all worker cores. We created some abstractions for worker and tasks.
Worker, on start, loading all files and other shared resources and after that checks count of logical processors and creates this number of task executors. If task executor throws an exception then worker should restart it again.

{% highlight csharp %}
public abstract class TasksRoleEntryPoint : RoleEntryPoint
{
    private readonly List<Task> _tasks = new List<Task>();

    private readonly CancellationTokenSource _tokenSource;

    private WorkerEntryPoint[] _workers;

    public TasksRoleEntryPoint()
    {
        _tokenSource = new CancellationTokenSource();
    }

    public void Run(WorkerEntryPoint[] arrayWorkers)
    {
        try
        {
            _workers = arrayWorkers;

            foreach (WorkerEntryPoint worker in _workers)
            {
                worker.OnStart(_tokenSource.Token).Wait();
            }

            foreach (WorkerEntryPoint worker in _workers)
            {
                var task = worker.ProtectedRun();
                _tasks.Add(task);
            }

            int completedTaskIndex;
            while ((completedTaskIndex = Task.WaitAny(_tasks.ToArray())) != -1 && _tasks.Count > 0)
            {
                _tasks.RemoveAt(completedTaskIndex);
                //Not cancelled so rerun the worker
                if (!_tokenSource.Token.IsCancellationRequested)
                {
                    _tasks.Insert(completedTaskIndex, _workers[completedTaskIndex].ProtectedRun());
                    Task.Delay(1000).Wait();
                }
            }
        }
        catch (Exception e)
        {
            Trace.TraceError(e.Message);
        }
        Trace.TraceError("Run workers exit");
    }

    public override bool OnStart()
    {
        return base.OnStart();
    }

    public override void OnStop()
    {
        Trace.TraceError("OnStop called");
        try
        {
            _tokenSource.Cancel();
            Task.WaitAll(_tasks.ToArray());
        }
        catch (Exception e)
        {
            Trace.TraceError(e.Message);
        }
        base.OnStop();
    }
}
public SomeWorkerClass : TasksRoleEntryPoint{
    ...
    for (int i = 0; i < Environment.ProcessorCount; i++)
    {
        workers.Add(new CrawlerWorker(stop_words, connectionString, RoleEnvironment.CurrentRoleInstance.Id + " " + i));
    }
    ...
}
{% endhighlight %}
So far so good. We have all what we need to use single type worker in azure. It can handle different types of messages and all seems to be fine. Unfortunately, we have new requirement to use the same type of worker but with a different data. In our case, it was different types of text classifiers. Both types of workers have the same pipeline. Crawl -> Extract features -> Feature hashing -> Vectorization -> Classification. Both classifiers takes a lot of memory and loads different files for classifiers and vectorizers. You may think that perfect way to do that is to have two different requests prefixes and one small auto-scaler worker instance, which will create classifier instance with a different configuration based on a queue prefix. Unfortunately, it takes too long to create an instance in azure and takes too long to load classifier into instance’s memory. In our case about half an hour. Nevertheless, clients of our web site uses synchronous interface and cannot wait too long. We need to keep one instance in running state for every type of classifier.  It is not a problem from developer point of view. 
Next feature request is to add some possibility to only one type of classifier. Now our worker differs not only by config but also by incoming message types and code. Now we need to separate both types of workers in code.  
After some profiling, we see that we have bottleneck in our performance and it is a crawling stage. Yes, it has async nature but we do not want to use an event loop in our processing thread in a worker and we have new requirement to implement some new pipeline, which uses crawling but not related to our classifiers. So we need to move crawling and tokenization into a new small worker instance.  It is cheap and now we auto scale only this worker. Classification workers has only one instance running they are too pricy. As you probably know your azure subscription has a limit in total count of cores which can be used. Our solution allows us to keep this count small. Now we have a problem in code: how can we express some pipeline? It is time for a pipeline message class. It is a simple message class with a message, which have to be sent to a response receiver. So worker, when it receives a message, should do some work and create some output value and invoke GetNextMessage(calculated output value). This method returns response message, which could has some fields populated initially by a client and some fields populated form the calculated output value. This response message should be sent to a response receiver. Also in case of any error, we need to break execution, find a final message, set error data and send to a final receiver. Why to send error message back to receiver. Because it could be not asynchronous and can be in waiting state and it needs to know when to show error or try again. Probably better way is to add ErrorMessageRecievers.

{% highlight csharp %}
public abstract class PipelineMessageBase : BaseMessage
{
    //normal way next step message
    public BaseMessage ResponseMessage { get; set; }
    public BaseMessage GetErrorMessage(string error, string where)
    {
        //find recursively final response message and set error data and send back to final response reciever 
        if (ErrorResponseMessage != null)
        {
            ErrorResponseMessage.Error = error;
            ErrorResponseMessage.StackTrace = where;
            ErrorResponseMessage.ReauestId = this.ReauestId;
        }
        return ErrorResponseMessage;
    }
}

public abstract class PipelineMessage<TO> : PipelineMessageBase
{
    public BaseMessage GetNextMessage(TO output) {
        // if next error waits response from a previous step than set it
        if (ResponseMessage != null && ResponseMessage is ISetter<TO>)
        {
            (ResponseMessage as ISetter<TO>).Set(output);
            ResponseMessage.ReauestId = this.ReauestId;
        }
        return ResponseMessage;
    }
}
//final response message. Pipeline is finished.
public class ResponseMessage<T> : BaseMessage, ISetter<T>
{
    public T Response { get; set; }
    
    public void Set(T value)
    {
        Response = value;
    }
}
Now we have a system, which allows us to split our work to distributed small pieces and express pipelines with message builders. For example, classification message builder will look like this. 
{% highlight csharp %}
public BaseMessage BuildClassificatioonMessage(Address responseReciever, string url, bool useStopWords, int ngramsLimit,...){
    return new CrawlerMessage(){
        UseStopWords = useStopWords,
        Url = url,
        ...
        ResponseRecievers = new List<Address>(){
            GetResponseRecieverAddress(WorkerType.Classifier1)}
        ,
        ResponseMessage = new ClassifyMessage(){
            NGramsLimit = nGramsLimit,
            ...
            ResponseRecievers = new List<Address>(){responseReciever},
            ResponseMessage = new ResponseMessage<ClassifierResultDto>()
        }
    };
}

{% endhighlight %}
#Big problems
We build some code, which allows us to express distributed computations for azure platform. Very interesting, but now we are using Microservices pattern. You can read more here http://microservices.io/patterns/microservices.html. We can decompose our algorithms into small pieces and they could be deployed separately without breaking currently executed pipelines. Our system described above have a lot of problems which are really difficult to solve.  With all that hype about Microserices I read many articles and unfortunately found no answers to my questions.
Let us start from our solution description. Our solution has possibility to express only SEQUENTIAL pipelines with a primitive error handling. That’s all. No possibility to express cancellation,  loops, if conditions and other control flow operators, fork/join parallelization. Even with only sequential pipelines, it is damn hard to use. 
1. It is hard to test. You can easily test pieces, but really difficult to abstract full execution context.
2. It is hard to add a new pipeline. There is many possible ways to shoot yourself in the foot. First you need to decompose pipeline into small pieces. Decide for every piece which worker should execute it. Create message classes for every piece of work. Decide what it is an incoming param from a client and what should be calculated, for every message class.
3. Caching. Just no words. Main strategy is to cache only in our api facade in other words on client side.
4. Debugging.
5. Message classes reuse for different tasks involves transport of unused data.
6. a lot of POCO classes for messages
7. Pipeline represented in message builder is hard to read and understand.
All problems solving increases time from requirement to release. It is just takes too long to decompose, workaround problems with control flow and test.

#Ideal solution in theory
So currently in our project, we have a lot of boilerplate code with workarounds, which are hard to support. Some time ago, I started to think how can we solve that problems. Main idea was is to use some actor framework like akka.net or Orleans. But after some attempts to emulate system on a local computer I found that they solves only part of problems. There is again small pieces of more complex pipelines. There is no abstract way to compose complex pipelines in code. Quote from @runarorama's twitter: Actors are an implementation detail for higher-order constructs, not a programming model. Also there are some problems from a types point of view. http://stew.vireo.org/posts/I-hate-akka/ Depression, depression, depression.
However, several days ago I received a question in twitter from @zahardzhan: "Do you use monads described in your blog in practice?" My answer was "NO". I decided to refresh my memories and read my posts just for fun. And after reading of a post about Workflow monad http://hodzanassredin.github.io/2014/07/07/real_world_monad_problem.html I decided to use a way described in this post to express pipelines as workflows, but after some thoughts and experiments it was transformed into something new.
So let’s, as we do in the post about workflow monad, start from an ideal solution which allows us to do:
1. Express pipeline as a simple code.
2. Separate What from How
3. Send in messages only information that we need for next steps.
4. Avoid a lot of POCOs (and replace them by POFFos(plai old fsharp functions :-)
5. Could be easily tested.
6. Keep workers as simple as possible.
7. Dynamically choose next worker based on available resources for execution of next step.
8. If resources of current system satisfies all current pipeline requirements do it right here in one place. Maybe in web site action without any queues and workers?

As you can see I did not list here other problems and are going to start from a something simple and after that add new possibilities one by one. 

#FSharp solution
I'm not going to write about whole way of thinking just describe my solution in short and show you some code.
I decided to write a computation expression builder, which could do everything described above.
1. We can write simple fsharp code inside a builder
2. A function expresses what and builder expresses how to build pipeline. In addition, run function expresses where and how.
3. Use fsharp possibility to serialize a closure.
4. The same as 3
5. We could implement different run function for testing
6. Our workers should only receive fsharp function as serialized message, deserialize it and invoke run function with func from the message as param.
7. Our builder should build a function, which accepts environment as an argument, and break execution if current environment did not satisfies requested resources and return to a run function information about requested resources and function which will continue work from resource request point, all previously executes code will be skipped and all calculated data will be stored as closure fields. Run function should check builder execution result and send not finished closure to a worker which satisfies requested resources.
8. Builder should return calculated value if no unsatisfied resources requested

As a starting point, I took Reader monad and changed a signature.
{% highlight fsharp %}
type Result<'r, 'a> = 
        | Ok of 'a
        | ResourceRequest of string * Reader<'r,'a>
 
and Reader<'r,'a> = Reader of ('r -> Async<Result<'r,'a>>)
{% endhighlight %}
As you can see I also used Async as return type because currently all our code is async.
It is easier to merge both monads from the scratch.
{% highlight fsharp %}
let runReader (Reader r) env = r env
//return
let ret v = Reader (fun _ -> async { return Ok(v)})
//return async
let ret_async v = Reader (fun _ -> async {
                                        let! vno_async= v 
                                        return Ok(vno_async)})
type ReaderBuilder() =
  member this.Return(a)  = ret a

  member this.ReturnFrom(a)  = a

  member this.Bind(m, k) = 
    Reader (fun r ->
        async{
        let! prev = runReader m r
        match prev with
            | Ok(prev) -> return! runReader (k prev) r
            | ResourceRequest(str, m) -> return ResourceRequest(str, this.Bind(m,k))})

let reader = ReaderBuilder()
//ask environment
let ask = Reader (fun env -> async { return Ok(env)})
// building block for environment resource getters
let rec asks name f =
    Reader(fun env ->
        async{
                printfn "requesting resource %A" name 
                match f env with
                    | Some(x) -> printfn "found resource %A" name
                                 return Ok(x)
                    | None -> printfn "not found resource %A" name
                              return ResourceRequest(name, asks name f)})
{% endhighlight %}
That’s all, now we can do all what we want to do.
{% highlight fsharp %}
let ser = new BinaryFormatter();

let serialize obj =
    use ms = new MemoryStream()
    ser.Serialize(ms, obj)
    ms.ToArray()

let deserialize<'a> (arr:byte[]) =
    use ms = new MemoryStream(arr)
    ser.Deserialize(ms)  :?> 'a

type Env = {stop_list : Set<string> option; name : string; big_res : string option}
let get_stop_words = asks "stop_list" (fun env -> env.stop_list)
let get_big_res = asks "big_res" (fun env -> env.big_res)

let download url = 
    reader{
        let! html = ret_async <| Http.AsyncRequestString(url)
        return html
    }

let tokenize (text:string) use_stop_words =
    reader{
        printfn "tokenizings" 
        let! big_res = get_big_res
        let! stop_words = if use_stop_words then get_stop_words else ret Set.empty   
        return text.Split([|' ';'<';'>'|]) |> Array.filter(fun x -> Set.contains x stop_words |> not)
}
     
let op url use_stop_words = reader{
    let! text = download url
    let! tokens = tokenize text use_stop_words
    return tokens |> Seq.take 20|> Seq.reduce (fun x y -> x + " | " + y)
}

let rec execute r env1 env2 =
    async{
    printfn "executing in %A" (env1.name)
    let! run_res = runReader r env1 
    match run_res with
        | Ok(res) -> printfn "executed %A" res
        | ResourceRequest(res_descr, res) -> 
                                printfn "env swap"
                                let arr = serialize res
                                let restored = deserialize<Reader<Env, string>> arr
                                return! execute restored env2 env1}
{% endhighlight %}
ready to go
{% highlight fsharp %}
let stop_env = {stop_list = None; name = "without stop_lst"; big_res = Some("big res")}
let other_env = {stop_list = Some(["a"; "no";"html"] |> Set.ofList); name = "with stop_lst"; big_res = None}
let r = op "http://ya.ru" true 
execute r stop_env other_env
{% endhighlight %}
Console output

executing in "without stop_lst"
tokenizing
requesting resource "big_res"
found resource "big_res"
requesting resource "stop_list"
not found resource "stop_list"
env swap
executing in "with stop_lst"
requesting resource "stop_list"
found resource "stop_list"
executed ...

I would really appreciate if you share your problems and solutions in comments.  
