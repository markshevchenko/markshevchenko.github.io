---
title: Практические проблемы решаем функционально
date: "2024-04-05 09:00:00 +0300"
id: solve-practical-problems-functionally
excerpt: "Сдвоенный воркшоп, прочитанный на DotNext 2023."
---

Сегодня организаторы DotNext опубликовали видео воркшопа, прочитанного мной на сентябрьской конференции.
Немедленно перевыкладываю видео из двух частей у себя.

## Официальная аннотация

Классический REST API тестируют с помощью curl или Postman. Более новый gRPC тестировать сложнее, потому что на входе и на выходе у него бинарные данные. Нужна утилита, которая умеет сериализовать текстовые данные в бинарные и десериализовать их обратно. Задача кажется сложной, потому что языки описания схемы и данных Protobuf — достаточно развитые. Но решается она просто, если пользоваться правильным инструментом. Мы напишем утилиту на языке программирования F#, используя библиотеку FParsec. Научимся по описанию грамматики писать код и тесты для разбора, построим абстрактное синтаксическое дерево и разберёмся, как применять его для сериализации.

Рекомендации по подготовке к воркшопу: ноутбук с установленными JetBrains Rider или Microsoft Visual Studio последней версии.

## Видео

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/HWNuMNmALHM?si=LDqWj9LEcGwrQJtG" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
</div>

<div class="video">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/p9gBoaABIVo?si=oNuHBY1x3d7C9Xl_" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
</div>
