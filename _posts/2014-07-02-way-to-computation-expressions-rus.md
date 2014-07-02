---
published: true
layout: post
title: Путь к computation expressions(Черновик).
tags : [lessons, csharp, fsharp, monad]
---

## {{page.title}} [ENG]({{ site.url }}/2014/07/02/way-to-computation-expressions.html)

<p class="meta">02 July 2014 &#8211; Karelia</p>

*Русский вариант поста не исправляется, и возможно немного устарел, поэтому если вы видите явную ошибку, то гляньте [английскую версию]({{ site.url }}/2014/07/02/way-to-computation-expressions.html) вернее всего там она исправлена. У меня просто нет времени на синхронизацию исправлений между двумя версиями.*

В предыдущем [посте]({{ site.url }}/2014/06/21/yet-another-monad-guide-rus.html) мы описали, кратко, что такое монада и как мы ее можем представить в C#. Теперь давайте рассмотрим проблемы применения нашей реализации монад и опишем приемлемое решение. Итак монады позволяют нам описывать некие асбтрактные вычисления линия за линией и потом запускать их поверх контейнерных типов которые имплементируют наш IMonad интерфейс. Также в c# мы можем использовать синтаксический сахар, который позволяет нам выражать наши монадические вычисления в более приемлемом виде. По сути монада это компьютер, и наши абстрактные монадические вычисления это описание программы для него. Это определнно так, но мы можем задаться вопросом: являются ли наш монадический компьютер Тюринг полным? Можем ли мы выразить любой алгоритм на этом монадическом языке? И ответ да. Без этого свойства IO монада в Haskell не могла бы имитировать любой возможный путь исполнения с сайд эффектами. Но достаточно слов, давайте подтвердим это в коде. Давайте попробуем описать обычные конструкции для управление путем исполнения: "if else" and "while do" для нашего интерфейса IMonad и определим их в терминах методов bind и return.
{% highlight csharp %}
public static class TuringMonad
{
	public static TR IfM<T,TM, TR> (this IMonad<bool,TM> m, 
										 Func<TR> x, 
										 Func<TR> y)
		where TR:TM, IMonad<T,TM>
	{
		return m.Bind (a => a ? x () 
							  : y ()).CastM<T,TR, TM> ();
	}

	public static TR WhileM<T,TM, TR> (this TR m, 
									   Func<T, IMonad<bool,TM>> p,
									   Func<T, TR> step)
		where TR : TM, IMonad<T,TM>
	{
		var res = m.Bind (x => 
			p (x).IfM<T,TM,TR> (() => 
				step (x).WhileM (p, step), () => 
					m.Return<T> (x).CastM<T,TR, TM> ()
			)
		);
		return res.CastM<T,TR, TM> ();
	}

	public static TR AndM<TM, TR> (this TR x, TR y)
		where TR:TM, IMonad<bool,TM>
	{
		return x.IfM<bool,TM,TR> (
								() => y, 
								() => x.Return (false)
									   .CastM<bool,TR, TM> ()
				);
	}

	public static TR OrM<TM, TR> (this TR x, TR y)
		where TR:TM, IMonad<bool,TM>
	{
		return x.IfM<bool,TM,TR> (
			() => x.Return (true).CastM<bool,TR, TM> (), 
			() => y
		);
	}

	public static TR NotM<TM, TR> (this TR m)
		where TR:TM, IMonad<bool,TM>
	{
		var res = m.Bind (x => m.Return (!x));
		return res.CastM<bool,TR, TM> ();
	}
}

{% endhighlight %}
Теперь мы можем описать любой поток исполнения для IMonad интерфейса. Давайте для примера расширим наш последний пример для монады Async < Check < > >.
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
		Func<string, CheckForT<Async>.CheckT<string>> getData =
			addr => Lift (() => GetData (addr)) ();
		Func<string, CheckForT<Async>.CheckT<Unit>> print = 
			str => { 
				Console.WriteLine (str);
				return Return (Unit.Value);
			};
		Func<List<String>, CheckForT<Async>.CheckT<List<String>>> 
			checkHeadAndReturnTail = 
			adressList => {
				var res = 
					from address in Return (Head (adressList))
				    //from unit in print ("Checking address:" + address)
					from response in getData (Head (adressList))
				    //from unit2 in print ("Checked address:" + address)
					select Tail (adressList);
				return res.CastM<List<String>, 
								 CheckForT<Async>.CheckT<List<String>>,
								 CheckForT<Async>> ();
			};

		Func<CheckForT<Async>.CheckT<List<string>>, 
			IMonad<Unit, CheckForT<Async>>> whileLoop = 
				addrsM => 
					from a in addrsM.WhileM<List<string>, 
											CheckForT<Async>,
											CheckForT<Async>
											.CheckT<List<string>>> (
				              addrsUnwrapped => 
				              	Return (addrsUnwrapped.Count > 0),
				              	checkHeadAndReturnTail
				    )
			        select Unit.Value;

		var res2 = 
			from addrs in Return (
				new List<string>{ "http://google.com", "http://yandex.ru" })
			let addrsM = Return (addrs)
			let predicate = Return (addrs.Count == 2)
			from r in predicate.IfM<Unit, 
									CheckForT<Async>,
								 	CheckForT<Async>
								 	.CheckT<Unit>> (
				    () => whileLoop (addrsM)
				    		.CastM<Unit, 
				    			   CheckForT<Async>.CheckT<Unit>,
				    			   CheckForT<Async>> (), 
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
Мы доказали, что мы можем описать любой алгоритм для наших монад. Но к сожалению этот код черезвычайно тяжело читать и понимать. Было бы прекрасно если бы мы могли использовать linq query для добавления синтаксического сахара. Давайте глянем на список возможных ключевых слов доступных для определния в linq queries:
Where, Select, SelectMany, Join, GroupJoin, OrderBy, OrderByDescending, ThenBy, ThenByDescending, GroupBy, and Cast. 
Полный список методов, с сигнатурами типов, которые должны быть реализованы  выглядит ([так](http://msdn.microsoft.com/en-us/library/bb308966.aspx#csharp3.0overview_topic19)):
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
Как можно видеть, здесь нет ключевых слов для прямого представления конструкций ветвления и цикла, так как изначально linq queries были созданы для описания семантики языков которые похожи на языки реляционных или иерархических запросов. И это нормально, но не для описания семантики императивного языка. 

Давайте суммируем наши наблюдения.

1. Мы можем выражать монады и монад трансформеры в c#.
2. Ограничения представления типов в CLR заставляет нас использовать паттерн "Единственный наследник" который в принципе может породить трудно отслеживаемые баги.
3. Использование монад трансформеров довольно проблематично и неэффективно. Типы наподобии CheckT<AsyncM<CheckM<T>>> не совсем то, что мы хотим видеть в нашем коде каждый день, просто представьте если мы обьеденим 3 или более монады в одну, это будет просто невозможно поддерживать.
4. В c# мы имеем поддержку для описания семантики для реляционных или иерархических языков запросов.

А что бы нам хотелось иметь?

1. Просто выражать монады и вычисления над ними.
2. Иметь синтаксическую поддержку для разных типов языков, включая императивные, в монадических вычислениях. 
3. Иметь лучший чем в linq запросах путь диспатчить вычисления в монаду. Что то что позволило бы нам отвязать монаду от типа контейнера, что то похожее на именованный регион. Также было бы прекрасно чтобы эти регионы были обьектами первого класса..

К счастью для нас, мы уже имеем все из перечисленного в дот нет и это "computation expressions" языка fsharp. Что же это такое? Это простой класс построитель в котором определены некоторые методы (наподобие SelectMany который мы определяли в linq). После этого мы можем использовать экземпляры этого класса, как имя для региона в котором мы можем использовать весь синтксис допустимый в "computation expressions" и этот синтаксис будет оттранслирован в выховы методов этого экземпляра. Давайте посмотрим на простой пример.
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
Этот код взят из прекрасной серии постов "Computation Expressions", я очень рекомендую вам прочитать материал целиком на сайте [Fsharp for fun and profit](http://fsharpforfunandprofit.com/posts/computation-expressions-intro/). В "computation expressions" мы можем использовать синтаксис, который очень близок к fsharp, но имеет другую семантику определенную классом построителем. Мы можем использовать набор ключевых слов, наподобии let! с предыдущего примера, которые достаточны для выражения любого возможного пути исполнения императивной программы. Список возможных конструкций включает в себы циклы, try catch блоки, let do биндинги и многое другое. Все ключевые слова транслируются на вызовы соответствующих методов определенных в билдер классе, вот стандартный список методов: Bind, Delay, Return, ReturnFrom, Run, Combine, For, TryFinally,TryWith, Using, While, Yield, YieldFrom, Zero. 
Для примера, если мы определим метод While в нашем билдере, тогда мы сможем использовать циклы в регионе маркированным экземпляром этого билдера.
{% highlight fsharp %}
some { 
    while predicate() do
        action() 
} 
{% endhighlight %}
И мы можем передавать билдеры как обыкновенные значения, так как по сути они являются простыми обьектами.
{% highlight fsharp %}
let divideByWorkflow (workflow:MaybeBuilder) init x y z = 
    workflow 
        {
        let! a = init |> divideBy x
        let! b = a |> divideBy y
        let! c = b |> divideBy z
        return c
        }    
{% endhighlight %}
 И еде одно, мы можем определять не только новую семантику для существующих ключевых слов, но и добавлять новые ключевые слова. [Например](http://stackoverflow.com/questions/9272285/f-is-there-a-way-to-extend-the-monad-keyword-list/9275289#9275289)
{% highlight fsharp %}
//sample from 
type SeqBuilder() =
    // Standard definition for 'for' and 'yield' in sequences
    member x.For (source : seq<'T>, body : 'T -> seq<'R>) =
      seq { for v in source do yield! body v }
    member x.Yield item =
      seq { yield item }

    // Define an operation 'select' that performs projection
    [<CustomOperation("select")>]
    member x.Select (source : seq<'T>, 
    				 [<ProjectionParameter>] f: 'T -> 'R) 
    				 : seq<'R> =
        Seq.map f source

    // Defines an operation 'reverse' that reverses the sequence    
    [<CustomOperation("reverse", MaintainsVariableSpace = true)>]
    member x.Expand (source : seq<'T>) =
        List.ofSeq source |> List.rev

let mseq = SeqBuilder()
{% endhighlight %}
"Computation expressions" в fsharp удовлетворяют все наши потребности из списка желаний. Теперь мы имеем возможность скрывать все нежелательные детали в билдере, а внутри вычислений писать астрактный код который не содержит ненужных подробностей. Построители позволяют нам определять новые языки с необходимой семантикой и синтаксисом. Мы должны четко понимать, что "computation expressions" не просто монадический синтаксический сахар, наподобии do нотации в Haskell, а путь для описания новых языков внутри fsharp кода. Для примера Do нотация в Haskell часто застваляет программистов использовать монаду, вместо более простых конструкций наподобии моноидов(мы рассмотрим их в следующих постах), просто для того, чтобы иметь возможность использовать синтаксический сахар. Do нотация всегда Тюринг полня, но что если мы хотим построить небольшой язык который не является Тюринг полным? Для примера тотальный язык оторый имеет интересное свойство, что любая программа написанная на нем всегда гарантированно останавливается? Безопасные языки как правило не полны по Тюрингу. Do нотация этого не может, но "computation expressions" могут. "Computation expressions" имеют свои собственные пролемы. Они не полиморфны. Наш костыль с наследованием не будет работать в fsharp из за ограничений системы типов fsharp (F# не позволяет описывать ограничения генерик типов которые включают два разных параметра типа). Как результат, мы не можем выразить полиморфные монады и монад трансформеры в fsharp, но в общем случае это не является серьезной проблемой. В fsharp у нас нет необходимости так часто создавать композиции монад, как например в Haskell, где у нас есть IO монада практически везде, и построить споциализированный построитель который будет являтеся композицией двух других обычно не сверх сложная задача и он будет гораздо быстрее, чем идентичная ему монада построенная с помощью монад трансформера и у нас не будет проблем с сумасшедшими сигнатурами типов. Вы можете взглянуть на [AsyncSeq](http://tomasp.net/blog/async-sequences.aspx/) и [Update](http://tomasp.net/blog/2014/update-monads/) построители созданыые Tomas Petricek-ом. Они являются прекрасными примерами композиции построителей. "Computation expressions" это хороший путь для выражения языков ориентированных на определнную область(internal DSL). Мы можем определять новый синтаксис и семантику. Однако отметим, что мы должны использовать операционную семантику(что ключевые слома должны делать), а не денотационную семантику(как ключевые слова должны быть переведены на другой язык). Если вам действительно необходимо выразить денотационную семантику, например построить транслятор fsharp кода в JavaScript, тгда вым лучше использовать другую фичу fsharp "цитирование кода". 