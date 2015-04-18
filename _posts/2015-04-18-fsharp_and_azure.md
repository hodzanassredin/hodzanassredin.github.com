---
published: false
layout: post
title: Fsharp workflows and azure.
tags : [azure, fsharp, monad]
---

## {{page.title}}

<p class="meta">18 April 2015 &#8211; Karelia</p>

#Intraduction

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
I\m not going to show client code but it almost the same. This simple abstraction works wery well for us.
Also we hide queues behind a REST api facade implemented as azure web site. So our external clients knows nothing about queues. First question was how to add possibility to use worker by multiple clients at the same time and give them identical bandwidth. Our abstraction gave us a way to solve it without changing worker code. For azure storage queue we implemented MultiQueue class.
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
Now woker use QueueStorageMulti instead of QueueStorage. Every time when it fetches message it uses next queue. There is a problem with refresh strategy, code above refreshes queues list only after when it finishes fetching from all existing queues. If we have a lot of clients then new one should wait a little bit more. But strategy could be easilly changed.
big messages, back address


{% highlight csharp %}
public abstract class BaseMessage {
        public string ReauestId { get; set; }
        public abstract string GetKey();
        public List<Address> ResponseRecievers { get; set; }

        public String Error { get; set; }
        public String StackTrace { get; set; }
    }

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

#Current problems

#Ideal solution in theory

#Fsharp solution
 
