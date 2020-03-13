---
layout: post
title: "Nadpisywanie Equals w C# - część III"
date: 2019-04-10
categories: C#
---

Artykuły wchodzące w skład serii:
1. Nadpisywanie Equals w C#, część I. Metoda Equals
2. Nadpisywanie Equals w C#, część II. Metoda GetHashCode
3. Nadpisywanie Equals w C#, część III. Implementacja metody Equals
4. Nadpisywanie Equals w C#, część IV. Implementacja metody GetHashCode
5. Nadpisywanie Equals w C#, część V. Porównanie wydajności różnych implementacji metody Equals

W opisywanych poniżej przykładach będę posługiwał się klasą oraz strukturą `Person` zdefiniowaną w następujący sposób.

``` csharp
public class/struct Person
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

# Nadpisywanie Equals dla typów referencyjnych
Nadpisywanie metody `Equals` nie jest niczym skomplikowany. Sprowadza się ono do wykonania rzutowania obiektu przekazanego przez parametr metody, sprawdzenia czy nie jest on równy *null* oraz porównania interesujących nas pól. W najprostszym przypadku metoda `Equals` może zostać nadpisana w następujący sposób:

``` csharp	
public override bool Equals(object obj)
{
	var other = obj as Person;
	if(other == null)
	{
		return false;
	}
	return Equals(_firstName, other._firstName)
		&& Equals(_lastName, other._lastName);
}
```

Jeśli jednak zależy nam na wydajności, warto pamiętać, że rzutowanie jest operacją bardziej złożoną obliczeniowo niż porównanie referencji. Mając to na uwadze zdefiniujmy uniwersalną listę kroków, która powinna być wykonana w celu efektywnego porównania dwóch instancji typów referencyjnych.

Lista kroków:

1. sprawdzenia czy przekazany obiekt nie jest równy *null*,
2. sprawdzenia czy przekazany obiekt nie jest równy *this* (jeśli dwa obiekty mają jednakowe referencje, muszą być sobie równe),
3. sprawdzenia czy przekazany obiekt ma pasujący typ,
4. porównanie przechowywanych wartości.

Warto również wyodrębnić osobną wersje metody `Equals` przyjmującą jako parametr instancję typu, dla którego nadpisywana jest metoda. Pozwala to na uniknięcie rzutowania w momencie gdy znane są typy porównywanych obiektów. 

``` csharp
bool Equals(Person other)
{
	// implementation here
}
```

To właśnie chęć uniknięcia niepotrzebnego rzutowania jest jednym z powodów wprowadzenia interfejsu `IEquatable<T>`.

``` csharp
public interface IEquatable<T>
{
	bool Equals(T other);
}
```

Ostatecznie nasza klasa `Person` wygląda następująco:

``` csharp
public class Person : IEquatable<Person>
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

	public bool Equals(Person other)
	{
		// Is null?
		if (ReferenceEquals(null, other))
		{
			return false;
		}
		
		// Is same instance?
		if (ReferenceEquals(this, other))
		{
			return true;
		}

		// Custom comparison here.
		return string.Equals(_firstName, other._firstName)
			   && string.Equals(_lastName, other._lastName);
	}

	public override bool Equals(object obj)
	{
		// Is null?
		if (ReferenceEquals(null, obj))
		{
			return false;
		}

		// Is same instance?
		if (ReferenceEquals(this, obj))
		{
			return true;
		}

		// Is same type?
		if (obj.GetType() != GetType())
		{
			return false;
		}

		return Equals((Person) obj);
	}
	
	// Remember to override method `GetHashCode`!
}
```

W przypadku, gdy chcemy pozwolić również na porównywanie instancji typów dziedziczących po naszej klasie, należy zmienić warunek sprawdzający typ obiektu na bardziej ogólny.

``` csharp
public override bool Equals(object obj)
{
	// Is null?
	if (ReferenceEquals(null, obj))
	{
	    return false;
	}
	
	// Is same instance?
	if (ReferenceEquals(this, obj))
	{
	    return true;
	}
	
	return obj is Person && Equals((Person)obj);
}
```

Korzystając z *pattern matching* ostatnia linia kodu może zostać zapisana w poniższy sposób.

``` csharp
return obj is Person person && Equals(person);
```

Analizują implementację metody `Equals(object obj)` można dojść do wniosku, że możliwe jest wykonanie niepotrzebnego sprawdzenia (dwukrotnego), czy przekazana instancja jest równa *null* lub *this*. W celu uniknięcia duplikacji można wprowadzić trzecią, prywatną wersję metody `Equals`, a następnie wywoływać ją jako ostatni krok z obu przedstawionych wcześniej metod.

``` csharp
private bool EqualsInternal(Person other)
{
    return string.Equals(_firstName, other._firstName)
           && string.Equals(_lastName, other._lastName);
}
```

Uważam jednak, że wprowadzenie jej nie wpływa praktycznie na prędkość porównania obiektów. Wpływa natomiast negatywnie na czytelność kodu.

# Nadpisywanie Equals dla typów wartościowych
Wykorzystanie metody `Equals(object obj)` w przypadku instancji typów wartościowych wiąże się z wystąpieniem zjawiska *boxingu*. Polega ono na "opakowaniu wartości w obiekt", czyli utworzeniu kopii wartości na stercie oraz nadanie jej referencji. Jest to dość kosztowna operacja, która w niektórych przypadkach może zostać ominięta dzięki implementacji interfejsu `IEquatable<T>`.

Ponieważ struktury nie mogą być równe *null* (z wyjątkiem `Nullable<T>`), a porównanie instancji dwóch typów wartościowych na podstawie referencji zawsze zwróci *false*, nadpisywanie metody `Equals` dla typów wartościowych jest łatwiejsze niż dla typów referencyjnych.

Lista kroków:

1. sprawdzenia czy przekazany obiekt nie jest równy *null*,
2. sprawdzenia czy przekazany obiekt ma pasujący typ,
3. porównanie przechowywanych wartości.

*Pomimo, iż typy wartościowe nie mogą być równe null, metoda `Equals(object obj)` przyjmuje jako parametr dowolny obiekt, którego instancja nie musi być typem wartościowym.*

Przykładowa implementacja metody `Equals` dla struktury `Person`:

``` csharp
public struct Person : IEquatable<Person>
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

	public bool Equals(Person other)
	{
	    return string.Equals(_firstName, other._firstName) 
	           && string.Equals(_lastName, other._lastName);
	}

	public override bool Equals(object obj)
	{
		// Instance of struct cannot be null, but object can.
	    if (ReferenceEquals(null, obj))
	    {
	        return false;
	    }
	
	    return obj is Person person && Equals(person);
	}

	// Remember to override method `GetHashCode`!
}
```

# Podsumowanie
Mam nadzieję, że artykuł ten wyjaśnił w jaki sposób poprawnie nadpisywać metodę `Equals`. Starałem się, aby opisane tu przykłady były uniwersalne i stanowiły pewnego rodzaju szablon, który możesz wykorzystać w przyszłości.

Porównanie wydajności różnych wersji metod `Equals` dla typów referencyjnych oraz wartościowych przedstawione zostanie w części V serii.
