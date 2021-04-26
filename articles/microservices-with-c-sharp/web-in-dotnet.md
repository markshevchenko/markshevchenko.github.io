---
title: Часть II. Веб в .NET
excerpt: "История веб-программирования на платформе .NET."
id: microservices-with-c-sharp
---

Сейчас, когда мы более-менее представляем, что такое микросервисы, самое время переходить к конкретике. Мы в курсе, что нам предстоит разрабатывать веб-приложения, так что надо узнать, какие средства для этого предлагает .NET.

### Исторический экскурс

Веб-приложения, то есть _динамические сайты_, в .NET можно было разрабатывать с самой первой версии, с 2003-го года. Тогда Microsoft предложила немного странную парадигму, которую сейчас принято называть WebForms. Она была понятна десктопным разработчикам, но веб-разработчикам приходилось с ней разбираться. Там были такие штуки, как события и цикл жизни страниц.

Сейчас подход WebForms [признали устаревшим](https://dzone.com/articles/net-5-is-the-future-of-net-what-every-net-develope) и не планируют далее поддерживать.

В 2006-м в .NET появилась библиотечно-инструментальная поддержка SOAP под названием WCF — Windows Communication Foundation.

В наше время «правильной» заменой SOAP считается gRPC, так что Microsoft в .NET Core [отказалась и от поддержки WCF](https://www.ben-morris.com/why-isnt-wcf-supported-in-net-core/). Впрочем, они запустили проект с открытым исходным кодом, который позволит перевести написанные приложения на новую платформу.

В 2005-м году Дэвид Ханссон опубликовал исходный код фреймворка Ruby on Rails, где предлагал простую и удобную концепцию разработки. Он взял за основу старый добрый паттерн Model-View-Controller (MVC), добавил толику здравого смысла и прикрутил всё это к HTTP.
Фреймворк вышел удачным. Команда разработчиков ASP.NET вовремя разглядела его потенциал и разработала похожее решение на C#.

Конечно, сама концепция MVC не нова, впервые о ней заговорили ещё в 1978-м. Но такие вещи, как маршрутизация запросов и хелперы — прямое заимствование из RoR.

ASP.NET MVC вышел в 2009-м и включал в себя Web API. Это была та же парадигма MVC, заточенная под разработку REST-сервисов.

Для развёртывания приложений всё ещё требовался IIS, и, если вы его настраивали, то знаете, что это непросто. IIS большой.

В 2013-м Microsoft разработала встраиваемый веб-сервер Katana, который позволил писать self-hosted web applications — автономные веб-приложения. Чтобы одно и то же приложение можно было запускать и автономно, и в IIS, Microsoft спроектировала общий интерфейс веб-серверов OWIN.

Наконец, в 2016-м появился .NET Core — переносимая инкарнация .NET Framework. Разработчикам .NET Core пришлось практически полностью переписать стандартную библиотеку, не рассчитывая больше на Windows API. Переработка затянулась на несколько лет и, фактически, всё ещё продолжается. ASP.NET Core вместе с Web API перенесли одним из первых, им можно пользоваться с 2016-го. От Katana пришлось отказаться в пользу нового переносимого асинхронного веб-сервера Kestrel. Для реализации асинхронности разработчики Kestrel использовали кросс-платформенную библиотеку [libuv](https://github.com/libuv/libuv) с открытым кодом, но сегодня все необходимые функции есть уже в .NET Core.

### Пример

Посмотрим, как выглядит типичный современный веб-проект. Представим, что у нас есть приложение, которое помогает вести список дел, to do list.

В соответствии с подходом REST каждое дело (to do item) представляет собой _ресурс_. Доступ к ресурсам осуществляется через базовый URI, у нас это будет `api/v1/todo-items`. Чтобы получить ресурс с идентификатором `100`, надо послать `GET`-запрос по адресу `api/v1/todo-items/100`.

```c#
[Route("api/v1/todo-items")]
public class TodoItemsController : ControllerBase
{
    private readonly TodoDbContext dbContext;

    [HttpGet(“{id}”)]
    public async Task<TodoItem> GetByIdAsync(int id)
    {
        return await dbContext.TodoItems
                              .SingleAsync(x => x.Id == id);
    }
}
```

Так выглядит обработчик запроса в .NET Core. Мы видим атрибуты `Route` и `HttpGet`, которые помогают фреймворку понять, что запрос `GET api/v1/todo-items/100` соответствует методу `GetByIdAsync`, а число **100** — параметру `id`.

Код работает асинхронно, но благодаря _магии_ компилятора C#, выглядит синхронным.
Возвращаемый объект класса TodoItem будет преобразован в JSON или XML, в зависимости от предпочтений клиента.

В целом, код не кажется сложным, особенно, если у вас есть опыт чтения исходников на C, C++ или Java.

### Пару слов об асинхронности

Не все программисты понимают, зачем нужна асинхронность в веб-приложениях. Об этом стоит поговорить.

Сравним производительность трёх версий веб-сервера: обычного, параллельного и асинхронного.

#### Обычный веб-сервер

Обычным мы будем называть не-параллельный и не-асинхронный веб-сервер.

Наш веб-сервер возвращает простой HTML-документ, если мы посылаем ему запрос `GET /`. Я убрал из примера весь лишний код, в частности, обрабтку ошибок.

```c#
var listener = new HttpListener();
listener.Prefixes.Add("http://localhost:8080/");
listener.Start();

while (true)
{
    var context = listener.GetContext();
    if (context.Request.HttpMethod == "GET" && context.Request.RawUrl == "/")
    {
//        Thread.Sleep(100);
        context.Response.StatusCode = 200;

        using (var writer = new StreamWriter(context.Response.OutputStream))
        {
            writer.WriteLine("<!DOCTYPE html>");
            writer.WriteLine("<html lang='en' xmlns='http://www.w3.org/1999/xhtml'>");
            writer.WriteLine("  <head>");
            writer.WriteLine("  <meta charset='utf-8' />");
            writer.WriteLine("  <title>Example HTTP server</title>");
            writer.WriteLine("  </head>");
            writer.WriteLine("  <body>");
            writer.WriteLine("    <p>Example HTTP server</p>");
            writer.WriteLine("  </body>");
            writer.WriteLine("</html>");
        }
    }
    else
        context.Response.StatusCode = 404;

    context.Response.OutputStream.Close();
}
```

Этот веб-сервер работает _последовательно_ и _синхронно_.

Вызов `GetContext()` «замораживает» выполнение программы, пока на вход не поступит запрос для обработки. Обработка означает проверку параметров, отправку HTML-кода клиенту и установку корректного статуса.

Несмотря на синхронность, или, возможно, благодаря ей, сервер работает очень быстро. Он в состоянии обрабатывать до пятиста запросов в секунду на обычном офисном компьютере.

Проблемы возникают, если мы имитируем какую-то «продолжительную обработку» на стороне сервера, например, обращение к базе данных.

Раскомментируем вызов `Thread.Sleep(100)`, чтобы сервер ждал 0,1 секунды, прежде чем ответить на запрос. Теперь сервер можно обрабаывать порядка пятидесяти запросов в секунду, то есть в десять раз меньше, чем без «нагрузки».

У нашего сервера есть ещё одна проблема. Он нагружает только один процессор в системе, даже если их у нас несколько.

Попытаемся исправить ситуацию — разработаем параллельную версию программы.

#### Параллельный веб-сервер

```c#
var listener = new HttpListener();
listener.Prefixes.Add("http://localhost:8080/");
listener.Start();

while (true)
{
    var context = listener.GetContext();

    var thread = new Thread(ProcessRequest);
    thread.Start(context);
}

. . .

void ProcessRequest(object parameter)
{
    var context = (HttpListenerContext) parameter;
    if (context.Request.HttpMethod == "GET" && context.Request.RawUrl == "/")
    {
//        Thread.Sleep(100);
        context.Response.StatusCode = 200;

        using (var writer = new StreamWriter(context.Response.OutputStream))
        {
            writer.WriteLine("<!DOCTYPE html>");
            writer.WriteLine("<html lang='en' xmlns='http://www.w3.org/1999/xhtml'>");
            writer.WriteLine("  <head>");
            writer.WriteLine("  <meta charset='utf-8' />");
            writer.WriteLine("  <title>Example HTTP server</title>");
            writer.WriteLine("  </head>");
            writer.WriteLine("  <body>");
            writer.WriteLine("    <p>Example HTTP server</p>");
            writer.WriteLine("  </body>");
            writer.WriteLine("</html>");
        }
    }
    else
        context.Response.StatusCode = 404;

    context.Response.OutputStream.Close();
}
```

Исходный код программы остался практически неизменным. Нам пришлось вынести код обработки запроса в отдельный метод, потому что «параллельное выполнение» в .NET означает запуск метода в новом _потоке_ (_thread_).
Веб-сервер, получив запрос, создаёт поток для его обработки, а сам возвращается в режим ожидания — вызов `GetContext()`, как мы помним, «замораживает» главный поток.

Новая версия программы будет равномерно загружать все наши процессоры. Но, поскольку запуск потока — ресурсоёмкая операция, сервер окажется чуть медленнее, чем простая последовательная версия. Вместо пятисот запросов в секунду он сможет обрабатывать триста.

Однако, он гораздо лучше справляется с «нагрузкой». Вернув в программу паузу в 0,1 секунды, мы увидим, что производительность сервера осталась на прежнем уровне — триста запросов.

Удивительно, но создаваемые нами потоки простаивают большую часть времени. Они отправляют клиенту HTML-код и большую часть времени ожидают, когда все данные будут переданы по сети.

Нам не обязательно создавать так много потоков (один на каждый запрос), чтобы успевать обрабатывать всё.

Созданим асинхронную версию веб-сервера и проверим её производительность.

#### Асинхронный веб-сервер

```c#
var listener = new HttpListener();
listener.Prefixes.Add("http://localhost:8080/");
listener.Start();

listener.BeginGetContext(AsyncProcessRequest, listener);
Console.ReadKey();
listener.Stop();

. . .

void AsyncProcessRequest(IAsyncResult ar)
{
    var listener = (HttpListener)ar.AsyncState;
    listener.BeginGetContext(AsyncProcessRequest, listener);

    var context = listener.EndGetContext(ar);
    if (context.Request.HttpMethod == "GET" && context.Request.RawUrl == "/")
    {
        context.Response.StatusCode = 200;
//        Thread.Sleep(100);
        using (var memoryStream = new MemoryStream())
        using (var writer = new StreamWriter(memoryStream, leaveOpen: true))
        {
            writer.WriteLine("<!DOCTYPE html>");
            writer.WriteLine("<html lang='en' xmlns='http://www.w3.org/1999/xhtml'>");
            writer.WriteLine("  <head>");
            writer.WriteLine("  <meta charset='utf-8' />");
            writer.WriteLine("  <title>Example HTTP server</title>");
            writer.WriteLine("  </head>");
            writer.WriteLine("  <body>");
            writer.WriteLine("    <p>Example HTTP server</p>");
            writer.WriteLine("  </body>");
            writer.WriteLine("</html>");

            var buffer = memoryStream.ToArray();
            context.Response.OutputStream.BeginWrite(buffer, 0, buffer.Length,
                AsyncWriteResponse, context.Response.OutputStream);
        }
    }
    else
    {
        context.Response.StatusCode = 404;
        context.Response.OutputStream.Close();
    }
}

void AsyncWriteResponse(IAsyncResult ar)
{
    var outputStream = (Stream)ar.AsyncState;
    outputStream.EndWrite(ar);
    outputStream.Close();
}
```

Первое, на что мы обращаем внимание — возросшая сложность программы. Асинхронный код никогда не считался простым, поскольку мы не можем держать всю логику обработки в одном или двух методах.

Мы вынуждены разбить программу на _методы обратного вызова_ (_callback methods_) и аккуратно передавать состояние между ними.

В начале мы вызываем метод `BeginGetContext()`. Сразу после этого мы вынуждены вызывать `Console.ReadKey()`, чтобы приостановить программу, пока пользователь не нажмёт какую-нибудь клавишу. Раньше у нас подобной проблемы не возникало, потому что код работал в бесконечном синхронном цикле.

`BeginGetContext()` возвращает управление сразу, это основа асинхронного кода. Выполнение будет продолжено, как только на вход сервера поступит HTTP-запрос.

В этот момент будет вызван метод `AsyncProcessRequest`. Первое, что мы должны сделать — организовать подобие бесконечного цикла. Мы снова вызываем `BeginGetContext()`, чтобы перевести сервер в режим ожидания запроса. И снова управление вернётся в наш метод, чтобы мы могли завершить обработку.

Вызов `EndGetContext()` уведомит объект-сервер, что с нашим запросом всё в порядке. Для передачи HTML нам приходится создать буфер, чтобы собрать там данные для отправки. Отправку мы тоже сделаем асинхронной с помощью пары методов `BeginWrite()` и `EndWrite()`.

Этот код и правда громоздкий. Но что мы получили взамен? Большую скорость.

Наша программа спокойно обрабатывает тысячу запросов в секунду. Если мы раскомментируем строку `Thread.Sleep(100)`, количество запросов немного снизится — где-то до девятисот пятидесяти.

На самом деле, производительность нашей программы осталась на прежнем уровне. Проблема возникла, потому что мы _приостанавливаем_ поток выполнения, чего не следует делать в асинхронной программе.

Корректный код в нашем случае получится ещё более громоздким, поэтому мы не будем тратить на него время. Вместо этого узнаем, как просто выглядит асинхронный код в современном C#.

#### Современный асинхронный веб-сервер

```c#
var listener = new HttpListener();
listener.Prefixes.Add("http://localhost:8080/");
listener.Start();

GetContextAsync(listener).Wait();

Console.ReadKey();
listener.Stop();

. . .

async Task GetContextAsync(HttpListener listener)
{
    await Task.Yield();
	
    var context = await listener.GetContextAsync();

    await GetContextAsync(listener);

    if (context.Request.HttpMethod == "GET" && context.Request.RawUrl == "/")
    {
//        await Task.Delay(100);
        context.Response.StatusCode = 200;

        using (var writer = new StreamWriter(context.Response.OutputStream))
        {
            await writer.WriteLineAsync("<!DOCTYPE html>");
            await writer.WriteLineAsync("<html lang='en' xmlns='http://www.w3.org/1999/   xhtml'>");
            await writer.WriteLineAsync("  <head>");
            await writer.WriteLineAsync("  <meta charset='utf-8' />");
            await writer.WriteLineAsync("  <title>Example HTTP server</title>");
            await writer.WriteLineAsync("  </head>");
            await writer.WriteLineAsync("  <body>");
            await writer.WriteLineAsync("    <p>Example HTTP server</p>");
            await writer.WriteLineAsync("  </body>");
            await writer.WriteLineAsync("</html>");
        }

        context.Response.OutputStream.Close();
    }
    else
    {
        context.Response.StatusCode = 404;
        context.Response.OutputStream.Close();
    }
}
```

Эта версия программы похожа на нашу первую версию, последовательную и синхронную. Но, благодаря магии компилятора, на самом деле она работает асинхронно.

Компилируя метод, помеченный ключевым словом `async`, компилятор разбивает его на части там, где встречает асинхронные вызовы, помеченные ключевым словом `await`.

Эти части становятся «как бы» самостоятельными методами обратного вызова. И компилятор  обеспечивает передачу значений между ними.

В результате условный код

```c#
A();
var context = await listener.GetContextAsync();
B(listener);
```

превращается в что-то похожее на

```c#
A();
listener.BeginGetContext(B, listener);
```

Конечно, всё намного сложнее, чем я описал. Компилятор умеет работать не только с последовательным кодом, но и с циклами, и с ветвлениями. Он учитывает поток управления, чтобы асинхронный код работал также, как синхронный.

Если вам интересны детали этой _магии_, обратитесь к пятой главе книги [«C# для профессионалов» Джона Скита](http://www.williamspublishing.com/Books/978-5-8459-1909-0.html).

Что с производительностью нового сервера? Он работает с той же скоростью, что и предыдущий — тысяча запросов в секунду. Раскомментировав строку `await Task.Delay(100)` — корректную замену `Task.Delay(100)` для асинхронного кода — мы не увидим никакой потери производительности.

Насколько наша программа стала быстрее? Производительность зависит от множества факторов и точных цифр я вам не назову. Условно можно считать, что асинхронный код на том же железе обрабатывает в 3–5 раз больше запросов, чем параллельный.

Производительность достигается не за счёт большей скорости, а за счёт большей утилизации процессорного времени.

_Продолжение следует._