---
title: "Пишем интернет-магазин на C#"
date: "2020-06-16 19:00:00 +0300"
id: write-online-store-on-csharp
excerpt: "Онлайн-курс по C#, ООП, DDD, тестам, SOLID, паттернам проектирования, Agile."
---

Если вы молодой программист и у вас не очень много опыта, некоторые темы в современном программировании могут показаться вам слишком абстрактными.
Например, такие, как паттерны проектирования, SOLID или DDD.

Когда меня спрашивают про SOLID, я понимаю, что все короткие ответы меня не удовлетворяют. В то же время, работая в команде, я могу рассказывать младшим программистам об абстракциях на примере нашего собственного кода. Такие рассказы гораздо доходчивее.

Промышленная разработка — лучшая ситуация для обучения, если вам повезло с проектом, командой и тим-лидом. Но что делать, если не повезло?

В московском клубе программистов регулярно делают доклады на такие темы. К сожалению, они носят слишком обзорный характер. Во-первых, продолжительность доклада всего сорок минут; а во-вторых, этот формат не позволяет раскрывать детали.

Нужен совместный проект, в котором можно столкнуться с реальными проблемами и решать их, попутно осваивая теорию. Проект должен быть достаточно условным и учебным, чтобы за деревьями мы не потеряли лес — наша задача не в запуске программы, а в том, чтобы научиться.

### Проект

В качестве учебного проекта мы возьмём книжный интернет-магазин. Каждый из нас покупал книги в интернете и хорошо представляет предметную область. Нам не придётся пару недель разбираться в том, что мы хотим сделать.

С другой стороны, интернет-магазин уже достаточно велик и практичен, чтобы мы могли столкнуться с проблемами и понять, как абстрактные знания помогают их решать.

### Формат

Место обучения: Zoom<br />
Длительность занятия: 1,5 часа<br />
Расписание занятий: каждый вторник с 19:00 до 20:30, начиная с  28 апреля  (28.04, 05.05, 12.05, 19.05, 26.05, 02.06, 09.06, 16.06)

Продолжительность: 8 встреч. Полный курс займёт два месяца.
По согласованию с участниками, количество встреч может быть изменено.

Курс бесплатный и это накладывает ограничение на участников. Вам должно быть интересно учиться, потому что мне трудно вас мотивировать. Вы должны помогать друг другу, потому что я один не успею.

Что конкретно будет происходить:
* Мы будем обсуждать задачи и их решения.
* Я буду расшаривать экран и писать код, попутно объясняя, что и зачем я делаю.
* Я буду отвечать на вопросы.
* Весь урок будет записан и выложен в YouTube, так что вы сможете позже просмотреть его и поэкспериментировать с кодом самостоятельно.

### Темы
* C#
* SQL
* git и GitHub
* Agile: бэклог, пользовательские истории, итерации
* Модульное тестирование
* Паттерны проектирования
* Принципы SOLID
* DDD (Domain Driven Design)
* ASP.NET MVC
* Entity Framework

У нас интернет-магазин, поэтому, конечно, нам придётся использовать HTML, CSS и JavaScript. Я не планирую глубоко погружаться в эти темы, в том числе и потому, что глубоко в них не разбираюсь. Тем не менее, они будут.

### Пожелания

Не всем этот курс может быть полезен. Если вы хотите научиться программировать с нуля, этот курс окажется слишком сложным. Если вы старший программист, то вряд ли узнаете что-то новое.

Курс подойдёт вам, если вы программируете несколько лет и знаете два-три языка программирования. Хорошо, если один из этих языков является наследником C, то есть это C++, C#, Java, PHP или JavaScript.

Важно, чтобы вы понимали основы объектно-ориентированного программирования, чтобы вас не пугали термины *класс*, *объект* и *наследование*.

Разработка будет вестись в Visual Studio 2019 Community Edition. Это бесплатная IDE от Microsoft, которая работает под Windows. Её можно запустить на MacOS. Разрабатывать код можно будет и под Linux в вашем любимом редакторе.

## Занятие 1

### Запись

[Чат](/download/write-online-shop-on-csharp-1.txt)

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/L2OC525fkGk" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

## Ответы на вопросы

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/HNrJTL42p44" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

## Занятие 2

Модульное тестирование. Как разбивать код, чтобы покрывать его тестами.

[Чат](/download/write-online-shop-on-csharp-2.txt)

### Запись

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/2oxrED5FJf0" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

### Ответы на вопросы

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/i2_lQ3d2EIg" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

## Занятие 3

Как писать веб-приложения: ASP.NET Core, Razor, Сессии, как хранить корзину на сервере.

[Чат](/download/write-online-shop-on-csharp-3.txt)

### Запись

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/RQzh8lOUCCo" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

### Ответы на вопросы

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/iLjNvo5i758" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

## Занятие 4

Большой рефакторинг, агрегаты, репозитории.

[Чат](/download/write-online-shop-on-csharp-4.txt)

### Запись

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/ZzHUD2R0eEY" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

### Ответы на вопросы

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/BsxJQ4NaCh0" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

## Занятие 5

Open Closed Principle (код, открытый для расширения и закрытый для изменения).
Объявляем интерфейсы инфраструктурных служб. Создаём реализации-заглушки.

Реализуем две истории: подтверждение номера телефона и доставка через постамат.

[Чат](/download/write-online-shop-on-csharp-5.txt)

### Запись

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/AFrg30Tk5To" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

## Занятие 6

Реализуем заглушку платёжного сервиса. Razor Class Library.
Плагины.

[Чат](/download/write-online-shop-on-csharp-6.txt)

### Запись

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/K4CpLIo_Eew" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

### Ответы на вопросы

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/ofFOcb6dlIM" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

## Занятие 7

Entity Framework. Полнотекстовый поиск в EF Core и MS SQL.
Как настроить время жизни служб, репозиториев и контекста БД.

[Чат](/download/write-online-shop-on-csharp-7.txt)

### Запись

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/zcMdZvWtzSM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

## Занятие 8

Обработка ошибок в ASP.NET Core. Зачем асинхронное приложение. Ответы на вопросы.

[Чат](/download/write-online-shop-on-csharp-8.txt)

### Запись

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/mXEK4hhcI80" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

## Ответы на вопросы

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/xc4uv83rkNo" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>
