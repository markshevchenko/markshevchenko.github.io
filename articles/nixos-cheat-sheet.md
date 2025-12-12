---
title: Шпаргалка по командам NixOS
subtitle: Обновление и чистка системы
id: nixos-cheat-sheet
---

## Обновление

```shell
sudo nix-channel --update
sudo nixos-rebuild switch
```

Время от времени не забываем проверять канал обновлений.
Заходим [страницу поиска пакетов NixOS](https://search.nixos.org/packages) и смотрим актуальную версию канала, например <kbd>25.05 <b>Deprecated</b></kbd><kbd>25.11</kbd><kbd>unstable</kbd>.

Текущая версия в нашей системе:

```shell
sudo nix-channel --list
# => nixos https://nixos.org/channels/nixos-25.05
# => rust-overlay https://github.com/oxalica/rust-overlay/archive/master.tar.gz
```

Удаляем старый канал и добавляем новый с тем же именем:

```shell
sudo nix-channel --remove nixos
sudo nix-channel --add https://nixos.org/channels/nixos-25.11 nixos
```

## Очистка

Смотрим, какие поколения системы не удалены:

```shell
sudo nixos-rebuild list-generations
```

Удаляем все поколения, кроме текущего:

```shell
sudo nix-collect-garbage -d
```
