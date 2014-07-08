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

Для начала определим классы активностей.

{% highlight csharp %}
public class Action
{
}
//показать текст пользователю
public class Show : Action
{
	public string What {
		get;
		set;
	}
}
//показать текст пользователю и спросить значение определенного типа
public class Ask<T> : Action
{
	public string What {
		get;
		set;
	}
}
{% endhighlight %}
Action это базовый класс для всех активностей. Ask и Show просто два примера конкретных активностей. Пока все просто.
Теперь определим стратегию сохранения процессов.
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
Очень простой вспомогательный класс который сериализует обьекты в json и сохраняет в текстовый файл и также делает обратную операцию. Теперь определим класс процесса и шага процесса. Вернее всего мы будем использовать процессы примерно так:

1. Определить процесс
1. Создать экземпляр процессаCreate the workflow, probably with some params.
2. Вызвать метод Execute
3. Проверить результат, он может содержать конечный результат или активность для исполнения. В случае с результатом мы можем закончить исполнение но в другом случае мы должны выполнить запрашиваемую активность.
4. Сохранить процесс в хранилищe.
5. При получении результата от активности загрузить процесс из хранилища и добавить результат.
6. Перейти к шагу 2. 
Давайте опишем это в коде и определим поведение команд Ask и Show.
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
	// не корректный код
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
Метод GetResult это суть нашего решения но в данный момент он не работает. Давайте посмотрим на этот метод внимательней и опишем поведение каждой строки. 
{% highlight csharp %}
var a = Ask<int> ("enter a");

//проверить если уже существует результат выполнение активности присвоить его переменной "а" и продолжить выполнение, если результата нет то прервать выполнение и вернуть необходимую активность. Эта линия не зависит от ранее вычисленных результатов.

var b = Ask<int> ("enter b");
//Поведение тоже но линия зависит от предыдущего результата "a" (мы не хотим исполныть эти линни параллельно).

var res = a + b;
Show ("Result= " + res);
//Первая линия должна быть выполнена как обычный код c#. 
//Мы можем добавить ее ко второй "Show" линии и рассматривать вместе. 
//Поведение тоже как у активности Ask но зависит от Tuple(a,b)

return res;
//просто вернуть результат маркировать его как вычисленный
{% endhighlight %}
Теперь мы можем выразить это в реальном коде.
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
Мы можем убрать код возвращения в специальную функцию return, но мы не можем убрать "if .. return .." в какую либо функцию. Континуации во спасение наших душ.  
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
Теперь все компилиться но это не выглядит как обычная функция c#. Можем ли мы сделать это лучше? Определенно да, наши функции return и bind это паттерн монады и мы может как и прежде добавить синтаксический сахар linq.
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
Теперь мы можем переписать нашу функцию определения процесса.
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
Мы все ще не знаем как написать функции IsExecuted и GetResult класса WokflowStep. Мы должны где то сохранять рузультаты уже выполненных активностей. Мы можем инкрементно присваивать номера линий на каждое создание активностей и использовать List<String> как хранилище сериализованных результатов. Мы будем использовать класс ExecutionСontext который будет использован как харнилище результатов и хранити и текущий номер исполняемой строки.
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
Теперь мы должны добавить контекст в классы Workflow и WokflowStep.
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
Сделано. Теперь мы можем варазить композицию процессов.
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
Чтож похоже мы смогли удовлетворить всем наши требования из списка.
Теперь мы можем исползовать этот движок дв консольных приложениях, asp.net сайтах или извращаться с SharePoint. И поаттерн монады помог нам решить эту проблему без особых костылей. Мы также можем добавить поддержку для циклов, условий([как]({{ site.url }}/2014/07/02/way-to-computation-expressions-rus.html)), паралллельного исполнения, транзакций и многого другого. [Полный код](https://gist.github.com/hodzanassredin/e6b5e70a46201251629e) для этого поста.
P.S.
Просто вообразите возможность запускать вычисления поверх баз данных или веб сервисов без идиотских IRepository, IService и т.п.
