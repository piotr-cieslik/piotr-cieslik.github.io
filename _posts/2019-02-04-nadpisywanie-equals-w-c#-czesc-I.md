---
title: "Nadpisywanie Equals w C# - część I"
date: 2019-02-04
---

Chciałbym, aby ten pięcioczęściowy artykuł był podsumowaniem tego, czego dowiedziałem się na temat nadpisywania metody `Equals` w C#. Mam nadzieję, że zawarta tu wiedza przyda się także i Tobie.

Artykuły wchodzące w skład serii:
1. Nadpisywanie Equals w C#, część I. Metoda Equals
2. Nadpisywanie Equals w C#, część II. Metoda GetHashCode
3. Nadpisywanie Equals w C#, część III. Implementacja metody Equals
4. Nadpisywanie Equals w C#, część IV. Implementacja metody GetHashCode
5. Nadpisywanie Equals w C#, część V. Porównanie wydajności różnych implementacji metody Equals

# Trochę teorii
W języku C# typy dzielimy na dwie podstawowe kategorie:
- typy wartościowe (ang. *value types*),
- typy referencyjne (ang. *reference types*).

Typy wartościowe to: 
- typy proste (`int`, `long`, `char`, `bool`, `decimal`, ...), 
- wyliczenia (ang. *enum types*), 
- struktury, 
- typ `Nullable<T>`.

Typami referencyjnymi są natomiast: 
- klasy (w tym klasa `String` oraz `Object`), 
- interfejsy, 
- tablice, 
- delegaty.

Dla przypomnienia dodam, że typy wartościowe najczęściej trzymane są bezpośrednio na stosie, a ich przypisanie skutkuje utworzeniem całkowicie nowej kopii wartości.
Natomiast typy referencyjne składają się z dwóch elementów: wskaźnika oraz obiektu. W przeciwieństwie do typów wartościowych nie przechowują bezpośrednio żadnych danych, a jedynie referencję na obiekt w pamięci. Przypisanie zmiennej typu referencyjnego do innej zmiennej skutkuje skopiowaniem referencji. W efekcie obie zmienne wskazywać mogą na ten sam obiekt. 

W języku C# rozróżniamy również dwa rodzaje równości:
- równość wartościowa (ang. *value equality*),
- równość referencyjna (ang. *referential equality*).

Równość wartościowa oznacza, że porównane zostają informacje przechowywane wewnątrz danego typu. Najprostszym przykładem opisywanej równości jest porównanie dwóch liczb.

``` csharp
int a = 1;
int b = 1;

bool result1 = a == b; // true
bool result2 = a.Equals(b); // true
```

Nic nie stoi jednak na przeszkodzie aby porównywać na podstawie przechowywanych informacji typy referencyjne. Przykładem typu implementującego takie rozwiązanie jest klasa `String`.

``` csharp
string a = "text";
string b = "text";

bool result1 = a == b; // true
bool result2 = a.Equals(b); // true
```

Drugi z wymienionych rodzajów równości, równość referencyjna, zarezerwowany jest jedynie dla typów referencyjnych. Polega ona na sprawdzeniu, czy obie zmienne posiadają referencję do tego samego obiektu.

``` csharp
int[] a = new[] { 1 };
int[] b = new[] { 1 };

bool result1 = a == b; // false
bool result2 = a.Equals(b); // false
```

Pomimo iż zmienne `a` oraz `b` przechowują dwie tablice o identycznej zawartości, w wyniku ich porównania otrzymamy wartość *false*. Dzieje się tak dlatego, że obie zmienne przechowują referencje do różnych obiektów w pamięci. Jeśli natomiast obie zmienne będą przechowywać referencję do tej samej tablicy, w wyniku ich porównania otrzymamy wartość *true*.

``` csharp
int[] a = new[] { 1 };
int[] b = a;

var result1 = a == b; // true
var result2 = a.Equals(b); // true
```

Zbieżność pomiędzy nazwami typów oraz nazwami równości nie jest przypadkowa. Domyślnie wszystkie typy wartościowe porównywane są na zasadach równości wartościowej, natomiast wszystkie typy referencyjne na zasadach równości referencyjnej.

W przedstawionych powyżej przykładach porównania zmiennych przeprowadzone były na dwa sposoby: z wykorzystaniem instancyjnej metody `Equals` oraz z wykorzystaniem operatora `==`. Należy jednak pamiętać, że nie są one równoważne. Różnice pomiędzy nimi opisane zostały w dalszej części artykułu. 

# Metoda Equals(object obj)
Wirtualna metoda `Equals(object obj)`, zdefiniowana wewnątrz typu `Object`, dostępna jest dla wszystkich typów w języku C# (zarówno referencyjnych jak i wartościowych). Ponieważ jest ona metodą wirtualna, może być nadpisywana podczas definicji własnych typów, a jej konkretna implementacja rozpoznawana jest dopiero w trakcie działania programu (ang. *runtime*).

*Choć może wydawać się to dziwne, typy wartościowe również dziedziczą po klasie `Object`. Pomimo iż język C# nie pozwala nam na dziedziczenie struktur, wszystkie one dziedziczą niejawnie po bazowej, abstrakcyjnej klasie `ValueType`, dziedziczącej po klasie `Object`.*

*`struct MyStruct` > `abstract class ValueType` > `class Object`*

Wspomniałem wcześniej, że w zależności od rodzaju typu rozróżniamy dwa rodzaje równości: wartościowa oraz referencyjna. Przyjrzyjmy się więc bliżej domyślnej implementacji metody `Equals` dla obu tych typów.

Dla typów referencyjnych domyślna implementacja metody `Equals` sprawdza, czy porównywane argumenty zawierają referencje na ten sam obiekt. Jeśli tak jest to są one sobie równe. Jeśli tak nie jest, pomimo iż obiekty wskazywane przez ich referencje mogą zawierać identyczne dane, metoda `Equals` zwróci wartość *false*. Wyjątkiem od tej reguły jest klasa `String`. Pomimo, iż jest ona typem referencyjnym, jej instancje porównywane są na zasadach równości wartościowej.

Domyślna implementacja metody `Equals` dla typów wartościowych jest zdecydowanie bardziej skomplikowana. Ponieważ zmienne tych typów porównywane są na podstawie przechowywanych informacji konieczne jest porównanie wszystkich pól należących do danego typu. Co ciekawe, tworząc własną strukturę będzie ona porównywana w taki właśnie sposób nawet, gdy nie napiszemy ręcznie metody `Equals`. 

```csharp
public struct Person
{
	public string Name { get; set; }
	public int Age { get; set; }
}
```

```csharp
Person a = new Person() { Name = "Piotr", Age = 29 };
Person b = new Person() { Name = "Piotr", Age = 29 };

bool result = a.Equals(b);	// true
```
Implementacja bazowej metody `Equals`, w czasie działania programu, wykrywa wszystkie pola struktury, a następnie wywołuje na każdym z nich metodę `Equals`. Odczytanie wszystkich pól typu możliwe jest dzięki wykorzystaniu mechanizmu refleksji. Jest to bardzo ciekawe jednak wolne rozwiązanie. W przypadku, gdy zależy nam na optymalizacji procesu porównywania dwóch struktur, dobrym pomysł jest nadpisanie metody `Equals` i porównywanie pól ręcznie.

Jednym z mankamentów metody `Equals`, jest możliwość wystąpienia wyjątku `NullReferenceException` w momencie, gdy zostanie ona wywołana na zmiennej przechowującej wartość *null*. W celu uniknięcie tego problemu twórcy języka dodali również statyczną wersję metody, zaimplementowaną również w klasie `Object`, która sprawdza czy przekazane do niej parametry są różne od *null* przed ich porównaniem.

``` csharp
public static bool Equals(object objA, object objB)
{
  if (objA == objB)
	return true;
  if (objA == null || objB == null)
	return false;
  return objA.Equals(objB);
}
```
Powyższa implementacja została skopiowana wprost z klasy `Object`. Warto dodać, że w przypadku, gdy oba obiekty są równe *null*, spełniony zostanie pierwszy warunek oraz zwrócona zostanie wartość *true*.

``` csharp
bool result = Equals(null, null); // true
```

Sposób działania operatora porównania na typie `Object` opisany został w dalszej części artykułu. 
 
# Operator porównania ==
Wszystkie operatory w języku C# zaimplementowane są jako zwykłe metody statyczne. Oznacza to, że decyzja na temat wykorzystanych typów podejmowana jest na etapie kompilacji programu, a nie jak w przypadku metod wirtualnych podczas jego wykonywania. Jest to jedna z głównych różnic pomiędzy działaniem operatora porównania, a metodą `Equals`. Przyjrzymy się opisanej różnicy w praktyce.

``` csharp
string a = "text";
string b = "text";

bool result1 = a == b; // true
bool result2 = a.Equals(b);	// true
```
W momencie, gdy kompilator wie, że zmienne `a` oraz `b` są typu `String`, wykorzystany zostanie operator porównania dla tego właśnie typu. Jeśli jednak zmienimy typ zmiennych z `String` na `Object`, kompilator porówna dwie wartości z wykorzystaniem domyślnego operatora dla typu `Object`.

``` csharp
object a = "text";
object b = "text";

bool result1 = a == b; // false
bool result2 = a.Equals(b);	// true
```
Metoda `Equals` odporna jest na tego typu zachowania, ponieważ wybór konkretnej implementacji następuje dopiero w momencie wykonywania programu (ang. *runtime*).

Domyślną implementacją operatora porównania dla typów referencyjnych jest sprawdzenie równości referencyjnej.

``` csharp
public class MyClass
{
}

...

var a = new MyClass();
var b = new MyClass();
var result = a == b; // false
```

Nie istnieje natomiast domyślna implementacja operatora porównania dla typów wartościowych.

``` csharp
public class MyStruct
{
}

...

var a = new MyStruct();
var b = new MyStruct();
var result = a == b; // error CS0019: Operator '==' cannot be applied to operands of type 'Program.MyStruct' and 'Program.MyStruct'
```

Operator porównania może zostać przeładowany dla dowolnego typu, jednak w praktyce dokonuje się tego najczęściej dla typów wartościowych.

``` csharp
public class MyStruct
{
	public static bool operator == (MyStruct a, MyStruct b)
	{
		return a.Equals(b);
	}
	
	public static bool operator != (MyStruct a, MyStruct b)
	{
		return !a.Equals(b);
	}
}

...

var a = new MyStruct();
var b = new MyStruct();
var result = a == b; // true
```

Nie zawsze jednak, dla typów wartościowych, wynik zwrócony przez operator porównania jest identyczny z wynikiem zwróconym z metody `Equals`. Przyjrzyjmy się wynikom porównania dwóch wartości `float.NaN`.

``` csharp
float a = float.NaN;
float b = float.NaN;

bool result1 = a == b; // false
bool result2 = a.Equals(b);	// true
```

Operator `==` zaimplementowany został w taki sposób, żeby dwie wartości *NaN* nigdy nie były sobie równe. Ma to swoje uzasadnienie z matematycznego punktu widzenia. Implementacja metody `Equals` została natomiast napisana w taki sposób, żeby`a.Equals(a)` zawsze zwracało wartość *true*.

# Metoda ReferenceEquals(object objA, object objB)
Ponieważ definiowane przez nas typy mogą w dowolny sposób nadpisywać metodę wirtualną `Equals` oraz przeciążać operator porównania `==`, twórcy języka udostępnili nam statyczną metodę `ReferenceEquals` pozwalającą na jawne sprawdzenie równości referencyjnej dwóch dowolnych wartości, bez względu na ich typy.

Metoda `ReferenceEquals` nie nadaje się do porównywania ze sobą dwóch typów wartościowych. Pomimo, iż techniczne jest to wykonalne, z uwagi na wystąpienie zjawiska *boxingu* zawsze zwrócona zostanie wartość *false*.

# Podsumowanie
Mam nadzieję, że powyższy artykuł pomógł Ci w przystępny sposób zrozumieć różnicę pomiędzy porównywaniem typów wartościowych i referencyjnych oraz wyjaśnił podstawowe różnice pomiędzy metodą `Equals` a operatorem `==`. W następnym artykule omówię czym jest metoda `GetHashCode` oraz dlaczego jest ona ważna.

---

- [https://docs.microsoft.com/en-us/dotnet/csharp/tour-of-csharp/types-and-variables](https://docs.microsoft.com/en-us/dotnet/csharp/tour-of-csharp/types-and-variables)
