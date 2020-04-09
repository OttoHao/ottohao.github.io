---
layout: post
title:  "JWT Authentication in ASP.NET Core 3.1"
date:   2020-04-08 18:03:18 +0800
categories: netcore auth
---

Authentication is the process of determining a user's identity. In this post we'll go through a simple example of JWT (JSON Web Token) authentication in an ASP.NET Core 3.1 API with C#.

The source code can be found in [Github](https://github.com/OttoHao/JwtAuthDemo).

The implementation can also be done in ASP.NET Core 2.2.

## Content

1. [Initialize ASP.NET Core 3.1 Web API Project](#1-initialize-aspnet-core-31-web-api-project)
2. [Authentication Scheme and Handler](#2-authentication-scheme-and-handler)
3. [Authentication Middleware](#3-authentication-middleware)
4. [JWT Authentication Service](#4-jwt-authentication-service)
5. [JWT Authentication Controller](#5-jwt-authentication-controller)
6. [Postman Test](#6-postman-test)

## 1. Initialize ASP.NET Core 3.1 Web API Project

Generate a new ASP.NET Core 3.1 Web API Project by running `dotnet new`.

```
dotnet new webapi -n JwtAuthDemo
```

The default `WeatherForecastController` provide a GET api which do not need any authentication yet.

![GetWithoutAuth]({{site.baseurl}}/assets/images/jwtAuthDemo-GetWithoutAuth.PNG)

## 2. Authentication Scheme and Handler

An authentication scheme is a name which corresponds to:

- An authentication handler.
- Options for configuring that specific instance of the handler.

An authentication handler:

- Is a type that implements the behavior of a scheme.
- Has the primary responsibility to authenticate users.

In `Startup.cs` class `ConfigureServices` method

```csharp
var jwtKey = Configuration.GetValue<string>("JwtKey"); // read from appsettings.json
var key = Encoding.ASCII.GetBytes(jwtKey);

services.AddAuthentication(x =>
{
    x.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    x.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
}).AddJwtBearer(x =>
{
    x.RequireHttpsMetadata = false;
    x.SaveToken = true;
    x.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(key),
        ValidateIssuer = false,
        ValidateAudience = false
    };
});
```

### 2.1 Source Code Analysis

In `AddAuthentication` extension method ([source code](https://github.com/aspnet/HttpAbstractions/blob/master/src/Microsoft.AspNetCore.Authentication.Core/AuthenticationCoreServiceCollectionExtensions.cs)), serveral services are registed, such as `AuthenticationService`, `AuthenticationHandlerProvider`, `AuthenticationSchemeProvider`. And the `AuthenticationOptions` is configed with `DefaultAuthenticateScheme` and `DefaultChallengeScheme`.

```csharp
services.TryAddScoped<IAuthenticationService, AuthenticationService>();
services.TryAddScoped<IAuthenticationHandlerProvider, AuthenticationHandlerProvider>();
services.TryAddSingleton<IAuthenticationSchemeProvider, AuthenticationSchemeProvider>();
```

In `AddJwtBearer` extension method ([source code](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication.JwtBearer/JwtBearerExtensions.cs))

```csharp
builder.AddScheme<JwtBearerOptions, JwtBearerHandler>(authenticationScheme, displayName, configureOptions)
```

The registed `JwtBearerHandler` is an authentication handler derived from `IAuthenticationHandler` ([source code](https://github.com/aspnet/HttpAbstractions/blob/master/src/Microsoft.AspNetCore.Authentication.Abstractions/IAuthenticationHandler.cs)).

```csharp
public interface IAuthenticationHandler
{
    Task InitializeAsync(AuthenticationScheme scheme, HttpContext context);
    Task<AuthenticateResult> AuthenticateAsync();
    Task ChallengeAsync(AuthenticationProperties properties);
    Task ForbidAsync(AuthenticationProperties properties);
}
```

## 3. Authentication Middleware

In ASP.NET Core, Middleware is software that's assembled into an app pipeline to handle requests and responses.
In `Startup.cs` class `Configure` method, add `UseAuthentication` after `UseRouting` while before `UseEndpoints`.

```csharp
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.UseEndpoints(endpoints =>
{
    endpoints.MapControllers();
});
```

### 3.1 Source Code Analysis

In `UseAuthentication` extension method ([source code](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication/AuthAppBuilderExtensions.cs))

```csharp
app.UseMiddleware<AuthenticationMiddleware>();
```

In `AuthenticationMiddleware` ([source code](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication/AuthenticationMiddleware.cs)), use injected `IAuthenticationSchemeProvider` to get authentication scheme, and call `AuthenticateAsync` extension method on `HttpContext` to authenticate.

```csharp
var defaultAuthenticate = await (IAuthenticationSchemeProvider)Schemes.GetDefaultAuthenticateSchemeAsync();
if (defaultAuthenticate != null)
{
    var result = await context.AuthenticateAsync(defaultAuthenticate.Name);
    if (result?.Principal != null)
    {
        context.User = result.Principal;
    }
}
```

The `AuthenticateAsync` ([source code](https://github.com/aspnet/HttpAbstractions/blob/master/src/Microsoft.AspNetCore.Authentication.Abstractions/AuthenticationHttpContextExtensions.cs)) extension method on `HttpContext` calls `AuthenticateAsync` method of `IAuthenticationService`.

```csharp
public static Task<AuthenticateResult> AuthenticateAsync(this HttpContext context, string scheme)
{
    return context.RequestServices.GetRequiredService<IAuthenticationService>().AuthenticateAsync(context, scheme);
}
```

`AuthenticationService` ([source code](https://github.com/aspnet/HttpAbstractions/blob/master/src/Microsoft.AspNetCore.Authentication.Core/AuthenticationService.cs)) just wraps `IAuthenticationSchemeProvider` and `IAuthenticationHandlerProvider`.

```csharp
var handler = await (IAuthenticationHandlerProvider)Handlers.GetHandlerAsync(context, scheme);
var result = await handler.AuthenticateAsync();
```

Here is a simple diagram to summarize all the relations.

<p align="center">
<img src="{{site.baseurl}}/assets/images/AspNetCoreAuthentication.PNG" alt="AspNetCoreAuthentication" width="600"/>
</p>

## 4. JWT Authentication Service

In `Models` folder create `UserCred` class.

```csharp
public class UserCred
{
    public string Username { get; set; }
    public string Password { get; set; }
}
```

In `Services` folder create `IJwtAuthService` interface and `JwtAuthService` class.

```csharp
public interface IJwtAuthService
{
    string Authenticate(string username, string password);
}
```

```csharp
public class JwtAuthService : IJwtAuthService
{
    private readonly byte[] _tokenKey;

    private readonly IDictionary<string, string> _users = new Dictionary<string, string>
    {
        { "test123", "123" },
        { "test234", "234" }
    };

    public JwtAuthService(byte[] tokenKey)
    {
        _tokenKey = tokenKey;
    }

    public string Authenticate(string username, string password)
    {
        if (!_users.Any(u => u.Key == username && u.Value == password))
        {
            return null;
        }

        var tokenHandler = new JwtSecurityTokenHandler();
        var tokenDescriptor = new SecurityTokenDescriptor
        {
            Subject = new ClaimsIdentity(new Claim[]
            {
                new Claim(ClaimTypes.Name, username)
            }),
            Expires = DateTime.UtcNow.AddHours(1),
            SigningCredentials = new SigningCredentials(
                new SymmetricSecurityKey(_tokenKey),
                SecurityAlgorithms.HmacSha256Signature)
        };
        var token = tokenHandler.CreateToken(tokenDescriptor);
        return tokenHandler.WriteToken(token);
    }
}
```

In `Startup.cs` class

```csharp
services.AddSingleton<IJwtAuthService>(new JwtAuthService(key)); // the same key in AddJwtBearer
```

## 5. JWT Authentication Controller

In `Controllers` folder create `JwtAuthController` which injects `JwtAuthService`. The `Authenticate` action should be decorated with `AllowAnonymous` attribute. And add `Authorize` attribute in controllers or methods as you want. In this case, `Authorize` attribute is added in `WeatherForecastController`.

```csharp
[Authorize]
[ApiController]
[Route("[controller]")]
public class JwtAuthController : ControllerBase
{
    private readonly IJwtAuthService _jwtAuthService;

    public JwtAuthController(IJwtAuthService jwtAuthService)
    {
        _jwtAuthService = jwtAuthService;
    }
    [AllowAnonymous]
    [HttpPost]
    public IActionResult Authenticate([FromBody] UserCred userCred)
    {
        var token = _jwtAuthService.Authenticate(userCred.Username, userCred.Password);

        if (token == null)
            return Unauthorized();

        return Ok(token);
    }
}
```

## 6. Postman Test

Since `Authorize` attribute is added in `WeatherForecastController`, the GET api will return 401 if there is no Bearer authorization header.

![GetWithoutAuth2]({{site.baseurl}}/assets/images/jwtAuthDemo-GetWithoutAuth2.PNG)

POST username and password to get JWT token.

![PostUserCredToGetToken]({{site.baseurl}}/assets/images/jwtAuthDemo-PostUserCredToGetToken.PNG)

Try GET api again with JWT token.

![GetWithAuth]({{site.baseurl}}/assets/images/jwtAuthDemo-GetWithAuth.PNG)