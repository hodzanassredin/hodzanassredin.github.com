---
published: true
layout: post
title: Monads and monad transformers for mere mortals in pure C#.
tags : [lessons, csharp, monad]
---

## {{page.title}} [RUS]({{ site.url }}/2014/06/21/yet-another-monad-guide-rus.html)

<p class="meta">25 June 2014 &#8211; Karelia</p>

During migration of a small project to fsharp, I thought a lot about using monads in practice. Subject fascinated me enough and I decided to describe my experience in a couple of posts. Unfortunately all the material demanded decent knowledge of monads and monad transformers, at the same time I wanted to make a material without any links to external materials, without a lot of words and simple for ordinary programmers who are not burdened by the knowledge of mathematics and functional programming language syntax. Thus was born the idea to write another explanation of what is a monad. Since this post is aimed at programmers, and it will be approached by code and not by the text or mathematics. Ready? Lets go.

Lets look at some simple code:

{% highlight csharp %}
public static bool Bool ()
{
	return new Random ().Next () % 2 == 0 ? true : false;
}

public static T GetData<T> () 
	where T : class, new()
{
	return Bool () ? null : new T ();
}

public static void Compare (string[] args)
{
	var a = GetData<Object> ();
	var b = GetData<Object> ();
	if (a == b)
		Console.WriteLine ("a == b");
	else
		Console.WriteLine ("a != b");
	Console.WriteLine ("finished!");
}

{% endhighlight %}

GetData function imitates a query to a data source and randomly returns instance of required data type or null. Compare function uses GetData for retrieving two values, compares them and writes output to console. What we as a good programmers should fix at first in this sample? Right, we should add null checks and use style of defensive programming.

{% highlight csharp %}
public static void DefesiveCompare (string[] args)
{
	var a = GetData<Object> ();
	if (a == null) {
		Console.WriteLine ("Can't compare");
		return;
	}
	var b = GetData<Object> ();
	if (b == null) {
		Console.WriteLine ("Can't compare");
		return;
	}
	if (a == b)
		Console.WriteLine ("a == b");
	else
		Console.WriteLine ("a != b");
	Console.WriteLine ("finished!");
}

{% endhighlight %}

Ok code now looks more safe but we have repeated code lets remove it to support DRY principle.

{% highlight csharp %}
public static bool Defend (object o)
{
	if (o == null) {
		Console.WriteLine ("Can't compare");
		return false;
	}
	return true;
}

public static void DefesiveCompareDry (string[] args)
{
	var a = GetData<Object> ();
	if (Defend (a))
		return;
	var b = GetData<Object> ();
	if (Defend (b))
		return;
	if (a == b)
		Console.WriteLine ("a == b");
	else
		Console.WriteLine ("a != b");
	Console.WriteLine ("finished!");
}
{% endhighlight %}

Looks better, but anyway we must add if check for every GetData invocation. Lets try to move if check into Defend function.

{% highlight csharp %}
public static string Defend (object a, Func<object, string> f)
{
	return a == null ? "Can't compare" : f (a);
}

public static void DefesiveCompareDry2 (string[] args)
{
	var res = Defend (GetData<Object> (), 
		          (a) => Defend (GetData<Object> (), 
			          (b) => a == b ? "a == b" : "a != b"));

	Console.WriteLine (res);
	Console.WriteLine ("finished!");
}
{% endhighlight %}

Just perfect but we have a problem in case of using some other type for example class Test. During execution it will be downcasted to Syste.Object and we will not be able to use its members.

{% highlight csharp %}
class Test
{
	public Test ()
	{

	}
	public string Text(){
		return "test";
	}

}
public static void Test(){
	Defend2(new Test(), a=>a.Text());
	// System.Object does not contain a defenition for Text
}
{% endhighlight %}
We can avoid this problem with generic parameters. 
{% highlight csharp %}
public static T Defend2Generic<T> (T a, Func<T, T> f)
	where T: class//applied only for TA which can be null
{
	return a == null ? "Can't compare" : f (a);
}
{% endhighlight %}
New problem.  We are trying to return type string instead of Test . And what to do? Lets create some type which could store some value or error string.
{% highlight csharp %}
public class Check<T> where T : class
{
	public Check (String errorMessage)
	{
		IsFailed = true;
		FailMesssage = errorMessage;
	}

	public Check (T val)
	{
		Value = val;
	}

	public bool IsFailed {
		get;
		private set;
	}

	public string FailMesssage {
		get;
		private set;
	}

	public T Value {
		get;
		private set;
	}

	public override string ToString ()
	{
		return string.Format (
			"[Check: IsFailed={0}, FailMesssage={1}, Value={2}]",
			 IsFailed, 
			 FailMesssage, 
			 Value);
	}
}

public static Check<TB> Defend<TA,TB> (TA a, Func<TA, TB> f)
			where TA : class
			where TB : class//applied only for TB and TA which can be null
{
	return a == null 
		? new Check<TB> ("Can't compare") 
		: new Check<TB> (f (a));
}
{% endhighlight %}
Looks beatiful. In action:
{% highlight csharp %}
public static void DefesiveCompareGeneric2 (string[] args)
{
	//expected Check<Test> but Check<Check<Test>>
	Check<Test> res = Defend (GetData<Test> (),
		a => Defend (GetData<Test> (),
			          b => a == b ? a : b));

	Console.WriteLine (res);
	Console.WriteLine ("finished!\n");
}
{% endhighlight %}
Problem again, our code returns Check<Check<Test>> instead of Check<Test> and it is not very good. We will change our code to avoid this problem.
{% highlight csharp %}
public class Check2<T>
	where T : class
{
	private Check2 (T val)
	{
		Value = val;
	}

	public static Check2<T> Success (T val)
	{
		return new Check2<T> (val){ IsFailed = false };
	}

	public static Check2<T> Fail ()
	{
		return new Check2<T> (null){ IsFailed = true };
	}

	public bool IsFailed {
		get;
		private set;
	}

	public T Value {
		get;
		private set;
	}

	public override string ToString ()
	{
		return string.Format (
			"[Check: IsFailed={0}, Value={2}]", 
			IsFailed, 
			Value);
	}
}

public static Check2<TB> Defend<TA,TB> (Check2<TA> a, Func<TA, Check2<TB>> f)
	where TA : class//applied only for TA which can be null
	where TB : class//applied only for TB which can be null
{
	return a.IsFailed ? Check2<TB>.Fail () : f (a.Value);
}

public static Func<Check2<T>> Lift<T> (Func<T> f)
	where T : class
{
	return () => {
		var res = f ();
		return res == null ? Check2<T>.Fail () : Check2<T>.Success (res);
	};
}
{% endhighlight %}
Now null check test is in List function. And the main task of this function is to wrap any function which retruns T into function which returns Check<T>. Problem solved. We can think about it in this way: we have some functions and our fucntion Defend. But to use them together we need to adapt all used functions. And our lift function solves that problem. Checking.
{% highlight csharp %}
public static void DefesiveCompare (string[] args)
{
	var getData = Lift<Test> (GetData<Test>);
	Check2<Test> res = Defend<Test,Test> (getData (),
		                   a => Defend<Test,Test> (getData (),
			                   b => a == b ? a : b));
	//unable to cast Test to Check<Test>
	Console.WriteLine (res);
	Console.WriteLine ("finished!\n");
}
{% endhighlight %}
Problem, problem, problem. Our end b => a == b ? a : b retruns result not wrapped into Check type. Lets write some helper function which wraps any type T into Check. The name of function will be return. Lets add it and do some refactoring
{% highlight csharp %}
public class Check3<T>
{
	private Check3 (T val)
	{
		Value = val;
	}

	public static Check3<T> Success (T val)
	{
		return new Check3<T> (val){ IsFailed = false };
	}

	public static Check3<T> Fail ()
	{
		return new Check3<T> (default(T)){ IsFailed = true };
	}

	public bool IsFailed {
		get;
		private set;
	}

	public T Value {
		get;
		private set;
	}

	public override string ToString ()
	{
		return string.Format (
			"[Check: IsFailed={0}, Value={1}]", 
			IsFailed, 
			Value);
	}
}

public static Check3<TB> Defend<TA,TB> (Check3<TA> a, Func<TA, Check3<TB>> f)
{
	return a.IsFailed ? Check3<TB>.Fail () : f (a.Value);
}

public static Func<Check3<T>> Lift2<T> (Func<T> f)
	where T : class
{
	return () => {
		var res = f ();
		return res == null ? Check3<T>.Fail () : Check3<T>.Success (res);
	};
}

public static Check3<T> Return <T> (T val)
{
	return Check3<T>.Success (val);
}

public static void DefesiveCompare (string[] args)
{
	var getData = Lift2<Test> (GetData<Test>);
	Check3<Test> res = Defend<Test,Test> (getData (), 
		                   a => Defend<Test,Test> (getData (), 
			                   b => Return (a == b ? a : b)));

	Console.WriteLine (res);
	Console.WriteLine ("finished!");
}
{% endhighlight %}
Awesome. It works as expected. So what do we have? Wrapper type Check over any type T. Two functions Defend and Return. And this is all what we need to write some defensive code without a lot of null checks. But we can write some other wrapper type and define functions Return and Defend over it with different functionality in function Defend. It will allow us to use the same code but now with different effect. For example instead of check we can implement async effect(and we do that later). This pattern is well known as monad and have a lot more possibilities to do, but instead of using Defend name for composion function usually used Bind name.  One minor problem that our consuming code looks not very beatiful in terms of wrapped functions and we as imperative developers prefere simple line by line code. Fortunately for us some solutions are already here. In some programming languages we have support for syntaxis sugar over monads: linq expressions in c#, do notation in Haskell and computation expressions in fsharp. Computation expressions is not only for monad syntax but we will discuss it in next posts. Lets try to adopt our code to linq expressions, we should implement extension function SelectMany for our wrapper type. 
{% highlight csharp %}
public class Check<T>
{
	private Check (T val)
	{
		Value = val;
	}

	public static Check<T> Success (T val)
	{
		return new Check<T> (val){ IsFailed = false };
	}

	public static Check<T> Fail ()
	{
		return new Check<T> (default(T)){ IsFailed = true };
	}

	public bool IsFailed {
		get;
		private set;
	}

	public T Value {
		get;
		private set;
	}

	public override string ToString ()
	{
		return string.Format (
			"[Check: IsFailed={0}, Value={1}]", 
			IsFailed, 
			Value);
	}
}

public static class CheckMonad
{
	public static Check<T> Return<T> (this T value)
	{
		return Check<T>.Success (value);
	}

	public static Check<U> Bind<T, U> (this Check<T> m, Func<T, Check<U>> k)
	{
		return m.IsFailed ? Check<U>.Fail () : k (m.Value);
	}

	public static Func<Check<T>> Lift<T> (Func<T> f)
		where T : class
	{
		return () => {
			var res = f ();
			return res == null ? Check<T>.Fail () : Check<T>.Success (res);
		};
	}

	public static Check<V> SelectMany<T, U, V> (
		this Check<T> id,
		Func<T, Check<U>> k,
		Func<T, U, V> s)
	{
		return id.Bind (x => k (x).Bind (y => s (x, y).Return ()));
	}
}

class Test
{
	public Test ()
	{

	}
}

class MainClass
{
	public static bool Bool ()
	{
		return new Random ().Next () % 2 == 0 ? true : false;
	}

	public static T GetData<T> () 
		where T : class, new()
	{
		return Bool () ? null : new T ();
	}

	static void Main (string[] args)
	{
		var getData = CheckMonad.Lift<Test> (GetData<Test>);
		var res = 
			from a in getData ()
			from b in getData ()
			select a == b ? a : b;

		Console.WriteLine (res);
		Console.WriteLine ("finished!");
	}
}
{% endhighlight %}
Now everything is ok. We can use this way to add syntatic shugare for other wrapper types. One of the advantages of monads is that describing the code over a monad, we can run it on top of other monads until monad carries within itself the same type. For example compare the code for the Async monad
{% highlight csharp %}
var getData = AsyncMonad.Lift (GetData);
var res = 
	from a in getData ()
	from b in getData ()
	select a.Substring (0, 10) + b.Substring (10, 20);
{% endhighlight %}
and Check monad
{% highlight csharp %}
var getData = CheckMonad.Lift (GetData);
var res = 
	from a in getData ()
	from b in getData ()
	select a.Substring (0, 10) + b.Substring (10, 20);
{% endhighlight %}
Very cool but there is a problem with the composition of monads. We would uses the same code with functions that return Async <Check <T>>. However, our code in the Bind function of type Async knows nothing about nested type Check, so our code will not work, our bind function unwraps only Async and returns Check <T> instead of T. Here monads transformers come into play. What is a monad transformer? This is a sort of thing which is taking an unknown monad as input adds some functionality of other monad and the result is a combined monad. Suppose in our case to monads Async <T> Check <T> which could not be used together, we can write monads transformers AsyncT <T,ParentMonad> and CheckT <T, ParentMonad>.For our case Async <Check <T> > we can safely do something like this:
{% highlight csharp %}
var getData = CheckT<T, Async<T>>.LiftT (AsyncMonad.Lift (GetData));
var res = 
	from a in getData ()
	from b in getData ()
	select a.Substring (0, 10) + b.Substring (10, 20);
{% endhighlight %}
Usually all "monad in c#"" tutorials ends here with words: this kind of things could exists in languages like Haskell which supports higher kinded types but not in c#. But we as a smart developers well knows that we could implement some workaround over any problem so lets try to create one. We will look on a typical example with Functor interface. When we solve that not so hard problem we will be able to use the same workaround for implementation of monad transformers in c#. So Functor interface looks like this:
{% highlight csharp %}
interface IFunctor<T> {
	T<B> FMap<A, B>(Func<A, B> f, T<A> a);
}
{% endhighlight %}
На первый взгляд ничего подозрительного, у нас есть контейнерный тип и далее мы определяем сигнатуру требуемого метода который применяет функцию над типом A к завернутому в T типу A и оборачивает результат функции типа B в контейнерный тип T. Все хорошо за искоючением того что в c# нельзя задать подобный тип T. Как видно из кода тип T является генериком T<_> и тут то собака и порылась. Я не буду обьяснять суть проблемы, тут проще взять и скопировать определение функтора данное выше и попробовать решить проблему самим в ide. Это отличная головолмка. И сразу становиться все ясно.
Попроовали? Решили? Отлично. Можете дальше не читать. Теперь решим вместе.
В чем суть типа T в нашем интерфейсе? Зачем он нужен, а нужен он для того чтобы пометить входящий и выходящий контейнеры одним маркером и наложить ограничение на то, что они должны быть одним и темже генерик типом. Чтобы не получилось что на входе List<AType>, а на выходе Check<BType>. Ок задача ясна пометить генерик тип негенерик типом. Как сделать? Да просто:
{% highlight csharp %}
public abstract class Wrapper
{
	private Wrapper ()
	{
		
	}

	public class WrapperImpl<T> : Wrapper
	{
	}
}
{% endhighlight %}
Хм интересно. Давайте подумаем какую гарантию нам это дает. А дает оно нам гарантию того что инстанс типа Wrapper всегда будет инстансом типа WrapperImpl, единственное в чем надо быть уверенными так это в том что при апкасте мы не промажем с типом сдержащимся в WrapperImpl. Давайте введем специальный тип контейнер для информации о генерик типе и немного перепишем определение враппер типа. 
{% highlight csharp %}
public interface IGeneric<T, TCONTAINER>
{

}
public class Wrapper{
	public class WrapperImpl<T> : Wrapper, IGeneric<T, Wrapper>
	{

	}
}
{% endhighlight %}
Ну и теперь добавить безопасный хелпер метод для кастов.
{% highlight csharp %}
public static class GenericExts
{
	public static TM UpCast<T, TM, TMB> (this IGeneric<T, TMB> m)
		where TM : IGeneric<T, TMB>
	{
		return (TM)m;//safe for single inheritance
	}

	public static IGeneric<T, TMB> DownCast<T, TM, TMB> (this TM m)
		where TM : IGeneric<T, TMB>
	{
		return (TM)m;//safe for single inheritance
	}
}
{% endhighlight %}
Ну и с учетом выше написанного перепишем определение интерфейса для функтора
{% highlight csharp %}
public interface IFunctor<T>
{
	CB FMap<A, B, CA, CB> (Func<A, B> f, CA a)
		where CA : IGeneric<A, T>
		where CB : IGeneric<B, T>;
}
{% endhighlight %}
Вуаля, ловкость пальцев и никакого обмана. Единственное условие, это следовать петтерну одиночного наследника, чтобы одним маркером нельзя было маркировать несколько классов. Давайте соберем все вместе и посмотрим работает ли.
{% highlight csharp %}
//	interface IFunctor<T<_>> {
//		T<B> FMap<A, B>(Func<A, B> f, T<A> a);
//	}
public interface IGeneric<T, TCONTAINER>
{

}

public static class GenericExts
{
	public static TM UpCast<T, TM, TMB> (this IGeneric<T, TMB> m)
		where TM : IGeneric<T, TMB>
	{
		return (TM)m;//safe for single inheritance
	}

	public static IGeneric<T, TMB> DownCast<T, TM, TMB> (this TM m)
		where TM : IGeneric<T, TMB>
	{
		return (TM)m;//safe for single inheritance
	}
}

public interface IFunctor<T>
{
	CB FMap<A, B, CA, CB> (Func<A, B> f, CA a)
		where CA : IGeneric<A, T>
		where CB : IGeneric<B, T>;
}

public interface IFunctorSelf<TGENERIC, TSELF, TVALUE>
	where TSELF : IGeneric<TVALUE, TGENERIC>
{
	CB FMap<B, CB> (Func<TVALUE, B> f)
		where CB : IGeneric<B, TGENERIC>;
}

public abstract class Wrapper
{
	private Wrapper ()
	{
		
	}

	public class WrapperImpl<T> : 
				Wrapper, 
				IGeneric<T, Wrapper>, 
				IFunctorSelf<Wrapper, WrapperImpl<T> , T>
	{
		#region IFunctorSelf implementation

		public CB FMap<B, CB> (Func<T, B> f) where CB : IGeneric<B, Wrapper>
		{
			var res = new WrapperImpl<B> (f (Value));
			return res.Cast<B, CB,Wrapper> ();
		}

		#endregion

		public WrapperImpl (T val)
		{
			Value = val;
		}

		public T Value {
			get;
			set;
		}


	}
}
class MainClass
{
	public static void Main (string[] args)
	{
		var a = new Wrapper.WrapperImpl<int> (1);
		var b = a.FMap<int, Wrapper.WrapperImpl<int>> (x => -x);
		Console.WriteLine ("Value is: " + b.Value);
		Console.ReadLine ();
	}
}
{% endhighlight %}
Ура. Мы сорвали джек пот, пора просить добавку к зарплате у начальства.
Теперь у нас есть все, чтобы для начала определить интерфейс для монад и наслждаться полиморфным кодом над ними, а также получить возможность их композиции.
{% highlight csharp %}
public interface IMonad<T, TMI>
{
	IMonad<TB,TMI> Return<TB> (TB val);
	IMonad<TB,TMI> Bind<TB> (Func<T, IMonad<TB,TMI>> f);
}
public static class MonadSyntax
{
	public static TM Cast<T, TM, TMB> (this IMonad<T, TMB> m)
		where TM : IMonad<T, TMB>
	{
		return (TM)m;//safe for single inheritance
	}

	public static IMonad<V, TMI> SelectMany<T, TMI, U, V> 
	(
		this IMonad<T, TMI> id,
		Func<T, IMonad<U, TMI>> k,
		Func<T, U, V> s)
	{
		return id.Bind (x => k (x).Bind (y => id.Return (s (x, y))));
	}
}
{% endhighlight %}
На данный момент в коде должно быть все понятно, мы взяли идею из функтора применили к интерефейсу для монад, который содержит определение методов bind и return. Теперь мы можем переписать Check монаду на основе этого интерфейса и реализовать Async монаду. Async монада может послужить примером того как можно сделать монадическую обертку над существующим генерик типом, который не имплементирует интерфейс монады. В нашем случае Async<T> это просто обертка над типом Task<T> и можно думать, что это адаптер типа Task к монадическому интерфейсу.
{% highlight csharp %}
public class Check
{
	Check ()
	{

	}

	public class CheckM<T>: Check, IMonad<T, Check>
	{
		#region IMonad implementation

		public IMonad<TB, Check> Return<TB> (TB val)
		{
			return CheckM<TB>.Success (val);
		}

		public IMonad<TB, Check> Bind<TB> (Func<T, IMonad<TB, Check>> f)
		{
			return this.IsFailed ? CheckM<TB>.Fail () : f (this.Value);
		}

		#endregion

		CheckM (T val)
		{
			Value = val;
		}

		public static CheckM<T> Success (T val)
		{
			return new CheckM<T> (val){ IsFailed = false };
		}

		public static CheckM<T> Fail ()
		{
			return new CheckM<T> (default(T)){ IsFailed = true };
		}

		public bool IsFailed {
			get;
			private set;
		}

		public T Value {
			get;
			private set;
		}

		public override string ToString ()
		{
			return string.Format (
				"[Check: IsFailed={0}, Value={1}]", 
				IsFailed, 
				Value);
		}
	}
}

public static class CheckMonad
{
	public static Func<Check.CheckM<TB>> Lift<TB> (this Func<TB> f)
		where TB : class
	{
		return () => {
			var res = f ();
			return res == null 
				? Check.CheckM<TB>.Fail () 
				: Check.CheckM<TB>.Success (res);
		};
	}
}
public class Async
{
	Async ()
	{

	}

	public class AsyncM<T>: Async, IMonad<T, Async>
	{
		#region IMonad implementation

		public IMonad<TB, Async> Return<TB> (TB val)
		{
			return new AsyncM<TB>(Task<TB>.FromResult(val));
		}
		//helper method two tasks composition
		private static async Task<TB> BindTasks<TB> (
			Task<T> m, 
			Func<T, Task<TB>> f)
		{
			var r = await m;
			return await f(r);
		}

		public IMonad<TB, Async> Bind<TB> (Func<T, IMonad<TB, Async>> f)
		{
			return new AsyncM<TB>(BindTasks(this.Task, 
				(t) => f(t).CastM<TB, AsyncM<TB>, 
				Async>().Task));
		}

		#endregion

		public AsyncM (Task<T> val)
		{
			Task = val;
		}
		public Task<T> Task {
			get;
			set;
		}
	}
}

public static class AsyncMonad
{
	public static Func<Async.AsyncM<TB>> Lift<TB> (this Func<Task<TB>> f)
		where TB : class
	{
		return () => {
			var res =  f ();
			return new Async.AsyncM<TB>(res);
		};
	}
}
{% endhighlight %}
Ну Check тип мы уже видели что работает, а как там насчет Async<T>?
{% highlight csharp %}
class MainClass
{
	public static Task<String> GetData () 
	{
		return new WebClient().DownloadStringTaskAsync(
			new Uri("http://google.com")
		);
	}

	static void Main (string[] args)
	{
		var getData = AsyncMonad.Lift (GetData);
		var res = 
			from a in getData ()
			from b in getData ()
			select a.Substring(0,10) + b.Substring(10,20);
		var task = res.CastM<string, Async.AsyncM<string>, Async> ().Task;
		Console.WriteLine (task.Result);
		Console.WriteLine ("finished!");
		Console.ReadLine ();
	}
}
{% endhighlight %}
Удивительно я бы сказал, но оно работает. Итак полиморфные монады у нас в кармане, теперь приступим к трансформерам. Итак есть тип Async<Check<T>> который является монадой Async над типом Check<T>, но нам бы хотелось его превратить в монаду Async<Check<_>> над типом T. Как это сделать? Все просто надо завернуть Async<Check<T>> в монаду над типом T. Допустим назовем этот тип трансформером и для Check монады назовем его CheckT. В конце концов мы получим конструкцию вида CheckT<Async<Check<T>>> этакий трехслойный бутерброд. Главная фишка что этот CheckT реализует интерфейс не IMonad над Async<Check<T>>>, а Imonad над T. Еще раз повторим: CheckT это враппер для типов вида SomeOtherMonad<CheckMonad<T>> и функция Lift для CheckT будет преобразовывать функции возвращающие SomeMonad<T> в функции возвращающие CheckT<SomeOtherMonad<CheckMonad<T>>>. В функции return он будет завертывать значение в тип Check, а потом его передавать методу return класа SomeOtherMonad и на выходе кастить в себя. В Bind поведение чуть сложнее и вся магия происходит там, но суть таже. Наверное это сложнее описать словами чем кодом. Чтож давайте реализуем CheckT. Я для облегчения понимания решил разделить определение типов CheckedVal это просто тип контейнер. Check.CheckM это монада над CheckedVal и
CheckForT<TMI>.CheckT<T> это трансформер над CheckedVal. Хотя такое разделение излишне и тип CheckedVal должен быть включен в тип Check.CheckM.
Особое внимание в коде надо обратить на функцию bind и место где описан маркер обертываемой монады TMI. Он определен в типе маркере трансформера CheckForT<TMI>.CheckT<T>, а не в типе трансформера CheckForT.CheckT<T,TMI>. Это сделано для того чтобы при реалзации методов интерфейса монады, у нас не терялся тип обернутой монады. Так мы застраховались от выхова bind на функциях которые возвращают разные обернутые и трансформированные в CheckT монады. Это вызвало бы ошибку в рантайм. Итак вся магия здесь.
{% highlight csharp %}
public class CheckForT<TMI>
{
	CheckForT ()
	{

	}

	public class CheckT<T>: CheckForT<TMI>, IMonad<T, CheckForT<TMI>>
	{
		#region IMonad implementation

		public IMonad<TB, CheckForT<TMI>> Return<TB> (TB val)
		{
			return new CheckT<TB> (
				Value.Return<CheckedVal<TB>> (
					CheckedVal<TB>.Success (val)
				)
			);
		}

		private IMonad<CheckedVal<TB>,TMI> BindInternal<TB> (
			CheckedVal<T> check, 
			Func<T, IMonad<TB, CheckForT<TMI>>> f)
		{
			return check.IsFailed 
				? Value.Return<CheckedVal<TB>> (CheckedVal<TB>.Fail ()) 
				: f (check.Value).CastM<TB, CheckT<TB>,CheckForT<TMI>> ().Value;
		}

		public IMonad<TB, CheckForT<TMI>> Bind<TB> (
			Func<T, IMonad<TB, CheckForT<TMI>>> f)
		{
			var tmp = Value.Bind<CheckedVal<TB>> (
				check => BindInternal (check, f)
			);
			return new CheckT<TB> (tmp);
		}

		#endregion

		public CheckT (IMonad<CheckedVal<T>,TMI> val)
		{
			Value = val;
		}

		public IMonad<CheckedVal<T>,TMI> Value {
			get;
			private set;
		}
	}
}

public static class CheckMonad
{
	public static Func<Check.CheckM<TB>> Lift<TB> (this Func<TB> f)
		where TB : class
	{
		return () => {
			var res = f ();
			return new Check.CheckM<TB> (CheckedVal<TB>.ToCheck (res));
		};
	}

	public static Func<Check.CheckM<TB>> Lift<TB> (
		this Func<CheckedVal<TB>> f)
		where TB : class
	{
		return () => {
			var res = f ();
			return new Check.CheckM<TB> (res);
		};
	}

	public static Func<CheckForT<TMI>.CheckT<TB>> LiftT<TB,TMI> (
		this Func<IMonad<TB,TMI>> f)
		where TB : class
	{
		Func<IMonad<CheckedVal<TB>,TMI>> checkF = () => {
			var m = f ();
			return m.Bind (val => m.Return (CheckedVal<TB>.ToCheck (val)));
		};

		return () => {
			var monad = checkF ();
			return new CheckForT<TMI>.CheckT<TB> (monad);
		};
	}
}
{% endhighlight %}
Ну что пора показать заказчику рабочий код для Async<CheckedValue> монады.
{% highlight csharp %}
public static Task<String> GetData ()
{
	//return Task<String>.FromResult ((string)null);//for check tests
	return new WebClient ().DownloadStringTaskAsync (
		new Uri ("http://google.com")
	);
}

static void Main (string[] args)
{
	//expected Check<Test> but Check<Check<Test>>
	var getData = CheckMonad.LiftT (AsyncMonad.Lift (GetData));
	var res = 
		from a in getData ()
		from b in getData ()
		select a.Substring (0, 10) + b.Substring (10, 20);
	var checkT = res.CastM<string, CheckForT<Async>
					.CheckT<string>, CheckForT<Async> > ();
	var task = checkT
					.Value
					.CastM<	CheckedVal<string>, 
							Async.AsyncM<CheckedVal<string>>,
							Async> ()
					.Task;
	Console.WriteLine (task.Result);
	Console.WriteLine ("finished!");
	Console.ReadLine ();
}
{% endhighlight %}
Полный код [здесь](https://gist.github.com/ hodzanassredin/28c4208206d9d88908f5 "полный код"). Опять работает. Ну мы добились того о чем грезили все генетики последние годы, мы скрестили ужа с ежом. Причем одна из монад была просто оболчкой над существующим классом Task. В коде видно как неудобно разворачивать завернутиые в кучу оберток типы, но это можно исправить перенеся финальный код в синтаксис монады или создав хелпер методы. Я надеюсь материал был не шибко запутан и мой стиль подачи вас не напугал. В следующей часте будет интересней: посмотрим на отличия computation expressions от монад, обнаружим что монады являются тюринг полными и обсудим возможности их применения в области определения семантики для монадического синтаксиса. 