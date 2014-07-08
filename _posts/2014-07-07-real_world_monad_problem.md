---
published: true
layout: post
title: Real world problem for monad(Draft).
tags : [lessons, csharp, fsharp, monad]
---

## {{page.title}} [RUS]({{ site.url }}/2014/07/07/real_world_monad_problem-rus.html)

<p class="meta">07 July 2014 &#8211; Karelia</p>

After publication of first two posts, some readers complains that all samples are too artifical and don't show any advantages of monads. This post is an attempt to resolve that issue. We will discuss a real world problem and will use manads to solve it. 

Sometimes in applications we need to use long running workflow processes. Workflow execution can take a lot of time and has asynchronous nature. For example after document creation we must send email to a manager with a link to page for document publication. This is a very simple example, in real world apps workflows could be very complicated. Typical case is an internet store and workflow of purchase of goods  We have some ready to use solutions like Microsoft Workflow Foundation. But for some applications it is an overkill. In most applications we don't want to allow users to create and edit workflows. Lets try to implement our lightweight solution and in the beginning we will write requirments list.

1. Workflow description should looks like a plain c# function.
2. Workflow is a composition of activities and oother workflows.
3. Activity describes what should be done in therms of domain model and should be represented as a POCO class.
4. Workflow execution and translation of activities to a real code should not be done by a workflow but by some executor entity which is orthogonal to workflows.
5. Workflow serialization should be easy and without black magic like serialization of expression trees and enumerators.
6. Ability to see current workflow state and all results of executed activities.  
7. Ability to extend workflow engine by other features like timeouts, last activity undo and so on.

Lets start and define command classes.

{% highlight csharp %}
public class Action
{
}
//show some text to user
public class Show : Action
{
	public string What {
		get;
		set;
	}
}
//show some text to aser and ask a value of specified type
public class Ask<T> : Action
{
	public string What {
		get;
		set;
	}
}
{% endhighlight %}
Action is a base class for all activities. Ask and Show just two examples of concrete activities. So far so good.
Next stop is a storage class.
{% highlight csharp %}
public class Storage
{
	static JsonSerializerSettings settings;

	static Storage ()
	{
		settings = new JsonSerializerSettings ();
	}

	public static void Save<T> (T wrkfl, String name)
	{
		var json = JsonConvert.SerializeObject (wrkfl, settings);
		File.WriteAllText (name, json);
	}

	public static T Load<T> (string name)
		where T:new()
	{
		if (File.Exists (name)) {
			var json = File.ReadAllText (name);
			return JsonConvert.DeserializeObject<T> (json, settings);
		}
		return new T ();
	}
}
{% endhighlight %}
Very simple helper class which serealize and deserialize objects to json on file system and vice versa. Now we should define workflow clas. We probably will use workflow in this way:

1. Define a workflow
1. Create the workflow, probably with some params.
2. Invoke Execute Method
3. Check result, it could be some activity for execution or final result. In case of final result we can stop but in other case we should do some asked activity.
4. Save workflow into storage.
5. On activity response load workflow from storage and set result.
6. Go to step 2. 
Lets descibe it in code and specify behaviour of Ask and Show commands.
{% highlight csharp %}
public class Unit//no result object
{
	Unit ()
	{

	}

	public static Unit Value = new Unit ();
}
public class WorkflowStep<T>
{
	public WorkflowStep (Action act)
	{
		...
	}

	public WorkflowStep (T value)
	{
		...
	}

	public Action Action {
		get;
		private set;
	}

	public bool IsExecuted ()
	{
		...
	}

	public T GetValue ()
	{
		...
	}
}
public abstract class Workflow<TB>
{
	protected WorkflowStep<T> Do<T> (Action act)
	{
		var step = new WorkflowStep<T> (act);
		return step;
	}

	protected WorkflowStep<T> Ask<T> (String what)
	{
		return Do<T> (new Ask<T> (){ What = what });
	}

	protected WorkflowStep<Unit> Show (String what)
	{
		return Do<Unit> (new Show (){ What = what });
	}

	public abstract WorkflowStep<TB> GetResult ();
}
public class SumWorkflow:Workflow<int>
{
	public SumWorkflow ()
	{
		
	}
	// not correct c# code
	public override WorkflowStep<int> GetResult ()
	{
		var a = Ask<int> ("enter a");
		var b = Ask<int> ("enter b");
		var res = a + b;
		Show ("Result= " + res);
		return res;
	}
}
class MainClass
{
	public static void Main (string[] args)
	{
		var fileName = "workflow.json";
		var wrkfl = Storage.Load<SumWorkflow> (fileName);
		Console.WriteLine ("Current workflow state");
		...
		var res = wrkfl.GetResult ();
		if (res.IsExecuted ()) {
			Console.WriteLine ("Finished");
			File.Delete (fileName);
		} else {
			if (res.Action is Show) {
				Console.WriteLine ((res.Action as Show).What);
				wrkfl.AddResult (Unit.Value);
			} else if (res.Action is Ask<int>) {
				Console.WriteLine ((res.Action as Ask<int>).What);
				var resp = Console.ReadLine ();
				var val = int.Parse (resp);
				wrkfl.AddResult (val);
			}
			Storage.Save (wrkfl, fileName);
		}
	}
}
{% endhighlight %}
GetResult method is the essence of our solution but it will not work in required way. lets exam that function and add descriptions of required behaviour. 
{% highlight csharp %}
//check if we alredy have result of execution, set it to "a" variable and continue function othervice break execution and return action. This line doesnt depends on any data.
var a = Ask<int> ("enter a");
//Behaviour is the same but it depends on previous result from "a" variable(we don't want to execute this lines in parallel).
var b = Ask<int> ("enter b");
//this line should be executed as a standard c# code. 
//We could add it to next "Show" line and use together. 
//Behaviour the same as for Ask action but depends on Tuple(a,b)
var res = a + b;
Show ("Result= " + res);
//simply return result and mark it as finished
return res;
{% endhighlight %}
Lets describe it in code.
{% highlight csharp %}
var a = Ask<int> ("enter a");
if (!a.IsExecuted ()) {
	return new WorkflowStep<WORKFLOWRESTYPE> (a.Action);
}
var aValue = a.GetValue ();
var b = Ask<int> ("enter b");
if (!b.IsExecuted ()) {
	return new WorkflowStep<WORKFLOWRESTYPE> (b.Action);
}
var bValue = b.GetValue ();
var res = aValue + bValue;
var c = Show ("Result= " + res);
if (!c.IsExecuted ()) {
	return new WorkflowStep<WORKFLOWRESTYPE> (c.Action);
}
return new WorkflowStep<WORKFLOWRESTYPE> (res){IsExecuted = true};;
{% endhighlight %}
We couuld wrap return boilerplate into a return fuction. But we can't wrap "if .. return .." constructions into some helper function. WContinuations to the rescue.  
{% highlight csharp %}
public class WorkflowStep<T>
{
	...
	public WorkflowStep<TB> Bind<TB> (Func<T,WorkflowStep<TB>> f)
	{
		if (this.IsExecuted ()) {
			return f (this.GetValue ());
		}
		return new WorkflowStep<TB> (this.Action, this.Context, this.Index);
	}

	public static WorkflowStep<TB> Return<TB> (TB x)
	{
		return new WorkflowStep<TB> (x);
	}
}

public class SumWorkflow:Workflow<int>
{
	public override WorkflowStep<int> GetResult ()
	{
		Ask<int> ("enter a").Bind(
			a=> Ask<int> ("enter b").Bind(
				b=>{
					var res = a + b;
					return Show ("Result= " + res).Bind(
						unit => unit.Return(res)
					);			
				}
			)
		);
	}
}
{% endhighlight %}
Now everything is ok but it doesnt look as a plain c# function. Could we do better? Defenitely yes our return and bind functions is a monad pattern and as usual we can use linq syntactic sugare.
{% highlight csharp %}
public static class WorkflowMonad
{
	public static WorkflowStep<T> Return<T> (this T value)
	{
		return new WorkflowStep<T> (value);
	}

	public static WorkflowStep<U> Bind<T, U> (this WorkflowStep<T> m, Func<T, WorkflowStep<U>> k)
	{
		if (m.IsExecuted ()) {
			return k (m.GetValue ());
		}
		return new WorkflowStep<U> (m.Action);
	}

	public static WorkflowStep<V> SelectMany<T, U, V> (
		this WorkflowStep<T> id,
		Func<T, WorkflowStep<U>> k,
		Func<T, U, V> s)
	{
		return id.Bind (x => k (x).Bind (y => s (x, y).Return ()));
	}

	public static WorkflowStep<B> Select<A, B> (this WorkflowStep<A> a, Func<A, B> select)
	{
		return a.Bind (aval => WorkflowMonad.Return (select (aval)));
	}
}
{% endhighlight %}
Now we could rewrite our workflow function.
{% highlight csharp %}
public override WorkflowStep<int> GetResult ()
{
	return 	from a in Ask<int> ("enter a")
	        from b in Ask<int> ("enter b")
	        let res = a + b
	        from u in Show ("Result= " + res)
	        select res;
}
{% endhighlight %}
Now we should think how can we implement functions of IsExecuted and GetResult of WokflowStep class. We should store results of already executed lines in workflow. Lets incrementally assign line numbers on each Action creation and use List<String> as a storage for serialized results.
{% highlight csharp %}
public class ExecutionContext
{
	public ExecutionContext ()
	{
		Memory = new List<string> ();
	}

	public List<string> Memory {
		get;
		set;
	}

	public int Index {
		get;
		private set;
	}

	public void Restart ()
	{
		Index = 0;
	}

	public void Inc ()
	{
		Index = Index + 1;
	}
}

public static class SerializationHelpers
{
	public static T ParseValue<T> (this string json)
	{
		return JsonConvert.DeserializeObject<T> (json);
	}

	public static string ValueToString<T> (this T value)
	{
		return JsonConvert.SerializeObject (value);
	}
}
{% endhighlight %}
Now we must add context into Workflow and WokflowStep classes
{% highlight csharp %}
public class WorkflowStep<T>
{
	public WorkflowStep (Action act, ExecutionContext context, int index)
	{
		Action = act;
		Context = context;
		Index = index;
	}

	public WorkflowStep (T value)
	{
		Context.Memory.Add (value.ValueToString ());
	}

	public ExecutionContext Context = new ExecutionContext ();

	public bool IsExecuted ()
	{
		return (this.Index) < Context.Memory.Count;
	}

	public T GetValue ()
	{
		return Context.Memory [this.Index].ParseValue<T> ();
	}

	public int Index {
		get;
		set;
	}
}
public abstract class Workflow<TB>
{
	public Workflow ()
	{
		Context = new ExecutionContext ();
	}

	public Workflow (ExecutionContext ctx)
	{
		Context = ctx;
		IsSubWorkflow = true;
	}

	public bool IsSubWorkflow {
		get;
		set;
	}

	public ExecutionContext Context {
		get;
		set;
	}

	protected WorkflowStep<T> Do<T> (Action act)
	{
		var step = new WorkflowStep<T> (act, Context, Context.Index);
		Context.Inc ();
		return step;
	}

	protected WorkflowStep<T> Ask<T> (String what)
	{
		return Do<T> (new Ask<T> (){ What = what });
	}

	protected WorkflowStep<Unit> Show (String what)
	{
		return Do<Unit> (new Show (){ What = what });
	}

	public abstract WorkflowStep<TB> GetResultInt ();

	public WorkflowStep<TB> GetResult ()
	{
		if (!IsSubWorkflow)
			Context.Restart ();
		return GetResultInt ();
	}

	public void AddResult (object val)
	{
		Context.Memory.Add (val.ValueToString ());
	}
}
{% endhighlight %}
We have done. Lets see some composable workflow in action.
{% highlight csharp %}
public class WorkflowComposition:Workflow<int>
{
	public override WorkflowStep<int> GetResultInt ()
	{
		return 	from a in new SumWorkflow (Context).GetResult ()
		        from b in new SumWorkflow (Context).GetResult ()
		        let res = a + b
		        from v in Show ("Result of two = " + (a + b))
		        select a + b;
	}
}
{% endhighlight %}
Hm seems that we meet all our requirments from the list.
Now we could use this engine in console apps, asp.net applications. Do some cool SharePoint stuff. And monad pattern helps us to solve that problem in a beautiful way. We also can add support for oops, conditions([how]({{ site.url }}/2014/07/02/way-to-computation-expressions.html)), parallel eecution, transactions and what ever you want. [Full code](https://gist.github.com/hodzanassredin/e6b5e70a46201251629e) for this post.
