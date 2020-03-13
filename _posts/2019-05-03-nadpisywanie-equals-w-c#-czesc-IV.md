---
layout: post
title: "Nadpisywanie Equals w C# - część IV"
date: 2019-05-03
categories: C#
---

Artykuły wchodzące w skład serii:
1. Nadpisywanie Equals w C#, część I. Metoda Equals
2. Nadpisywanie Equals w C#, część II. Metoda GetHashCode
3. Nadpisywanie Equals w C#, część III. Implementacja metody Equals
4. Nadpisywanie Equals w C#, część IV. Implementacja metody GetHashCode
5. Nadpisywanie Equals w C#, część V. Porównanie wydajności różnych implementacji metody Equals

Sens istnienia metody `GetHashCode` opisałem w jednym z wcześniejszych artykułów. Dla przypomnienia wymienię jednak najważniejsze wymagania stawiane przed tą metodą:

- powinna być obliczana możliwe jak najszybciej;
- powinna minimalizować wystąpienia kolizji (wystąpienia tej samej wartości *hash code* dla dwóch obiektów, które porównane metodą `Equals` nie będą sobie równe);
- nie może rzucać wyjątków;
- jeśli wynik porównania dwóch obiektów za pomocą metody `Equals` zwróci  wartość *true*, muszą one zwrócić identyczny wyniki z metod `GetHashCode`.

Metodę `GetHashCode` nadpisuje się w identyczny sposób dla typów referencyjnych oraz wartościowych. W omawianym poniżej przykładach skupie się jedynie na tych pierwszych. Podobnie jak w poprzedniej części artykułu posłużę się typem `Person` zdefiniowanym w następujący sposób:

``` csharp
public class Person
{
	private readonly string _firstName;
	private readonly string _lastName;
	
	public Person(
		string firstName,
		string lastName)
	{
		_firstName = firstName;
		_lastName = lastName;
	}
	
	// implementation here
}
```

# Metoda GetHashCode
Przypomnę raz jeszcze: **jeśli nadpisujesz metodę `Equals`, musisz również nadpisać metodę `GetHashCode`**. Nie robiąc tego, musisz liczyć się z tym, że twój program może nie działać poprawnie. Jako ciekawostkę dodam, że zwrócenie stałej wartości równej 0 jest lepszym rozwiązaniem niż nie nadpisanie jej wcale. Sprawi to, że twój typ będzie bezużyteczny jako klucz dla słowników, nie doprowadzi to jednak do błędnego działania systemu. Ucierpi na tym jedynie jego wydajność.

``` csharp
public override int GetHashCode()
{
	return 0; // It's bad, but better than nothing!
}
```

Jeśli jednak nie interesują Cię tak drastyczne rozwiązania, zapraszam do dalszej części artykułu.

Wymagania stawiane metodzie `GetHashCode` sprawiają, że jest ona w dużej mierze zależna od metody `Equals`. Rozsądnym jest więc założenie, że powinny być one obliczane na podstawie tych samych pól.

Dla przypomnienia metoda `Equals` dla typu `Person` wygląda następująco:

``` csharp
public bool Equals(Person other)
{
	if (ReferenceEquals(null, other))
	{
		return false;
	}
	if (ReferenceEquals(this, other))
	{
		return true;
	}
	return string.Equals(_firstName, other._firstName)
		   && string.Equals(_lastName, other._lastName);
}
```

Taka definicja metody `Equals` sugeruje, że metoda `GetHashCode` powinna bazować na polach `_firstName` oraz `_lastName`. Ponieważ każdy typ w języku C# zawiera metodę `GetHashCode`, najprostszą z możliwych jej implementacji jest zsumowanie wartości *hash code* dla każdego z pól.

``` csharp
public override int GetHashCode()
{
	return _firstName.GetHashCode() + _lastName.GetHashCode();
}
```

To proste rozwiązanie nie jest dalekie od ideału, ma jednak swoje wady:

1. w przypadku gdy wynik dodawania dwóch wartości *hash code* przekroczy zakres `int`, metoda może rzucić wyjątek `System.OverflowException`;
2. w przypadku gdy przynajmniej jedno z pól będzie równe *null*, metoda rzuci wyjątek `System.NullReferenceException`;
3. ponieważ dodawanie jest przemienne, obiekty `new Person("a", "b")` oraz `new Person("b", "a") ` zwrócą identyczną wartość *hash code*, pomimo iż są różne.

Rozwiązaniem pierwszego z wspomnianych problemów jest wykorzystanie słowa kluczowego `unchecked`. Jego użycie wyłącza kontrolę przepełnień arytmetycznych w wybranym fragmencie kodu.

``` csharp
unchecked
{
	var x = int.MaxValue; // 2147483647
	var y = x + 1;	// -2147483648 - it's int.MinValue
}
```

Metoda `GetHashCode` z wykorzystaniem `unchecked`:

``` csharp
public override int GetHashCode()
{
	unchecked
	{
		return _firstName.GetHashCode() + _lastName.GetHashCode();
	}
}
```

Rozwiązanie drugiego problemu sprowadza się do sprawdzenia czy obiekt, na którym wywołujemy `GetHashCode` jest różny od *null*. Jeśli jest on równy *null*, zamiast wywoływać metodę `GetHashCode`, zwracamy od razu wartość zero.

``` csharp
public override int GetHashCode()
{
	unchecked
	{
		var hashCode = 0;
		hashCode = hashCode + !ReferenceEquals(null, _firstName) ? _firstName.GetHashCode() : 0;
		hashCode = hashCode + !ReferenceEquals(null, _lastName) ? _lastName.GetHashCode() : 0;
		return hashCode;
	}
}
```

*Ponieważ operator porównania `==` może zostać przeładowany dla dowolnego typu, bezpieczniejszym sposobem na sprawdzenie czy obiekt nie jest równy null jest skorzystanie z statycznej metody `ReferenceEquals`*

Ostatni z omawianych problemów to duplikacja wartości *hash code* w momencie, gdy zmienimy miejscami argumenty przekazane do konstruktora klasy `Person`.

``` csharp
var a = new Person("a", "b").GetHashCode(); // -1684705411
var b = new Person("b", "a").GetHashCode(); // -1684705411
```

Jego rozwiązanie jest trochę mniej intuicyjne, niż wcześniejsze dwa. Sprowadza się ono do pomnożenia częściowego wyniku *hash code* o pewną wartość, przed dodaniem do niego wartości wynikających z kolejnych zmiennych. 

```
hashCode = (hashCode * x) + y
```

Wykorzystaniem w tym celu liczb pierwszych minimalizuje ryzyko wystąpienia kolizji. W omawianym przypadku skorzystam z wiedzy i doświadczenia twórców aplikacji *ReSharper* i podobnie jak oni przemnożę wartość *hash code* razy *397*. Uczciwie przyznam, że nie wiem dlaczego zdecydowali się wybrać akurat tę wartość [3].

``` csharp
public override int GetHashCode()
{
	unchecked
	{
		var hashCode = 0;
		hashCode = (hashCode * 397) + (!ReferenceEquals(null, _firstName) ? _firstName.GetHashCode() : 0);
		hashCode = (hashCode * 397) + (!ReferenceEquals(null, _lastName) ? _lastName.GetHashCode() : 0);
		return hashCode;
	}
}
```

*Mnożenie wartości 0 przez 397 wykonywane w pierwszym kroku jest zbędne. Postanowiłem jednak go nie usuwać w celu uwidocznienia omawianego mechanizmu.*

Wartości *hash code* dla zmodyfikowanej wersji metody:

``` csharp
var a = new Person("a", "b").GetHashCode(); // -248927503
var b = new Person("b", "a").GetHashCode(); // -248927899
```

Wprowadzenie dodatkowego mnożnika pozwoliło na rozróżnienie dwóch instancji typu `Person` z jednakowymi argumentami przekazanymi w odwrotnej kolejności.

Otrzymaliśmy w ten sposób w pełni działającą implementację metody `GetHashCode`. Ostatnią operacją jaką można wprowadzić jest zastąpienie dodawania przez alternatywę rozłączną (XOR). W wyniku czego otrzymamy algorytm FNV[5], wykorzystywane między innymi przez wspomnianą wcześniej aplikację *Resharper*.

``` csharp
public override int GetHashCode()
{
	unchecked
	{
		var hashCode = 0;
		hashCode = (hashCode * 397) ^ (!ReferenceEquals(null, _firstName) ? _firstName.GetHashCode() : 0);
		hashCode = (hashCode * 397) ^ (!ReferenceEquals(null, _lastName) ? _lastName.GetHashCode() : 0);
		return hashCode;
	}
}
```

# Podsumowanie
Jak widać nadpisanie metody `GetHashCode` nie jest zbyt skomplikowane, może być jednak dość nużące. Jest to jedna z tych operacji, która z powodzeniem może zostać zautomatyzowana przez narzędzie generujące kod. Jednak dobrą praktyką korzystania z generatorów kodu jest rozumienie wygenerowanego kodu.

Mam nadzieję, że artykuł ten pomógł Ci zrozumieć w jaki sposób efektywnie zaimplementować metodę `GetHashCode`.

---

1. https://docs.microsoft.com/en-us/dotnet/api/system.object.gethashcode?redirectedfrom=MSDN&view=netframework-4.7#System_Object_GetHashCode
2. https://stackoverflow.com/questions/371328/why-is-it-important-to-override-gethashcode-when-equals-method-is-overridden
3. https://stackoverflow.com/questions/102742/why-is-397-used-for-resharper-gethashcode-override
4. https://stackoverflow.com/questions/263400/what-is-the-best-algorithm-for-an-overridden-system-object-gethashcode/263416#263416
5. https://en.wikipedia.org/wiki/Fowler–Noll–Vo_hash_function
