---
title: Binateq.JsonRestClient
date: "2018-04-25 18:10:00 +0300"
id: json-rest-client
excerpt: Пакет для отправки REST запросов.
---

Разработал и опубликовал пакет для отправки REST-запросов. Сделал его, когда в четвёртый или пятый раз
писал однотипное расширение класса [`HttpClient`](https://msdn.microsoft.com/ru-ru/library/system.net.http.httpclient(v=vs.118).aspx).

## Проблема

При разработке REST-клиента нужно отсылать на сервере запросы. Выглядят они приблизительно так:

```
POST http://api.domain.tld/v1/resources
GET http://api.domain.tld/v1/resources?from=2018-04-17T11:37:00&ends_with=bar
PUT http://api.domain.tld/v1/resources/2
```

В теле запроса могут присутствовать дополнительные параметры, которые чаще всего закодированы в формате JSON.
Иногда запросы возвращают результаты, также в формате JSON.

Уровень класса `HttpClient` чуть ниже требуемых задач, поэтому его приходится обвязывать расширениями.

## Решение

Разработан класс `JsonRestClient` с методами отправки REST-запросов и поддержкой JSON.

```c#
class Resource
{

	public int Id { get; set;}

	public string Name { get; set; }
}

var httpClient = new HttpClient();
var baseUri = new Uri("https://api.domain.tld/v1");
var jsonRestClient = new JsonRestClient(httpClient, baseUri);

var resourceId = 100;

resource = await jsonRestClient.GetAsync<Resource>($"resources/{resourceId}");

await jsonRestClient.PostAsync($"resources", new Resource { Name = "foo" });
```

Для работы объекту `JsonRestClient` нужен `HttpClient`, чтобы отправлять запросы; и базовый `Uri`, чтобы не дублировать часть
URI, общую для всех методов.

## Параметры строки запроса

Параметры запроса можно передавать либо в пути (до знака вопроса `?`), либо в строке запроса (после знака вопроса `?`):

```
GET http://api.domain.tld/v1/resources/314/subresources?from=2018-04-17T11:37:00&ends-with=bar
```

В примере `314`&nbsp;&mdash; это параметр, переданный в пути `/v1/resources/314/subresources`, а `2018-04-17T11:37:00`
и `bar`&nbsp;&mdash; параметры, переданные в строке запроса. В отличие от `314`, каждый из них имеет имя,
в нашем примере `from` и `ends-with`.

Вот так мы можем отправить запрос из примера:

```c#
var resourceId = 314;
var from = new DateTime(2018, 04, 17, 11, 37, 00);
var endsWith = "bar";

var resource = await jsonRestClient.GetAsync<Resource>($"resources/{resourceId}"
    new Dictionary<object, string>
    {
        { nameof(from), from },
        { "ends-with" }, endsWith,
    }
);
```

## Статус 404

Для методов `Get` в `JsonRestClient` предусмотрена форма `GetOrDefault`. Если запрос возвращает статус 404, метод `GetOrDefault`
возвратит значение по умолчанию для заданного типа.

## С глаз долой&nbsp;&mdash; из сердца вон

Для методов `Put` и `Post` в `JsonRestClient` предусмотрена форма `PutAndForget`. Обычные методы `Put`/`Post` выбрасывают исключение, если
статус ответа отличается от 2xx. Методы `AndForget` не проверяют статус ответа совсем.

[nuget.org](https://www.nuget.org/packages/Binateq.JsonRestClient/)

[Исходный код](https://github.com/binateq/json-rest-client)
