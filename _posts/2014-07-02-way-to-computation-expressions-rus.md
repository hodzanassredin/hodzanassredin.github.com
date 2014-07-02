---
published: true
layout: post
title: Путь к computation expressions(Черновик).
tags : [lessons, csharp, fsharp, monad]
---

## {{page.title}} [ENG]({{ site.url }}/2014/07/02/2014-07-02-way-to-computation-expressions.html)

<p class="meta">02 July 2014 &#8211; Karelia</p>

*Русский вариант поста не исправляется, и немного устарел, поэтому если вы видите явную ошибку, то гляньте [английскую версию]({{ site.url }}/2014/07/02/2014-07-02-way-to-computation-expressions.html) вернее всего там она исправлена. У меня просто нет времени на синхронизацию исправлений между двумя версиями.*

В предыдущем [посте]({{ site.url }}/2014/06/21/yet-another-monad-guide-rus.html) мы описали, в кратце, что такое монады и как их можно представить в c#. Теперь я предлагая посмотреть что мы можем с ними сделать на практике.
Итак монады позволяют описывать нам некоторые абстрактные вычисления строка за строкой и запускать их поверх любых контейнерных типов которые поддерживают наш IMonad интерфейс. 
Также в c# мы можем использовать синтаксицеский сахар который позволяет писать такие вычисления в более удобной форме.
По сути конкретная монада это вычислитель, а заданные вычисления это программа для нее. Отсюда сразу вопрос, а является ли наш монадический вычислитель тюринг полным? Тоесть сможем ли мы закодировать любой возможный алгоритм на этом монадическом языке вычислений? И ответ да. Без этого свойства монада IO в языке Хаскель не могла бы имитировать любой алгоритм с сайд эффектами и лишила бы Хаскелл тюринг полноты. Но перейдем от слов к делу.
Давайте попробуем описать как мы можем представить ветвление и циклы для нашего интерфеса IMonad.


{% highlight csharp %}
public static class TuringMonad
{
	public static TR IfM<T,TM, TR> (this IMonad<bool,TM> m, Func<TR> x, Func<TR> y)
		where TR:TM, IMonad<T,TM>
	{
		return m.Bind (a => a ? x () : y ()).CastM<T,TR, TM> ();
	}

	public static TR WhileM<T,TM, TR> (this TR m, Func<T, IMonad<bool,TM>> p, Func<T, TR> step)
		where TR : TM, IMonad<T,TM>
	{
		var res = m.Bind (x => p (x).IfM<T,TM,TR> (() => step (x).WhileM (p, step), () => m.Return<T> (x).CastM<T,TR, TM> ()));
		return res.CastM<T,TR, TM> ();
	}

	public static TR AndM<TM, TR> (this TR x, TR y)
		where TR:TM, IMonad<bool,TM>
	{
		return x.IfM<bool,TM,TR> (() => y, () => x.Return (false).CastM<bool,TR, TM> ());
	}

	public static TR OrM<TM, TR> (this TR x, TR y)
		where TR:TM, IMonad<bool,TM>
	{
		return x.IfM<bool,TM,TR> (() => x.Return (true).CastM<bool,TR, TM> (), () => y);
	}

	public static TR NotM<TM, TR> (this TR m)
		where TR:TM, IMonad<bool,TM>
	{
		var res = m.Bind (x => m.Return (!x));
		return res.CastM<bool,TR, TM> ();
	}
}

{% endhighlight %}
Теперь мы можем выразить практически любой поток исполнения для монад. Давайте для примера расширим наш последний пример для монады Async < Check < > >.
{% highlight csharp %}
//class representation of void result
public class Unit
{
	Unit ()
	{
		
	}

	public static Unit Value = new Unit ();

	public static Unit Start ()
	{
		return Value;
	}
}
//...
public class MainClass{
	//... 

	//helper methods
	static CheckForT<Async>.CheckT<T> Return<T> (T val)
	{
		Func<T,Task<T>> f = t => Task<T>.FromResult (t);
		return Lift (() => f (val)) ();
	}

	static T Head<T> (List<T> lst)
	{
		return lst [0]; 
	}

	static List<T> Tail<T> (List<T> lst)
	{
		lst.RemoveAt (0);
		return lst; 
	}
	//...
	//main 
	static void Main (string[] args)
	{
		//expected Check<Test> but Check<Check<Test>>
		Func<string, CheckForT<Async>.CheckT<string>> getData = addr => Lift (() => GetData (addr)) ();
		Func<string, CheckForT<Async>.CheckT<Unit>> print = str => { 
			Console.WriteLine (str);
			return Return (Unit.Value);
		};
		Func<List<String>, CheckForT<Async>.CheckT<List<String>>> checkHeadAndReturnTail = 
			adressList => {
				var res = from address in Return (Head (adressList))
				          //from unit in print ("Checking address:" + address)
			from response in getData (Head (adressList))
				          //from unit2 in print ("Checked address:" + address)
				select Tail (adressList);
				return res.CastM<List<String>, CheckForT<Async>.CheckT<List<String>>,CheckForT<Async>> ();
			};

		Func<CheckForT<Async>.CheckT<List<string>>, IMonad<Unit, CheckForT<Async>>> whileLoop = 
			addrsM => from a in addrsM.WhileM<List<string>, CheckForT<Async>, CheckForT<Async>.CheckT<List<string>>> (
				              addrsUnwrapped => Return (addrsUnwrapped.Count > 0), checkHeadAndReturnTail)
			          select Unit.Value;

		var res2 = 
			from addrs in Return (new List<string>{ "http://google.com", "http://yandex.ru" })
			let addrsM = Return (addrs)
			let predicate = Return (addrs.Count == 2)
			from r in predicate.IfM<Unit, CheckForT<Async>, CheckForT<Async>.CheckT<Unit>> (
				    () => whileLoop (addrsM).CastM<Unit, CheckForT<Async>.CheckT<Unit>, CheckForT<Async>> (), 
				    () => Return (Unit.Value))
			select r;
			
		var result = 
			res2
			.CastM<Unit, CheckForT<Async>.CheckT<Unit>,CheckForT<Async>> ()
			.Value
			.CastM<CheckedVal<Unit>, Async.AsyncM<CheckedVal<Unit>>,Async> ()
			.RunSync ();
		if (result.IsFailed)
			Console.WriteLine ("Fail!");
		else
			Console.WriteLine ("Success!");
		Console.ReadLine ();
	}
}
{% endhighlight %}
Now we can express any possible alghorithm for our monads. But it is very difficult to read and unerstand. It would be great to have some additional support from linq to add some additional syntatic shugar. Lets check which keywords we can use in linq queries:
Where, Select, SelectMany, Join, GroupJoin, OrderBy, OrderByDescending, ThenBy, ThenByDescending, GroupBy, and Cast. 
With type signatures it looks like this([more info](http://msdn.microsoft.com/en-us/library/bb308966.aspx#csharp3.0overview_topic19)):
{% highlight csharp %}
delegate R Func<T1,R>(T1 arg1);
delegate R Func<T1,T2,R>(T1 arg1, T2 arg2);
class C
{
   public C<T> Cast<T>();
}
class C<T>
{
   public C<T> Where(Func<T,bool> predicate);
   public C<U> Select<U>(Func<T,U> selector);
   public C<U> SelectMany<U,V>(Func<T,C<U>> selector,
      Func<T,U,V> resultSelector);
   public C<V> Join<U,K,V>(C<U> inner, Func<T,K> outerKeySelector,
      Func<U,K> innerKeySelector, Func<T,U,V> resultSelector);
   public C<V> GroupJoin<U,K,V>(C<U> inner, Func<T,K> outerKeySelector,
      Func<U,K> innerKeySelector, Func<T,C<U>,V> resultSelector);
   public O<T> OrderBy<K>(Func<T,K> keySelector);
   public O<T> OrderByDescending<K>(Func<T,K> keySelector);
   public C<G<K,T>> GroupBy<K>(Func<T,K> keySelector);
   public C<G<K,E>> GroupBy<K,E>(Func<T,K> keySelector,
      Func<T,E> elementSelector);
}
class O<T> : C<T>
{
   public O<T> ThenBy<K>(Func<T,K> keySelector);
   public O<T> ThenByDescending<K>(Func<T,K> keySelector);
}
class G<K,T> : C<T>
{
   public K Key { get; }
}
{% endhighlight %}
As we can see there is no direct representations for if and loop because linq queries was build mainly as a way to describe semantics of a language that is similar to relational and hierarchical query languages. And for some situations it is ok, but not for describing semantics of an imperative language 
Lets sum up our observations.
1. We can build monads and monad transformers in c#.
2. Limitations of representation for types in CLR forces us to use the Single Inheritance hack which can be a cause of errors.
3. Usage of monads transformers can be difficult. Types like CheckT<AsyncM<CheckM<T>>> is not what we really need.
4. In c# we have support for describing semantics of languages that is similar to relational and hierarchical query languages.

And what would be great to have?
1. Express monads in a simple way.
2. Have syntatic support for imperative languages. 
3. Have better way to dispatch current monad into computation. Like setting named scope. Would be great to have this named scopes to be first class values.

Fortunately for us, we already have all of that in clr and this is computation expressions of fsharp language. What is a computation expression? It is a simple builder class which defines some methods(like we defined SelectMany in query expressions). After that we can use instance of this class as a name for a scope in which we can use all syntax alowed by computation expressions and it will be translated into method calls of this instance. Lets check simple example.
{% highlight fsharp %}
type MaybeBuilder() =
    member this.Bind(x, f) = 
        match x with
        | None -> None
        | Some a -> f a

    member this.Return(x) = 
        Some x

let maybe = new MaybeBuilder()

let divideBy bottom top =
    if bottom = 0
    then None
    else Some(top/bottom)

let divideByWorkflow init x y z = 
    maybe 
        {
        let! a = init |> divideBy x
        let! b = a |> divideBy y
        let! c = b |> divideBy z
        return c
        }    
{% endhighlight %}
This code is a copy from the great "Computation Expressions" series, I strongly recommend you to read it on a site [Fsharp for fun and profit](http://fsharpforfunandprofit.com/posts/computation-expressions-intro/). In computation expressions we could use syntax which are very close to fsharp, but has different semantics defined by a builder. We can use limited set of keywords like let! from previous example which is good enought to express any possible imperative workflow. List of possible constructs includes for loops, try catch blocks, let and do bindings. All predefined keywords maps into invocations of methods defined in builder: Bind, Delay, Return, ReturnFrom, Run, Combine, For, TryFinally,TryWith, Using, While, Yield, YieldFrom, Zero. 
For example if we define While method in our a builder then we could use while loops inside the builder scope.
{% highlight csharp %}
some { 
    while predicate() do
        action() 
} 
{% endhighlight %}
Also we can pass builders as parameters because they are first class sitizens.
{% highlight csharp %}
let divideByWorkflow (workflow:MaybeBuilder) init x y z = 
    workflow 
        {
        let! a = init |> divideBy x
        let! b = a |> divideBy y
        let! c = b |> divideBy z
        return c
        }    
{% endhighlight %}
 And one more thing, we can add our own keywords as a result  we can define not only new semantics for existing keywords but also can add new ones. [Example from](http://stackoverflow.com/questions/9272285/f-is-there-a-way-to-extend-the-monad-keyword-list/9275289#9275289)
{% highlight csharp %}
//sample from 
type SeqBuilder() =
    // Standard definition for 'for' and 'yield' in sequences
    member x.For (source : seq<'T>, body : 'T -> seq<'R>) =
      seq { for v in source do yield! body v }
    member x.Yield item =
      seq { yield item }

    // Define an operation 'select' that performs projection
    [<CustomOperation("select")>]
    member x.Select (source : seq<'T>, [<ProjectionParameter>] f: 'T -> 'R) : seq<'R> =
        Seq.map f source

    // Defines an operation 'reverse' that reverses the sequence    
    [<CustomOperation("reverse", MaintainsVariableSpace = true)>]
    member x.Expand (source : seq<'T>) =
        List.ofSeq source |> List.rev

let mseq = SeqBuilder()
{% endhighlight %}
Computation expressions in fsharp meets all our needs from the list. Now we have possibility to hide all boilerplate code in builders and inside this builder write a code that doesnt contains technical details and could be as generic as possible. Builders allows us to define new llanguage with custom semantics and syntax. And you should understand that computation expresiions is not a monadic shuga like do notation in haskell it is a way to express new language inside our fsharp code. For example do notation in haskell often forces people to use monad insted of more simple cunstructs like monoid(we will see what it is in following posts) just to use syntatic shuga. Do notations is always turing complete but what if we want to build some little not turing complete language for example total language which has interesting property that for program written in that language termination is guaranteed? Do notation cant do tht but computation expressians can. Computation expression has it's own limitation. It's not polymorphic. Our workaround will not work in fsharp due to limitations of fsharp type system (F# does not allow type constraints involving two different type parameters). As a result we can't express polymorphic monads and monad transformers in fsharp but usually it is not a prolem at all. We don't need to compose different monads like in haskell where we have IO monad everywhere and it is not a problem to build specialized builder which will combine two other builders in a much more efficien way that monat transformers and without crayzy type signatures. You can see as example [AsyncSeq](http://tomasp.net/blog/async-sequences.aspx/) and [Update](http://tomasp.net/blog/2014/update-monads/) builders created by Tomas Petricek. 
Conclusion.
Computation expressions is a perfect way to express some domain specific language in F#. We can define new syntax and semantics. One note is that we should use operational semantics(what language keywords must do) but not denotational semantics(how keywords can must be translated to other language). if you need to describe denotational semantics for program in fsharp, for example some fsharp to js translator then you should look at another feature of fsharp "code quotations". 

