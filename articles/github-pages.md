---
title: Сайт на GitHub Pages
description: GitHub Pages, инструкция для начинающих.
id: github-pages
---

## Что это

[GitHub Pages](https://pages.github.com/)&nbsp;&mdash; служба хостинга сайтов, которая позволяет вести персональные сайты, сайты организаций и сайты
отдельных проектов GitHub.

Сайты GitHub Pages статические. Текст страниц в формате [Markdown](https://daringfireball.net/projects/markdown/) лежит в репозитории GitHub.
Мы пишем новые статьи, фиксируем (commit) изменения и проталкиваем (push) их в ветку `master` центрального репозитория. GitHub самостоятельно
запускает генератор [Jekyll](https://jekyllrb.com/), и через несколько секунд публикует готовые HTML-страницы на сайте.

Возможности статических сайтов ограничены. Тем не менее Jekyll позволяет:

1. Вести блог.
1. Анонсировать записи в формате [Atom](https://tools.ietf.org/html/rfc4287).
1. Подсвечивать код.
1. Контролировать внешний вид сайта.

## Быстрый старт

Исходный код сайта нужно разместить в правильном репозитории. Название организации Московский клуб программистов на [GitHub](https://github.com/)
**progmsk**, следовательно, URI страницы клуба [https://github.com/**progmsk**](https://github.com/markshevchenko), а для сайта я
должен создать репозиторий [**progmsk**.github.io](https://github.com/markshevchenko/progmsk.github.io).

Сразу после создания репозитория сайт становится доступен по адресу http://progmsk.github.io. Сейчас он совершенно пуст.

![Пустой сайт на GitHub Pages]({{ "/img/github-pages-404.png" | relative_url }})

Нажмём кнопку <kbd>Create new file</kbd> в заголовке репозитория на сайте GitHub. В качестве имени нового файала укажем **\_layouts/default.html**.
По правилам GitHub это означает, что в корне проекта будет создана папка **\_layouts**, а в ней&nbsp;&mdash; файл **default.html**.
Разместим внутри основной шаблон сайта:
{% raw %}
```liquid
<!doctype html>
<html lang="{{ page.lang | default: site.lang }}">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{% if page.title %}{{ page.title | escape }}{% else %}}{{ site.title | escape }}{% endif %}</title>
    <meta name="description" content="{{ page.excerpt | default: site.description | strip_html | normalize_whitespace | truncate: 160 | escape }}">
  </head>
  <body>
    {{ content }}
  </body>
</html>
```
{% endraw %}

В корне создадим файл **index.md**, где запишем:

```markdown
---
layout: default
lang: ru-RU
title: Московский клуб программистов
excerpt: Тяжела и неказиста жизнь простого программиста
---

Как выяснилось, в Москве есть большая потребность в неформальных встречах программистов,
без привязки к конкретным технологиям и языкам.

Ниша «посидеть и поговорить» оказалась незаполненной.
Этот пробел требует немедленной ликвидации, которой мы и занимаемся.
```

Наконец, в корне репозитория создадим пустой файл **\_config.yml**.

После сохранения изменений (после фиксации их в ветке `master`), GitHub запускает Jekyll, чтобы сгенерировать
набор статических HTML-страниц. Как правило, в течение нескольких секунд вам доступна новая версия сайта.

```html
<!doctype html>
<html lang="ru-RU">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Московский клуб программистов</title>
    <meta name="description" content="Тяжела и неказиста жизнь простого программиста">
  </head>
  <body>
    <p>Как выяснилось, в Москве есть большая потребность в неформальных встречах программистов,
без привязки к конкретным технологиям и языкам.</p>

<p>Ниша «посидеть и поговорить» оказалась незаполненной.
Этот пробел требует немедленной ликвидации, которой мы и занимаемся.</p>

  </body>
</html>
```

Что у нас получилось? Во-первых, мы подготовили шаблон сайта. Ещё раз внимательно взгляните на **default.html**. Вместе с тегами HTML, шаблоны
позволяют вставлять в текст страницы переменные или использовать управляющие конструкции, чтобы скрывать или повторять части страниц.

Дальше мы рассмотрим шаблонизатор детально.

Во-вторых, мы поняли, как описывать наши переменные. Всё, что относится к странице, можно описать в её заголовке, выделив его тройными
дефисами.

```markdown
---
layout: default
lang: ru-RU
title: Московский клуб программистов
excerpt: Мы такие же, как все, только на работу ездим в метро
---
```

Переменная `layout` показывает, что при генерации HTML надо применить шаблон **default.html**.
Переменные `lang`, `title` и `excerpt` в тексте страницы доступны как `page.lang`, `page.title` и `page.excerpt`.

После всех мытарств, мы видим правильно оформленный результат генерации сайта.

## Конфигурация сайта

Конечно, встречаются в этой жизни полиглоты, но вряд ли вы будете вести свой сайт на разных языках. Скорее всего, язык всегда
будет один и тот же, скажем, русский. Вставлять в текст каждой страницы конструкцию `lang: ru-RU` вам быстро надоест. Что делать?

Делать надо переменные сайта. Описать их можно в том самом пока пустом файле **\_config.yml**:

```yaml
lang: ru-RU
encoding: utf-8
url: http://progmsk.github.io
permalink: pretty
highlighter: rouge
markdown: kramdown
```

Здесь мы присвоили переменной `lang` значение `ru-RU`, в шаблонах она доступна нам как `site.lang`. Теперь становится понятен смысл конструкции:

{% raw %}
```liquid
<html lang="{{ page.lang | default: site.lang }}">
```
{% endraw %}

Если не указать `lang` в тексте страницы, `page.lang` окажется пустым; и вместо него Jekyll подставит `site.lang`.

Часть переменных это на самом деле параметры. Они определяют, как должен выглядеть сайт.

permalink
: Задаёт структуру ссылок, то есть, по сути, структуру сайта. `pretty`&nbsp;&mdash; предопределённое значение, описанное в [документации](https://jekyllrb.com/docs/permalinks/).
  &laquo;Красивые&raquo; (pretty) ссылки означают, что для страницы **foo.md** будет создана папка **foo** с файлом **index.html** внутри. В результате адресом страницы будет красивый **/foo/**, а не страшный **/foo.html**.

highlighter
: Указывает инструмент, с помощью которого Jekyll подсвечивает код. По умолчанию применяется [Rouge](http://rouge.jneen.net/), который понимает
  больше [60-ти языков](https://github.com/jneen/rouge/wiki/List-of-supported-languages-and-lexers).

markdown
: Указывает инструмент конвертации Markdown в HTML. Мы предпочли [kramdown](https://kramdown.gettalong.org/).

Если мы сейчас уберём `lang: ru-RU` из теста страницы **index.md**, язык документа всё равно будет указан, потому что теперь он берётся из **\_config.yml**.

## Оформление сайта&nbsp;&mdash; шаблоны

Большинство страниц вашего сайта похожи друг на друга. У них одинаковы заголовки, подвалы, панель навигации, а если вас подводит творческое начало, то и содержание.
Чтобы не повторять всё это на каждой странице, применяют *шаблоны*. GitHub позволяет описывать шаблоны страниц на языке [Liquid](http::/shopify.github.io/liquid/).

Создадим в папке **\_layouts** файл **article.html** рядом с **default.html**:
{% raw %}
```liquid
---
layout: default
---
<header>
  <h1>{{ page.title }}</h1>
  {% if page.subtitle %}
  <h2>{{ page.subtitle }}</h2>
  {% endif %}
</header>

<main>
  {{ content }}
</main>

<footer>
  <p>&copy; Московский клуб программистов.</p>
</footer>

```
{% endraw %}
В заголовке шаблона мы видим констуркцию `layout: default`, которая означает, что в качестве основы мы используем шаблон **default.html**. Содержимое нашего шаблона
будет встроено в **default.html** там, где указана конструкция {% raw %}`{{ content }}`{% endraw %}.

Теперь откроем **index.md** и изменим шаблон на новый `article`:

```markdown
---
layout: article
title: Московский клуб программистов
excerpt: Тяжела и неказиста жизнь простого программиста
---

. . .
```

Наша страница будет вставлена в **article.html**, который, в свою очередь, будет вставлен в **default.html**, и в результате мы получим такой HTML:

```html
<!doctype html>
<html lang="ru-RU">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Московский клуб программистов</title>
    <meta name="description" content="Тяжела и неказиста жизнь простого программиста">
  </head>
  <body>
    <header>
  <h1>Московский клуб программистов</h1>
  
</header>

<main>
<p>Как выяснилось, в Москве есть большая потребность в неформальных встречах программистов,
без привязки к конкретным технологиям и языкам.</p>

<p>Ниша «посидеть и поговорить» оказалась незаполненной.
Этот пробел требует немедленной ликвидации, которой мы и занимаемся.</p>

</main>

<footer>
  <p>&copy; Московский клуб программистов.</p>
</footer>

  </body>
</html>
```

Если шаблон оказался слишком большим, отдельные блоки можно *вынести* в другие файлы, и подключить их с помощью инстуркции `include`:

Например создадим в репозитории файл **\_includes/header.html**:

{% raw %}
```liquid
<header>
  <h1>{{ page.title }}</h1>
  {% if page.subtitle %}
  <h2>{{ page.subtitle }}</h2>
  {% endif %}
</header>
```
{% endraw %}
И файл **\_includes/footer.html**:
{% raw %}
```liquid
<footer>
  <p>&copy; Московский клуб программистов.</p>
</footer>
```
{% endraw %}
Поскольку оба они размещены в каталоге **\_includes**, мы можем включить их в шаблон **article.html** без указания пути:
{% raw %}
```liquid
---
layout: default
---
{% include header.html %}
<main>
{{ content }}
</main>
{% include footer.html %}
```
{% endraw %}
## Оформление сайта&nbsp;&mdash; стили
Настало время рассмотреть детали генерации сайта. Каталоги, название которых начинается с подчёркивания, используются при генерации, но сами
на сайт не попадают. Обычные каталоги копируются &laquo;как есть&raquo; вместе со всем содержимым. При этом Jekyll обрабатывает файлы с расширением
**.md** и **.html** так, как мы разобрали выше.

Добавим на сайт описание стилей **css/main.css**:
```css
body, h1, h2, h3, h4, h5, h6,
p, blockquote, pre, hr,
dl, dd, ol, ul, figure {
  margin: 0;
  padding: 0;
}

body {
  font-family: Corbel, Trebuchet MS, Helvetica, Trebuchet, sans-serif;
  font-size: 18pt;
  line-height: 1.2;
  font-weight: normal;
  color: black;
  background-color: white;
}

h1, h2, h3, h4, h5, h6,
p, blockquote, pre,
ul, ol, dl, figure {
  margin-bottom: 0.25em;
}

li,
li > ul,
li > ol {
  margin-bottom: 0;
}

h1, h2, h3, h4, h5, h6 {
  font-weight: normal;
}

a {
  color: tomato;
}

a:visited {
  color: indianred;
}
```

Подключим стили в шаблоне **default.html**:

{% raw %}
```liquid
<!doctype html>
<html lang="{{ page.lang | default: site.lang }}">
  <head>
    . . .
    <link rel="stylesheet" href="{{ "/css/main.css" | relative_url }}">
  </head>
  <body>
    {{ content }}
  </body>
</html>
```
{% endraw %}

Немного странный `href` означает, что Jekyll будет генерировать корректный относительный путь к файлу стилей для каждой страницы.
После фиксации изменений вы увидите, что сайт стал выглядеть &laquo;красиво&raquo;.

Помимо CSS Jekyll умеет запускать шаблонизатор [Sass](https://sass-lang.com/) для генерации стилей.
Начнём редактирование **main.css** и заодно переименуем его в **main.scss**:


```scss
---
---

@charset "utf-8";

$base-font-family: Corbel, Trebuchet MS, Helvetica, Trebuchet, sans-serif;
$base-font-size:   18pt;
$small-font-size:  $base-font-size * 0.8;
$base-line-height: 1.2;

$foreground-color: black;
$background-color: white;
$link-color: tomato;

body, h1, h2, h3, h4, h5, h6,
p, blockquote, pre, hr,
dl, dd, ol, ul, figure {
  margin: 0;
  padding: 0;
}

body {
  font-family: $base-font-family;
  font-size: $base-font-size;
  line-height: $base-line-height;
  font-weight: normal;
  color: $foreground-color;
  background-color: $background-color;
}

h1, h2, h3, h4, h5, h6,
p, blockquote, pre,
ul, ol, dl, figure {
  margin-bottom: 0.25em;
}

li {
  > ul,
  > ol {
    margin-bottom: 0;
  }
}

h1, h2, h3, h4, h5, h6 {
  font-weight: normal;
}

a {
  color: $link-color;
  
  &:visited {
    color: $link-color/2;
  }
}
```

Обратите внимание, что в начале файла должны находиться две строки из трёх дефисов. Встретив файл с таким заголовком,
Jekyll запустит Sass для его обработки и результат&nbsp;&mdash; **main.css**&nbsp;&mdash; скопирут в итоговый сайт в каталог **css**.

В **\_config.yml** добавим строки:

```yaml
sass:
  style: :compressed
```
В результате содержимое **main.css** будет [максимально утрамбовано](https://web-design-weekly.com/2014/06/15/different-sass-output-styles/).

## Своё доменное имя

Мы в клубе программистов зарегистрировали себе доменное имя **prog.msk.ru** и хотим, чтобы сайт был доступен по адресу **http:/prog.msk.ru**. Для
этого надо выполнить два шага:
1. На NS-сервере прописать две А-записи для **prog.msk.ru**, одну на IP 192.30.252.153, вторую на 192.30.252.154.
   После этого все обращения к **prog.msk.ru** будут вести на GitHub.
2. В репозитории GitHub зайти в раздел **Settings**, найти настройки GitHub Pages и в поле **Custom domain** ввести **prog.msk.ru**.

## Блог

Страницы, которые вы разместите в каталоге **\_posts** при генерации сайта превратятся в записи блога. Названия страницам надо дать
правильные, например **2018-04-02-first-post.md**. В начале имени файла мы пишем дату, а через дефис&nbsp;&mdash; заголовок на английском языке,
как он должен выглядеть в адресной строке браузера.

Создадим в GitHub файл **\_posts/2018-04-02-first-post.md**:

{% raw %}
```liquid
---
layout: default
title: Тестовая запись
---

Проверка работы GitHub Pages.

<!--more-->

Создали тестовую запись.

```
{% endraw %}

Осталось показать запись в списке. Мы можем создать отдельную страницу, которая будет центральной страницей блога. Пока ограничимся тем, что
внесём необходимые изменения в **index.md**:

{% raw %}
```liquid
---
layout: article
title: Московский клуб программистов
excerpt: Тяжела и неказиста жизнь простого программиста
---

Как выяснилось, в Москве есть большая потребность в неформальных встречах программистов,
без привязки к конкретным технологиям и языкам.

Ниша «посидеть и поговорить» оказалась незаполненной.
Этот пробел требует немедленной ликвидации, которой мы и занимаемся.

{% for post in site.posts %}
  <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
  {{ post.excerpt }}
{% endfor %}
```
{% endraw %}

Несколько секунд, и на нашем сайте появится ссылка на первый пост:

```html
<!doctype html>
. . .
<p>Ниша «посидеть и поговорить» оказалась незаполненной.
Этот пробел требует немедленной ликвидации, которой мы и занимаемся.</p>

<h2><a href="/2018/04/02/first-post/">Тестовая запись</a></h2>
<p>Проверка работы GitHub Pages.</p>

. . .
```

Посмотрим на то, какой код создал Jekyll. Дата **2018-04-02** (2 апреля 2018 года) была преобразована в часть пути.
Сама страница получила имя **first-post**. Полное имя нашей записи на сервере&nbsp;&mdash; **/2018/04/02/first-post/index.html**. Так как раньше мы
установили свойство сайта **permalink** в **pretty**, Jekyll для каждой страницы создаёт папку с файлом **index.html** внутри.

Взглянем на код, который выводит список записей:

{% raw %}
```liquid
{% for post in site.posts %}
  <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
  {{ post.excerpt }}
{% endfor %}
```
{% endraw %}

Очевидно `site.posts` содержит все записи из каталога **\_posts**. `post.title`&nbsp;&mdash; содержимое свойства `title` страницы, а `post.url`&nbsp;&mdash;
её адрес. `post.excerpt` это *анонс*. Обычно Jekyll использует в качестве анонса первый абзац записи. Мы можем изменить это поведение,
вписав в код страницы специальный маркер `<!--more-->`. Анонсом станет всё, что находится в тексте выше этого маркера.

## Постраничый список записей

Если записей станет слишком много, например, сто, разбираться в их списке станет неудобно. Чтобы сократить количество одновременно доступных записей,
их разбивают на страницы.

Jekyll разбивает записи на страницы с помощью плагина **jekyll-paginate**. Подключим и настроим его в конфигурационном файле **\_config.yml**:

```yaml
lang: ru-RU
encoding: utf-8
url: http://prog.msk.ru
permalink: pretty
highlighter: rouge
markdown: kramdown

gems:
  - jekyll-paginate

paginate: 10
paginate_path: "/:num"
```

Параметр `paginate` позволяет указать количество записей на странице, а `paginate_path` устанавливает формат пути. Скажем, формат **/blog/page:num** означает,
что для всех страниц будут созданы каталоги **/blog/page1**, **/blog/page2**, и так далее.

Нашему формату **/:num** соответствуют каталоги **/1**, **/2**, **/3**, &hellip;

Теперь организуем постраничный вывод записей. Первое, что нам потербуется: переделать **index.md** в **index.html**, потому что **jekyll-paginate** работает
только с **.html**-файлами. Второе: вместо переменной `site.posts` мы будем получать записи из `paginator.posts`. Итак, откроем **index.md**, переименуем его в
**index.html** и перепишем содержимое так:

{% raw %}
```liquid
---
layout: article
title: Московский клуб программистов
excerpt: Тяжела и неказиста жизнь простого программиста
---

<p>Как выяснилось, в Москве есть большая потребность в неформальных встречах программистов,
без привязки к конкретным технологиям и языкам.</p>

<p>Ниша «посидеть и поговорить» оказалась незаполненной.
Этот пробел требует немедленной ликвидации, которой мы и занимаемся.</p>

{% for post in paginator.posts %}
  <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
  {{ post.excerpt }}
{% endfor %}

{% if paginator.total_pages > 1 %}
<ul>
  {% if paginator.previous_page %}
    <li><a href="{{ paginator.previous_page_path }}">Назад</a></li>
  {% else %}
    <li>Назад</li>
  {% endif %}
    <li>Страница {{ paginator.page }} из {{ paginator.total_pages }}</li>
  {% if paginator.next_page %}
    <li><a href="{{ paginator.next_page_path }}">Вперёд</a></li>
  {% else %}
    <li class="next">Вперёд</li>
  {% endif %}
</ul>
{% endif %}
```
{% endraw %}

Как и прежде, публикация списка записей реализована просто. Зато ниже мы видим сложный код, который отвечает отображает ссылки на
следующую и предыдующую страницы.

## Синдикация новостей

Jekyll умеет создавать ленту новостей в формате [Atom](https://tools.ietf.org/html/rfc4287). В список `gems` файла **\_config.yml** добавим плагин **jekyll-feed**
и опишем имя XML-файла, куда попадут анонсы записей:

```yaml
lang: ru-RU
encoding: utf-8

url: http://prog.msk.ru
title: Московский клуб программистов

permalink: pretty
highlighter: rouge
markdown: kramdown

gems:
  - jekyll-paginate
  - jekyll-feed

paginate: 10
paginate_path: "/:num"
```

Наконец, нам потребуется подставить ссылку Atom в заголовок наших страниц. Добавим мета-тег {% raw %}`{% feed_meta %}`{% endraw %} в заголовок
основного HTML-шаблона **default.html**:

{% raw %}
```liquid
<!doctype html>
<html lang="{{ page.lang | default: site.lang }}">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{% if page.title %}{{ page.title | escape }}{% else %}}{{ site.title | escape }}{% endif %}</title>
    <meta name="description" content="{{ page.excerpt | default: site.description | strip_html | normalize_whitespace | truncate: 160 | escape }}">
    <link rel="stylesheet" href="{{ "/css/main.css" | relative_url }}">
    {% feed_meta %}
  </head>
  <body>
    {{ content }}
  </body>
</html>
```
{% endraw %}

После компиляции сайта в коде страницы мы увидим ссылку на документ Atom как раз на месте {% raw %}`{% feed_meta %}`{% endraw %}.

```html
<link type="application/atom+xml" rel="alternate" href="http://prog.msk.ru/feed.xml" title="Московский клуб программистов" />
```

Наконец, вот и сам документ:
```xml
<feed xmlns="http://www.w3.org/2005/Atom" xml:lang="ru-RU">
  <generator uri="https://jekyllrb.com/" version="3.7.3">Jekyll</generator>
  <link href="http://prog.msk.ru/feed.xml" rel="self" type="application/atom+xml"/>
  <link href="http://prog.msk.ru/" rel="alternate" type="text/html" hreflang="ru-RU"/>
  <updated>2018-04-03T15:39:58+00:00</updated>
  <id>http://prog.msk.ru/</id>
  <title type="html">progmsk.github.io</title>
  <subtitle>Сайт московского клуба программистов</subtitle>
  <entry>
    <title type="html">Тестовая запись</title>
    <link href="http://prog.msk.ru/2018/04/02/first-post/" rel="alternate" type="text/html" title="Тестовая запись"/>
    <published>2018-04-02T00:00:00+00:00</published>
    <updated>2018-04-02T00:00:00+00:00</updated>
    <id>http://prog.msk.ru/2018/04/02/first-post</id>
    <content type="html" xml:base="http://prog.msk.ru/2018/04/02/first-post/">
      <p>Проверка работы GitHub Pages.</p> <!--more--> <p>Создали тестовую запись.</p>
    </content>
    <author>
      <name/>
    </author>
    <summary type="html">Проверка работы GitHub Pages.</summary>
  </entry>
</feed>
```

## Страницы

Если наш сайт будет не только блогом (а он не будет), удобно на главной странице разместить общую информацию, а для блога выделить
отдельную страницу.

Создадим в корне репозитория файл **blog/index.html** и перенесём туда код генерации списка записей:

{% raw %}
```liquid
---
layout: article
title: Блог клуба программистов
---
<p>
  <a href="/">На главную</a>
</p>
{% for post in paginator.posts %}
  <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
  {{ post.excerpt }}
{% endfor %}

{% if paginator.total_pages > 1 %}
<ul>
  {% if paginator.previous_page %}
    <li><a href="{{ paginator.previous_page_path }}">Назад</a></li>
  {% else %}
    <li>Назад</li>
  {% endif %}
    <li>Страница {{ paginator.page }} из {{ paginator.total_pages }}</li>
  {% if paginator.next_page %}
    <li><a href="{{ paginator.next_page_path }}">Вперёд</a></li>
  {% else %}
    <li class="next">Вперёд</li>
  {% endif %}
</ul>
{% endif %}
```
{% endraw %}

В корневом **index.html** поставим ссылку на блог:

```html
---
layout: article
title: Московский клуб программистов
excerpt: Тяжела и неказиста жизнь простого программиста
---

<p>Как выяснилось, в Москве есть большая потребность в неформальных встречах программистов,
без привязки к конкретным технологиям и языкам.</p>

<p>Ниша «посидеть и поговорить» оказалась незаполненной.
Этот пробел требует немедленной ликвидации, которой мы и занимаемся.</p>

<p><a href="/blog/">Блог клуба программистов</a>.</p>

```

Чтобы ссылки на страницы блога привести к общему виду, в файле **\_config.yml** изменим параметр `paginate_path`:

```yaml
paginate_path: "blog/:num"
```

## Динамическое содержимое

Поскольку наш сайт статический, кажется, что мы не можем показывать динамическое содержимое, например, ленту другого блога. Но мы&nbsp;&mdash;
можем, с помощью JavaScript.

Разместим на главной странице **index.html** вот этот код:

```html
---
layout: article
title: Московский клуб программистов
excerpt: Тяжела и неказиста жизнь простого программиста
---

<p>Как выяснилось, в Москве есть большая потребность в неформальных встречах программистов,
без привязки к конкретным технологиям и языкам.</p>

<p>Ниша «посидеть и поговорить» оказалась незаполненной.
Этот пробел требует немедленной ликвидации, которой мы и занимаемся.</p>

<p><a href="/blog/">Блог клуба программистов</a>.</p>

<div id="posts">
</div>

<script>
  window.onload = function() {
    var request = new XMLHttpRequest();
    request.open('GET', 'http://markshevchenko.pro/feed.xml', true);
    request.onload = function(data) {
      if (data.target.status >= 200 && data.target.status < 300) {
        var feed = data.target.responseXML;
        var entries = feed.getElementsByTagName('entry');
        makeHtml(entries);
      }
    };

    request.send();

    function makeHtml(entries) {
      for (var i = 0; i < entries.length; i++) {
        var entry = parseEntry(entries[i]);

        var titleLink = document.createElement('a');
        titleLink.setAttribute('href', entry.href);
        titleLink.appendChild(document.createTextNode(entry.title));
            
        var title = document.createElement('h2');
        title.appendChild(titleLink);

        var date = document.createElement('p');
        date.setAttribute('class', 'date');
        date.appendChild(document.createTextNode(entry.published.toLocaleString()));

        var author = document.createElement('p');
        author.setAttribute('class', 'author');
        author.appendChild(document.createTextNode(entry.author));

        var summary = document.createElement('p');
        summary.appendChild(document.createTextNode(entry.summary));

        document.getElementById('posts').appendChild(title);
        document.getElementById('posts').appendChild(date);
        document.getElementById('posts').appendChild(author);
        document.getElementById('posts').appendChild(summary);
      }
    }

    function parseEntry(entry) {
      return {
        title: entry.getElementsByTagName('link')[0].getAttribute('title'),
        href: entry.getElementsByTagName('link')[0].getAttribute('href'),
        published: new Date(entry.getElementsByTagName('published')[0].innerHTML),
        author: entry.getElementsByTagName('name')[0].innerHTML,
        summary: entry.getElementsByTagName('summary')[0].innerHTML
      };
    }
  }
</script>
```

Этот простейший скрипт на ванильном JavaScript читает и разбирает ленту одного блога, и затем вставляет её на стринцу.