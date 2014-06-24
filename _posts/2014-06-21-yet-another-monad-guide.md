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
Очень круто но есть проблема с композицией монад. Допестим хотелось бы нам работать с функциями которые возвращают Async<Check<T>> с нашим красивым синтацсисом. Однако наши фунции Bind в типе Async ничего не знают о вложенном типе Check поэтому наш красивый код работать не будет наша функция bind развернет нам только async и вернет Сheck<T> вместо T. Вот тут в дело вступают монад трансформеры. Что такое монад трансформер? это по сути штука которая беря на воход ему неизвестную монаду добавляет к ней функциональность другой и в результате получаем комбинированную монаду. Допустим в нашем случае к монадам Async<T> и Check<T> которые мы не можем использовать вместе мы можем написать монад трансформеры AsyncT<T,ParantMonad> и CheckT<T, ParentMonad> и для нашего случая Async<Check<T>> мы спокойно можем сделать что то типа такого:
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
{% highlight csharp %}

{% endhighlight %}