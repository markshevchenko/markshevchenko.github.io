---
title: "Практические задачи решаем функционально"
date: 2024-08-30 19:00:00+03:00
excerpt: "Доклад на DotNext 2023."
id: grpc-parser-and-serde
---

В прошлом году сделал двойной воркшоп по языку F#. В преддверии следующей конференции DotNext организаторы выложили его в публичный доступ, так что теперь могу поделиться.

## Аннотация

Классический REST API тестируют с помощью curl или Postman. Более новый gRPC тестировать сложнее, потому что на входе и на выходе у него бинарные данные. Нужна утилита, которая умеет сериализовать текстовые данные в бинарные и десериализовать их обратно. Задача кажется сложной, потому что языки описания схемы и данных Protobuf — достаточно развитые. Но решается она просто, если пользоваться правильным инструментом. Мы напишем утилиту на языке программирования F#, используя библиотеку FParsec. Научимся по описанию грамматики писать код и тесты для разбора, построим абстрактное синтаксическое дерево и разберёмся, как применять его для сериализации.

Рекомендации по подготовке к воркшопу: ноутбук с установленными JetBrains Rider или Microsoft Visual Studio последней версии.

<div class="video">
    <iframe src="https://vk.com/video_ext.php?oid=-65845767&id=456239952&hd=2&autoplay=1" width="853" height="480" allow="autoplay; encrypted-media; fullscreen; picture-in-picture; screen-wake-lock;" frameborder="0" allowfullscreen></iframe>
</div>

<div class="video">
    <iframe src="https://vk.com/video_ext.php?oid=-65845767&id=456239957&hd=2&autoplay=1" width="853" height="480" allow="autoplay; encrypted-media; fullscreen; picture-in-picture; screen-wake-lock;" frameborder="0" allowfullscreen></iframe>
</div>
