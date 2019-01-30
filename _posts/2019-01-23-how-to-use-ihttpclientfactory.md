---
title: Как использовать IHttpClientFactory
date: "2019-01-23 12:00:00 +0300"
id: how-to-use-ihttpclientfactory
excerpt: Надо ли вызывать Dispose у HttpClient, если создал его с помощью IHttpClientFactory?
---

## Резюме

[`Dispose`](https://docs.microsoft.com/ru-ru/dotnet/standard/garbage-collection/implementing-dispose) у [`HttpClient`](https://docs.microsoft.com/ru-ru/dotnet/api/system.net.http.httpclient?view=netcore-2.2), полученного из [`IHttpClientFactory.CreateClient`](https://docs.microsoft.com/en-us/dotnet/api/system.net.http.ihttpclientfactory.createclient?view=aspnetcore-2.2) вызывать не обязательно, но можно.

## Как было раньше

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
Если у нас есть специфичные методы с очень большим временем ожидания, нам снова нужен отдельный `HttpClient`. Если у нас есть клиентский сертификат&nbsp;&mdash; сюрприз&nbsp;&mdash; без отдельного экземпляра `HttpClient` опять не обойтись.

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

Этот способ специфичен для Autofac. Чтобы он работал, нам придётся подключить Autofac к проекту, где реализован `RestClient`, что создаёт лишнюю зависимость. Такое решение нельзя назвать идеальным, но оно кажется вполне рабочим.

## Что не так с `HttpClient`?

Почему в Microsoft придумали `IHttpClientFactory`? Потому что [решение оказалось не таким уж и рабочим](https://github.com/dotnet/corefx/issues/11224). *Постоянные* `HttpClient` не умеют обновлять кэш DNS, что приводит к ошибкам, когда приложение работает *долго*, а записи в DNS меняются *часто*.

> Один из способов распределения нагрузки в интернет-приложениях это метод [Round robin DNS](https://ru.wikipedia.org/wiki/Round_robin_DNS). В Round robin DNS одному доменному имени соответствую несколько серверов с разными IP-адресами. Разные клиенты, запрашивая адрес сервера по имени, получают разные значения, после чего обращаются к разным физическим серверам, что и помогает распределить нагрузку.

> Для того, чтобы такая схема работала хорошо, адреса нельзя кэшировать надолго. Предполагается, что если сервер перестал отвечать, клиент должен запросить новый адрес, и продолжить работу. Как раз этот сценарий не работает с постоянным объектом `HttpClient`.

По какой-то причине в компании Microsoft решили не исправлять ошибку на уровне реализации, а изменили сам сценарий взаимодействия. Подозреваю, дело в том, что нам действительно нужен переносимый способ получать именованные экземпляры `HttpClient`:

```c#
public interface IHttpClientFactory
{
    HttpClient CreateClient(string name);
}
```

Теперь класс `RestClient`, который мы используем для примеров, будет выглядеть так:

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

У нас есть конструкция `using` благодаря которой у экземпляра `httpClient` вызывается метод `Dispose`. Он освобождает физические ресурсы, выделенные программой при отправке запросов.

Но, если мы заглянем в [примеры от Microsoft](https://docs.microsoft.com/ru-ru/aspnet/core/fundamentals/http-requests?view=aspnetcore-2.1#basic-usage), то увидим, что в них нет конструкции `using`. Метод `Dispose` там просто не вызывается.

В *обычных условиях* метод `Dispose` *должен быть вызван*.

Надо ли его вызывать в случае, если `HttpClient` получен из `IHttpClientFactory`?

* Возможно, вызывать надо обязательно, как это всегда и делали. В приведённом примере ошибка.
* Возможно, вызывать нельзя ни в коем случае. Для освобождения ресурсов применяется неочевидный способ, и, если мы вызовем `Dispose`, то всё сломаем.

Какой из этих вариантов правильный?

## Чуть глубже в кроличью нору

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

Здесь `HttpClient` создан с помощью альтернативного конструктора. Поскольку флаг `disposeHandler` получает значение `false`, метод `HttpClient.Dispose` в действительности не делает ничего. Мы можем его вызывать, можем не вызывать&nbsp;&mdash; неуправляемые ресурсы в лице `HttpMessageHandler` будут освобождены классом `DefaultHttpClientFactory`.

Это означает, что объекты `HttpClient` не требуют длительной инициализации и их можно создавать без потери производительности в неограниченном количестве. Проблему с кэшем DNS тоже решает класс `DefaultHttpClientFactory`. Он держит *пул обработчиков* `HttpMessageHandler` и пересоздаёт их время от времени, чтобы очистить кэш.

Кажется, на этом можно было бы и закончить, поскольку мы теперь знаем, что всю интересную работу делает волшебный класс `HttpMessageHandler`. Правда, если заглянуть в его [исходный код](https://github.com/dotnet/corefx/blob/master/src/System.Net.Http/src/System/Net/Http/HttpMessageHandler.cs), мы увидим, что он&nbsp;&mdash; абстрактный, и метод там всего один&nbsp;&mdash; `SendAsync`. При этом, если судить по названию, ответственность класса заключется в том, чтобы *обрабатывать сообщения HTTP*. Вам не кажется это странным?

## `HttpMessageHandler` в деталях

В протоколе HTTP существует два вида сообщений. Первое из них это *запрос*, которое *клиент* отправляет *серверу*. Второе&nbsp;&mdash; *ответ*, который *сервер* возвращает *клиенту*.

В .NET Core сообщение-запрос представляет класс `HttpRequestMessage`, а сообщение-ответ&nbsp;&mdash; `HttpResponseMessage`.

Асинхроныый метод `SendAsync` выглядит так, будто он отправляет запрос и возвращает ответ:

```c#
protected internal abstract Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken);
```

Странность с `HttpMessageHandler` происходит из-за того, что объекты класса применяются в двух разных сценариях.

Первый сценарий реализован в наследнике `HttpMessageHandler` который называется `HttpClientHandler`. В .NET Core 2.1 вместо него написали `SocketsHttpHandler`, но суть осталась прежней.

Здесь метод `SendAsync` делает именно то, что мы от него ожидаем&nbsp;&mdash; формирует из `HttpRequestMessage` запрос, отправляет его на сервер, дожидается ответа и упаковывает результат в `HttpResponseMessage`.

К сожалению, при таком сценарии непонятно, почему класс называют *обработчиком сообщений*. Он, скорее, *приёмник-передатчик*.

Обработчиками являются другие наследники `HttpMessageHandler`, которые реализуют паттерн *Декоратор*. Декораторы, обладая идентичным интерфейсом, могут быть объеденены в *Цепочку обязанностей* (Chain of responsibility). Можно создать несколько наследников `HttpMessageHandler`, каждый из которых будет выполнять свою задачу.

Каждый класс в цепочке делает приблизительно одно и то же:

1. Он получает управление из метода `SendAsync` предыдущего объекта в цепочке.
1. Он вносит изменение в запрос `HttpRequestMessage`.
1. Он вызывает `SendAsync` следующего объекта в цепочке.
1. В конце цепочки находится класс, который физически отправляет запрос на сервер и возвращает ответ.
1. Управление возвращается по цепочке назад. Каждый метод `SendAsync`, прежде чем вернуть управление, может обработать сообщение-ответ `HttpResponseMessage`.

О каких изменения речь? HTTP-запрос [состоит](https://www.w3.org/Protocols/rfc2616/rfc2616-sec5.html) из строки запроса, заголовков и содержимого. При внесении изменений в подавляющем большинстве случае речь идёт об изменении заголовков.

В простейшем сценарии, скажем, при авторизации, обработчик может добавлять заголовок `Authorization`. В более сложном сценарии работы с cookie, обработчик сначала сохраняет содержимое заголовков `Set-Cookie`, полученных в `HttpResponseMessage`, а затем добавляет это содержимое к заголовкам `HttpRequestMessage` в последующих запросах.

Именно в таком сценарии понятно, почему `HttpMessageHandler`&nbsp;&mdash; это *обработчик сообщений*.

## Закрываем гештальт

Осталось осветить тему, как настраивать параметры именованных экземпляров `HttpClient`. Пока мы знаем, что контейнер IoC нам передаёт реализацию интерфейса `IHttpClientFactory`, у которой мы можем вызвать метод `CreacteClient`.

Чтобы эта схема работала, надо в том месте программы, которое назвыается *Корнем композиции* (Composite Root), и где регистрируются службы и интерфейсы, зарегистрировать клиент HTTP и указать его имя:

```c#
services.AddHttpClient("foo");
```

Метод расширения `AddHttpClient` находится в NuGet-пакете [Microsoft.Extensions.Http](https://www.nuget.org/packages/Microsoft.Extensions.Http/), не забудьте его подключить.

Напомню, что в классе `DefaultHttpClientFactory` под этим именем будет хранится не экземпляр `HttpClient`, а набор данных, необходимых для инициализации создаваемых клиентов.

Например, вот так мы можем создать запись о клиенте, который будет добавлять заголовок `Authorization` к каждому запросу:

```c#
services.AddHttpclient("authorized", httpClient =>
{
    var lastToken = tokenProvider.GetLastToken();

    httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("bearer", lastToken);
});
```

Здесь `tokenProvider`&nbsp;&mdash; гипотетический объект, который отвечает за хранение токена доступа.

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

В каждом из этих случаев новый экземпляр `HttpClient` будет создан при каждом вызове `CreateClient`, однако, `DefaultHttpClientFactory` будет хранить связанные с клиентами обработчики `HttpMessageHandler`, поэтому создание клиентов не займёт много времени.
