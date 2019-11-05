---
title: IdentityServer, HttpClient и всё такое
date: "2019-10-10 12:00:00 +0300"
id: identity-server-http-client-and-everything
excerpt: Два рабочих дня потречены, чтобы разобраться, почему HttpClient так странно работает.
---

Давно надо было опробовать **IdentityServer4**, который Microsoft выпустила три года назад, в декабре 2016-го.

**IdentityServer4** — это полуфабрикат, из которого легко собрать работающий сервер аутентификации за двадцать минут. Если вы знаете, как его собирать.

Он реализует стандарты OpenID Connect и OAuth 2.0, он стыкуется с Google, Facebook и Azure Active Directory. Если вы разрабатываете Web API, берите **IdentityServer4** для аутентификации — не ошибётесь.

Так я думал вначале недели. Сегодня четверг, и я у меня только что всё заработало. Почему так долго?

### Радужное начало

В интернете есть [документация](http://docs.identityserver.io/en/latest/), с помощью которой можно быстро разработать серверный и клиентский код аутентификации.

Кратко процесс выглядит так: создаём ASP.NET Core приложение, устанавливаем пакет [IdentityServer4](https://www.nuget.org/packages/IdentityServer4/), подключаем готовые модули, конфигурируем и — наслаждаемся работающей аутентификацией.

Первый сценарий, о котором рассказывает руководство — это [Client Credentials](http://docs.identityserver.io/en/latest/quickstarts/1_client_credentials.html). В этом сценарии нет даже пользователей, он применяется, когда два независимых сервиса должны безопасно работать вместе.

Решение из этой главы состоит из трёх проектов — **IdentityServer**, **Api** и **Client**. **IdentityServer** — это приложение Web API, которое отвечает за аутентификацию и умеет создавать токены. **Api** — веб-приложение, которое отдаёт инофрмацию только доверенному клиенту. **Client** — консольное приложение, тот самый клиент.

> В терминах стандартов OpenID и OAuth *Клиент* — это программа, а не пользователь.

Вначале клиент должен подключиться к серверу аутентификации и получить токен доступа. Код в статье выглядит так:

```c#
var client = new HttpClient();

var disco = await client.GetDiscoveryDocumentAsync("http://localhost:5000");
if (disco.IsError)
{
    Console.WriteLine(disco.Error);
    return;
}

// request token
var tokenResponse = await client.RequestClientCredentialsTokenAsync(new ClientCredentialsTokenRequest
{
    Address = disco.TokenEndpoint,
    ClientId = "console",
    ClientSecret = "secret",

    Scope = "api1"
});
```

И он не работет. После вызова `GetDiscoveryDocumentAsync` объект `disco` содержит сообщение об ошибке: Error connecting to http://localhost:5000/.well-known/openid-configuration: Service Unavailable.

Если я захожу по адресу http://localhost:5000/.well-known/openid-configuration, то вижу, что сервис работает и возвращает мне JSON.

```javascript
{
  "issuer":"http://localhost:5000",
  "jwks_uri":"http://localhost:5000/.well-known/openid-configuration/jwks",
  "authorization_endpoint":"http://localhost:5000/connect/authorize",
  "token_endpoint":"http://localhost:5000/connect/token",
  "userinfo_endpoint":"http://localhost:5000/connect/userinfo",
  "end_session_endpoint":"http://localhost:5000/connect/endsession",
  "check_session_iframe":"http://localhost:5000/connect/checksession",
  "revocation_endpoint":"http://localhost:5000/connect/revocation",
  "introspection_endpoint":"http://localhost:5000/connect/introspect",
  "device_authorization_endpoint":"http://localhost:5000/connect/deviceauthorization",
  "frontchannel_logout_supported":true,
  "frontchannel_logout_session_supported":true,
  "backchannel_logout_supported":true,
  "backchannel_logout_session_supported":true,
  "scopes_supported": [ "openid", "api1","offline_access" ],
  "claims_supported": [ "sub" ],
  "grant_types_supported": [ "authorization_code", "client_credentials", "refresh_token", "implicit", "urn:ietf:params:oauth:grant-type:device_code" ],
  "response_types_supported": [ "code", "token", "id_token", "id_token token", "code id_token", "code token", "code id_token token" ],
  "response_modes_supported": [ "form_post", "query", "fragment" ],
  "token_endpoint_auth_methods_supported": [ "client_secret_basic", "client_secret_post" ],
  "id_token_signing_alg_values_supported": [ "RS256" ],
  "subject_types_supported": [ "public" ],
  "code_challenge_methods_supported": [ "plain","S256" ],
  "request_parameter_supported":true
}
```

Почему же не работает консольная программа?

### Первая попытка

К счастью, объект `disco` содержит не только поля, возвращаемые сервером, но и данные HTTP-ответа. А HTTP-ответ в .NET ссылается на HTTP-запрос, так что мы можем попытаться понять, что у нас не так.

Поле `HttpResponse.RequestMessage.RequestUri` имеет значение **http://192.168.7.24/**. И, кажется, этот адрес совершенно не похож на **http://localhost:5000**, который мы видим в коде. **192.168.7.24** — IP-адрес моей машины, но почему вдруг **localhost** разрешается в него, а не в **127.0.0.1** и куда пропал порт 5000?

Сначала я решил проверить, а прослушивает ли **IdentityServer** внешний порт. Оказалось, что нет, потому что в настройках явно был указан хост **http://localhost:5000**.

**launchSettings.json**
```javascript
{
  "iisSettings": {
    "windowsAuthentication": false,
    "anonymousAuthentication": true,
    "iisExpress": {
      "applicationUrl": "http://localhost:5000",
      "sslPort": 0
    }
  },
  "profiles": {
    "IIS Express": {
      "commandName": "Project",
      "launchBrowser": true,
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "SelfHost": {
      "commandName": "Executable",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      },
      "applicationUrl": "http://localhost:5000",
      "executablePath": "IdentityServer.exe"
    }
  }
```

Чтобы сервер прослушивал все интерфейсы IPv4, в качестве `applicaitonUri` надо указать **http://0.0.0.0:5000**, а чтобы все интерфейсы IPv4 и IPv6 — **http://*:5000**.

Но сервер продолжал быть недоступным. Я решил, что, возможно, дело в файрволе и открыл порт 5000 на внешнем интерфейсе, вызвав команду `netsh`.

```
netsh advfirewall firewall add rule name="HTTP 5000" dir=in action=allow protocol=TCP localport=5000
```

Chrome показывал мне правильный JSON, а клиент продолжал возвращать ошибку **503 Service Unavailable**.

### Вторая попытка

Я предположил, что дело в классе `HttpClient`, который осуществляет запрос к **IdentityServer** из приложения **Client**. На каком-то этапе он *искажает* URI, который я ему передал, и мне надо всего лишь выяснить, почему.

Обычно я неплох в Google. Однако, наступил момент, когда весь мой опыт мне не помог. Я придумывал десятки формулировок, пытаясь описать проблему разными способами, но интернет был безмолвен. `HttpClient` не изменяет переданный ему URI. Точка.

Тогда почему изменяет у меня? Что я сделал не так?

### Прозрение

Теперь я знаю, что я сделал не так. Я поменял работу. На старой работе у нас не было файрвола, мы сидели в интернете напрямую. Теперь я работаю в банке и файрвол у меня есть. Его параметры настроены в Windows и Chrome их использует. А `HttpClient` из коробки — нет.

Чтобы включить прокси, надо добавить к `HttpClient` обработчик с настроенными параметрами.

**Client\Program.cs**
```c#
var httpHandler = new HttpClientHandler
{
    Proxy = new WebProxy("192.168.7.100:8080")
    {
        BypassProxyOnLocal = true,
        UseDefaultCredentials = true,
    },
};

var client = new HttpClient(handler: httpHandler, disposeHandler: true);
```

Для обращения к серверу **Api** консальная программа **Client** использует ещё один экземпляр `HttpClient`, который называется `apiClient`. Его тоже надо проинициализировать нашим обработчиком.

```c#
var apiClient = new HttpClient(handler: httpHandler, disposeHandler: true);
```

Наконец, сервер **Api** сам обращается к **IdentityServer**, чтобы узнать, можно ли доверять клиенту. Это другой проект, поэтому мы должны продублировать код обработчика.

**Api\Startupcs**
```c#
services.AddAuthentication("Bearer")
        .AddJwtBearer("Bearer", options =>
        {
            options.Authority = "http://localhost:5000/";
            options.RequireHttpsMetadata = false;
            options.Audience = "api1";
            options.BackchannelHttpHandler = new HttpClientHandler
            {
                Proxy = new WebProxy("192.168.7.100:8080")
                {
                    BypassProxyOnLocal = true,
                    UseDefaultCredentials = true,
                },
            };
        });
```

Вот теперь консольное приложение показывает мне именно то, что я жду.

![Вывод программы Client из документации по IdentityServer4](/img/identity-server-console.png)

### Заключение

Как я догадался, что дело в файрволе? Я не помню. Мне бы хотелось поделиться секретом решения нетривиальных задач, но как и в других случаях, я его не знаю. Я просто перебирал в голове разные варианты, пока не наткнулся на очевидный. Я его проверил и он сработал.

Такова работа программиста: делаешь что-то и надеешься, что рано или поздно что-нибудь получится. Странный вывод через тридцать лет работы программистом, не правда ли?