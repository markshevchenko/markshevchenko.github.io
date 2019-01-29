---
title: Хранение сертификата HttpClient в Azure Key Vault
date: "2019-01-29 12:00:00 +0300"
id: storing-http-client-certificate-in-azure-key-vault
excerpt: Неделя потрачена на то, чтобы победить хранение сертификатов HttpClient в Azure Key Vault. Краткое резюме битвы. 
---

## О чём речь

При работе с внешними REST API мы применяем `HttpClient`. Например, чтобы построить маршрут с помощью Google Maps Directions API мы пишем приблизительно такой код:

```c#
public class GoogleMapsDirectionsClient
{
    private const string _baseUri = "https://maps.googleapis.com/maps/api/directions/json";

    private readonly string _apiKey;

    public GoogleMapsDirectionsClient(string apiKey)
    {
        this._apiKey = apiKey;
    }

    public async Task<Route> BuildRoute(Location from, Location to)
    {
        var queryString = new NameValueCollection();
        queryString["key"] = _apiKey;
        queryString["origin"] = $"{from.Latitude},{from.Longitude}";
        queryString["destination"] = $"{to.Latitude},{to.Longitude}";
        
        var uriBuilder = new UriBuilder(_baseUri);
        uriBuilder.Query = queryString.ToString();

        using (var httpClient = new HttpClient())
        {
            var httpResponseMessage = await httpClient.GetAsync(uriBuilder.Uri);

            httpResponseMessage.EnsureSuccessStatusCode();

            var json = await httpResponseMessage.Content.ReadAsString();

            return JsonConvert.Deserialize<Route>(json);
        }
    }
}
```

Google Maps Directions API не нало параноидально защищаит, но некоторые финансовые API&nbsp;&mdash; надо. Одним из средств защиты является *клиентский сертификат*.

Вы заключаете договор с финансовой компанией, которая, например, принимает оплату для вашего интернет-магазина. Компания выдаёт вам сертификат и пароль к нему.

> Сертификаты могут храниться в разных форматах. Azure поддерживает только формат PFX. Если финансовая компания выдала сертификат в другом формате, его нужно перекодировать с помощью утилиты **openssl**.

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

Файл можно хранить локально, но что, если вам надо общаться с финансовым API из Azure?

## Импорт сертификата в Хранилище ключей

Для хранения секретных данных в Azure применяется *Хранилище ключей*, оно же Key Vault.
Для начала добавим наш сертификат в хранилище. Откроем портал Azure и выберем хранилище Key Vault. В панели слева мы должны увидеть вкладку *Certificates*.

![Список сертификатов](/img/key-vault-1.png)

Откроем её и нажмём в верхней панели кнопку *Generate/Import*. Укажем метод *Import*, загрузим сертификат, дадим ему имя (из букв английского алфавита и дефиса) и введём пароль.

![Импорт сертификата](/img/key-vault-2.png)

После того, как сертификат загружен, мы увидим его в списке сертификатов. Откроем:

![Новый сертификат](/img/key-vault-3.png)

Дважды щёлкнем на записи *Current Version*, чтобы посмотреть параметры сертификата.

![Параметры нового сертификата](/img/key-vault-4.png)

Нам понадобится параметр *Secret Identifier*, поэтому сохраним его в текстовом файле.

## Регистрация приложения

Доступ к Хранилищу должен быть предоставлен нашему приложению, а для этого его надо зарегистрировать в *Azure Active Directory*.
Наберите *Azure Active Directory* в строке поиска в самом верху Портала:

![Azure Active Directory](/img/key-vault-5.png)

В панели справа выберите закладку *App registrations*:

![Список зарегистрированных приложений](/img/key-vault-6.png)

Нажмите кнопку *New application registration* и введите параметры своего приложения:

![Регистрация приложения](/img/key-vault-7.png)

Что вы укажите в поле *Sign-on URL*, неважно. Главное, сохраните название *Name*, поскольку оно потребуется нам на следующем шаге. Добавив приложение, вы не увидите его в списке, поскольку установлен фильтр *My apps*. Переключитесь на *All apps*, найдите своё приложение и откройте его.

![Параметры нового приложения](/img/key-vault-8.png)

Нам потребуется параметр, который называется *Application ID*.
Запишите, но не покидайте закладку. Нажмите кнопку *Settings* в заголовке. Вы увидите панель с дополнительным настройками:

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

Мы готовы к тому, чтобы запрограммировать загрузку сертификата из Хранилища и подключение его к `HttpClient`.
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

Как видим, подключиться к Хранилищу непросто. Я ожидал, что это будет не сложнее, чем подлюкчение к SQL серверу, но на деле код пришлось разбить на два метода. В методе `CreateHttpClientAsync` вызывается конструктор `KeyVaultClient`, которому мы передаём обработчик для получения токена `GetToken`. Если честно, я и сам не понимаю, почему здесь надо делать именно так: документация не блещет подробностями.

Буду благодарен за обоснование.

В любом случае, в январе 2019-го года этот код работает. Чтобы разобраться, времени ушла неделя, потому что информацию пришлось собирать по крупицам с десятка ресурсов.

## Оптимизация

В этом коде я вижу две проблемы. Первая заключается в том, что сейчас создавать вручную подключения `HttpClient` считается моветоном. Весто этого надо через DI-контейнер получать `IHttpClientFactory` и вызывать метод `CreateClient`.

Чтобы это работало, надо зарегистрировать клиент с уникальным именем приблизительно так:

```c#
services.AddHttpClient("certified")
        .ConfigurePrimaryHttpMessageHandler(() => 
        {
            return new HttpClientHandler
            {
                ClientCertificates = { certificate },
            };
        });
```

Проблма в том, что этот код синхронно вызывается при старте нашего приложения, а загрузить `certificate` через `KeyVaultClient` мы можем только асинхронно.

Решения, кроме как дожидаться завершения асинхронной операции на старте, я найти не смог. Если придумаете, пишите.

Вторая проблема заключается в том, что сертификат загружается при каждом запросе к внешней защищённой системе, что может быть медленно. Попробуем закэшировать экземпляр `HttpClientHandler` для чего воспользуемся паттерном ленивой инициализации.

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
