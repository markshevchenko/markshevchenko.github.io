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

Исходный код сайта нужно разместить в правильном репозитории. Название организации Московский клуб программистов на [GitHub](https://github.com/)
**progmsk**, следовательно, URI страницы клуба [https://github.com/**progmsk**](https://github.com/markshevchenko), а для сайта я
должен создать репозиторий [**progmsk**.github.io](https://github.com/markshevchenko/progmsk.github.io).

Сразу после создания репозитория сайт становится доступен по адресу http://progmsk.github.io. Сейчас он совершенно пуст.

![Пустой сайт на GitHub Pages]({{ "/img/github-pages-404.png" | relative_url }})

Нажмём кнопку <kbd>Create new file</kbd> в заголовке репозитория на сайте GitHub. В качестве имени нового файала укажем **_layouts/default.html**.
По правилам GitHub это означает, что в корне проекта будет создана папка **_layouts**, а в ней&nbsp;&mdash; файл **default.html**.
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

Наконец, в корне репозитория создадим пустой файл **_config.yml**.

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

Делать надо переменные сайта. Описать их можно в том самом пока пустом файле **_config.yml**:

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

Если мы сейчас уберём `lang: ru-RU` из теста страницы **index.md**, язык документа всё равно будет указан, потому что теперь он берётся из **_config.yml**.

## Оформление сайта&nbsp;&mdash; шаблоны

Большинство страниц вашего сайта похожи друг на друга. У них одинаковы заголовки, подвалы, панель навигации, а если вас подводит творческое начало, то и содержание.
Чтобы не повторять всё это на каждой странице, применяют *шаблоны*. GitHub позволяет описывать шаблоны страниц на языке [Liquid](http::/shopify.github.io/liquid/).

Создадим в папке **_layouts** файл **article.html** рядом с **default.html**:

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

Например создадим в репозитории файл **_includes/header.html**:

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

И файл **_includes/footer.html**:

{% raw %}
```liquid
<footer>
  <p>&copy; Московский клуб программистов.</p>
</footer>
```
{% endraw %}

Поскольку оба они размещены в каталоге **_includes**, мы можем включить их в шаблон **article.html** без указания пути:

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
Jekyll запустит Sass для его обработки и результат&nbsp;&mdash; **main.css**&nbps;&mdash; скопирут в итоговый сайт в каталог **css**.

В **_config.yml** добавим строки:

```yaml
sass:
  style: :compressed
```

В результате содержимое **main.css** будет [максимально утрамбовано](https://web-design-weekly.com/2014/06/15/different-sass-output-styles/).

## Своё доменное имя

Мы в клубе программистов зарегистрировали себе доменное имя **prog.msk.ru** и хотим, чтобы сайт был доступен по адресу http:/prog.msk.ru. Для
этого надо выполнить два шага:

1. На NS-сервере прописать две А-записи для **prog.msk.ru**, одну на IP 192.30.252.153, вторую на 192.30.252.154.
   После этого все обращения к **prog.msk.ru** будут вести на GitHub.
2. В репозитории GitHub зайти в раздел <kbd>Settings</kbd>, найти настройки GitHub Pages и в поле <kbd>Custom domain</kbd> ввести **prog.msk.ru**.

Всё.
