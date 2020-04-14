---
title: "Dlaczego null jest zły"
date: 2018-09-13
---

Zacznijmy od przykładu.

``` csharp
public int PersonAge(long pesel)
{
	return _persons
		.Where(x => x.Pesel == pesel)
		.Select(x => x.Age)
		.First();
}

public Address PersonAddress(long pesel)
{
	return _persons
		.Where(x => x.Pesel == pesel)
		.Select(x => x.Address)
		.First();
}

public sealed class Address
{
	public string Street { get; set; }
	public string HouseNumber { get; set; }
	...
}
```

Powyższe dwie metody wyglądają bardzo podobnie. Obie wyszukują w kolekcji osoby o podanym numerze PESEL, a następnie zwracają pewną informację na jej temat. Ponieważ korzystają one z metody `First()`, w przypadku gdy nie zostanie znaleziona ani jedna osoba o podanym numerze PESEL, rzucony zostanie wyjątek `InvalidOperationException`.

W celu uniknięcia wyjątku zmodyfikujmy powyższe metody w taki sposób, aby w przypadku gdy osoba nie zostanie znaleziona zwróciły wartość pustą.

Ponieważ nie istnieje możliwość przypisania wartości `null` do zmiennej typu `int`, modyfikacja pierwszej metody skutkować będzie zmianą typu zwracanego z `int` na `Nullable<int>` (w skrócie `int?`). Spowoduje to konieczność dokonania dalszych zmian w programie, ponieważ każda linijka kodu bazująca na wyniku metody `PersonAge` musi teraz obsłużyć sytuacje, w której wiek osoby jest nieznany.

``` csharp
public int? PersonAge(long pesel)
{
	return _persons
		.Where(x => x.Pesel == pesel)
		.Select(x => (int?)x.Age)
		.FirstOrDefault();
}
```

Niestety, sprawa wygląda odmiennie w przypadku typów referencyjnych. W języku C# istnieje możliwość przypisania wartości `null` do dowolnej zmiennej takiego typu. Oznacza to, że modyfikacja drugiej z metod nie wymaga dalszych zmian w kodzie. Zwrócenie wartości `null` nie wpływa na wynik kompilacji. Nasz program zbuduje się, a wszystkie problemu wynikające z braku wartości wystąpią dopiero w momencie korzystania z programu.

``` csharp
public Address PersonAddress(long pesel)
{
	return _persons
		.Where(x => x.Pesel == pesel)
		.Select(x => x.Address)
		.FirstOrDefault();
}
```

W aktualnej wersji języka C# (wersja 7.0) nie istnieje możliwość rozróżnienia na poziomie systemu typów przypadku, w którym obiekt ma lub nie ma wartości. Nie istnieje typ analogiczny do `Nullable<T>` dla typów referencyjnych.

``` csharp
public Address? PersonAddress(long pesel) // error
{
	...
}
```

Efektem braku jawnego rozróżnienia typów referencyjnych na te, które muszą posiadać wartość oraz na te, które mogą ją posiadać jest konieczność sprawdzania w kodzie, czy obiekt z którym pracujemy jest faktyczną instancją klasy czy nie. Ostatecznie nie chcą ryzykować wystąpienia wyjątku `NullReferenceException` kod zaczyna puchnąć od warunków sprawdzających, czy obiekt nie jest równy `null`.

``` csharp
if(person == null)
{
	return null;
}
return person.Address;
```

Dobra, koniec narzekania. Jak w takim razie unikać wartości `null`?

## Alternatywa 0
Najprostszym sposobem na poprawę sytuacji jest zmiana nazwy metody na taką, która sugeruje możliwość zwrócenia `null`. Tylko tyle i aż tyle. Przykładowo, biblioteka LINQ udostępnia nam dwie metody służące do wyciągnięcie pierwszego elementu z kolekcji: `First()` oraz `FirstOrDefault()`. Nie trzeba czytać dokumentacji aby zrozumieć zasadę ich działania.

Wracając do analizowanego wcześniej przypadku możemy zmienić nazwę metody `PersonAddress` na `PersonAddressOrNull`. Chociaż rozwiązanie to nie naprawia problemu, wciąż zwracamy `null`, może pozwolić na minimalizacje strat w przyszłości. Osobiście nie jestem zwolennikiem defensywnego podejścia do programowania. Jeśli nazwa metody nie sugeruje tego, zakładam że nie zwraca ona `null`.

## Alternatywa 1
Pierwszym realnym sposobem na uniknięcie zwracania `null` z funkcji jest wykorzystanie wzorca projektowego `Null Object`. W dużym skrócie polega on na stworzenie specjalnej, domyślnej instancji oczekiwanej klasy i zwrócenie jej zamiast `null`.
``` csharp
public static readonly Address EmptyAddress = new Address();

public Address PersonAddress(long pesel)
{
	var address = _persons
		.Where(x => x.Pesel == pesel)
		.Select(x => x.Address)
		.FirstOrDefault();
	if(address == null)
	{
		return EmptyAddress;
	}
	return address;
}
```

Rozwiązanie to ma jednak swoje wady mogące prowadzić do przekłamań w programie. Ukrywa ono przed wywołującym metodę fakt, że osoba nie została znaleziona. Jeżeli zapomnimy sprawdzić czy otrzymany przez nas wynik nie jest równy pustemu elementowi, obiekt ten będzie traktowany jak prawdziwa instancja oczekiwanej klasy. Może to skutkować wydrukowaniem faktury dla firmy bez danych lub wysłaniem wiadomości e-mail pod pusty adres.

``` csharp
var address = PersonAddress(0);
if(address.Equals(EmptyAddress)
{
	...
}
...
```

## Alternatywa 2
Kolejnym sposobem na uniknięcie zwracania `null` jest rzucenie wyjątku niezwłocznie w momencie wystąpienia problemu. Podejście to nosi nazwę Fail Fast i uważam je za rozwiązanie lepsze niż zwracanie pustych obiektów. Sprowadza się to do prostego, jednak skutecznego założenia: jeśli program zachowuje się w nieprzewidziany sposób, natychmiast przerywamy jego wykonanie. Nigdy nie maskujemy dziwnego zachowania przed resztą systemu. Największą zaletą tego podejścia jest błyskawiczna identyfikacja problemu. Może wydawać się czymś nienaturalnym chęć celowego przerwania działania systemu, jednak podejście to pozwala na szybkie zauważenie problemu oraz jego poprawę.

W omawianym przypadku sprowadzi się to do zmiany metody `FirstOrDefault `na `First`.

``` csharp
public Address PersonAddress(long pesel)
{
	return _persons
		.Where(x => x.Pesel == pesel)
		.Select(x => x.Address)
		.First();
}
```

## Alternatywa 3
Ostatnią z proponowanych alternatyw jest zdefiniowanie typu analogicznego do `Nullable` działającego również dla typów referencyjnych. Jest to niezwykle popularny wzorzec w językach funkcyjnych, nazywany często Optional, Option lub Maybe. Na potrzeby omawianego przykładu nazwijmy go Maybe i zdefiniujmy w następujący sposób.

``` csharp
public struct Maybe<T>;
{
	private readonly bool _hasValue;
	private readonly T _value;
	
	public Maybe(T value)
	{
		_value = value;
		_hasValue = true;
	}

	public bool HasValue()
	{
		return _hasValue;
	}

	public T Value()
	{
		if(!_hasValue)
		{
			throw new InvalidOperationException();
		}
		return _value;
	}
}
```

Przykład wykorzystania Maybe.

``` csharp
public Maybe<Address> PersonAddress(long pesel)
{
	var address = _persons
		.Where(x => x.Pesel == pesel)
		.Select(x => x.Address)
		.FirstOrDefault();
	if(address == null)
	{
		return new Maybe<Address>(); // no value
	}
	return new Maybe<Address>(address);
}
```

Dokładniejsze wyjaśnienie typu Maybe zamieszczone będzie w kolejnym artykule. Póki co polecam zerknąć na projekt nlk/Optional dostępny na platformie GitHub.

Na koniec ciekawostka. Tony Hoare, człowiek który uznawany jest za twórcę wartości null przyznał, że wprowadził ją nie dlatego, że była mu potrzebna ale dlatego, że było to takie proste. Nie mógł oprzeć się pokusie wprowadzenia null. Po ponad 40 latach, podczas jednej z londyńskich konferencji nazwał swój pomysł "The Billion Dollar Mistake".

---

- [https://en.wikipedia.org/wiki/Tony_Hoare](https://en.wikipedia.org/wiki/Tony_Hoare)
- [https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare](https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare)
- [https://www.yegor256.com/2014/05/13/why-null-is-bad.html](https://www.yegor256.com/2014/05/13/why-null-is-bad.html)
- [https://www.yegor256.com/2015/08/25/fail-fast.html](https://www.yegor256.com/2015/08/25/fail-fast.html)
- [https://blogs.msdn.microsoft.com/dotnet/2017/11/15/nullable-reference-types-in-csharp/](https://blogs.msdn.microsoft.com/dotnet/2017/11/15/nullable-reference-types-in-csharp/)
- [https://github.com/nlkl/Optional](https://github.com/nlkl/Optional)