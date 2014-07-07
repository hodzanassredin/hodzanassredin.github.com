---
published: true
layout: post
title: Путь к computation expressions.
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
public class SumWorkflow:Workflow<int>
{
	public SumWorkflow ()
	{
		
	}

	public SumWorkflow (ExecutionContext context) : base (context)
	{
	}
	// pseudo c# code
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

{% highlight csharp %}
{% endhighlight %}

{% highlight csharp %}
{% endhighlight %}

{% highlight csharp %}
{% endhighlight %}

{% highlight csharp %}
{% endhighlight %}

{% highlight csharp %}
{% endhighlight %}
