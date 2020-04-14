---
title: "Extension methods"
date: 2019-01-03
---


Razem z pojawieniem się C# w wersji 3.0 Microsoft wprowadził mechanizm extension method pozwalający na dodawania nowych metod do istniejących już typów bez potrzeby modyfikacji samego typu. Extension method są zwykłymi metodami statycznymi, które dzięki specjalnemu wsparciu ze strony języka mogą być wywoływane w sposób identyczny do metod instancyjnych. Jest to szczególnie przydatne gdy zależy nam na dodaniu często wykorzystywanej funkcjonalności dla typu, który nie może być przez nas zmodyfikowany.

<!--more-->

Definicja extension method odbywa się w sposób niemal identyczny do definicji statycznej metody. Jedyną różnicą jest dodanie modyfikatora `this` przed typem pierwszego parametru. To właśnie wspomniany modyfikator pozwoli nam na wywoływanie extension method w sposób jednakowy do metod instancyjnych. Jest on wskazówką dla kompilatora dla obiektów jakiego typu metoda powinna zostać dodana. Warto również dodać, że extension method może być zdefiniowana jedynie wewnątrz statycznej, niezagnieżdżonej, niegenerycznej klasy, a modyfikator `this` wstawić możemy jedynie przed pierwszym parametrem metody.

Poniżej przykładowa implementacja extension method zwracającej liczbę wyrazów w tekście.

``` csharp
public static class StringExtensions
{
	public static int NumberOfWords(this string text)
	{
		var counter = 0;
		foreach (var x in text.Split(' '))
		{
			if (!string.IsNullOrEmpty(x))
			{
				counter++;
			}
		}
		return counter;
	}
}
```

Zdefiniowaną wcześniej metodę możemy wykorzystać w sposób identyczny jak metody należące do obiektów typu `String`, podając po kropce jej nazwę. Osoba czytająca kod korzystający z extension method ma wrażenia, że metoda `NumberOfWords` należy do obiektu typu `String`.

``` csharp
var m = "Some text".NumberOfWords();
```

Należy jednak zwrócić uwagę na fakt, że extension method dodawane są do typu jedynie w określonym fragmencie kodu, nie globalnie. Oznacza to, że aby mieć do nich dostęp musimy pamiętać o użyciu odpowiedniej dyrektywy `using` w pliku.

``` csharp
using ExtensionMethods.Extensions
```

Niewątpliwą zaletą opisywanego mechanizmu, jest możliwość wykorzystania go dla dowolnego typu. Może być to klasa, interfejs, typ wartościowy lub generyczny.

``` csharp
public static class GeneralExtensions
{
	public static string SayHello(this object value)
	{
		return "Hello, I'm " + value.ToString();
	}

	public static string SayHello(this int value)
	{
		return "Hello, I'm " + value.ToString();
	}

	public static string SayHello(this IEnumerable value)
	{
		return "Hello, I'm " + value.ToString();
	}

	public static string SayHello<T>(this T value)
	{
		return "Hello, I'm " + value.ToString();
	}

	public static string SayHello<T>(this IEnumerable<T> value)
	{
		return "Hello, I'm " + value.ToString();
	}
}
```

Opisywany mechanizm ma jednak swoje ograniczenia. Ponieważ extension method jest metodą statyczną, nie ma ona dostępu do prywatnych zmiennych oraz metod typu.

Kolejną wadą jest brak możliwości nadpisywania istniejących już metod. W przypadku próby zdefiniowania extension method o sygnaturze identycznej z metodą zdefiniowaną dla typu, kompilator nie pozwoli nam na wykorzystanie nowo zdefiniowanej metody.

``` csharp
public static class IntegerExtensions
{
	public static string ToString(this int value)
	{
		return "Value is " + value.ToString();
	}
}
```

Próba wykorzystania powyższej metody zakończy się wywołaniem metody `ToString` zdefiniowanej wewnątrz typu `int` (dokładniej `Int32`).

``` csharp
using ExtensionMethods.Extensions;

namespace ExtensionMethods
{
    public sealed class Example
    {
        public void MethodB()
        {
            var a = 1.ToString(); // Instance method .ToString() is called here.
        }
    }
}
```

Istnieje jednak możliwość wywołania extension method jak zwykłej metody statycznej. Jest to szczególnie przydatne w momencie gdy jest ona zakryta przez jedną z metod obiektu lub istnieją dwie extension method o identycznej sygnaturze zdefiniowane w różnych przestrzeniach nazw. Wspomniana wcześniej extension method `ToString` może zostać wywołana w następujący sposób.

``` csharp
var a = IntegerExtensions.ToString(1);
```

## LINQ

Przykładem użycia mechanizmu extension method w języku C# jest popularna biblioteka LINQ. Pozwala ona na deklaratywne pisanie kodu udostępniając szereg dodatkowych metod do typów `IEnumerable<T>` oraz `IQueryable<T>`.

``` csharp
var number = Enumerable.Range(1, 100);
var oddNumbers = number.Where(x => x % 2 != 0);
```

Wykorzystana powyżej metoda`Where` jest zdefiniowana w następujący sposób.

``` csharp
public static IEnumerable<TSource> Where<TSource>(
	this IEnumerable<TSource> source,
	Func<TSource, bool> predicate);
```

Słowo `this` znajdujące się przed typem pierwszego parametru oznacza, że jest to extension method działający dla typu generycznego `IEnumerable<T>`, przyjmującą jako drugi parametr predykat (funkcje, która dla zadanej wartości zwraca wartość prawda lub fałsz).

## Extension method a IL

Wspomniałem wcześniej, że extension method to nic innego jak metody statyczne z rozbudowanym wsparciem na poziomie języka. Poniżej, jako dowód, przedstawiam przykład wykorzystania tej samej metody jako metody statycznej oraz jako extension method, wraz z odpowiadającym im kodem IL.

``` csharp
public void MethodA()
{
	var n = StringExtensions.NumberOfWords("Some text");
	var m = "Some text".NumberOfWords();
}
```

Kod w IL.

```
.method public hidebysig 
	instance void MethodA () cil managed 
{
	// Method begins at RVA 0x2050
	// Code size 24 (0x18)
	.maxstack 1
	.locals init (
		[0] int32,
		[1] int32
	)

	IL_0000: nop
	IL_0001: ldstr "Some text"
	IL_0006: call int32 ExtensionMethods.Extensions.StringExtensions::NumberOfWords(string)
	IL_000b: stloc.0
	IL_000c: ldstr "Some text"
	IL_0011: call int32 ExtensionMethods.Extensions.StringExtensions::NumberOfWords(string)
	IL_0016: stloc.1
	IL_0017: ret
} // end of method Example::MethodA
```

Kod w IL nie jest wprawdzie tak czytelny jak C#, należy jednak zwrócić uwagę na identyczne wywołania metod w liniach 14 oraz 17. Pierwsza z nich odpowiada za wywołanie statycznej metody, druga za wywołanie extension method.

## Podsumowanie

Extension method to prosty mechanizm dodający nowy sposób na wywoływanie statycznych metod zdefiniowanych dla obiektów konkretnych typów. Jak każde narzędzie ma ono swoje wady i zalety. Główną wadą opisywanego mechanizmu jest działanie na statycznych metodach, których użycie w języku obiektowym powinno budzić zastrzeżenia. Z drugiej strony, jego mądre wykorzystanie może znacznie poprawić jakość pisanego kodu, czego świetnym przykładem jest biblioteka LINQ. Mam nadzieję, że powyższy artykuł przybliżył Ci ideę stojącą za extension method oraz sprawił, że nie będą one czymś tajemniczym.

---

- [https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods)
