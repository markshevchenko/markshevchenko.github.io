---
title: Сайт на GitHub Pages
excerpt: Инструкция для начинающих
id: github-pages
---

## Что это

[GitHub Pages](https://pages.github.com/)&nbsp;&mdash; служба хостинга сайтов, которая позволяет вести персональные сайты, сайты организаций и сайты
отдельных проектов GitHub.

Сайты GitHub Pages статические. Текст страниц в формате [Markdown](https://daringfireball.net/projects/markdown/) хранится в репозитории GitHub.
При проталкивании (push) изменений в ветку `master` GitHub запускает генератор [Jekyll](https://jekyllrb.com/), и через несколько секунд
публикует готовые HTML-страницы на сайте.

Возможности статических сайтов ограничены, но тем не менее Jekyll позволяет:

1. Вести блог.
1. Анонсировать записи в формате [Atom](https://tools.ietf.org/html/rfc4287).
1. Подсвечивать код.
1. Контролировать внешний вид сайта.

## Быстрый старт

Исходный код сайта нужно разместить в правильном репозитории. Мой логин на [GitHub](https://github.com/) markshevchenko, следовательно,
URI моей страницы https://github.com/markshevchenko, а для своего сайта я должен создать репозиторий
[markshevchenko.github.io](https://github.com/markshevchenko/markshevchenko.github.io).

Сразу после создания сайт становится доступен по адресу http://markshevchenko.github.io. Сейчас он совершенно пуст.

Нажмём кнопку <kbd>Create new file</kbd> и в качестве имени укажем **_layouts/default.html**. По правилам GitHub это означает, что
в корне проекта будет создана папка **_layouts**, а в ней&nbsp;&mdash; файл **default.html**. Разместим внутри шаблон:
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
    <p>История языков программирования полна <em>трагедий</em> и <strong>разочарования</strong>.</p>
  </body>
</html>
```

Что у нас получилось? Во-первых, мы подготовили шаблон сайта, где в парных фигурных скобках поместили названия переменных, выражения и даже управляющие конструкции.

Во-вторых, мы обнаружили, что переменные могут относиться как к сайту целиком (`site.description`), так и к отдельным страницам (`page.excerpt`).

В-третьих, мы увидели, что значения переменных старницы можно указать в её заголовке, отделив их от основного текста строками из трёх дефисов.
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

Но не все переменные здесь нужны для шаблонов. Часть из них определяют, как будут создаваться страницы сайта.

`permalink`
: Задаёт структуру ссылок, то есть, по сути, структуру сайта. `pretty`&nbsp;&mdash; предопределённое значение, описанное в [документации](https://jekyllrb.com/docs/permalinks/).
  &laquo;Красивые&raquo; (pretty) ссылки означают, что для страницы **foo.md** будет создана папка **foo** с файлом **index.html** внутри. В результате адресом страницы будет **/foo/**,
  а не **/foo.html**.

`highlighter`
: Указывает инструмент, с помощью которого Jekyll создаёт подсветку кода. По умолчанию применяется [Rouge](http://rouge.jneen.net/), которые умеет подсвечивать синтаксис
  более чем у [60-ти языков](https://github.com/jneen/rouge/wiki/List-of-supported-languages-and-lexers).

`markdown`
: Указывает инструмент конвертации Markdown в HTML. Мы предпочли [kramdown](https://kramdown.gettalong.org/).

