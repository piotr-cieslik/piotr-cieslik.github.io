---
title: "C# 7.0 - value tuples"
date: 2018-11-08
---

Chcąc omówić wprowadzone razem z C# 7.0 generyczne typy `ValueTuple<T>` warto przypomnieć czym właściwie jest struktura danych Tuple oraz opisać jej implementację w środowisku .Net.

<!--more-->

## Struktura danych Tuple
Struktura danych Tuple, zwana po polsku krotką, pozwala na przechowanie stałej liczby niezmiennych elementów o różnych typach danych. Abstrahując od konkretnych języków programowania, przykładowy Tuple może wyglądać następująco.

```
person = (1, "Piotr", 29)
...
id = person[0]
name = person[1]
age = person[2]
```

Struktura ta jest szczególnie przydatna, gdy chcemy zwrócić z metody więcej niż jeden wynik

```
function GetPerson : (number, string)
{
	return (1, "Piotr")
}
```

lub zwracany typ jest kolekcją złożonych elementów.

```
persons = [
	(1, "Piotr" 29),
	(2, "Ola", 25)
]
```


## System.Tuple
Wsparcie dla struktury Tuple zostało wprowadzone w .Net Framework 4.0 przez dodanie statycznej klasy `Tupl`e oraz 8 generycznych typów `Tuple<>`:
- `Tuple<T1>`,
- `Tuple<T1, T2>`,
- ...,
- `Tuple<T1, T2, T3, T4, T5, T6, T7>`,
- `Tuple<T1, T2, T3, T4, T5, T6, T7, T8>`.

Najważniejsze właściwości typów `Tuple<>`:
- są klasami (są typami referencyjnymi, nie wartościowymi),
- przechowywane w nich wartości nie mogą zostać zmienione (ang. immutable),
- mogą być serializowane (ang. serializable),
- mogą być porównywane (ang. equatable),
- mogą być sortowane (ang. comparable).

Microsoft umożliwił nam tworzenie obiektu typu `Tuple` na dwa różne sposoby. Pierwszym jest wykorzystanie konstruktora jednego z generycznych typów `Tuple<>`.

``` csharp
var person = new Tuple<int, string>(1, "Piotr");
```

Drugim sposobem jest skorzystanie z jednej z statycznych metod `Create` klasy `Tuple`.

``` csharp
var person = Tuple.Create(1, "Piotr");
```

Zaletą drugiego rozwiązania jest brak konieczności ręcznego definiowania typów przechowywanych wartości. W zdecydowanej większości przypadków kompilator poprawnie domyśli się o jakie typy nam chodzi. Warto również pamiętać, że maksymalna liczba elementów przechowywana w obiekcie typu `Tuple<>` wynosi 8, istnieje jednak możliwość zagnieżdżania `Tuple<>` w `Tuple<>`.

``` csharp
var tuple = new Tuple<int, int, int, int, int, int, int, Tuple<int, int>>(1,2,3,4,5,6,7, new Tuple<int, int>(8, 9));
```

Skoro omówiliśmy już sposoby tworzenia obiektów klas `Tuple<>`, zobaczmy w jaki sposób odwołać się do przechowywanych wartości. Odwołanie się do wartości odbywa się przez właściwości o nazwach Item1, Item2,  ..., Item8, których liczba uzależniona jest od typu `Tuple<>`.

``` csharp
var person = Tuple.Create(1, "Piotr");
var id = person.Item1;
var name = person.Item2;
```

### Przykłady użycia
Typy `Tuple<>` wykorzystane mogą być wszędzie tam, gdzie zależy nam na przekazaniu pewnych paczek informacji bez konieczności definiowania nowego typu. 

``` csharp
public Tuple<int, string> Person()
{
	return Tuple.Create(1, "Piotr");
}
```

Osobiście odradzam jednak ich nadużywanie. Odwoływanie się do właściwości obiektów przez nazwy Item1, Item2, Item3, ... , bardzo negatywnie wpływa na czytelność kodu.

``` csharp
public int DoMath() {
	var data = Data();
	return (data.Item2 * data.Item2) - 4 * data.Item1 * data.Item3;
}
```

Analizując powyższy przykład można się domyślić, że metoda `DoMath` oblicza deltę równania kwadratowego. Jednak w ogólnym przypadku ciężko zrozumieć kod bazujący na właściwościach o nazwach w formacie ItemX. Jest to szczególnie trudne, gdy metoda zwracająca Tuple znajduje się w innym miejscu w kodzie.

Nie twierdzę jednak, że należy całkowicie rezygnować z typów `Tuple<>`. Dobrym miejscem na ich wykorzystanie jest przekazywanie zmiennych pomiędzy prywatnymi metodami klasy. Uważam, że typy te nie powinny być wykorzystywane w publicznych metodach z powodu niewiele mówiących nazw elementów. Jednak z uwagi na fakt, że obiekty typów `Tuple<>` porównywane są na podstawie przechowywanych wartości, są one wygodnym narzędziem do definiowania złożonych kluczy słowników.

``` csharp
var dictionary = new Dictionary<Tuple<int, int>, string>();
...
dictionary[Tuple.Create(1, 1)] = "Some value";
...
var value = dictionary[Tuple.Create(1,1)];
```

### Porównywanie
Dwa obiekty klasy `Tuple<>` są równe, gdy posiadają taką samą liczbę elementów oraz odpowiadające sobie elementy są równe.

``` csharp
var tuple1 = Tuple.Create(1, "Piotr");
var tuple2 = Tuple.Create(1, "Piotr");
var result = Equals(tuple1, tuple2); // true
```

Uwaga, nie należy korzystać z operatora == do porównywania dwóch obiektów klasy `Tuple`. Operator ten porówna obiekty po referencji!

``` csharp
var tuple1 = Tuple.Create(1, "Piotr");
var tuple2 = Tuple.Create(1, "Piotr");
var result = tuple1 == tuple2; // false
```

### Sortowanie
Podobnie jak w przypadku porównywania, sortowanie odbywa się na podstawie wartości poszczególnych elementów. Najpierw porównywane są elementy znajdujące się na pierwszej pozycji, następnie na pozycji drugiej (o ile istnieje), operacja ta jest powtarzana aż do momentu wystąpienia pierwszej różnicy w wartościach lub wyczerpania liczby elementów.

``` csharp
var array = new[]{
	Tuple.Create(2, "a"),
	Tuple.Create(1, "a"),
	Tuple.Create(1, "b")
};
Array.Sort(array);

// array:
// 0: (1, a)
// 1: (1, b)
// 2: (2, a)
```

## System.ValueTuple
Dodane razem z C# 7.0 typy `ValueTuple<>` są na pierwszy rzut oka bardzo podobne do typów `Tuple<>`. Podobnie jak w przypadku `Tuple<>` Microsoft zdecydował się na definicję ośmiu typów
- `ValueTuple<T1>`
- `ValueTuple<T1, T2>`
- ...
- `ValueTuple<T1, T2, T3, T4, T5, T6, T7>`
- `ValueTuple<T1, T2, T3, T4, T5, T6, T7, T8>`
oraz statycznej klasy `ValueTuple`.

Najważniejsze właściwości typów `ValueTuple<>`:
- są strukturami (typ wartościowy),
- przechowywane w nich wartości mogą zostać zmienione (ang. mutable),
- mogą być serializowane (ang. serializable),
- mogą być porównywane (ang. equatable),
- mogą być sortowane (ang. comparable).

Warto zwrócić uwagę na dwie różnice pomiędzy omawianymi typami. `ValueTuple<>` są strukturami, nie klasami, a przechowywane w nich wartości mogą zostać zmienione.

``` csharp
var person = (Id: 1, Name: "Piotr");
person.Id = 2;	// mutable
```

### Przykłady użycia
Wartości typów `ValueTuple<>` możemy tworzyć w analogiczny sposób jak obiekty typów `Tuple<>`, oznacza to wykorzystanie konstruktora określonego typu lub przez jedną z ośmiu statycznych metod `Create` zdefinowanych w strukturze `ValueTuple`.

``` csharp
var person = new ValueTuple<int, string>(1, "Piotr");
```

``` csharp
var person = ValueTuple.Create(1, "Piotr");
```

Korzystając jednak z C# w wersji 7.0 i nowszej powyższy zapis można zastąpić znacznie krótszym.

``` csharp
var person = (1, "Piotr");
var name = person.Item2;
```

Wsparcie dla szybkiego definiowania wartości typu `ValueTuple` znacznie zwiększa czytelność kodu. Wciąż nie rozwiązuje to jednak problemu odwoływania się do jego właściwości nazwami Item1, Item2, ..., Item8. 

Na szczęście i tutaj Microsoft wykonał dobrą robotę dodając możliwość definiowania nazw dla przechowywanych wartości. Nazwy elementów zdefiniowane mogą zostać podczas tworzenie elementu

``` csharp
var person = (Id: 1, Name: "Piotr");
var name = person.Name;
```

lub podczas definicji typu.

``` csharp
(int Id, string Name) person = (1, "Piotr");
var name = person.Name;
```

Warto zauważyć, że w przypadku zdefiniowania nazwy dwukrotnie, pierwszeństwo ma nazwa zdefiniowana dla typu, nie dla konkretnej wartości.

``` csharp
(int Id, string Name) person = (Id2: 1, Name2: "Piotr");
var name = person.Name; // not person.Name2
```

Znając już sposoby na definicję typu oraz konkretnej wartości, przyjrzyjmy się wykorzystaniu `ValueTuple<>` jako typu zwracanego z metody.

``` csharp
public (int Id, string Name) Person()
{
	return (1, "Piotr");
}
...
var person = Person();
var name = person.Name;
```

Powyższy przykład pokazuje, że typy `ValueTuple<>` są świetnym sposobem na zwrócenie więcej niż jednego wyniku z metody. Dodatkowo mogą być one wykorzystane w połączeniu z  typami generycznymi.

``` csharp
public IEnumerable<(int Id, string Name)> Persons()
{
	yield return (1, "Piotr");
}
...
var persons = Persons();
var names = persons.Select(x => x.Name);
```

Jako ciekawostkę dodam, że kompilator przechowuje zdefiniowane przez nas nazwy elementów z wykorzystaniem atrybut `TupleElementNames`. Poniżej znajduje się kod powstały w wyniku dekompilacji metody Person() z przedostatniego przykładu.

``` csharp
[return: TupleElementNames(new string[] {
	"Id",
	"Name"
})]
public ValueTuple<int, string> Person()
{
	return new ValueTuple<int, string>(1, "Piotr");
}
...
ValueTuple<int, string> valueTuple = Person();
string item = valueTuple.Item2;
```

### Dekonstrukcja
Kolejną nowością wprowadzoną w C# 7.0 jest dekonstrukcja typów, pozwala ona na rozbicie elementów typu `ValueTuple<>` na poszczególne zmienne lokalne.

``` csharp
public (int Id, string Name, int Age) Person()
{
	return (1, "Piotr", 29);
}
```

Przykład ręcznego przypisania wartości.

``` csharp
var tuple = Person();
var id = tuple.Item1;
var name = tuple.Item2;
var age = tuple.Item3;
```

Przykład wykorzystania dekonstrukcji.

``` csharp
(var id, var name, var age) = Person();
```

Powyższy kod zapisać można jeszcze krócej.

``` csharp
var(id, name, age) = Person();
```

Co ciekawe dekonstrukcja nie jest zarezerwowana jedynie dla wartości typów `ValueTuple<>`. Może ona zostać zaimplementowana na dowolnym typie definiując metodę o nazwie `Deconstruct` z odpowiednimi parametrami.

``` csharp
public sealed class Person
{
	public void Deconstruct(out int id, out string name)
	{
		id = 1;
		name = "Piotr";
	}
}
...
var person = new Person();
var (id, name) = person;
```

### Porównywanie i sortowanie
Porównywanie i sortowania wartości typu `ValueTuple<>` jest identyczne jak w przypadku obiektów typu `Tuple<>`. Obie operacje możliwe są tylko dla obiektów z identyczną liczbą elementów oraz odbywają się one bazując na wartościach odpowiadających sobie elementów.

## Podsumowanie
Wprowadzenie typów `ValueTuple<>` oraz wsparcia dla nich na poziomie składni języka to moim zdaniem krok w dobrą stronę. Odciąża nas to od niepotrzebnego pisania klas, których jedynym zadaniem jest przeniesienie danych z jednego miejsca w inne, pozwalając skupić się na rozwiązywaniu poważniejszych problemów.

Mam nadzieje, że powyższy artykuł przybliżył Ci idee struktury danych Tuple oraz rozwiązań zaimplementowanych w .Net Framework pozwalających na jej sprawne wykorzystanie.

---

[https://blogs.msdn.microsoft.com/mazhou/2017/05/26/c-7-series-part-1-value-tuples/](https://blogs.msdn.microsoft.com/mazhou/2017/05/26/c-7-series-part-1-value-tuples/)
[https://pl.wikipedia.org/wiki/Krotka_(struktura_danych)](https://pl.wikipedia.org/wiki/Krotka_(struktura_danych))
