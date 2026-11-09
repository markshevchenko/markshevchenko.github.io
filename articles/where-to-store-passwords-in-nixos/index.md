---
title: Где хранить пароли в NixOS?
subtitle: На примере OpenVPN и ANTHROPIC_AUTH_TOKEN.
---

NixOS — дистрибутив Linux, который позволяет восстанавливать рабочее окружение за считанные минуты.

Когда вам выдают новый рабочий ноут, вы тратите пару дней, чтобы установить и настроить нужный софт.

Когда вы включаетесь в новый проект, ваш первый день уходит на борьбу с Postgres, Redis и Kafka.

Но только не в NixOS.
Да, и здесь программы скачиваются и устанавливаются.
Это занимает время.
Но уже не день, и не два, а тридцать минут.

Секрет в том, что и рабочее окружение, и нужные в проекте настройки описаны на специальном языке — но называется Nix — благодаря чему конфигурацию можно восстановить буквально с точностью до байта.

Правда, и в мире NixOS возникают сложности.
Многие вещи здесь делаются по особому, в соответствии с *Nix way*.

Например, конфигурация OpenVPN, как и многое прочее, находится в файле **configuration.nix**, который многие никсоводы держат в открытом репозитории git.
А, поскольку для запуска OpenVPN нужны ключи и пароли, кажется, что и они окажутся в открытом доступе.

Не будем ходить вокруг да около — действительно, окажутся.
Чтобы этого не случилось, потребуется немного магии.
Она называется SOPS и Age.

## SOPS

NixOS — не единственная система, где надо шифровать конфигурацию.
Любой бекенд, который мы пишем, читает свои настройки из файлов YAML и JSON, которые, очевидно, удобно хранить вместе с проектом.
Наряду с безобидными открытыми параметрами настроечные файлы содержат пароли и токены, которые очень не хотелось бы «светить».

И чтобы их не «светить», их нужно зашифровать.

Программа SOPS (Secrets OPerationS) разработана как раз для шифровки и расшифровки настроечных файлов.

При этом структура файла сохраняется.
Если, скажем, незашифрованный JSON выглядит так:

```json
{ 
  "db": { 
    "host": "localhost:4032",
    "username": "basil",
    "password": "Dk2Kl9!a^"
  }
}  
```

то зашифрованный так:

```json
{
  "db": {
    "host": "ENC[AES256_GCM,data:QsWSapvLYXRFqLtGeww=,iv:QFrAyJVSYH5XklLNWewUlK8MPPCbbiDT87+Rmua7m3I=,tag:GgGt5jJz7PAdCT0AIHsBfw==,type:str]",
    "username": "ENC[AES256_GCM,data:sEbiU7A=,iv:kiqiNt9Oly8LqotOVauNiIfcAvvRpubAY9kAMw3zX7Y=,tag:OOkvjAUAmY5zj53USGEqvw==,type:str]",
    "password": "ENC[AES256_GCM,data:yGSuKEWf/9Aw,iv:NIvdOVh4W+uld3sh3j2s6+keaWwlvb5M0uu6FygqQNw=,tag:J4PPcmH4wNLvWIigLyDqJQ==,type:str]"
  }
}
```

Для того, чтобы магия работала, важно расшифровывать настроечные файлы перед запуском приложений, скажем, в пайплайнах CI/CD.
Именно поэтому SOPS умеет интегрироваться с различными системами развёртывания, такими, как Ansible.

Однако, не поверите, SOPS не умеет зашифровывать и расшифровывать данные.
Если это и кажется странным, то только на первый взгляд.
Существуют десятки, возможно, сотни алгоритмов шифрования, среди которых есть и проприетарные.
Реализовать даже самые популярные и открытые — большая и долгая задача.
Однако, можно поступить в соответствии с Open Closed Principle и предусмотреть интерфейс для подключения внешних утилит шифрования, что авторы SOPS и сделали.

Одна из этих утилит — это Age.

## Age

В давние времени для шифрования чаще всего использовали рабочую лошадку PGP, но сейчас ей на смену приходит Age.
PGP рассчитана на самых разные, в том числе очень редкие сценарии.
Чтобы её запустить, с ней надо разбираться, даже если вы не собираетесь делать ничего сложного.
Кроме того, PGP спроектирована как интерактивная программа.
Запускать её из скриптов не всегда просто, приходится трюкачить.

Age гораздо проще в настройке и гораздо дружелюбнее к скриптам, что очень важно для *Nix way*.

Нам, на самом деле не придётся работать с Age, поскольку SOPS берёт все тяготы шифрования на себя.

Один-единственный раз вы запустим утилиту age-keygen, чтобы сгенерировать пару ключей, и это всё.

## Начальная настройка

Перед тем, как приступить к конкретным задачам, подготовим плацдарм.
Нам предстоит:

1. Установить модуль sops-nix, чтобы NixOS вызывала sops для расшифровки
   настроечных файлов при сборке конфигурации.
2. Установить пакеты sops и age, чтобы получить доступ к утилитам sops и age-keygen.
3. Сгенерировать ключ.
4. Настроить SOPS, чтобы он использовал Age для шифровки и расшифровки.

### Установка sops-nix

Модуль sops-nix находится в канале **https://github.com/Mic92/sops-nix/archive/master.tar.gz**.
Добавьте его в список каналов:

```bash
sudo nix-channel --add https://github.com/Mic92/sops-nix/archive/master.tar.gz sops-nix
sudo nix-channel --update
```

Затем в начале файла **/etc/nixos/configuration.nix** импортируйте модуль `<sops-nix/modules/sops>`:

```nix
imports = [
  ./hardware-configuration.nix
  # Другие импорты в заголовке файла
  # . . .
  <sops-nix/modules/sops>
];
```

### Установка sops и age

Добавьте пакеты sops и age в `systemPackages`:

```nix
environment.systemPackages = with pkgs; [
  # Другие пакеты
  # . . .
  sops
  age
];
```

Пересоберите систему.
Убедитесь, что всё прошло без ошибок.

```bash
sudo nixos-rebuild switch
```

### Генерация ключа

Мы можем использовать несколько ключей для шифрования разных частей системы — ключ пользователя, ключ хоста, ключ команды и так далее.

Чтобы настраивать системную конфигурацию, ключ должен быть доступен пользователю **root**, так что мы сгенерируем ключ для суперпользователя.

Ключи Age для SOPS по соглашению хранят в файле **~/.config/sops/age/keys.txt**, что для **root** соответствует **/root/.config/sops/age/keys.txt**:

```bash
sudo mkdir -p /root/.config/sops/age
sudo age-keygen -o /root/.config/sops/age/keys.txt
```

Сохраните копию файла в надёжном месте.
Если вы его потеряете, секреты останутся зашифрованными навечно.

Чтобы «подсмотреть» открытый ключ, нужный для шифрования, запустите команду:

```bash
sudo age-keygen -y /root/.config/sops/age/keys.txt
```

Программа напечатате строку вида «age1...».
Скопируйте её в буфер обмена, она потребуется для настройки SOPS.

### Настройка SOPS

Перейдите в каталог **/etc/nixos**.
В любимом редакторе создайте файл **.sops.yaml** следующего вида:

```yaml
keys:
  - &root age1...dla3
creation_rules:
  - path_regex: secrets/secrets.yaml$
    key_groups:
    - age:
      - *root
```

Вместо `root` можно использовать любое имя, но мы придерживаемся соглашений.
Для пользователя с именем `eve` рекомендуют что-то вроде `eve` или `user_eve`, а для хоста `webserver` — что-то вроде `host_webserver`.

Строка `age1...dla3` — открытый ключ, который вы скопировали в буфер обмена на предыдущем шаге.

В файле **secrets/secrets.yaml** (полный путь **/etc/nixos/secrets/secrets.yaml**) будут храниться наши пароли в зашифрованном виде.
Его пока не существует, потому что у нас нет ни одного пароля.

Давайте настроим OpenVPN, сложив в файл с секретами сертификаты, ключи и пароли.

## OpenVPN

На работе, в компании, которую из-за NDA я назову xxx, мне выдали файл с настройками OpenVPN и пароль.
В файле лежали сертификаты `ca` и `cert`, а так же ключи `key` и `tls-crypt`:

```ovpn
client
dev tun
tun-mtu 1372
proto udp
remote openvpn.xxx.ru 1194
resolv-retry 10
nobind
persist-key
persist-tun
remote-cert-tls server
key-direction 1
cipher AES-256-GCM
auth SHA256
verb 4
push-peer-info
<ca>
-----BEGIN CERTIFICATE-----
MIIB...
-----END CERTIFICATE-----
</ca>
<cert>
-----BEGIN CERTIFICATE-----
MIIB...
-----END CERTIFICATE-----
</cert>
<key>
-----BEGIN ENCRYPTED PRIVATE KEY-----
MIGj...
-----END ENCRYPTED PRIVATE KEY-----
</key>
<tls-crypt>
-----BEGIN OpenVPN Static key V1-----
5166...
-----END OpenVPN Static key V1-----
</tls-crypt>
```

Пароль я должен был вводить при запуске VPN, впрочем, файл с паролем можно указать в этом же файле в параметре `askpass`.
Тогда подключаться к VPN можно будет автоматически.

Чтобы настроить OpenVPN, мы:

1. Перенесём сертификаты, ключи и пароль в файл **/etc/nixos/secrets/secrets.yaml**.
   Зашифруем его.
2. Опишем все значения в файле конфигурации **configuration.nix**,
   чтобы sops-nix при сборке расшифровал каждое из них в отдельный файл.
3. Расставим ссылки на файлы там, где раньше нам бы пришлось писать
   чувствительные данные в открытом виде.

### Создание secrets/secrets.yaml

Проверьте, что вы всё ещё в каталоге **/etc/nixos**.
Именно в этом каталоге в файле **.sops.yaml** хранится конфигурация SOPS, которая неявным образом потребуется нам прямо сейчас.

Запустите программу:

```bash
sops edit secrets/secrets.yaml
```

Программа sops откроет ваш любимый редактор, где в формате YAML вам предстоит записать всё, что вы хотели бы скрыть.

Но, прежде чем предаться графомании, продумайте структуру файла с секретами.
Можно хранить все секреты в общем списке, а можно разбить их на логические блоки.

Будучи программистом старой школы, я предпочитаю второй вариант.
Больше структуры богу структуры!

```yaml
openvpn:
  xxx:
    ca:
    cert:
    key:
    tls-crypt:
    key-pass:
```

Здесь `xxx` — это название компании.
Если мне потребуется добавить несколько серверов, я добавлю их в раздел `openvpn`, каждый со своим именем.

Значения параметров `ca`, `cert`, `key` и `tls-crypt` — многострочные, в YAML они выглядят немного громоздко.
Параметра `key-pass` — пароль для расшифровки поля `key`.
При настройке OpenVPN он передаётся в параметре `askpass`.

```yaml
openvpn:
  xxx:
    ca: |
      -----BEGIN CERTIFICATE-----
      MIIB...
      -----END CERTIFICATE-----
    cert: |
      -----BEGIN CERTIFICATE-----
      MIIB...
      -----END CERTIFICATE-----
    key: |
      -----BEGIN ENCRYPTED PRIVATE KEY-----
      MIGj...
      -----END ENCRYPTED PRIVATE KEY-----
    tls-crypt: |
      -----BEGIN OpenVPN Static key V1-----
      5166...
      -----END OpenVPN Static key V1-----
    key-pass: xxxxxxxxx
```

Сохраним файл и закроем редактор.
Посмотрим, что на самом деле попало в **secrets.yaml**:

```bash
cat secrets/secrets.yaml
```

Если всё работает так, как задумывалось, в YAML появится несколько новых служебных полей, а все наши поля окажутся зашифрованными!

Такой файл уже не страшно доверить гитхабу.

### Описание значений

Для начала добавим в **configuration.nix** раздел `sops`.
Содержимое этого раздела обрабатывается модулем sops-nix:

```nix
sops = {
  defaultSopsFile = ./secrets/secrets.yaml;
  
  # Не файл, а строка, значит, не попадёт в /nix/store!
  age.keyFile = "/root/.config/sops/age/keys.txt";
};
```

Здесь мы подсказываем sops-nix, где искать секреты и что использовать для расшифровки.

Опишем каждый ключ из файла **secrets.yaml**.
Ключ `ca` из подраздела `xxx`, раздела `openvpn` получает имя `"openvpn/xxx/ca"`:

```nix
sops = {
  defaultSopsFile = ./secrets/secrets.yaml;
  age.keyFile = "/root/.config/sops/age/keys.txt";

  secrets = {
    "openvpn/xxx/ca" = {
      owner = "root";
    };
  };
};
```

Косую черту нельзя использовать в идентификаторах, поэтому мы заключаем идентификатор в кавычки.

При сборке конфигурации модуль sops-nix расшифрует каждое описанное значение в отдельный файл.
Сертификат из `"openvpn/xxx/ca"` окажется в файле **/run/secrets/openvpn/xxx/ca**, владельцем которого будет пользователь `root`.

Опишем оставшиеся параметры:

```nix
secrets = {
  "openvpn/xxx/ca" = {
    owner = "root";
  };

  "openvpn/xxx/cert" = {
    owner = "root";
  };

  "openvpn/xxx/key" = {
    owner = "root";
  };

  "openvpn/xxx/tls-crypt" = {
    owner = "root";
  };

  "openvpn/xxx/key-pass" = {
    owner = "root";
  };
};
```

Мы почти закончили.
Осталось использовать описанные секреты при настройке OpenVPN.

### Настройка OpenVPN

Настройки подключения хранятся в файле с расширением **.ovpn**.
Скопируем его содержимое в атрибут `config`:

```nix
services.openvpn.servers = {
  xxx = {
    config = ''
      client
      dev tun
      tun-mtu 1372
      proto udp
      remote openvpn.xxx.ru 1194
      resolv-retry 10
      nobind
      persist-key
      persist-tun
      ca ${config.sops.secrets."openvpn/xxx/ca".path}
      cert ${config.sops.secrets."openvpn/xxx/cert".path}
      key ${config.sops.secrets."openvpn/xxx/key".path}
      tls-crypt ${config.sops.secrets."openvpn/xxx/tls-crypt".path}
      askpass ${config.sops.secrets."openvpn/xxx/key-pass".path}
      remote-cert-tls server
      key-direction 1
      cipher AES-256-GCM
      auth SHA256
      verb 4
      push-peer-info
    '';
    updateResolvConf = true;
    autoStart = true;
  };
};
```

Напомню, что `"openvpn/xxx/ca"` — это не строка, а набор атрибутов, в котором атрибут `path` содержит полное имя файла.
Таким образом, `${config.sops.secrets."openvpn/xxx/ca".path}` — путь к файлу сертификата `ca`.

При сборке системы NixOS сгенерирует конфигурационный файл, шаблон которого находится в атрибуте `config`, и поместит его в каталог **/nix/store**.
Файлы в **/nix/store** досупны для чтения всем, поэтому в них нельзя хранить чувствительные данные.

Модуль soap-nix поместит секреты в отдельные файлы в каталоге **/run/secrets**, ограничив к ним доступ — их не сможет прочитать никто, кроме владельца.

При старте системы (или при первом запуске) OpenVPN прочитает настройки из конфигурационного файла, а пароли — из файлов в **/run/secrets**.

При этом пароли больше не хранятся в открытом виде ни в одном из файлов конфигурации.

Что ж, осталось сделать два важных шага.
Во-первых, пересоберём систему:

```bash
sudo nixos-rebuild switch
```

Исправив неизбежные опечатки, получим в результате работающий корпоративный VPN.

Во-вторых, сохраним изменения в git:

```bash
sudo git add .
sudo git commit -m "Настроил OpenVPN 😎😎😎"
sudo git push
```

Напомню, что в репозитории появились новые файлы **.sops.yaml** и **secrets/secrets.yaml**.
Они — тоже часть конфигурации.

## Clause Code

Рассмотрим ещё один сценарий.
Представим, что для работы над проектом команда использует Claude Code.

Клиент ищет адрес подключения в переменной `ANTHROPIC_BASE_URL`, а токен — в `ANTHROPIC_AUTH_TOKEN`.

Обе настройки локальные, их нет смысла выносить в конфигурационный файл системы.
Идеальным местом для инициализации переменных окружения будет файл **shell.nix** в каталоге проекта.

Переменную `ANTHROPIC_BASE_URL`, с некоторыми оговорками, можно хранить в открытом виде, а вот `ANTHROPIC_AUTH_TOKEN` должна быть зашифрована.

```nix
{ pkgs ? import <nixpkgs> { } }:

pkgs.mkShell {
  shellHook = ''
    export ANTHROPIC_AUTH_TOKEN="???"
    export ANTHROPIC_BASE_URL="https://api.anthropic.com"
    export ANTHROPIC_API_KEY=""
  '';

  nativeBuildInputs = with pkgs.buildPackages; [
    go
    claude-code
  ];
}
```

К сожалению, модуль sops-nix работает только с глобальной конфигурацией **configuration.nix**.
Мы не сможем воспользоваться им в **shell.nix**.

Поэтому разобьём извлечение секрета на два этапа.
Вспомним, что sops-nix, собирая конфигурацию, помещает расшифрованные файлы в каталог **/run/secrets**.

К этому файлу можно обратиться из **shell.nix**.
Значит, нам надо:

1. Добавить новый секрет в **secrets.yaml**.
2. Описать его в разделе `sops.secrets` файла **configuration.nix**.
   Пересобрать систему, чтобы в каталоге **/run/secrets** появился файл с токеном.
3. Проинициализировать `ANTHROPIC_AUTH_TOKEN` содержимым этого файла в **shell.nix**.

### secrets.yaml

Запустим наш любимый редактор через sops.

```bash
cd /etc/nixos
sops edit secrets/secrets.yaml
```

Sops расшифровывает файл с секретами для редактирования.
Добавим в конец YAML токен с именем, например, `claude-token`, сохраним изменения и закроем редактор:

```yaml
# настройка OpenVPN...
claude-token: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### sops.secrets

Опишем новый ключ в разделе `sops.secrets`:

```yaml
"claude-token" = {
  owner = "me";
};
```

В качестве владельца файла следует указать себя, а не пользователя `root`.
Вмето `me` впишите реальное имя пользователя.

Пересоберём систему.

```bash
sudo nixos-rebuild switch
```

### shell.nix

В каталоге **/run/secrets** появился файл **claude-token**, в котором находится токен для доступа к Claude Code.

```bash
cat /run/secrets/claude-token
```

Используем его для инициализации переменной `ANTHROPIC_AUTH_TOKEN`:

```nix
{ pkgs ? import <nixpkgs> { } }:

pkgs.mkShell {
  shellHook = ''
    if [ -f /run/secrets/claude-token ]; then
      export ANTHROPIC_AUTH_TOKEN="$(cat /run/secrets/claude-token)"
    else
      echo "There's no ANTHROPIC_AUTH_TOKEN in /run/secrets"
      echo "Fix your configuration.nix"
    fi
    export ANTHROPIC_BASE_URL="https://api.anthropic.com"
    export ANTHROPIC_API_KEY=""
  '';

  nativeBuildInputs = with pkgs.buildPackages; [
    go
    claude-code
  ];
}
```

Чтобы этот метод работал у всех членов команды, надо, чтобы каждый участник настроил свою конфигурацию так, как описано выше.

## Заключение

Вооружённые новыми знаниями, теперь вы можете держать секреты в секрете.
Установите sops-nix, sops и age, создайте ключ, пропишите секреты в файле **secrets.yaml**, бесстрашно добавьте его в git и пересоберите систему.

Вы перешли на новый уровень.