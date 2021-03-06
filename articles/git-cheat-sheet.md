---
title: Шпаргалка по командам git
subtitle: Что делать, когда под руками нет SourceTree
id: git-cheat-sheet
---

## Основной сценарий

### Шаг 1

У нас есть задача (issue), которую мы планируем сделать. В центральном репозитории создаём ветку для решения задачи. Например, для задачи с номером 234 создаём ветку **features/issue-234**.

### Шаг 2

Теперь в удалённом репозитории есть ветка, где мы будем работать над задачей. Нам нужна такая же ветка в локальном репозитории. Локальную ветку надо создать и связать с удалённой.

Сначала скачиваем из удалённого репозитория изменения, включая новые ветки

```
git fetch
```

Чтобы убедиться, что удалённая ветка доступна локально, выполняем команду

```
git branch -a
```

Создаём локальную ветку, которая указывает на удалённую ветку

```
git checkout -b features/issue-234 origin/features/issue-234
```

или, короче

```
git checkout -t origin/features/issue-234
```

### Шаг 3

Далее работаем над задачей, в локальной ветке, время от времени фиксируя изменения.

Прежде, чем фиксировать, смотрим, что успели сделать.

```
git status
```

* *Untracked* — git не отслеживает изменения в этих файлах и не хранит их.
* *Changes not staged for commit* — git отлеживает и хранит эти файлы. Сейчас эти файлы изменены,
  но git не будет сохранять эти изменения при фиксации.
* *Changes to be committed* — файлы изменены и git сохранит эти изменения при следующей фиксации.

Помечаем для фиксации файлы **file₁..fileₙ**.

```
git add <file₁> ... <fileₙ>
```

Помечаем для фиксации *все* файлы в текущем каталоге и его подкаталогах. Помеченными окажутся в том числе все *новые* файлы.

```
git add .
```

Фиксируем все файлы, помеченные для фиксации. Git попросит ввести комментарий.

```
git commit
```

Команда `git commit -a` работает также, как `git add .` и `git commit` вместе за тем исключением, что *новые* файлы не будут зафиксированы.

Команда `git commit -m "Комментарий к коммиту"` фиксирует изменения с указанным комментарием, не запуская внешний редактор.

Откатываем изменения в конкретных изменённых файлах.

```
git checkout --  <file₁> ... <fileₙ>
```

Откатывем изменения во всех файлах:

```
git checkout -- .
```
или
```
git reset --hard
```

Удаляем с диска *новые* файлы

```
git clean -f -d
```

После фиксации корректным считается не удаление коммита, а создание парного к нему *отменяющего* коммита.

```
git revert <commit-id>
```

Отправляем изменения в центральный репозиторий.

```
git push
```

### Шаг 4

Решив задачу мы должны слить все изменения в основную ветку — **master**. Это делается через pull request в центральном репозитории. Ставим галочку *удалить исходную ветку после слияния*, чтобы не пришлось удалять эти ветку вручную.

Теперь надо избавиться от локальной ветки, где мы работали. Сначала переключимся на основную ветку.

```
git checkout master
```

«Подтягиваем» в неё изменения из центрального репозитория.

```
git pull
```

Смотрим, какие ветки живы локально, но при этом удалены на сервере.

```
git remote prune origin
```

Удаляем ветки, которые показала предыдущая команда.

```
git branch -d features/issue-234
```

## Вливаем изменения из центрального репозитория

Если в ветке **master** появились изменения, мы можем влить в нашу ветку, чтобы в будущем упростить pull request. Переключаемся на ветку **master**, загружаем обновления, переключаемя на рабочую ветку, сливаемся с **master** и выгружаем результат в центральный репозиторий. Если конфликтов не было, pull request пройдёт без проблем.

```
git checkout master
git pull
git checkout features/issue-234
git merge master
git push
```