---
title: Сертификат HttpClient в Azure Key Vault
date: "2019-01-29 12:00:00 +0300"
id: storing-http-client-certificate-in-azure-key-vault
excerpt: Неделя потрачена на то, чтобы победить хранение сертификатов HttpClient в Azure Key Vault. Краткое резюме битвы.
---

## О чём речь

При работе с внешними REST API мы применяем `HttpClient`. Чтобы построить маршрут с помощью Google Maps Directions API мы пишем код, похожий на этот:

```c#
public class GoogleMapsDirectionsClient
{
    private const string _baseUri = "https://maps.googleapis.com/maps/api/directions/json";

    private readonly string _apiKey;

    public GoogleMapsDirectionsClient(string apiKey)
    {
        _apiKey = apiKey;
    }

    public async Task<Route> BuildRouteAsync(Location from, Location to)
    {
        var parameters = new Dictionary<string, string>
        {
            { "key", _apiKey },
            { "origin", string.Format(CultureInfo.InvariantCulture, "{0},{1}", from.Latitude, from.Longitude) },
            { "destination", string.Format(CultureInfo.InvariantCulture, "{0},{1}", to.Latitude, to.Longitude) },
        };

        var uri = BuildUri(parameters);

        using (var httpClient = new HttpClient())
        {
            var httpResponseMessage = await httpClient.GetAsync(uri);

            httpResponseMessage.EnsureSuccessStatusCode();

            var json = await httpResponseMessage.Content.ReadAsStringAsync();

            return JsonConvert.Deserialize<Route>(json);
        }
    }

    private static Uri BuildUri(IReadOnlyDictionary<string, string> queryStringParameters)
    {
        var builder = new UriBuilder(_baseUri);
        builder.Query = BuildQueryString(queryStringParameters);

        return builder.Uri;
    }

    private static string BuildQueryString(IReadOnlyDictionary<string, string> queryStringParameters)
    {
        var assigns = queryStringParameters.Select(x => x.Key + "=" + x.Value);

        return string.Join("&", assigns);
    }
}
```

Код простой, поскольку для работы с Google Maps Directions API не требует параноидальной защиты. Но когда дело касается денег, параноидальная защита становится нужна. Одним из средств такой защиты является *клиентский сертификат*.

Вы заключаете договор с финансовой компанией, например с такой, которая принимает оплату для вашего интернет-магазина. Компания выдаёт вам сертификат и пароль к нему.

> Сертификаты могут храниться в разных форматах. Azure поддерживает только формат PFX. Если финансовая компания выдала сертификат в другом формате, его нужно перекодировать с помощью утилиты [OpenSSL](https://www.openssl.org/).

Вот так выглядит подключение сертификата из файла:

```c#
public HttpClient CreateHttpClient()
{
    var certificate = new X509Certificate2(filename, password);
    var handler = new HttpClientHandler
    {
        ClientCertificates = { certificate }
    }

    return new HttpClient(handler);
}
```

Хранение сертификата вместе с программой не является хорошей идеей. Клиентский сертификат является средством проверки подлинности, так что его нельзя разбрасывать где попало.

Куда положить сертификат, если наше клиентское приложение живёт в Azure?

## Импорт сертификата в Хранилище ключей

В Azure для защищённого хранения сертификатов используют Key Vault, то есть *Хранилище ключей*.

Чтобы добавить сертификат в хранилище, откроем портал Azure, а в нём&nbsp;&mdash; Key Vault. В панели слева мы должны увидеть вкладку *Certificates*.

![Список сертификатов](/img/key-vault-1.png)

На вкладке в верхней панели нажмём *Generate/Import*. Укажем метод *Import*, загрузим сертификат, дадим ему имя и введём пароль.

![Импорт сертификата](/img/key-vault-2.png)

Мы увидим сертификат в списке. Откроем:

![Новый сертификат](/img/key-vault-3.png)

Дважды щёлкнем на записи *Current Version*, чтобы посмотреть параметры сертификата.

![Параметры нового сертификата](/img/key-vault-4.png)

Нам понадобится параметр *Secret Identifier*, поэтому сохраним его в текстовом файле.

## Регистрация приложения

Доступ к Хранилищу должен быть предоставлен нашему приложению, а для этого его надо зарегистрировать в *Azure Active Directory*.
Наберём *Azure Active Directory* в строке поиска в самом верху Портала:

![Azure Active Directory](/img/key-vault-5.png)

В панели слева выберем закладку *App registrations*:

![Список зарегистрированных приложений](/img/key-vault-6.png)

Нажмём кнопку *New application registration* и введём параметры приложения:

![Регистрация приложения](/img/key-vault-7.png)

Не имеет значения, что мы введём в *Sign-on URL*. Главное, сохраним поле *Name*&nbsp;&mdash; оно нам понадобится на следующем шаге. Добавив приложение, мы не увидим его в списке, поскольку установлен фильтр *My apps*. Переключимся на *All apps*, найдём своё приложение и откроем его.

![Параметры нового приложения](/img/key-vault-8.png)

Нам потребуется параметр, который называется *Application ID*.
Запишем и нажмём кнопку *Settings* в заголовке. Увидим панель с настройками:

![Дополнительные параметры нового приложения](/img/key-vault-9.png)

Нам нужна вкладка *Keys*. Как видите, мы можем создать здесь *пароли* (*Passwords*) и *публичные ключи* (*Public Keys*). Нам нужен новый пароль. Введём описание, выберем время жизни пароля и нажмём кнопку *Save* наверху:

![Создание пароля](/img/key-vault-10.png)

Здесь будьте осторожны. Портал показывает вам значение созданного пароля, чтобы вы могли его скопировать и сохранить. Сейчас самое время это сделать, потому что потом увидеть пароль в Портале будет нельзя. Об этом и предупреждение вверху экрана.

![Значение пароля](/img/key-vault-11.png)

## Политика доступа

Мы на предпоследнем шаге. Вернёмся в Хранилище ключей. В левой панели откроем вкладку *Access policies*:

![Список политик доступа](/img/key-vault-12.png)

Нажмём кнопку *Add new*. Выберем шаблон *Key, Secret, & Certificate Management*. В поле *Select principal* выберем приложение, которое мы зарегистрировали на предыдущем шаге. Я просил вас запомнить его название, сейчас оно пригодится.

![Список политик доступа](/img/key-vault-13.png)

В поле *Authorized application* оставим существующее значение *None selected*. Нажмём кнопку *Ok* и вернёмся к список политик доступа. **Очень важно** нажать кнопку *Save* в верхней части панели.

## Мы готовы к коду

Сейчас у вас на руках должны быть *Secret Identifier*, *Application ID* и пароль, созданный для доступа к нашему приложению.

Мы готовы к тому, чтобы запрограммировать загрузку сертификата из Хранилища и подключить его к `HttpClient`.
Предположим, за обращение к финансовому API в нашем приложении отвечает гипотетический класс `FinanceClient`. Я добавил в него гипотетический метод `GetCurrentWalletAsync`, который *как бы* получает из внешнего API кошелёк пользователя, обращаясь по защищённому каналу, подписанному клиентским сертификатом.

```c#
public class FinanceClient
{
    private readonly string _secretIdentifier;
    private readonly string _applicationId;
    private readonly string _password;

    public FinanceClient(string secretIdentifer, string applicationId, string password)
    {
        _secretIdentifier = secretIdentifier;
        _applicationId = applicationId;
        _password = password;
    }

    private async Task<HttpClient> CreateHttpClientAsync()
    {
        // https://docs.microsoft.com/ru-ru/azure/key-vault/key-vault-use-from-web-application
        // https://stackoverflow.com/questions/37033073/how-can-i-create-an-x509certificate2-object-from-an-azure-key-vault-keybundle
        using (var keyVaultClient = new KeyVaultClient(GetToken))
        {
            var secret = await keyVaultClient.GetSecretAsync(_secretIdentifier); // <- Secret Identifier
            var bytes = Convert.FromBase64String(secret.Value);
            var certificate = new X509Certificate2(bytes, (string)null, X509KeyStorageFlags.MachineKeySet);
            var handler = new HttpClientHandler
            {
                ClientCertificates = { certificate },
            };

            return new HttpClient(handler);
        }
    }

    private async Task<string> GetToken(string authority, string resource, string scope)
    {
        var authenticationContext = new AuthenticationContext(authority);
        ClientCredential clientCred = new ClientCredential(_applicationId, _password); // <- Application ID и пароль
        AuthenticationResult result = await authenticationContext.AcquireTokenAsync(resource, clientCred);

        if (result == null)
            throw new InvalidOperationException("Failed to obtain the JWT token");

        return result.AccessToken;
    }

    public async Task<Wallet> GetCurrentWalletAsync()
    {
        using (var httpClient = await CreateHttpClientAsync())
        {
            var httpResponseMessage = await httpClient.GetAsync("http://api.some-finance.tld/wallets/current");

            httpResponseMessage.EnsureSuccessStatusCode();

            var json = await httpResponseMessage.Content.ReadAsString();

            return JsonConvert.Deserialize<Wallet>(json);
        }
    }
}
```

Как видим, подключиться к Хранилищу непросто. Я ожидал, что это будет не сложнее, чем подлюкчение к SQL серверу, но на деле код пришлось разбить на два метода. В методе `CreateHttpClientAsync` вызывается конструктор `KeyVaultClient`, которому мы передаём обработчик для получения токена `GetToken`. Я и сам не понимаю, почему здесь надо делать именно так&nbsp;&mdash; документация не изобилует подробностями.

В любом случае, в январе 2019-го года этот код работает. Чтобы разобраться, времени ушла неделя, потому что информацию пришлось собирать по крупицам с десятка ресурсов.

## Оптимизация

В этом коде есть две проблемы. Первая заключается в том, что .NET Core 2 создавать вручную подключения `HttpClient` считается моветоном. Весто этого надо через DI-контейнер получать `IHttpClientFactory` и вызывать метод `CreateClient`.

Чтобы это работало, зарегистрируем клиент с уникальным именем. В классе `Startup` напишем:

```c#
services.AddHttpClient("certified")
        .ConfigurePrimaryHttpMessageHandler(() => 
        {
            using (var keyVaultClient = new KeyVaultClient(GetToken))
            {
                var secretIdentifier = Configuration.GetValue("CERTIFICATE_SECRET_IDENTIFIER", "");
                var secret = keyVaultClient.GetSecretAsync(secretIdentifier)
                                           .Result;
                var bytes = Convert.FromBase64String(secret.Value);
                var certificate = new X509Certificate2(bytes, (string)null, X509KeyStorageFlags.MachineKeySet);
                
                return new HttpClientHandler
                {
                    ClientCertificates = { certificate },
                };
            }
        });
```

Метод `GetToken` надо разместить в том же классе `Startup`. Выглядит он так:

```c#
private async Task<string> GetToken(string authority, string resource, string scope)
{
    var authenticationContext = new AuthenticationContext(authority);

    var applicationId = Configuration.GetValue("CERTIFICATE_APPLICATION_ID", "");
    var password = Configuration.GetValue("CERTIFICATE_PASSWORD", "");

    ClientCredential clientCred = new ClientCredential(applicationId, password);
    AuthenticationResult result = await authenticationContext.AcquireTokenAsync(resource, clientCred);

    if (result == null)
        throw new InvalidOperationException("Failed to obtain the JWT token");

    return result.AccessToken;
}
```

Мы обращаемся к конфигурации приложения, чтобы загрузить значения *Secret Identifer*, *Application ID* и пароль.

Проблема этого кода в том, что он синхронно вызывается при старте нашего приложения, а загрузить `certificate` через `KeyVaultClient` мы можем только асинхронно. Поэтому нам приходится приоставноить загрузку, чтобы должаться выполнения асинхронной операции.

Это не очень здорово, но найти хорошего обходного решения я не смог. Если знаете, напишите.

Реализация `IHttpClientFactory` решает проблему с производительностью, поскольку кэширует экземпляры `HttpClientHandler`. Если мы откажемся от фабрики в данном сценарии, мы можем сами кэшировать обработчик и повысим производительность. Для этого воспользуемся ленивой инициализацией.

```c#
public class FinanceClient : IDisposable
{
    private readonly string _secretIdentifier;
    private readonly string _applicationId;
    private readonly string _password;
    private readonly Lazy<Task<HttpClientHandler>> _lazyHandler;

    public FinanceClient(string secretIdentifer, string applicationId, string password)
    {
        _secretIdentifier = secretIdentifier;
        _applicationId = applicationId;
        _password = password;
        _lazyHandler = new Lazy<Task<HttpClientHandler>>(CreateHttpClientHandlerAsync);
    }

    protected async Task<HttpClientHandler> CreateHttpClientHandlerAsync()
    {
        using (var keyVaultClient = new KeyVaultClient(GetToken))
        {
            var secret = await keyVaultClient.GetSecretAsync(secretIdentifier);
            var bytes = Convert.FromBase64String(secret.Value);
            var certificate = new X509Certificate2(bytes, (string)null, X509KeyStorageFlags.MachineKeySet);
            return new HttpClientHandler
            {
                ClientCertificates = { certificate },
            };
        }
    }

    protected async Task<HttpClient> CreateHttpClientAsync()
    {
        var handler = await lazyHandler.Value;

        return new HttpClient(handler, disposeHandler: false);
    }
    . . .
    public void Dispose()
    {
        if (_lazyHandler.IsValueCreated)
            _lazyHandler.Value.Dispose();
    }
}
```

Код метода `CreateHttpClientAsync` упростился, благодаря тому, что существенная его часть перебралась в `CreateHttpClientHandlerAsync`. Нам приходится реализовать `IDisposable` потому что `HttpClient` теперь не освобождает `HttpClientHandler`.

Этот код не &laquo;подвисает&raquo; на старте приложения и работает быстро, но у него тоже есть проблема. Одной из причин появления фабрики `IHttpClientFactory` была ошибка в коде `HttpClientHandler`, из-за которой запросы к DNS кэшировались и не обновлялись. Чтобы с этим справиться, &laquo;встроенная&raquo; реализация `DefaultHttpClientFactory` пересоздаёт обработчики один раз в несколько минут.

Мы можем сделать то же самое в нашем коде. Он станет сложнее, но будет стабильнее работать с сервисами, у которых DNS-записи регулярно обновляются.

Если это кажется слишком сложным, можно вернуться к самому первому варианту, где сертификат загружается при каждом обращении к внешней системе. Мы предположили, что это будет *очень медленно*, но серьёзных оснований для такого предположения у нас нет.

Так что мы можем начать с простой реализации, провести нагрузочное тестирование, и принять решение на основании фактических данных.