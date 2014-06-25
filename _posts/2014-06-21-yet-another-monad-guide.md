---
published: true
layout: post
title: Monads and monad transformers for mere mortals in pure C#. DRAFT
tags : [lessons, csharp, monad]
---

## {{page.title}} [RUS]({{ site.url }}/2014/06/21/yet-another-monad-guide-rus.html)

<p class="meta">25 June 2014 &#8211; Karelia</p>

During migration of a small project to fsharp, I thought a lot about using monads in practice. Subject fascinated me enough and I decided to describe my experience in a couple of posts. Unfortunately all the material demanded decent knowledge of monads and monad transformers, at the same time I wanted to make a material without any links to external materials, without a lot of words and simple for ordinary programmers who are not burdened by the knowledge of mathematics and  syntax of some functional programming language. Thus was born the idea to write another explanation of what is a monad. Since this post is aimed at programmers, and it will be approached by code and not by the text or mathematics. Ready? Let's go.

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
public static void Compare (string[] args)
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

Ok, code now looks more safe, but we have repeated code, lets remove it to support DRY principle.

{% highlight csharp %}
public static bool Defend (object o)
{
	if (o == null) {
		Console.WriteLine ("Can't compare");
		return false;
	}
	return true;
}

public static void Compare (string[] args)
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

Looks better, but anyway we must add if check for every GetData invocation. Lets move if check into Defend function.

{% highlight csharp %}
public static string Defend (object a, Func<object, string> f)
{
	return a == null ? "Can't compare" : f (a);
}

public static void Compare (string[] args)
{
	var res = Defend (GetData<Object> (), 
		          (a) => Defend (GetData<Object> (), 
			          (b) => a == b ? "a == b" : "a != b"));

	Console.WriteLine (res);
	Console.WriteLine ("finished!");
}
{% endhighlight %}

Just perfect, but we have a problem in case of using some other type, for example class Test. During execution it will be downcasted to System.Object and we will not be able to use its members. 

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
public static T Defend<T> (T a, Func<T, T> f)
	where T: class//applied only for T which can be null
{
	return a == null ? "Can't compare" : f (a);
}
{% endhighlight %}
New problem, we are trying to return a type string instead of a type Test. And what to do? Lets create some type which could store a value or an error string.
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
Looks beautiful. In action:
{% highlight csharp %}
public static void Compare (string[] args)
{
	//expected Check<Test> but Check<Check<Test>>
	Check<Test> res = Defend (GetData<Test> (),
		a => Defend (GetData<Test> (),
			          b => a == b ? a : b));

	Console.WriteLine (res);
	Console.WriteLine ("finished!\n");
}
{% endhighlight %}
Problem again, our code returns Check< Check< T > > instead of Check< T > and it is not very good. We will change our code to avoid this problem.
{% highlight csharp %}
public class Check<T>
	where T : class
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
		return new Check<T> (null){ IsFailed = true };
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

public static Check<TB> Defend<TA,TB> (Check<TA> a, Func<TA, Check<TB>> f)
	where TA : class//applied only for TA which can be null
	where TB : class//applied only for TB which can be null
{
	return a.IsFailed ? Check<TB>.Fail () : f (a.Value);
}

public static Func<Check<T>> Lift<T> (Func<T> f)
	where T : class
{
	return () => {
		var res = f ();
		return res == null ? Check<T>.Fail () : Check<T>.Success (res);
	};
}
{% endhighlight %}
Now null check test lives in a Lift function. And the main task of the function is to wrap any function which returns T into function which returns Check< T >. Problem solved. We can think about it in this way: we have some functions and our function Defend. But to use them together we need to adapt all used functions to Defent function. And this is main purpose of the lift function. 
{% highlight csharp %}
public static void Compare (string[] args)
{
	var getData = Lift<Test> (GetData<Test>);
	Check<Test> res = Defend<Test,Test> (getData (),
		                   a => Defend<Test,Test> (getData (),
			                   b => a == b ? a : b));
	//unable to cast Test to Check<Test>
	Console.WriteLine (res);
	Console.WriteLine ("finished!\n");
}
{% endhighlight %}
Problem, problem, problem. At the end we return result not wrapped into the Check type. Lets write some helper function which wraps any type T into the Check. The name of function will be return. Lets add it and do some refactoring
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

public static Check<TB> Defend<TA,TB> (Check<TA> a, Func<TA, Check<TB>> f)
{
	return a.IsFailed ? Check<TB>.Fail () : f (a.Value);
}

public static Func<Check<T>> Lift<T> (Func<T> f)
	where T : class
{
	return () => {
		var res = f ();
		return res == null ? Check<T>.Fail () : Check<T>.Success (res);
	};
}

public static Check<T> Return <T> (T val)
{
	return Check<T>.Success (val);
}

public static void DefesiveCompare (string[] args)
{
	var getData = Lift<Test> (GetData<Test>);
	Check<Test> res = Defend<Test,Test> (getData (), 
		                   a => Defend<Test,Test> (getData (), 
			                   b => Return (a == b ? a : b)));

	Console.WriteLine (res);
	Console.WriteLine ("finished!");
}
{% endhighlight %}
Awesome, it works as expected. So what do we have? Wrapper type Check over any type T, two functions Defend and Return. And this is all what we need to write some defensive code without null checks. But we can write some other wrapper type and define functions Return and Defend over it with a different functionality in function Defend. It will allow us to use the same code, but now with different effect. For example instead of checking we can implement async effect(and we do that later). This pattern is well known as monad, and function Defnd has name Bind by convention in monad pattern.  One minor problem is that our consuming code looks not very beautiful in terms of wrapped functions and we as imperative developers prefer simple line by line code. Fortunately for us, some solutions are already here. In some programming languages we have support for syntactic sugar over monads: linq expressions in c#, do notation in Haskell and computation expressions in fsharp. Computation expressions is not only for monad syntax, but we will discuss it in next posts. Lets try to adapt our code to linq expressions, we should implement extension function SelectMany for our wrapper type. 
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
Now everything is ok. We can use this pattern to add syntactic sugar for other wrapper types. We can use the same code for differnet monads, until monad carries within itself the same type. For example, compare the code for the Async monad
{% highlight csharp %}
var getData = AsyncMonad.Lift (GetData);
var res = 
	from a in getData ()
	from b in getData ()
	select a.Substring (0, 10) + b.Substring (10, 20);
{% endhighlight %}
and the Check monad
{% highlight csharp %}
var getData = CheckMonad.Lift (GetData);
var res = 
	from a in getData ()
	from b in getData ()
	select a.Substring (0, 10) + b.Substring (10, 20);
{% endhighlight %}
Very cool, but there is a problem with the composition of monads. We would use the same code with functions that return Async< Check < T > >. However, our code in the Bind function of the type Async knows nothing about nested type Check, so our code will not work, our bind function unwraps only Async and returns Check < T > instead of T. Here monads transformers come into play. What is a monad transformer? This is a sort of thing which is taking an unknown monad as input, adds some functionality of other monad and returns a combined monad. Suppose in our case with monads Async< T > and Check< T > which could not be used together, we can write monads transformers AsyncT< T,ParentMonad > and CheckT< T, ParentMonad >.For our case Async< Check< T > > we can safely do something like this:
{% highlight csharp %}
var getData = CheckT<T, Async<T>>.LiftT (AsyncMonad.Lift (GetData));
var res = 
	from a in getData ()
	from b in getData ()
	select a.Substring (0, 10) + b.Substring (10, 20);
{% endhighlight %}
Usually all "monad in c#"" tutorials ends here with words: this kind of things could exists in languages like Haskell which supports higher kinded types but not in c#. But we as a smart developers well know that we could implement some workaround over any problem, so lets try to create one. We will look on a typical example with Functor interface. When we solve that problem we will be able to use the same workaround for implementation of monad transformers in c#. So Functor interface looks like this:
{% highlight csharp %}
interface IFunctor<T> {
	T<B> FMap<A, B>(Func<A, B> f, T<A> a);
}
{% endhighlight %}
Nothing special is here, it describes a function which takes "a" value wrapped into a type T, unwraps it, applies function f to unwrapped value and finally wraps result into the type T. Everything seems to be ok, but we can't write this code in C#. C# doesn't support usage of type variable T as type constructor. I don't want to describe whole problem here and better way to understand this restriction is to copy interface definition into IDE and play with it. It is a good puzzle. Lets try to analyse that problem and solve it step by step. Why do we need type T here? We need it as a constraint to input and output of FMap function. They should have the same wrapper type over different wrapped types. It guards us from incorrect implementations which takes Check< AType > and returns List< BType >. So we need to mark generic type by some other non generic type. How can we do that. It is simple.
{% highlight csharp %}
public abstract class Wrapper
{
	private Wrapper ()
	{
		
	}

	public sealed class WrapperImpl<T> : Wrapper
	{
	}
}
{% endhighlight %}
Interesting. First of all we can be sure that instance of type Wrapper is the instance of type WrapperImpl. But we need to keep wrapped type somewhere to do safe upcast. Lets introduce special type container, which stores generic type marker with wrapped type. Also we need to rewrite WrapperImpl to support it. 
{% highlight csharp %}
public interface IGeneric<T, TCONTAINER>
{

}
public class Wrapper{
	public sealed class WrapperImpl<T> : Wrapper, IGeneric<T, Wrapper>
	{

	}
}
{% endhighlight %}
Now we can add helper method for safe upcasts.
{% highlight csharp %}
public static class GenericExts
{
	public static TM UpCast<T, TM, TMB> (this IGeneric<T, TMB> m)
		where TM : TMB, IGeneric<T, TMB>
	{
		return (TM)m;//safe for single inheritance
	}
}
{% endhighlight %}
And now we can solve our Functor interface problem with type T.
{% highlight csharp %}
public interface IFunctor<T>
{
	CB FMap<A, B, CA, CB> (Func<A, B> f, CA a)
		where CA : IGeneric<A, T>
		where CB : IGeneric<B, T>;
}
{% endhighlight %}
Bingo. One restriction is to follow single inheritance pattern when describing our container classes. Lets try to use out Functor interface in csharp's idiomatic way.
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
		where TM : TMB, IGeneric<T, TMB>
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

	public sealed class WrapperImpl<T> : 
				Wrapper, 
				IGeneric<T, Wrapper>, 
				IFunctorSelf<Wrapper, WrapperImpl<T> , T>
	{
		#region IFunctorSelf implementation

		public CB FMap<B, CB> (Func<T, B> f) where CB : IGeneric<B, Wrapper>
		{
			var res = new WrapperImpl<B> (f (Value));
			return res.UpCast<B, CB,Wrapper> ();
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
Now we have everything to implement IMonad interface and later build monad transformers on top of it. 
{% highlight csharp %}
public interface IMonad<T, TMI>
{
	IMonad<TB,TMI> Return<TB> (TB val);
	IMonad<TB,TMI> Bind<TB> (Func<T, IMonad<TB,TMI>> f);
}
public static class MonadSyntax
{
	public static TM UpCast<T, TM, TMB> (this IMonad<T, TMB> m)
		where TM : TMB, IMonad<T, TMB>
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
It should be clear what is going on here. We took our workaround for functor interface and applied it to our IManad interface. Now we can rewrite our Check monad and adapt it to out IMonad interface. Also now we can implement Async monad. Async monad implementation can be used as an example of how to adapt some existing type to monadic interface. In our case we will build Async monad on top of Task< T > type.
{% highlight csharp %}
public class Check
{
	Check ()
	{

	}

	public sealed class CheckM<T>: Check, IMonad<T, Check>
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

	public sealed class AsyncM<T>: Async, IMonad<T, Async>
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
			return await f(r);//could be rewriten as return f(r);
		}

		public IMonad<TB, Async> Bind<TB> (Func<T, IMonad<TB, Async>> f)
		{
			return new AsyncM<TB>(BindTasks(this.Task, 
				(t) => f(t).UpCast<TB, AsyncM<TB>, 
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
Ok Check monad works but what about Async< T >?
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
		var task = res.UpCast<string, Async.AsyncM<string>, Async> ().Task;
		Console.WriteLine (task.Result);
		Console.WriteLine ("finished!");
		Console.ReadLine ();
	}
}
{% endhighlight %}
It works as expected, we have polymorphic monads. Time for transformers. We have type Async< Check< T > > it is an Async monad over type Check< T >, but we want to convert it into a monad Async<Check<_>> over the type T. How to do that? We need to wrap Async<Check<T>> into a monad over type T. Lets name it as CheckT transformer for the Check monad. At the end we will have a type like this CheckT< Async< Check< T >>>, it is very similar to a sliced bread. Main thing is that CheckT implements interface IMonad over T and not over Async< Check< T > > >. Repeat one more time: CheckT is a wrapper for types like SomeOtherMonad< CheckMonad< T > > and Lift function for CheckT convert functions which returns SomeMonad< CheckMonad< T > > into functions which returns CheckT< SomeOtherMonad< CheckMonad< T > > >. In Return function it will wrap value into the Check type, and after that will use Return function of other monad to wrap it one more time and finally cast result to type CheckT. Bind function is a little bit harder to understand, but logic is the same. So lets implement the CheckT type. For better understanding I separated Check types into: a container CheckedVal< T >, a monad adapter CheckM for the type CheckedVal< T > and a monad transformer CheckT for the type CheckedVal< T >. Check.CheckM. This separation is artificial and you can merge CheckedVal< T > and CheckM into the single one. Most attention should be paid to place where we put internal monad marker in type CheckT. It is defined in parent type CheckForT< TMI >.CheckT< T >, but not in generic type CheckForT.CheckT< T,TMI >. This constraints our IMonad functions to use the same internal monad marker everywhere. And it is similar to partial type construction. So magic lives here:
{% highlight csharp %}
public class CheckForT<TMI>
{
	CheckForT ()
	{

	}

	public sealed class CheckT<T>: CheckForT<TMI>, IMonad<T, CheckForT<TMI>>
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
				: f (check.Value).UpCast<TB, CheckT<TB>,CheckForT<TMI>> ().Value;
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
And now we are ready to implement code for Async< CheckedValue > monad.
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
	var getData = CheckMonad.LiftT (AsyncMonad.Lift (GetData));
	var res = 
		from a in getData ()
		from b in getData ()
		select a.Substring (0, 10) + b.Substring (10, 20);
	var checkT = res.UpCast<string, CheckForT<Async>
					.CheckT<string>, CheckForT<Async> > ();
	var task = checkT
					.Value
					.UpCast<	CheckedVal<string>, 
							Async.AsyncM<CheckedVal<string>>,
							Async> ()
					.Task;
	Console.WriteLine (task.Result);
	Console.WriteLine ("finished!");
	Console.ReadLine ();
}
{% endhighlight %}
Full code [here](https://gist.github.com/ hodzanassredin/28c4208206d9d88908f5 "code"). So we composed two monads into single one and this is a real benefit for us, now we can write generic code which is polymorphic for different monad types and have a way to cmpose monads. One of the monads was just a wrapper over existing type Task<T>. It is clear that we have problems with result unwrapping, but it can be avoided by moving final code into the monad syntax or by creating helper methods like runAsync. I hope this post was helpful for you and now you will be able to read articles about interesting problems solutions, like parsing, described in therms of monads. In the next chapter we will look at the differences between computation expressions and monads, will find that monads are Turing complete and will see how to use monads to solve real world problems and to create DSLs. 