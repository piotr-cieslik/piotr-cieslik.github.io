---
title: "Generowanie własnego tokenu JWT w ASP.NET Core"
date: 2023-05-13
tags: ["c#"]
---

Niedawno otrzymałem zadanie dodania REST API dla istniejącej już aplikacji ASP.NET MVC 5, którą na potrzeby tego wpisu nazywać będę aplikacją X. Zadanie nie wydawało się zbyt skomplikowane. Nowe API miało być osobną aplikacja, udostępnionym pod nowym adresem URL, która obsługiwać miała dziennie maksymalnie kilkaset zapytań.

Otwartą kwestią było jednak to, w jaki sposób autoryzować otrzymane zapytania. Ponieważ jestem wielkim fanem tokenów JWT ([link do opisu](https://jwt.io/introduction)) naturalnym wyborem wydawało się skorzystanie z jednego z darmowych serwisów pozwalających na ich generowanie, takich jak [Auth0.com](https://auth0.com/) lub platformy [Azure](https://azure.microsoft.com/).

Nasz aplikacja X sama zarządza kontami użytkowników, co rodzi pewne problem z autentykacją poprzez zewnętrzne serwisy. Jest to zadanie wykonalne ale sytuacja zaczynała się komplikować. Dodatkowo zależało mi na prostocie. Chciałem, aby użytkownicy naszego API pobierali token bezpośrednio od nas, a następnie dodawali go do nagłówków zapytań.

W związku z powyższym porzuciłem pomysł autentykacji z wykorzystaniem tokenu JWT oraz zabrałem się za implementacje autentykacji na podstawie generowanych przez nas kluczy API. Było to klasyczne podejście stosowane w milionach projektów na cały świecie. Jednak i to rozwiązanie okazało się dalekie od ideału. Klucze API trzeba gdzieś przechowywać, a nie chciałem dodawać tej informacji do głównej bazy danych projektu. Oznaczało to, że potrzebowałem nowej bazy. Nowa baza to więcej kodu do napisania, bardziej skomplikowany projekt oraz dodatkowe  koszty związane z jej utrzymaniem.

Mając przed sobą dwa nieodpowiednie rozwiązania zacząłem się zastanawiać co mogę zrobić. Skłaniałem się do rozwiązania numer dwa. Wychodząc jednak z założenia, że pisanie własnego mechanizmu autoryzacji to zawsze zły pomysł zacząłem szukać innych opcji. I tak natrafiłem na paczkę [`Microsoft.IdentityModel.Tokens`](https://www.nuget.org/packages/Microsoft.IdentityModel.Tokens) pozwalającą na wystawianie własnych tokenów JWT. Pomyślałem, że warto spróbować.

Kilka uwag zanim zerkniemy na implementacje:
- W dalszej części artykułu niezbędna jest podstawowa biegłość w tworzeniu API w platformie ASP.NET Core. Jeśli nie tworzyłeś nigdy takiego API, polecam zapoznać się wcześniej z jakimś kursem dostępnym w sieci.
- W dalszej części artykułu niezbędna jest podstawowa wiedza na temat budowy oraz zasady działania tokenu JWT. Garść przydatnych informacji można znaleźć na stronie [https://jwt.io/introduction](https://jwt.io/introduction).
- Wygenerowany przez nas token podpisany będzie z wykorzystaniem klucza symetrycznego. Oznacza to, że do podpisu oraz walidacji tokenu wykorzystany jest ten sam klucz.


# Implementacja

*Całość omawianego poniżej kodu dostępna jest tutaj: [Blog.SelfSignedJWT](https://github.com/piotr-cieslik/Blog.SelfSignedJWT).*

Zacznijmy od zainstalowania dwóch potrzebnych nam paczek:
1. `Microsoft.IdentityModel.Tokens`
1. `Microsoft.AspNetCore.Authentication.JwtBearer`

Pierwsza z nich posłuży nam do generowania tokenu JWT, druga do jego walidacji. W moim przypadku, dla aplikacji pisanej w .net 6, były to paczki w wersjach 6.30.1 oraz 6.0.16.

Mając zainstalowane niezbędne paczki, omówmy szybko podstawowe elementy tworzonej aplikacji.
- Klasa `JwtOptions` zawierająca konfiguracje naszego tokenu, w tym klucz służący do jego podpisu.
- Klasa `TestController` wystawiająca endpoint `/test`, służący do testowania naszego tokenu.
- Klasa `AuthController` wystawiająca endpoint `/auth/token` służący do generowania tokenu na podstawie podanego adresu email oraz hasła.
- Klasy `TokenRequestDto` oraz `TokenResponseDto` będące modelami.

Zacznijmy od implementacji klasy `JwtOptions`. Wystawia ona jedynie trzy pola `Key`, `Issuer` oraz `Audience`. Klasa zaimplementowana została z wykorzystaniem [options pattern](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options?view=aspnetcore-6.0).

``` csharp
public class JwtOptions
{
    public const string Jwt = "Jwt";

    public string Key { get; set; }

    public string Issuer { get; set; }

    public string Audience { get; set; }
}
```

Po jej zdefiniowaniu musimy dodać odpowiednie wpisy w pliku `appsettings.json`
``` json
{
  // ... Rest of the file
  "Jwt": {
    "Key": "12345678901234567890123456789012", // At lest 32 digits key
    "Issuer": "https://example.com/",
    "Audience": "https://example.com/"
  }
}

```

oraz skonfigurować `IOption<JwtOptions>` w pliku `Program.cs`.
``` csharp
builder.Services.Configure<JwtOptions>(
    builder.Configuration.GetSection(JwtOptions.Jwt));
```

Następnie zacznijmy pisać implementacje naszego *endpointu* `/auth/token`. Na ten moment bez kodu, który rzeczywiście generuje token.
``` csharp
[ApiController]
[Route("auth")]
public class AuthController : ControllerBase
{
    private readonly IOptions<JwtOptions> _jwtOptions;

    public AuthController(IOptions<JwtOptions> jwtOptions)
    {
        _jwtOptions = jwtOptions;
    }

    [AllowAnonymous]
    [HttpPost("token")]
    public IActionResult Token([FromBody] TokenRequestDto dto)
    {
        // Check if user is valid here.
        // Please do it in more secure way :)
        const string expectedEmail = "user@example.com";
        const string expectedPassword = "1234";

        var authenticated =
            dto.Email== expectedEmail &&
            dto.Password == expectedPassword;

        if (!authenticated)
        {
            return BadRequest("Invalid email or password");
        }

        // TODO Generate access token here.
        var accessToken =
            default(string);
        var expiresAt =
            default(DateTime);
        return Ok(new TokenResponseDto
        {
            AccessToken = accessToken,
            ExpiresAt = expiresAt,
        });
    }
}

public sealed class TokenRequestDto
{
    public string Email { get; set; }

    public string Password { get; set; }
}

public sealed class TokenResponseDto
{
    public string AccessToken { get; set; }

    public DateTime ExpiresAt { get; set; }
}
```

Jak pewnie się domyślasz, cały kod generujący token umieszczony będzie w miejscu, gdzie teraz znajduje się komentarz z TODO.

Implementacja *endpointu* `/test` jest banalnie prosta, jednak warto ją tutaj pokazać:
``` csharp
[ApiController]
[Route("test")]
[Authorize]
public class TestController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        return NoContent();
    }
}
```

## Generowanie tokenu JWT

Zajmijmy się teraz generacją samego tokenu. W tym celu skorzystamy z dwóch klas [`JwtSecurityTokenHandler`](https://learn.microsoft.com/en-us/dotnet/api/system.identitymodel.tokens.jwt.jwtsecuritytokenhandler?view=msal-web-dotnet-latest) oraz [`JwtSecurityToken`](https://learn.microsoft.com/en-us/dotnet/api/system.identitymodel.tokens.jwt.jwtsecuritytoken?view=msal-web-dotnet-latest). Pierwsza z nich służy do generowania ciągu znaków będących naszym token. Druga z kolei stanowi model naszego tokenu i to na niej musimy się skupić.

Klasa `JwtSecurityToken` zawiera aż cztery konstruktory, przy czym ja posłużę się ostatnim z nich, przyjmującym następujące wartości:

``` csharp
public JwtSecurityToken (
    string issuer = default,
    string audience = default,
    System.Collections.Generic.IEnumerable<System.Security.Claims.Claim> claims = default,
    DateTime? notBefore = default,
    DateTime? expires = default,
    Microsoft.IdentityModel.Tokens.SigningCredentials signingCredentials = default
);
```

Wartości dla parametrów `issuer` oraz `audience` przepiszemy bezpośrednio z konfiguracji, wartości dla `notBefore` oraz `expires` policzmy na podstawie aktualnej daty. Pozostają nam jedynie dwa parametry `claims` oraz `signingCredentials`, które są trochę bardziej złożone.

Jeśli pracowałeś kiedyś z użytkownikami w ASP.NET stworzenie kolekcji `claims`, nie powinno być żadnym problemem. Jest to zbiór elementów (klucz, wartość), które dodane będą do danych przenoszonych przez nasz token. Jeśli nie chcemy wymyślać koła na nowo, warto posłużyć się predefiniowaną kolekcją kluczy dostępną w klasach `JwtRegisteredClaimNames` lub `ClaimTypes`.

``` csharp
// Assume our user has id: 1.
const int userId = 1;
var claims =
    new List<Claim>()
    {
        new (JwtRegisteredClaimNames.Sub, userId.ToString()),
        new (ClaimTypes.Email, dto.Email),
    };
```

Przechodzimy następnie do zdefiniowania `signingCredentials` na podstawie naszego sekretnego klucza oraz wybranego przez nas algorytmu szyfrowania. 

``` csharp
// Read options from configuration.
var options = _jwtOptions.Value;
var signingCredentials =
    new SigningCredentials(
        new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(options.Key)),
        SecurityAlgorithms.HmacSha256);
```

Wiedząc już w jaki sposób stworzyć wszystkie wartości niezbędne do stworzenia obiektu klasy `JwtSecurityToken` zajmijmy się wygenerowaniem naszego tokenu.

``` csharp
// Assume our user has id: 1.
const int userId = 1;

// Read options from configuration.
var options = _jwtOptions.Value;

// Take snapshot of current time and calculate when token expires.
var now = DateTime.UtcNow;
var expiresAt = now.AddMinutes(15);

// Create SigningCredentials using symmetric key from settings.
var signingCredentials =
    new SigningCredentials(
        new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(options.Key)),
        SecurityAlgorithms.HmacSha256);

// Optionally define list of extra claims for token.
var claims =
    new List<Claim>()
    {
        new (JwtRegisteredClaimNames.Sub, userId.ToString()),
        new (ClaimTypes.Email, dto.Email),
    };

// Create a token model.
var token =
    new JwtSecurityToken(
        issuer: options.Issuer,
        audience: options.Audience,
        claims: claims,
        notBefore: now,
        expires: expiresAt,
        signingCredentials: signingCredentials);

// Generate an access token.
var accessToken =
    new JwtSecurityTokenHandler().WriteToken(token);
```

Podsumowując, nasz endpoint generujący token wygląda jak poniżej.

``` csharp
using Blog.SelfSignedJWT.Models;
using Microsoft.AspNetCore.Authorization;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using Microsoft.AspNetCore.Mvc;
using Microsoft.IdentityModel.Tokens;
using Microsoft.Extensions.Options;

namespace Blog.SelfSignedJWT.Controllers;

[ApiController]
[Route("auth")]
public class AuthController : ControllerBase
{
    private readonly IOptions<JwtOptions> _jwtOptions;

    public AuthController(IOptions<JwtOptions> jwtOptions)
    {
        _jwtOptions = jwtOptions;
    }

    [AllowAnonymous]
    [HttpPost("token")]
    public IActionResult Token([FromBody] TokenRequestDto dto)
    {
        // Check if user is valid here.
        // Pleas do it in more secure way :)
        const string expectedEmail = "user@example.com";
        const string expectedPassword = "1234";

        var authenticated =
            dto.Email== expectedEmail &&
            dto.Password == expectedPassword;

        if (!authenticated)
        {
            return BadRequest("Invalid email or password");
        }
        
        // Assume our user has id: 1.
        const int userId = 1;
        
        // Read options from configuration.
        var options = _jwtOptions.Value;

        // Take snapshot of current time and calculate when token expires.
        var now = DateTime.UtcNow;
        var expiresAt = now.AddMinutes(15);

        // Create SigningCredentials using symmetric key from settings.
        var signingCredentials =
            new SigningCredentials(
                new SymmetricSecurityKey(
                    Encoding.UTF8.GetBytes(options.Key)),
                SecurityAlgorithms.HmacSha256);

        // Optionally define list of extra claims for token.
        var claims =
            new List<Claim>()
            {
                new (JwtRegisteredClaimNames.Sub, userId.ToString()),
                new (ClaimTypes.Email, dto.Email),
            };

        // Create a token model.
        var token =
            new JwtSecurityToken(
                issuer: options.Issuer,
                audience: options.Audience,
                claims: claims,
                notBefore: now,
                expires: expiresAt,
                signingCredentials: signingCredentials);

        // Generate an access token.
        var accessToken =
            new JwtSecurityTokenHandler().WriteToken(token);

        return Ok(new TokenResponseDto
        {
            AccessToken = accessToken,
            ExpiresAt = expiresAt,
        });
    }
}
```

Pozostaje nam jedynie przetestowanie naszego kodu. Uruchamiamy nasz program (w moim przypadku jest on hostowany pod adresem https://localhost:7141), a następnie wysyłamy zapytanie HTTP jak poniżej.

```
POST https://localhost:7141/auth/token
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "1234"
}
```

Otrzymana odpowiedź.
``` json
{
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxIiwiaHR0cDovL3NjaGVtYXMueG1sc29hcC5vcmcvd3MvMjAwNS8wNS9pZGVudGl0eS9jbGFpbXMvZW1haWxhZGRyZXNzIjoidXNlckBleGFtcGxlLmNvbSIsIm5iZiI6MTY4Mzg3NjYyNywiZXhwIjoxNjgzODc3NTI3LCJpc3MiOiJodHRwczovL2V4YW1wbGUuY29tLyIsImF1ZCI6Imh0dHBzOi8vZXhhbXBsZS5jb20vIn0.kt_25dyrScfDhRxa5oOFRLPaB4B6dAArXbaJ3b8Sce0",
    "expiresAt": "2023-05-12T07:45:27.4154647Z"
}
```

Jak widać wszystko działa. Dodatkowo poprawność naszego tokenu możemy sprawdzić na stronie [https://jwt.io/](https://jwt.io).

## Walidacja tokenu JWT

Walidacja poprawności tokenu JWT w ASP.NET Core jest dość prosta i zapewne znajdziesz w sieci wiele artykułów na ten temat. Postanowiłem jednak wspomnieć o niej tutaj, ponieważ nasz token podpisany jest za pomocą klucza symetrycznego. Oznacza to, że musimy przekazać informacje o kluczu oraz wykorzystywanym algorytmie szyfrowania do mechanizmu walidacji.

Całość konfiguracji odbywa się w pliku `Program.cs`.

``` csharp
// Configure Authentication
builder.Services
    .AddAuthentication(
        options =>
        {
            options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
            options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
        })
    .AddJwtBearer(
        options =>
        {
            options.TokenValidationParameters =
                new()
                {
                    ValidateIssuer = true,
                    ValidateAudience = true,
                    ValidateLifetime = true,
                    ValidateIssuerSigningKey = true,
                    // Keep in mind to specify audience, issuer, and key
                    ValidAudience = builder.Configuration["Jwt:Audience"],
                    ValidIssuer = builder.Configuration["Jwt:Issuer"],
                    IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]))
                };
        });
```

Dodatkowo należy upewnić się, że w naszej aplikacji wykorzystujemy zarówno *AuthenticationMiddleware* jak i *AuthorizationMiddleware*. Bez nich uwierzytelnienie nie będzie możliwe.

``` csharp
// Turn on Authentication service.
app.UseAuthentication();

// Add Authorization middleware.
app.UseAuthorization();
```

Pozostało nam więc tylko wysłanie zapytania do naszego testowego *endpointu* bez tokenu oraz z wygenerowanym właśnie tokenem i sprawdzenie jaką otrzymamy odpowiedź.

Zapytanie bez tokenu, zwraca odpowiedź z kodem 401: Unauthorized.
```
GET https://localhost:7141/auth/token
```

Zapytanie z tokenem, zwraca odpowiedź 204: No Content.
```
GET https://localhost:7141/auth/token
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxIiwiaHR0cDovL3NjaGVtYXMueG1sc29hcC5vcmcvd3MvMjAwNS8wNS9pZGVudGl0eS9jbGFpbXMvZW1haWxhZGRyZXNzIjoidXNlckBleGFtcGxlLmNvbSIsIm5iZiI6MTY4Mzg3NjYyNywiZXhwIjoxNjgzODc3NTI3LCJpc3MiOiJodHRwczovL2V4YW1wbGUuY29tLyIsImF1ZCI6Imh0dHBzOi8vZXhhbXBsZS5jb20vIn0.kt_25dyrScfDhRxa5oOFRLPaB4B6dAArXbaJ3b8Sce0
```

PS: Warto w tym miejscu zauważyć, że domyślna konfiguracja walidacji tokenu JWT dopuszcza 5 minutowe przesunięcie czasowe (ang. clock skew). Oznacza to, że wystawiony przez nas token będzie wciąż możliwy do użycia przez 5 minut od momentu, w którym wygasł. Opcja ta może być zmieniona podczas konfiguracji mechanizmu walidacji tokenu.

# Podsumowanie
W powyższym artykule przedstawiłem sposób generowania tokenu JWT w .NET Core, a następnie jego walidacji. Pomimo, iż powyższy kod jest dość prost, samo zagadnienie autoryzacji i autentykacji użytkownika jest bardzo złożone.

Chciałbym wyraźnie podkreślić, iż przedstawiona powyżej implementacja była sensownym rozwiązaniem dla naszej aplikacji. Nie oznacza to jednak, że sprawdzi się ona w każdym projekcie. Osobiście, zaczynając nowy projekt, staram się korzystać z dostępnych na rynku narzędzi firm trzecich, które wyręczą mnie w tym temacie.