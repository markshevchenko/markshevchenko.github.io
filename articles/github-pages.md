---
title: Сайт на GitHub Pages
excerpt: Инструкция для начинающих
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

Исходный код сайта нужно разместить в правильном репозитории. Мой логин на [GitHub](https://github.com/) **markshevchenko**, следовательно,
URI моей страницы [https://github.com/**markshevchenko**](https://github.com/markshevchenko), а для своего сайта я должен создать репозиторий
[**markshevchenko**.github.io](https://github.com/markshevchenko/markshevchenko.github.io).

Сразу после создания репозитория сайт становится доступен по адресу http://markshevchenko.github.io. Сейчас он совершенно пуст.

Нажмём кнопку <kbd>Create new file</kbd> и в качестве имени укажем **_layouts/default.html**. По правилам GitHub это означает, что
в корне проекта будет создана папка **_layouts**, а в ней&nbsp;&mdash; файл **default.html**. Разместим внутри основной шаблон сайта:
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
    <main>
      {{ content }}
    </main>
  </body>
</html>
```
{% endraw %}

В корне создадим файл **index.md**, где запишем:

```markdown
---
layout: default
lang: ru-RU
title: История языков программирования
excerpt: От Ады до наших дней
---

История языков программирования полна *трагедий* и **разочарования**.
```

После сохранения изменений на сайте появится первая страница:

```html
<!doctype html>
<html lang="ru-RU">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>История языков программирования</title>
    <meta name="description" content="От Ады до наших дней">
  </head>
  <body>
    <main>
      <p>История языков программирования полна <em>трагедий</em> и <strong>разочарования</strong>.</p>
    </main>
  </body>
</html>
```

Что у нас получилось? Во-первых, мы подготовили шаблон сайта, где в парных фигурных скобках поместили названия переменных, выражения и даже управляющие конструкции.

Во-вторых, мы обнаружили, что переменные могут относиться как к сайту целиком (`site.description`), так и к отдельным страницам (`page.excerpt`).

В-третьих, мы увидели, что значения переменных страницы можно указать в её заголовке, отделив их от основного текста строками из трёх дефисов.
После всех мытарств, мы увидели правильно оформленный результат генерации сайта.

## Конфигурация сайта

Мы описываем переменные страницы, такие как `page.excerpt`, непосредственно в коде самой страницы:

```markdown
---
excerpt: От Ады до наших дней
---
```

А где же описываются переменные сайта? В файле **_config.yml**, который должен быть размещён в корне репозитория. Попробуем:

```yaml
---
lang: ru-RU
encoding: utf-8
author: "Mark Shevchenko"
url: http://markshevchenko.github.io
permalink: pretty
highlighter: rouge
markdown: kramdown
```

Здесь мы присвоили `lang` значение `ru-RU`. Теперь, если в тексте страницы не определено значение `lang`, в шаблоне **default.html** сработает второе правило:

{% raw %}
```liquid
<html lang="{{ page.lang | default: site.lang }}">
```
{% endraw %}

Если `page.lang` окажется пустым, вместо него Jekyll подставит `site.lang`.

Не все параметры в **_config.yml** нужны для шаблонов. Часть из них определяют, как будут создаваться страницы сайта.

`permalink`
: Задаёт структуру ссылок, то есть, по сути, структуру сайта. `pretty`&nbsp;&mdash; предопределённое значение, описанное в [документации](https://jekyllrb.com/docs/permalinks/).
  &laquo;Красивые&raquo; (pretty) ссылки означают, что для страницы **foo.md** будет создана папка **foo** с файлом **index.html** внутри. В результате адресом страницы будет **/foo/**,
  а не **/foo.html**.

`highlighter`
: Указывает инструмент, с помощью которого Jekyll подсвечивает код. По умолчанию применяется [Rouge](http://rouge.jneen.net/), который понимает
  больше [60-ти языков](https://github.com/jneen/rouge/wiki/List-of-supported-languages-and-lexers).

`markdown`
: Указывает инструмент конвертации Markdown в HTML. Мы предпочли [kramdown](https://kramdown.gettalong.org/).

## Оформление сайта&nbsp;&mdash; шаблоны

Большинство страниц вашего сайта похожи друг на друга. У них одинаковы заголовки, подвалы, панель навигации, а если вас подводит творческое начало, то и содержание
Чтобы не повторять всё это на каждой странице, применяют *шаблоны*. GitHub позволяет описывать шаблоны страниц на языке [Liquid](http::/shopify.github.io/liquid/).

Создадим в папке **_layouts** файл **article.html** рядом с **default.html**:

{% raw %}
```liquid
---
layout: default
---

<h1>{{ page.title }}{% if page.subtitle %}<br /><small>{{ page.subtitle }}</small>{% endif %}</h1>
{% if page.description %}<p>{{ page.description }}</p>{% endif %}

{{ content }}

{% if page.previous or page.up or page.next %}
<div id="relations">
  <ul>
    <li>{% if page.previous %}<a href="{{ page.previous }}">Назад</a>{% else %}Назад{% endif %}</li>
    <li>{% if page.up %}<a href="{{ page.up }}">Содержание</a>{% else %}Содержание{% endif %}</li>
    <li>{% if page.next %}<a href="{{ page.next }}">Вперёд</a>{% else %}Вперёд{% endif %}</li>
  </ul>
</div>
{% endif %}
```
{% endraw %}

В заголовке шаблона мы видим констуркцию `layout: default`, которая означает, что в качестве основы мы используем шаблон **default.html**. Содержимое нашего шаблона
будет встроено в **default.html** там, где указана конструкция {% raw %}`{{ content }}`{% endraw %}.

Если шаблон оказался слишком большим, отдельные блоки можно *вынести* в другие файлы, и подключать их с помощью инстуркции `include`:

Например создадим в репозитории файл **_includes/header.html**:

{% raw %}
```liquid
<header>
  <a id="home" href="{{"/" | relative_url}}"><img src="{{ "/img/logo.png" | relative_url}}" alt="На главную" /></a>
</header>

<nav id="menu">
  <ul>
    <li><a href="{{"/articles" | relative_url}}"><span class="icon-articles"></span> Статьи</a></li>
    <li><a href="{{"/posts" | relative_url}}"><span class="icon-blog"></span> Блог</a></li>
  </ul>
</nav>
```
{% endraw %}

И файл **_includes/footer.html**:

{% raw %}
```liquid
<nav id="profiles">
  <ul>
    <li><a href="https://github.com/markshevchenko"><span class="icon-github"></span> GitHub</a></li>
    <li><a href="http://stackoverflow.com/users/1051621/mark-shevchenko"><span class="icon-stackoverflow"></span> Stack Overflow</a></li>
    <li><a href="http://ru.stackoverflow.com/users/182162/mark-shevchenko"><span class="icon-stackoverflow"></span> Stack Overflow/рус</a></li>
  </ul>
</nav>

<footer>
  &copy; Писал и много раз переписывал Марк Шевченко, 1992&ndash;2018.
</footer>
```
{% endraw %}

Поскольку оба они размещены в каталоге **_includes**, мы можем включить их в шаблон **default.html** без указания пути:

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
    {% include header.html %}
    <main>
      {{ content }}
    </main>
    {% include footer.html %}
  </body>
</html>
```
{% endraw %}

## Оформление сайта&nbsp;&mdash; стили

[Sass](https://sass-lang.com/)&nbsp;&mdash; аналог шаблонов для CSS. Он позволяет упростить описание стилей и исключить дублирование. Для его корректной работы
при генерации сайта необходимо запускать утилиту, которая на основании **.scss** файлов создаст файлы **.css**.

Чтобы всё заработало, мы должны разместить описания стилей в правильном месте. Первое: создадим файл **css/main.scss**:

```scss
---
---

@charset "utf-8";

$base-font-family: Corbel, Trebuchet MS, Helvetica, Trebuchet, sans-serif;
$base-font-size:   18pt;
$small-font-size:  $base-font-size * 0.8;
$base-line-height: 1.2;

$monospace-font-family: Consolas, American Typewriter, Courier New, monospace;

$spacing-unit:     8px;
$aside-width:      280px;
$text-color:       black;
$background-color: white;
$brand-color:      crimson;
$brand-color-dark: firebrick;

$grey-color:       grey;
$grey-color-light: whitesmoke;
$grey-color-dark:  darkslategrey;

@import "base";
@import "layout";
@import "syntax-highlighting";
```

Обратите внимание, что в начале файла должны находиться две строки из трёх дефисов. Мы разобьём наши стили на четыре части. В **main.scss** мы задаём
значения основных переменных: стандартные цвета, стандартные отступы, стандартные шрифты. Их мы используем при описании стилей.

В конце мы *импортируем* файлы **base.scss**, **layout.scss** и **syntax-highlighting.scss**. Они должны размещаться в папке **_sass**. Вот так, например, выглядит файл
**_sass/base.scss**:

```scss
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
  color: $text-color;
  background-color: $background-color;
}

h1, h2, h3, h4, h5, h6,
p, blockquote, pre,
ul, ol, dl, figure,
%vertical-rhythm {
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
  color: $brand-color;

  &:visited {
    color: $brand-color-dark;
  }
}

%clearfix {
  &:after {
    content: "";
    display: table;
    clear: both;
  }
}
```

Чтобы упростить оформление, мы описываем стили базовых тегов, как `body` и `h1` в файле **base.scss**, и отдельно, в файле
[**layout.scss**](https://github.com/markshevchenko/markshevchenko.github.io/blob/master/_sass/layout.scss) описываем стили классов и идентификаторов,
специфичные именно для нашего дизайна.

В [**syntax-highlighting.scss**](https://github.com/markshevchenko/markshevchenko.github.io/blob/master/_sass/syntax-highlighting.scss) вынесем описания стилей подсветки кода.

В **_config.yml** добавим строки:

```yaml
sass:
  sass_dir: _sass
  style: :compressed
```

Финальным аккордом будет подключение стилей в заголовке шаблона **default.html**:

{% raw %}
```liquid
<!doctype html>
<html lang="{{ page.lang | default: site.lang }}">
  <head>
    . . .
    <link rel="stylesheet" href="{{ "/css/main.css" | relative_url }}">
  </head>
  . . .
</html>
```
{% endraw %}

Сохраняем изменения и через несколько секунд наслаждаемся новым оформлением нашего сайта.