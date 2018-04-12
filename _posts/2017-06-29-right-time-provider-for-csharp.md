---
layout: post
title: Правильный поставщик времени для C#
date: 2017-06-29 17:15:00 +0300
excerpt: Как протестировать, что метод корректно вызывает DateTime.Now, ведь мы не знаем, какое значение будет установлено? Простое решение для C#.
image: https://github.com/jekyll/brand/blob/master/jekyll-notext-light-solid.png?raw=true
---

### Проблема

Покрыть модульными тестами код, который зависит от вызова `DateTime.Now` или `DateTimeOffset.Now`. Например, при создании
нового *поста*, *фабрика постов* заполняет поле *дата/время создания*:

```c#
public class PostFactory
{
    public Post Create(string title, string content)
    {
        return new Post(DateTimeOffset.Now, title, content);
    }
}
```

Как протестировать, что метод корректно заполняет поле, ведь при каждом вызове `DateTime.Now` возращает
разные значения? Один из способов заключается в том, чтобы спрятать зависимость от недетерменированного вызова
в отдельный класс. Он даже имеет устоявшееся название&nbsp;&mdash; *поставщик времени* (time provider).

### Классическое решение

```c#
public class TimeProvider
{
    public virtual DateTimeOffset Now { get { return DateTimeOffset.Now; } }
}
```

Мы регистрируем его в контейнере IoC, например, в Autofac:

```c#
builder.RegisterType<TimeProvider>()
       .AsSelf()
       .SingleInstance();
```

И *внедряем через конструктор* в фабрику постов:

```c#
public class PostFactory
{
    private readonly TimeProvider timeProvider;
        
    public PostFactory(TimeProvider timeProvider)
    {
        this.timeProvider = timeProvider;
    }
    
    public Post Create(string title, string content)
    {
        return new Post(timeProvider.Now, title, content);
    }
}
```

Основное преимущества класса перед `DateTime.Now` в том, что мы можем превратить его в заглушку:

```c#
public class StubTimeProvider : TimeProvider
{
    public DateTimeOffset DefaultNow { get; set; }
        
    public override DateTimeOffset Now => DefaultNow;
}
```

Эта заглушка всегда возвращает одно и то же значение текущей даты/времени, что не соответствует реальности,
зато позволяет нам тестировать код:

```c#
[TestMethod]
public void Create_WhenCalled_FillsCreatedAt()
{
    var timeProvider = new StubTimeProvider { DefaultNow = new DateTimeOffset(2017, 06, 29, 17, 32, 10, TimeSpan.Zero) };
    var postFactory = new PostFactory(timeProvider);
        
    var post = postFactory.Create("foo", "bar");
        
    Assert.AreEqual(new DateTimeOffset(2017, 06, 29, 17, 32, 10, TimeSpan.Zero), post.CreatedAt);
}
```

### Проблема классического решения

Поставщик времени оказывается очень фундаментальной штукой, которая нужна практически во всех проектах. Чтобы не дублировать код,
вы вынуждены завести специальный общий проект, в котором собираете все такие фундаментальные штуки. И этот проект очень скоро: а)
превращается в свалку плохо структурированного кода; б) подключается к каждой вашей программе.

В соответствии с [одним из 12-ти факторов](https://12factor.net/ru/dependencies), поставщик времени нужно вынести в отдельный
проект, и превратить его в NuGet-пакет. Кстати, такой пакет уже [существует](https://www.nuget.org/packages/rg.TimeProvider/),
но не поддерживает `DateTimeOffset`, только `DateTime`.

Microsoft между тем [утверждает](https://docs.microsoft.com/en-us/dotnet/standard/datetime/choosing-between-datetime),
что:

> These uses for `DateTimeOffset` values are much more common than those for `DateTime` values.
As a result, `DateTimeOffset` should be considered the default date and time type for application development.

То есть `DateTime` это прошлый век, все используем `DateTimeOffset`. Пора создавать свой собственный пакет. Или нет?

### Новое решение

Конечно, нет. Вместо `TimeProvider` мы можем использовать `Func<DateTimeOffset>`:

```c#
public class PostFactory
{
    private readonly Func<DateTimeOffset> now;
        
    public PostFactory(Func<DateTimeOffset> now)
    {
        this.now = now;
    }
    
    public Post Create(string title, string content)
    {
        return new Post(now(), title, content);
    }
}
```

Вот так регистрируем функцию в Autofac:

```c#
builder.RegisterInstance<Func<DateTimeOffset>(() => DateTimeOffset.Now);
```

И вот так используем в тестах:

```c#
[TestMethod]
public void Create_WhenCalled_FillsCreatedAt()
{
    Func<DateTimeOffset> now = () => new DateTimeOffset(2017, 06, 29, 17, 32, 10, TimeSpan.Zero);
    var postFactory = new PostFactory(now);
        
    var post = postFactory.Create("foo", "bar");
        
    Assert.AreEqual(new DateTimeOffset(2017, 06, 29, 17, 32, 10, TimeSpan.Zero), post.CreatedAt);
}
```

В результате нам удалось обойтись только стандартными средствами .NET Framework, не создавать NuGet-пакетов, и не множить зависимости.
