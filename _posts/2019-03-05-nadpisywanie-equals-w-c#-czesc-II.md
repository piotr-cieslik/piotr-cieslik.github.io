---
layout: post
title: "Nadpisywanie Equals w C# - część II"
date: 2019-03-05
categories: C#
---

Artykuły wchodzące w skład serii:
1. Nadpisywanie Equals w C#, część I. Metoda Equals
2. Nadpisywanie Equals w C#, część II. Metoda GetHashCode
3. Nadpisywanie Equals w C#, część III. Implementacja metody Equals
4. Nadpisywanie Equals w C#, część IV. Implementacja metody GetHashCode
5. Nadpisywanie Equals w C#, część V. Porównanie wydajności różnych implementacji metody Equals

# Metoda GetHashCode
Metoda `GetHashCode`, podobnie jak metoda `Equals`, jest wirtualną metodą zdefiniowaną wewnątrz typu `Object`. Sprawia to, że jest dostępna dla wszystkich typów w języku C#.

Pomimo, iż jest ona wykorzystywana głównie w słownikach (lub ogólniej w typach pozwalający na szybki dostęp do obiektu na podstawie zadanego klucza), twórcy języka stwierdzili, że jest ona na tyle ważna żeby umieścić ją w klasie `Object`.

Zanim przejdziemy do zdefiniowania czym jest oraz do czego służy metoda `GetHashCode` przeanalizujmy w jaki sposób można zaimplementować prosty słownik, który dla zadanego numeru PESEL zwróci przechowywane informację na temat osoby.

Wpis w naszym słownik definiowany będzie przez parę `(Pesel, Person)`.

``` csharp
public sealed class Pesel
{
    public Pesel(long value)
    {
        Value = value;
    }
    
    public long Value { get; }
}

public sealed class Person
{
	// any implementation here
}
```

Najprostszym rozwiązaniem jest umieszczenie wszystkich danych w pojedynczej liście, a następnie przeszukiwanie jej element po elemencie aż do momentu znalezienia pierwszego wystąpienia interesującego nas wpisu.

```csharp
public sealed class MyDictionary
{
	private readonly List<(Pesel Pesel, Person Person)> _set = new List<(Pesel, Person)>();

	public void Add(Pesel pesel, Person person)
	{
		if(_set.Any(x => x.Pesel.Value == pesel.Value))
		{
			throw new InvalidOperationException();
		}
		_set.Add((pesel, person));
	}

	public Person Get(Pesel pesel)
	{
		return _set
			.Where(x => x.Pesel.Value == pesel.Value)
			.Select(x => x.Person)
			.First();
	}
}
```

Wadą tego rozwiązania jest liniowa złożoność obliczeniowa *O(n)*. Oznacza to, że w pesymistycznym wariancie zmuszeni będziemy przeszukać wszystkie elementy kolekcji w celu znalezienia interesującego nas wpisu. Zakładając, że pojedyncze porównanie trwa 1ms, przeszukanie słownika zawierającego 1 000 000 elementów możemy zając ponad 16 minut!

Rozwiązaniem opisanego problemu jest podział kolekcji na mniejsze podzbiory, wewnątrz których wyszukamy interesujący nas numer PESEL. Dzieląc kolekcję na odpowiednią liczbę podzbiorów oraz wykorzystując fakt, że odczyt wartości przechowywanej w tablicy pod danym adresem jest szybki, możemy zredukować złożoność obliczeniową naszego problemu z *O(n)* do *O(1)*. Oznacza to, że przeszukanie słownika zawierającego 1 000 000 elementów będzie tak samo proste, jak przeszukanie słownika jednoelementowego.

Wspomniane rozwiązanie rodzi dwa pytania:
- W jaki sposób wybrać liczbę podzbiorów?
- W jaki sposób przyporządkować numer PESEL do konkretnego podzbioru.

Odpowiedz na pierwsze pytanie wybiega poza zakres tego artykułu. Liczba podzbiorów najczęściej obliczana jest dynamicznie w momencie dodawania nowych elementów do słownika. Jednak dla naszych potrzeb przyjmijmy, że nasz zbiór dzielimy na znaną z góry liczbę podzbiorów wynoszącą *N*. Dla osób, które nie lubią przykładów bez podanych wartości proponuję przyjąć *N = 10*, jednak może być to dowolna liczba większa do zera mieszcząca się w zakresie  typu `Int32`.

```csharp
var N = 10;
var subsets = new List<(Pesel, Person)>[N];
```

Odpowiedzią na drugie pytanie jest zdefiniowanie funkcji, która na podstawie numeru PESEL (klucza) zwróci nam numer podzbioru  *n* z przedziału `[0,N-1]`.

*n = f(Pesel)*

Aby uniezależnić poszukiwaną funkcję od liczby podzbiorów możemy zdefiniować ją w taki sposób, aby zwracała ona dowolną liczbę całkowitą, dla której następnie policzymy wynik reszty z dzielenia przez N w celu otrzymania numeru podzbioru.

*m = f(Pesel)*
*n = m % N*

Poszukiwana przez nas funkcja *f(Pesel)* to właśnie `GetHashCode`.

W naszym przypadku funkcja *f(Pesel)* może zwracać wartość składającą się z pierwszych 6 cyfr numeru pesel reprezentujących rok, miesiąc oraz dzień urodzenia.

*f(Pesel) = round(Pesel / 100000)*

Może zostać ona zaimplementowana w poniższy sposób.

``` csharp
public int GetHashCode(Pesel pesel)
{
	return (int) pesel.Value / 100000;
}
```

Poniżej przedstawiam kompletną implementację słownika z uwzględnieniem podziału na N podzbiorów.

``` csharp
public sealed class MyDictionary
{
    private const int N = 10;
    private readonly List<(Pesel Pesel, Person Person)>[] _subsets = new List<(Pesel, Person)>[N];
    
    public void Add(Pesel pesel, Person person)
    {
        var numberOfSubset = Math.Abs(GetHashCode(pesel) % N);
        var subset = _subsets[numberOfSubset];
        if(subset == null)
        {
            subset = new List<(Pesel Pesel, Person Person)>();
            _subsets[numberOfSubset] = subset;
        }
        if(subset.Any(x => x.Pesel.Value == pesel.Value))
        {
            throw new InvalidOperationException();
        }
        subset.Add((pesel, person));
    }
    
    public Person Get(Pesel pesel)
    {
        var numberOfSubset = Math.Abs(GetHashCode(pesel) % N);
        var subset = _subsets[numberOfSubset];
        return subset
            .Where(x => x.Pesel.Value == pesel.Value)
            .Select(x => x.Person)
            .First();
    }
    
    private int GetHashCode(Pesel pesel)
    {
        return (int) pesel.Value / 100000;
    }
}
```

Ponieważ w realnym świecie słowniki napisane zostały w taki sposób, aby nie musiały znać z góry typu przechowywanych obiektów, projektanci języka zdecydowali się dodać metodę `GetHashCode` wprost do typu `Object`. Jednak jej przeznaczenie jest dokładnie takie, jak w przedstawionym powyżej przykładzie.

Metoda `GetHashCode` powinna być obliczana możliwe jak najszybciej zapewniając jednocześnie minimalne ryzyko wystąpienia kolizji. Powinna być ona również niezmienna dla tych samych parametrów wejściowych.

Implementując opisywaną metodę należy pamiętać, że w przypadku, gdy wyniki metod `GetHashCode` są sobie równe jest to warunek konieczny, jednak nie wystarczający do stwierdzenia faktu, że porównywane wartości lub obiekty są sobie równe.

Ponieważ metoda `GetHashCode` jest ściśle powiązana z metodą `Equals` należy zawsze pamiętać o nadpisaniu obu z nich. Nie powinniśmy nigdy dopuścić do sytuacji, w której wynik porównania wartości zwróconych z metod `GetHashCode` będzie inny niż wynik wywołania metody `Equals`.

# Podsumowanie
Mam nadzieję, że przykład prostego słownika przedstawiony w tym artykule pomógł Ci zrozumieć dlaczego metoda `GetHashCode` jest ważna oraz dlaczego należy nadpisywać ją razem z metodą `Equals`.

---

- [https://docs.microsoft.com/en-us/dotnet/api/system.object.gethashcode](https://docs.microsoft.com/en-us/dotnet/api/system.object.gethashcode)
- [https://en.wikipedia.org/wiki/Hash_function](https://en.wikipedia.org/wiki/Hash_function)
