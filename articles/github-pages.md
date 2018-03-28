---
title: Сайт на GitHub Pages
id: github-pages
---

## Что это

[GitHub Pages](https://pages.github.com/)&nbsp;&mdash; служба хостинга сайтов, которая позволяет вести персональные сайты, сайты организаций и сайты
отдельных проектов GitHub.

Сайты GitHub Pages статические. Текст страниц в формате [Markdown](https://daringfireball.net/projects/markdown/) хранится в репозитории GitHub.
При проталкивании (push) изменений в ветку `master` GitHub запускает генератор [Jekyll](https://jekyllrb.com/), и через несколько секунд
публикует готовые HTML-страницы на сайте.

Конечно, возможности статических сайтов ограничены, тем не менее Jekyll позволяет:

1. Вести блог.
1. Анонсировать записи в формате [Atom](https://tools.ietf.org/html/rfc4287).
1. Подсвечивать код.
1. Контролировать внешний вид сайта.

## Быстрый старт

Исходный код сайта нужно разместить в правильном репозитории. Мой логин на [GitHub](https://github.com/) markshevchenko, следовательно,
URI моей страницы https://github.com/markshevchenko, а для своего сайта я должен создать репозиторий
[markshevchenko.github.io](https://github.com/markshevchenko/markshevchenko.github.io).

Сразу после создания репозитория, сайт становится доступен по адресу http://markshevchenko.github.io, но он пока совершенно пуст.

Нажмём кнопку <kbd>Create new file</kbd> и в качестве имени укажем `_layouts/default.html`. По правилам GitHub это означает, что
в корне проекта будет создана папка `_layouts` в которой будет создан пустой файл `default.html`. Разместим внутри шаблон:

```html
<!doctype html>
<html lang="{{{{ page.lang | default: site.lang | default: "en" }}}}">
	<head>
		<meta charset="utf-8">
		<meta http-equiv="X-UA-Compatible" content="IE=edge">
		<meta name="viewport" content="width=device-width, initial-scale=1">
 		<title>{{% if page.title %}}{{{{ page.title | escape }}}}{{% else %}}{{{{ site.title | escape }}}}{{% endif %}}</title>
 		<meta name="description" content="{{{{ page.excerpt | default: site.description | strip_html | normalize_whitespace | truncate: 160 | escape }}}}">
	</head>
	<body>
	{{{{ content }}}}
	</body>
</html>
```

В корне создадим файл `index.md`, где запишем:

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

Что у нас получилось? Во-первых, мы подготовили шаблон сайта, где в парных фигурных скобках разместили названия переменных, выражения и даже конструкцию
**if**. Во-вторых, мы обнаружили, что переменные могут относиться как к сайту целиком (`site.description`), так и к отдельным страницам (`page.excerpt`).

В-третьих, мы увидели, что значения переменных старницы можно указать в её заголовке, отделив их от основного текста строками из трёх дефисов.
После всех мытарств, мы увидели правильно оформленный результат генерации сайта.