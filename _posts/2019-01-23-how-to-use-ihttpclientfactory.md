---
title: Как использовать IHttpClientFactory
date: "2019-01-23 12:00:00 +0300"
id: how-to-use-ihttpclientfactory
excerpt: Наконец, разобралася, надо ли вызывать Dispose у HttpClient, если создал его с помощью IHttpClientFactory.CreateCient.
---

## Резюме

[`Dispose`](https://docs.microsoft.com/ru-ru/dotnet/standard/garbage-collection/implementing-dispose) у [`HttpClient`](https://docs.microsoft.com/ru-ru/dotnet/api/system.net.http.httpclient?view=netcore-2.2), полученного из [`IHttpClientFactory.CreateClient`](https://docs.microsoft.com/en-us/dotnet/api/system.net.http.ihttpclientfactory.createclient?view=aspnetcore-2.2) вызывать не обязательно, но можно.

## Из-за чего сыр-бор?

По ряду причин Microsoft предложила создавать экземпляры `HttpClient` не вручную или через IoC-контейнер, а через фабричный метод `IHttpClientFactory.CreateClient`. В [примерах](https://docs.microsoft.com/ru-ru/aspnet/core/fundamentals/http-requests?view=aspnetcore-2.1#basic-usage) Microsoft приводт, в частности, такой код:

```c#
public async Task OnGet()
{
    var request = new HttpRequestMessage(HttpMethod.Get, "https://api.github.com/repos/aspnet/docs/branches");
    request.Headers.Add("Accept", "application/vnd.github.v3+json");
    request.Headers.Add("User-Agent", "HttpClientFactory-Sample");

    var client = _clientFactory.CreateClient();
    var response = await client.SendAsync(request);

    if (response.IsSuccessStatusCode)
    {
        Branches = await response.Content
                                 .ReadAsAsync<IEnumerable<GitHubBranch>>();
    }
    else
    {
        GetBranchesError = true;
        Branches = Array.Empty<GitHubBranch>();
    }                               
}
```

Как видим, у `client` не вызыван метод `Dispose`. Предполагается, что он освобождает неуправляемые ресурсы, которые важно освобождать, поэтому *обычно* метод `Dispose` *должен быть вызван*.

Надо его вызывать или не надо? Возможны три варианта.

* Вызывать обязательно надо, как всегда и было. В приведённом примере ошибка.
* Вызывать нельзя ни в коем случае. Для освобождения ресурсов применяется неочевидный способ, и, если мы вызовем `Dispose`, то всё сломаем.
* Можно вызывать, можно не вызвыать. Для освобождения ресурсов действительно применяется неочевидный способ, но вызов `Dispose` ничего не ломает.

## Разберёмся?

Последние несколько лет для обращения к REST-службам из C# мы применяли класс `HttpClient`. Он реализует интерфейс `IDisposable`, поэтому &laquo;правильный&raquo; сценарий использования выглядит так:

```c#
public class RestClient
{
    . . .
    public async GetAsync<T>(Uri uri)
    {
        using (var httpClient = new HttpClient())
        {
            var httpResponseMessage = await httpClient.GetAsync(uri);
            var content = await httpResponseMessage.Content.ReadAsStringAsync();

            return JsonConvert.Deserialize<T>(content);
        }
    }
    . . .
}
```

Такой способ требуется постоянной инициализации объекта `HttpClient`, и считается не очень производительным.
Экземпляр `HttpClient` вполне можно создать один раз, и использовать во всех вызовах вашей программы:

```c#
public class RestClient
{
    private readonly HttpClient _httpClient = new HttpClient();
    . . .
    public async GetAsync<T>(Uri uri)
    {
        var httpResponseMessage = await _httpClient.GetAsync(uri);
        var content = await httpResponseMessage.Content.ReadAsStringAsync();

        return JsonConvert.Deserialize<T>(content);
    }
    . . .
}
```

В современных приложениях, где применяется *инверсия зависимостей*, ссылку на `HttpClient` обычно *внедряют через конструктор*:

```c#
public class RestClient
{
    private readonly HttpClient _httpClient;

    public RestClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }
    . . .
}
```

В этом случае `HttpClient` будет создан *контейнером IoC* в одном экземпляре. Освободит его&nbsp;&mdash; то есть вызовет метод `Dispose`&nbsp;&mdash; тоже IoC-контейнер в конце работы приложения.

Схема выглядит слишком простой? Да, в реальных программах она чуть сложнее, поскольку иногда нам требуется несколько экземпляров `HttpClient` одновременно.

> Например, если мы получили токен доступа у REST-службы, то должны в каждом HTTP-запросе отправлять заголовок [`Authorization`](https://developer.mozilla.org/ru/docs/Web/HTTP/Заголовки/Authorization). Для этого нужен отдельный `HttpClient`.
Если у нас есть специфичные методы с очень большим временем ожидания, нам снова нужен отдельный `HttpClient`. Если у нас есть клиентский сертификат&nbsp;&mdash; сюрприз&nbsp;&mdash; без отдельного экземпляра `HttpClient` снова не обойтись.

Чтобы отличать экземпляры `HttpClient` друг от друга, IoC-контейнеры предлагают разные способы. Например, в Autofac можно [дать им имена](https://autofaccn.readthedocs.io/en/latest/advanced/keyed-services.html). Нужный экземпляр можно выбрать с помощью атрибута [`KeyFilter`](https://autofac.org/apidoc/html/9305214F.htm).

```c#
public class RestClient
{
    private readonly HttpClient _httpClient;

    public RestClient([KeyFilter("authorized")] HttpClient httpClient)
    {
        _httpClient = httpClient;
    }
    . . .
```

Этот способ специфичен для Autofac. Чтобы он работал, нам придётся подключить Autofac к проекту, где реализован `RestClient`, что создаёт лишнюю зависимость. Такое решение нельзя назвать идеальным, но оно кажется вполне рабочим. Или нет?

## Что не так с `HttpClient`?

Почему в Microsoft придумали `IHttpClientFactory`? Потому что [решение оказалось не таким уж и рабочим](https://github.com/dotnet/corefx/issues/11224). *Постоянные* `HttpClient` не умеют обновлять кэш DNS, что приводит к ошибкам, когда приложение работает *долго*, а записи в DNS меняются *часто*.

> Один из способов распределения нагрузки в интернет-приложениях это метод [Round robin DNS](https://ru.wikipedia.org/wiki/Round_robin_DNS). В Round robin DNS одному доменному имени соответствую несколько серверов с разными IP-адресами. Разные клиенты, запрашивая адрес сервера по имени, получают разные значения, после чего обращаются к разным физическим серверам, что и помогает распределить нагрузку.

> Для того, чтобы такая схема хорошо работала, адреса нельзя кэшировать надолго. Предполагается, что если сервер перестал отвечать, клиент должен запросить новый адрес, и продолжить работу. Как раз этот сценарий не работает с классическим постоянным объектом `HttpClient`.

По какой-то причине в компании Microsoft решили не исправлять ошибку на уровне реализации, а изменили сам сценарий взаимодействия. Подозреваю, дело в том, что нам действительно нужен переносимый способ получать именованные экземпляры `HttpClient`:

```c#
public interface IHttpClientFactory
{
    HttpClient CreateClient(string name);
}
```

Теперь наш `RestClient` может выглядеть так:

```c#
public class RestClient
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly string _httpClientName;

    public RestClient(IHttpClientFactory httpClientFactory, string httpClientName)
    {
        _httpClientFactory = httpClientFactory;
        _httpClientName = httpClientName;
    }
    . . .
    public async GetAsync<T>(Uri uri)
    {
        using (var httpClient = _httpClientFactory.CreateClient(_httpClientName))
        {
            var httpResponseMessage = await httpClient.GetAsync(uri);
            var content = await httpResponseMessage.Content.ReadAsStringAsync();

            return JsonConvert.Deserialize<T>(content);
        }
    }
    . . .
}
```

Однако, мы вернулись к проблеме одноразовых `HttpClient`&nbsp;&mdash; они гораздо медленнее постоянных. Впрочем, Microsoft утверждает, что на самом деле мы не теряем в скорости, а ещё нам не обязательно вызывать `Dispose`.

## Немного о `HttpMessageHandler`

Чтобы разобраться с тем, почему так, взглянем на альтернативный конструктор `HttpClient`:

```c#
public HttpClient(HttpMessageHandler handler, bool disposeHandler)
{
    . . .
}
```

Класс `HttpClient` довольно прост. Он позволяет создать заготовку HTTP-запроса, а реальную работу по его отправке делегирует объекту [`HttpMessageHandler`](https://docs.microsoft.com/ru-ru/dotnet/api/system.net.http.httpmessagehandler?view=netcore-2.2). Это означает, что неуправляемые ресурсы находятся в `HttpMessageHandler`. Метод `HttpClient.Dispose` всего лишь вызывает `HttpMessageHandler.Dispose`, чтобы их освободить.

В .NET Core есть реализация `IHttpClientFactory` по умолчанию, которая так и называется&nbsp;&mdash; `DefaultHttpClientFactory`. Эта реализация сама управляет созданием объектов `HttpMessageHandler`. Создание нового экземпляра `HttpClient` там выглядит так:

```c#
public class DefaultHttpClientFactory : IHttpClientFactory
{
    . . .
    public HttpClient CreateClient(string name)
    {
        var httpMessageHandler = CreateOrGetHttpMessageHandler(name);

        return new HttpClient(httpMessageHandler, disposeHandler: false);
    }
    . . .
}
```

Поскольку флаг `disposeHandler` получает значение `false`, метод `HttpClient.Dispose` в действительности не делает ничего. Мы можем его вызывать, можем не вызывать&nbsp;&mdash; неуправляемые ресурсы в лице `HttpMessageHandler` будут освобождены классом `DefaultHttpClientFactory`.

Именно этот класс решает проблему с кэшированием DNS. Он время от времени пересоздаёт обработчики `HttpMessageHandler`, что приводит к сбросу кэша.

Кажется, на этом можно было бы и закончить, поскольку мы теперь знаем, что всю интересную работу делает волшебный класс `HttpMessageHandler`. Правда, если заглянуть в его [исходный код](https://github.com/dotnet/corefx/blob/master/src/System.Net.Http/src/System/Net/Http/HttpMessageHandler.cs), мы увидим, что он&nbsp;&mdash; абстрактный, и метод там всего один&nbsp;&mdash; `SendAsync`. При этом, если судить по названию, ответственность класса заключется в том, чтобы *обрабатывать сообщения HTTP*. Вам не кажется всё это странным?

## Подробнее о `HttpMessageHandler`

Странность с `HttpMessageHandler` происходит из-за того, что объекты класса применяются в дрвух разных сценариях. Первый сценарий послностью соответствует названию класса&nbsp;&mdash; *обработчик сообщений*. Вот так выглядит сигнатура метода `SendAsync`:

```c#
protected internal abstract Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken);
```

В протоколе HTTP существует два вида сообщений. Первое из них это *запрос*, которое *клиент* отправляет *серверу*. Второе&nbsp;&mdash; *ответ*, который *сервер* возвращает *клиенту*.

В .NET Core сообщение-запрос представляет класс `HttpRequestMessage`, а сообщение-ответ&nbsp;&mdash; `HttpResponseMessage`.
Прежде, чем отправить запрос, мы можем внести в него изменения. HTTP-запрос [состоит](https://www.w3.org/Protocols/rfc2616/rfc2616-sec5.html) из строки запроса, заголовков и содержимого. При внесении изменений в подавляющем большинстве случае речь идёт об изменении заголовков.

В простейшем сценарии, скажем, при авторизации, обработчик мог бы просто добавлять заголовок `Authorization`. В более сложном сценарии работы с cookie, обработчик сначала сохраняет содержимое заголовков `Set-Cookie`, полученных в `HttpResponseMessage`, а затем добавляет это содержимое к заголовкам `HttpRequestMessage` в последующих запросах.

Такое поведение соответсвует паттерну *Декоратор* (Decorator). Декораторы, обладая идентичным интерфейсом, могут быть объеденены в *Цепочку обязанностей* (Chain of responsibility). Мы можем создать несколько наследников `HttpMessageHandler`, каждый из которых будет выполнять одну задачу.

В самом начале `HttpClient` сформирует заготовку запроса, объект `HttpRequestMessage` и вызовет метод `SendAsync` первого обработчика. Каждый из обработчиков будет сначала вносить изменения в запрос, а затем вызывать следующий `SendAsync` в цепочке. Получив управление назад, он обработает объект ответа, `HttpResponseMessage`, и вернёт управление методу `SendAsync` предыдущего обработчика.

Именно в таком сценарии понятно, почему `HttpMessageHandler`&nbsp;&mdash; это *обработчик сообщений*. Однако, сигнатура метода `SendAsync` такова, что позволяет реализовать не только обработку сообщений, но и непосрядственно их отправку и приём.

Авторы библиотеки не стали описывать отдельный интерфейс `HttpSender` с похожим методом, вместо этого они создали наследника `HttpMessageHandler`, который занимается не обработкой сообщений, а отправкой запросов и получением ответов. Это класс [`HttpClientHandler`](https://github.com/dotnet/corefx/blob/master/src/System.Net.Http/src/System/Net/Http/HttpClientHandler.cs).

В результате мы имеем нарушенный принцип **Interface segregation** и связанную с этим путаницу в библиотеке.

## Настройка именованных экземпляров `HttpClient`

К счастью, нам не нужно заглядывать под капот `HttpClient` слишком часто. Обычно достаточно подключить к проекту NuGet [Microsoft.Extensions.Http](https://www.nuget.org/packages/Microsoft.Extensions.Http/) и вызвать метод `AddHttpClient`, чтобы зарегистировать новый именованный `HttpClient`:

```c#
services.AddHttpClient("foo");
```

Напомню, что в классе `DefaultHttpClientFactory` под этим именем будет хранится не экземпляр `HttpClient`, а набор данных, необходимых для инициализации создаваемых клиентов.

Например, вот так мы можем создать запись о клиенте, который будет добавлять заголовок `Authorization` к каждому запросу:

```c#
services.AddHttpclient("authorized", httpClient =>
{
    var lastToken = tokenProvider.GetLastToken();

    httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("bearer", lastToken);
});
```

Здесь `tokenProvider`&nbsp;&mdash; гипотетический класс, который отвечает за хранение токена доступа.

Вот так может быть подключен клиент с сертификатом:

```c#
services.AddHttpClient("signed")
        .ConfigurePrimaryMessageHandler(() =>
        {
            var handler = new HttpClientHandler();

            handler.ClientCertificateOptions = ClientCertificateOptions.Manual;
            handler.ClientCertificates.Add(...);
        });
```

В каждом из этих случаев новый экземпляр `HttpClient` будет создан при каждом вызове `CreateClient`, однако, `DefaultHttpClientFactory` будет хранить связанные с клиентами обработчики `HttpMessageHandler`.