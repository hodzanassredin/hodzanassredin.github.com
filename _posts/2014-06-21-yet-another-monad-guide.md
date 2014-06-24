---
published: false
layout: post
title: Monads, a little bit of higher kinded types and monad transforers for mere mortals.
tags : [lessons, csharp]
---

## {{page.title}}

<p class="meta">24 June 2014 &#8211; Karelia</p>

В процессе переписи небольшого проекта на fsharp я задумался на тему использования монад на практике. Тема меня достаточно увлекла и я решел описать свой опыт в паре другой постов. К сожалению весь материал требовал неплохих знаний о монадах и монад трансформерах но в тоже время материал хотелось сделать цельным, самодосточным,без кучи слов и простым для обычных программистов без знаний математики и синтаксиса функциональных языков программирования.
Так родилась идея написать еще одно обьяснение что же такое монады.
Так как пост ориентирован на программистов то и будем подходить не со стороны обьяснений или математика а со стороны кода. Готовы? Поехали.

Давайте рассмотрим простой код:

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

Функция GetData имитирует запрос к источнику данных и случайным образом возвращает null или значение.
Функция Compare использует функцию GetData для получения двух значений, сравнивает их между собой и пишет ответ на консоль.
Что мы как ответственные программисты хотим добавить в этот код первым делом? Правильно проверку на null в стиле defensive programming.

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

Да программа стала безопасней но видно повторяющийся код. Берем на вооружение DRY принцип и убираем повторы.

% highlight csharp %}
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

Стало немного лучше но все равно после каждого выхова надо добавлять проверку if. Давайте попробуем убрать управление потоком исполнения в функцию Defend.

% highlight csharp %}
		public static string Defend2 (object a, Func<object, string> f)
		{
			return a == null ? "Can't compare" : f (a);
		}
 
		public static void DefesiveCompareDry2 (string[] args)
		{
			var res = Defend2 (GetData<Object> (), 
				          (a) => Defend2 (GetData<Object> (), 
					          (b) => a == b ? "a == b" : "a != b"));
 
			Console.WriteLine (res);
			Console.WriteLine ("finished!");
		}
{% endhighlight %}

Выглядит замечательно но есть проблема в том что если мы будем спользовать как значение например тип Test то информация о нем пропадет и мы не сможем использовать его свойства и методы.

% highlight csharp %}
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
			Defend2(new Test(), a=>a.Text());// System.Object does not contain a defenition for Text
		}
{% endhighlight %}
Попробуем обойти эту проблемы с помощью переменных типа. 
% highlight csharp %}
		public static T Defend2Generic<T> (T a, Func<T, T> f)
			where T: class//applied only for TA which can be null
		{
			return a == null ? "Can't compare" : f (a);
		}
{% endhighlight %}
Хм новая ошибки дело в том что мы пытаемся вернуть строку вместо типа Test. Что делать, а давайте создадим тип Check<T> который будет содежрать или значение или строку с ошибкой.
% highlight csharp %}
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
				return string.Format ("[Check: IsFailed={0}, FailMesssage={1}, Value={2}]", IsFailed, FailMesssage, Value);
			}
		}
 
		public static Check<TB> Defend<TA,TB> (TA a, Func<TA, TB> f)
					where TA : class
					where TB : class//applied only for TB and TA which can be null
		{
			return a == null ? new Check<TB> ("Can't compare") : new Check<TB> (f (a));
		}
{% endhighlight %}
Вроде выглядит красиво. Попробуем в деле.
{% highlight csharp %}
		public static void DefesiveCompareGeneric2 (string[] args)
		{
			//expected Check<Test> but Check<Check<Test>>
			Check<Test> res = Defend2Generic2 (GetData<Test> (),
				a => Defend2Generic2 (GetData<Test> (),
					          b => a == b ? a : b));

			Console.WriteLine (res);
			Console.WriteLine ("finished!\n");
		}
{% endhighlight %}
Опять проблема вместо того чтобы вернуть в res тип Check<Test> туда приходит типа Check<Check<Tese>>. Попробуем изменить немного логику работы чтобы предотвратить эту ошибку.
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
				return string.Format ("[Check: IsFailed={0}, Value={2}]", IsFailed, Value);
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
Теперь проверка на нулл происходит в функции лифт задача которой преобразовать функциюю возвращающуу T в фукцию возвращающую Check<T>. Проблема решена. Можно думать об этом так есть обычные функции и есть наша функция дефенд. Но для того чтобы фукция дефенд могла работаь с данными фукциями их надо адаптировать. Вот как раз адаптацией фукция lift и занимается. Проверяем.
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
Опять проблема концовка вида b => a == b ? a : b возвращает тип Test тоесть она не адаптирована к фукции  Defend. Напишем простую вспомогающую функция Return которую будем дергать в самом конце и немного отрефакторим код.
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
				return string.Format ("[Check: IsFailed={0}, Value={1}]", IsFailed, Value);
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
 
		public static void DefesiveCompareGeneric4 (string[] args)
		{
			//expected Check<Test> but Check<Check<Test>>
			var getData = Lift2<Test> (GetData<Test>);
			Check3<Test> res = Defend<Test,Test> (getData (), 
				                   a => Defend<Test,Test> (getData (), 
					                   b => Return (a == b ? a : b)));
 
			Console.WriteLine (res);
			Console.WriteLine ("finished!");
		}
{% endhighlight %}
Ура все заработало! Итак что мы имеем: тип обертку Check над  T, две функции Defend, Return и все это вместе позволяет нам писать защищенный код без кучи if проверок. На самом деле мы можем определить другой враппер класс и описать функцию return и defend над ним и получить довольно интересные возможности по исполнению dry кода в цепочке функций. На самом деле возможностей у этого паттерна намного больше и он давно известен под названием монада вот только функция defend там называется bind. 
Вот есть только один недостаток очень уж неудобно записывать подобным образом код в виде вложенных функций. К счастью решение есть в некоторых языках есть поддержка монадического синтаксиса например в csharp это linq queries, в Haskell Do нотация, в fsharp computation expressions. К слову сказать возможности computation expressions выходят далеко за пределы монад но об этом в другой раз. Давайте попробуем адаптировать наш код под linq expressions для этого нам надо над нашим враппер типом описать статическую екстеншн функцию SelectMany. Экстеншн функция это статическая функция которую можно подцепить к существующему классу.
Давайте ее определим и заодно отрефакторим код:

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
			return string.Format ("[Check: IsFailed={0}, Value={1}]", IsFailed, Value);
		}
	}
 
	public static class CheckMonad
	{
		public static Check<T> Return<T> (this T value)
		{
			return Check<T>.Success (value);
		}
 
		public static Check<U> Bind<T, U> (this Check<T> m, Func<T, Check<U>> k) //bind
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
Вот теперь все в порядке. И подобным образом мы можем определить другие монады например async которая позволит нам выполнять наш код асинхронно(мы ее напишем чуть позже). Мир чудесен. Но не в нашем туториале. Одно из достоинств монад в том что описав код над монадой мы можем его запускать поверх других монад до тех пор пока монада внутри себя проносит тот же тип. Например сравните код для монады Async
{% highlight csharp %}
			var getData = AsyncMonad.Lift (GetData);
			var res = 
				from a in getData ()
				from b in getData ()
				select a.Substring (0, 10) + b.Substring (10, 20);
{% endhighlight %}
и монады Check
{% highlight csharp %}
			var getData = CheckMonad.Lift (GetData);
			var res = 
				from a in getData ()
				from b in getData ()
				select a.Substring (0, 10) + b.Substring (10, 20);
{% endhighlight %}
Очень круто но есть проблема с композицией монад. Допестим хотелось бы нам работать с функциями которые возвращают Async<Check<T>> с нашим красивым синтаксисом. Однако наши фунции Bind в типе Async ничего не знают о вложенном типе Check поэтому наш красивый код работать не будет наша функция bind развернет только async и вернет Сheck<T> вместо T. Вот тут в дело вступают монад трансформеры. Что такое монад трансформер? это по сути штука которая беря на вход неизвестную монаду добавляет к ней функциональность известной и в результате получаем комбинированную монаду. Допустим в нашем случае к монадам Async<T> и Check<T> которые мы не можем использовать вместе мы можем написать монад трансформеры AsyncT<T,ParantMonad> и CheckT<T, ParentMonad> и для нашего случая Async<Check<T>> мы спокойно можем сделать что то типа такого:
{% highlight csharp %}
var getData = CheckT<T, Async<T>>.LiftT (AsyncMonad.Lift (GetData));
			var res = 
				from a in getData ()
				from b in getData ()
				select a.Substring (0, 10) + b.Substring (10, 20);
{% endhighlight %}
Обычно все туториалы в этом месте оканчиваются словами о том что подбные штуки есть в Хаскелл и Скала, но в C# нету higher kinded types и тут их реализовать невозможно. Занавес.
Но мы то прожженные SharePoin энтерпрайз разработчики мы не знаем слов любви и жалости. Заказчику НАДО значит надо. Ок давайте посмотрим на эту проблему поближе. Мы разберем типичный пример. На самом деле он не показателен так как эту проблему можно обойти несколькими способами но мы решим ее механизмом который позволит в дальнейшем решить проблему трансформеров. Итак есть интерфейс:
{% highlight csharp %}
	interface IFunctor<T> {
		T<B> FMap<A, B>(Func<A, B> f, T<A> a);
	}
{% endhighlight %}
На первый взгляд ничего подозрительного, у нас есть контейнерный тип и далее мы определяем сигнатуру требуемого метода который применяет функцию над типом A к завернутому в T типу A и оборачивает результат функции типа B в контейнерный тип T. Все хорошо за искоючением того что в c# нельзя задать подобный тип T. Как видно из кода тип T является генериком T<_> и тут то начинются проблемы. Я не буду обьяснять суть проблемы тут вам проще взять и скопировать определение функтора данные выше и попробовать решить проблему самим. Это отличная головолмка. И сразу становиться все ясно.
Попроовали? Решили? Отлично. Теперь решим вместе.
В чем суть типа T в нашем интерфейсе? Зачем он нужен а нужен он для того чтобы пометить входящий и выходящий контейнеры одним маркером и наложить ограничение на то что они должны быть одним и темже генерик типом. Чтобы не получилось что на входе List<A> а на выходе Check<B>. Ок задача ясна пометить генерик типа негенерик типом. Как сделать? Да просто:
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
Хм интересно. Давайте подумаем какие гарантию нам это дает. То что инстанс типа Wrapper всегда будет инстансом типа WrapperImpl единственное в чем надо быть уверенными так это в том что при апкасте мы не промажем с типом сдержащимся в WrapperImpl. Давайте введем специальный тип контейнер для информации о генерик типе и немного перепишем определение враппер типа. 
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
Вуаля ловкость пальцев и никакого обмана. Единственное условие это следовать петтерну одиночного наследника, чтобы одним маркером нельзя было маркировать несколько классов. Давайте соберем все вместе и посмотрим работает ли.
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
 
		public class WrapperImpl<T> : Wrapper, IGeneric<T, Wrapper>, IFunctorSelf<Wrapper, WrapperImpl<T> , T>
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
Теперь у нас есть все чтобы для начала определить интерфейс для монад и наслждаться полиморфным кодом над монадами и получить возможность их композиции.
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
На данный момент в коде должно быть все понятно, мы взяли идею из функтора применили к интерефейса для монад который содержит определение методов bind и return. Теперь мы можем переписать Check монаду на основе этого интерфейса и реализовать Async монаду. Async монада может послужить пример того как можно сделать монадическую обертку над существующим генерик типом который не имплементирует интерфейс монады. В нашем случае Async<T> это просто обертка над типом Task<T> и можно думать что это адаптер типа Task к монадическому интерфейсу.
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
				return string.Format ("[Check: IsFailed={0}, Value={1}]", IsFailed, Value);
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
				return res == null ? Check.CheckM<TB>.Fail () : Check.CheckM<TB>.Success (res);
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
			private static async Task<TB> BindTasks<TB> (Task<T> m, Func<T, Task<TB>> f)
			{
				var r = await m;
				return await f(r);
			}
 
			public IMonad<TB, Async> Bind<TB> (Func<T, IMonad<TB, Async>> f)
			{
				return new AsyncM<TB>(BindTasks(this.Task, (t) => f(t).CastM<TB, AsyncM<TB>, Async>().Task));
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
			return new WebClient().DownloadStringTaskAsync(new Uri("http://google.com"));
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
Удивительно я бы сказал, но оно работает. Итак полиморфные монады у нас в кармане, теперь приступим к трансформерам. Итак есть тип Async<Check<T>> который является монадой Async над типом Check<T>, но нам бы хотелось его превратить в монаду Async<Check<_>> над типом T. Как это сделать? Все просто надо завернуть Async<Check<T>> в монаду над типом T. Допустим назовем этот тип трансформером и для Check монады назовем его CheckT. В конце концов мы получим конструкцию вида CheckT<Async<Check<T>>> этакий трехслойный бутерброд. Главная фишка что этот CheckT реализует интерфейс не IMonad над Async<Check<T>>> а Imonad над T. Еще раз повторим: CheckT это враппер для типов вида SomeOtherMonad<CheckMonad<T>> и функция Lift для CheckT будет преобразовывать функции возвращающие SomeMonad<T> в функции возвращающие CheckT<SomeOtherMonad<CheckMonad<T>>>. В функции return в начале он будет звертывать значение в тип Check а потом его передавать методу return класа SomeOtherMonad и навыходе кастить его в себя. В Bind поведение чуть сложнее и вся магия происходит там но суть таже. Наверное это сложнее описать словами чем кодом. Чтож давайте реализуем CheckT. Я для облегчения понимания решил разделить определение типов CheckedVal это просто тип контейнер. Check.CheckM это монада над CheckedVal и
CheckForT<TMI>.CheckT<T> это трансформер над CheckedVal. Хотя такое разделение излишне и тип CheckedVal должен быть включен в тип Check.CheckM.
Особое внимание в коде надо обратить на функцию bind и место где описан маркер обертываемой монады TMI. Он определен в типе маркере трансформера CheckForT<TMI>.CheckT<T>, а не в типе трансформера CheckForT.CheckT<T,TMI>. Это сделано для того чтобы при реалзации методо интерфейса монады, у нас не терялся тип обернутой монады. Так мы застраховались от выхова bind на функциях которые возвращают разные обернутые и трансформированные монады. Это вызвало бы ошибку в рантайм. Итак вся магия здесь.
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
				return new CheckT<TB> (Value.Return<CheckedVal<TB>> (CheckedVal<TB>.Success (val)));
			}
 
			private IMonad<CheckedVal<TB>,TMI> BindInternal<TB> (CheckedVal<T> check, 
			                                                     Func<T, IMonad<TB, CheckForT<TMI>>> f)
			{
				return check.IsFailed 
					? Value.Return<CheckedVal<TB>> (CheckedVal<TB>.Fail ()) 
						: f (check.Value).CastM<TB, CheckT<TB>,CheckForT<TMI>> ().Value;
			}
 
			public IMonad<TB, CheckForT<TMI>> Bind<TB> (Func<T, IMonad<TB, CheckForT<TMI>>> f)
			{
				var tmp = Value.Bind<CheckedVal<TB>> (check => BindInternal (check, f));
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
 
		public static Func<Check.CheckM<TB>> Lift<TB> (this Func<CheckedVal<TB>> f)
			where TB : class
		{
			return () => {
				var res = f ();
				return new Check.CheckM<TB> (res);
			};
		}
 
		public static Func<CheckForT<TMI>.CheckT<TB>> LiftT<TB,TMI> (this Func<IMonad<TB,TMI>> f)
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
			return new WebClient ().DownloadStringTaskAsync (new Uri ("http://google.com"));
		}
 
		static void Main (string[] args)
		{
			//expected Check<Test> but Check<Check<Test>>
			var getData = CheckMonad.LiftT (AsyncMonad.Lift (GetData));
			var res = 
				from a in getData ()
				from b in getData ()
				select a.Substring (0, 10) + b.Substring (10, 20);
			var checkT = res.CastM<string, CheckForT<Async>.CheckT<string>, CheckForT<Async> > ();
			var task = checkT.Value.CastM<CheckedVal<string>, Async.AsyncM<CheckedVal<string>>, Async> ().Task;
			Console.WriteLine (task.Result);
			Console.WriteLine ("finished!");
			Console.ReadLine ();
		}
{% endhighlight %}
Опять работает. Ну добились того о чем просили коммунисты 20 века мы скомпозили 2 монады. Причем одна из них была просто оболчкой над существующим классом Task. Я надеюсь материал был не шибко запутан и мой стиль подачи вас не напугал. В следующей часте будет интересней посмотрим на отличия computation expressions от монад, обнаружим что монады являются тюринг полными. и обсудим возможности их применения в областе определения семантики для монадического синтаксиса. 
P.S. Это реализация полиморфного computation expression DO для fsharp чтобы теперь можно было писать как в Хаскелл.
{% highlight fsharp %}

{% endhighlight %}


{% highlight csharp %}

{% endhighlight %}


{% highlight csharp %}

{% endhighlight %}