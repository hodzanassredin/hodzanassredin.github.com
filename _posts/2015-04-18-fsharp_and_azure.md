---
published: false
layout: post
title: Fsharp workflows and azure.
tags : [azure, fsharp, monad]
---

## {{page.title}}

<p class="meta">18 April 2015 &#8211; Karelia</p>

#Intraduction
microsvices hype
I decided to write a history and current state of our distributed azure project. We started from a simple azure worker and finished vith a very complex system. I suppose it will be interesting for developers who is going to build some distributed application and hope this post will help them to avoid some mistakes. Also I'm going to describe two possible solutions in FSharp for our current problems.

#History of an Azure project

##Start
We started form a single azure worker, it had only two queues one for input and other for output and it was single threaded. Many azure tutorials shows the way how to build this kind of worker. We decided to use azure storage queue instead of Azure Service Bus queue. It is cheaper and has better SLA. For possible future changes we abstracted queue in simple C# interface:

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

As you can see main interface is push based but for example storage queue is pull based so we need additional one to let worker know what it should do some additional work. We have implementations for both types of azure queues and for RabbitMQ. Woker uses it this way:
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

I'm not going to show client code but it almost the same. This simple abstraction works wery well for us. All queues implementations on message fetching invokes subscribed action with fetched message and in case of no exception deletes massage from queue. if an exception was thrown then our queue wrapper do nothing and queue provider after some time will return this message to the queue so it can be handled second time by some worker. When something is going wrong with you worker you can easy see this by checking you queue in visual studio's server explorer. In case of problems there will be some messages with non zero fetching count. 

Also we hide queues behind a REST api facade implemented as azure web site. So our external clients know nothing about queues. First question was how to add possibility to use worker by multiple clients at the same time and give them identical bandwidth. Our abstraction gave us a way to solve it without changing worker code. For azure storage queue we implemented MultiQueue class.
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
Now woker use QueueStorageMulti instead of QueueStorage. Every time when it fetches message it uses next queue. There is a problem with refresh strategy, code above refreshes queues list only after when it finishes fetching from all existing queues. If we have a lot of clients then new one should wait a little bit more. But strategy could be easilly changed. Now problem which queue should be used by a worker to send result back to client. Probably we can use some client id in message and use some convention about queues naming. So if we have requests queue with worker1 prefix then request queue has name worker1_clientId and response queue has name responses_worker_clientid. But we decided to use some base message class with response address property. Back address also has a type in our case types are: redis,storage queue, email. So answer could be send not only to some queue but to also to other supported back address types. Also we should know about errors so there is Error and StackTrace fields. 
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
New problem different types of queues has differet limits to maximum message size. We solved that problem by impllementing Ref message type. if queue wrapper see that message is too bit then it converts it to a ref message. Ref message stores message data in aazure blob storage and during fething if it recieves ref message then it loads content from blob storage and deserializes it.
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
Uff a lot of work done and our simple worker can work without any issues. But after some time you understand that your worker is not enought and you need to scale it. Scaling should be no problem because this is a Cloud. Right? No. It is extremely hard in azure Yes there is some possibilities to do autoscaling which depends on some metric But we need to scale depending on queue length and our queue could be changed to other type unsupported by autoscaling. For example there is no autoscaling support for azure storag queue. After some googling I found autoscaling application block. It was outdated and after several days without any success i decided to write my own. Main idea is to use azure management api. You think it is simpe as installing nuget package and writing several lines of code. NOOOOO. It is extremely undoccumented and not easy to understand proccess. In two words you need to load security certificate by thumbprint and after that do some black magick with downloading some xml and changing instance count in it and uploading it back. I'll not write a code here becouse it is a theme for a different blog post. 


    //role which uses several subworkers entry point
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

    for (int i = 0; i < Environment.ProcessorCount; i++)
                {
                    workers.Add(new CrawlerWorker(stop_words, connectionString, RoleEnvironment.CurrentRoleInstance.Id + " " + i));
                }

                public class ResponseMessage<T> : BaseMessage, ISetter<T>
    {
        public T Response { get; set; }
        
        public void Set(T value)
        {
            Response = value;
        }

        public override string GetKey()
        {
            return "";
        }
    }

    public abstract class PipelineMessageBase : BaseMessage
    {
        public BaseMessage ResponseMessage { get; set; }
        public BaseMessage GetNextErrorMessage(string error, string where)
        {

            if (ResponseMessage != null)
            {
                ResponseMessage.Error = error;
                ResponseMessage.StackTrace = where;
                ResponseMessage.ReauestId = this.ReauestId;
            }
            return ResponseMessage;
        }

        public string GetKeyRecursive()
        {
            if (this.ResponseMessage == null) this.GetKey();
            var subKey = this.ResponseMessage is PipelineMessageBase ?
                (this.ResponseMessage as PipelineMessageBase).GetKeyRecursive() :
                this.ResponseMessage.GetKey();
            return String.Format("{0}_{1}", this.GetKey(), subKey);
        }
    }

    public abstract class PipelineMessage<TO> : PipelineMessageBase
    {

        public BaseMessage GetNextMessage(TO output) {

            if (ResponseMessage != null && ResponseMessage is ISetter<TO>)
            {
                (ResponseMessage as ISetter<TO>).Set(output);
                ResponseMessage.ReauestId = this.ReauestId;
            }
            return ResponseMessage;
        }
    }

 
{% endhighlight %}

#Current problems

#Ideal solution in theory

#Fsharp solution
 
