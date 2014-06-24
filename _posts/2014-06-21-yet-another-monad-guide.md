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
% highlight csharp %}
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
