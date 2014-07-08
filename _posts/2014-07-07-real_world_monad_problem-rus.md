---
published: true
layout: post
title: Решение реальной проблемы с помощью монад(Черновик).
tags : [lessons, csharp, fsharp, monad]
---

## {{page.title}} [ENG]({{ site.url }}/2014/07/07/real_world_monad_problem.html)

<p class="meta">07 Июля 2014 &#8211; Карелия</p>

*Русский вариант поста не исправляется, и возможно немного устарел, поэтому если вы видите явную ошибку, то гляньте [английскую версию]({{ site.url }}/2014/07/07/real_world_monad_problem.html) вернее всего там она исправлена. У меня просто нет времени на синхронизацию исправлений между двумя версиями.*

После публикации первых двух постов некоторые читатели мнение что все примеры с монадами немного надуманные и не показывают какого либо преимущества перед простым кодом с повторениями. Чтож попытаемся исправить это упущение. Давайте опишем проблему приближенную к реальности.

#Проблема.
Иногда в приложениях нам необходимо создавать рабочие процессы. Исполнение которых занимает продолжительное время. Например после создания документа отправить письмо менеджеру с сылкой на подтверждение публикации. Это самый простой пример, в реальном мире рабочие процессы могут быть черезвычайно сложны. В веб приложениях оипичный пример при оформление покупок в интернет магазине. На рынке существуют различные готовые решения этой проблемы например Microsoft Workflow Foundation. Но для некоторых задач это явный оверкил. В большинстве приложений не требуется позволять пользователям создавать и редактировать свои кастомные процессы. Давйте же попробуем написать свое легковесное решение. Для начала определим список требований.

1. Описание Процесса должно быть похоже на простую функцию.
2. Процессы это композиция других процессов и активностей.
3. Активность описывающия что должно быть сделано в терминах доменной модели должна быть простым POCO классом.
4. Конкретный код который будет исполнять активности и процессы должен быть определен вне процесса, в сущности под названием Исполнитель.
5. Сохранине и серилизация Процесса должно быть легко реализуемо.
6. Возможность просмотра текущего состояния процесса и всех его выполненных шагов.  
7. Легость добавления новых возможностей. Например отмена последней осуществленной активности, таймауты и т. д.

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
Now we could use this engine in console apps, asp.net applications. Do some cool SharePoint stuff. And monad pattern helps us to solve that problem in a beautiful way. We also can add support for oops, conditions([how]({{ site.url }}/2014/07/02/way-to-computation-expressions.html)), parallel eecution, transactions and what ever you want.
