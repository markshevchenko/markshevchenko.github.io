---
title: Где хранить пароли в NixOS?
subtitle: На примере OpenVPN и ANTHROPIC_AUTH_TOKEN.
id: where-to-store-passwords-in-nixos
---

NixOS — интересная и даже заворащивающая операционная система, которая позволяет восстанавливать рабочее окружение за считанные минуты.

Если вам выдают новый рабочий компьютер Windows или Debian, вы тратите два дня на то, чтобы установить и настроить нужный софт.

Если вы включаетесь в новый проект, ваш первый день уходит на борьбу с Postgres, Redis и Kafka.

Но только не в NixOS.
Да, и здесь программы скачиваются и устанавливаются.
Это занимает время.
Но уже не день, и не два, а тридцать минут.

Секрет в том, что и рабочее окружение, и нужные в проекте настройки, вы описали на языке Nix.

NixOS в чём то похожа на Docker, но ей не нужны котейнеры.

Правда, и с ней возникают сложности.
Многие вещи в NixOS делаются по особому, в соответствии с *Nix way*.

Например, конфигурация OpenVPN, как и многое прочее, находится в файле **configuration.nix**, который многие никсоводы держат в открытом репозитории git.
А, поскольку для запуска OpenVPN нужны ключи и пароли, кажется, что и они окажутся в открытом доступе.

Не будем ходить вокруг да около — действительно, окажутся.
Чтобы этого не случилось, нам потребуется немного магии.
Она называется SOPS и Age.

## SOPS

NixOS не единственная система, где требуется шифровать конфигурационные данные.
Любой бекенд, который мы пишем, читает свои настройки, включая пароли, из файлов YAML и JSON.
Которые, очевидно, удобно хранить вместе с проектом.
Но так делать нельзя, потому что тогда они становятся доступны всему миру или, по крайней мере, всем командам вашей компании.

Если их, конечно, не зашифровать.

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
Так что SOPS интегрируется с системами развёртывания приложений, такими, как Ansible.

Удивительно, что программа SOPS не умеет шифровать и расшифровывать данные.
Для этого она запускает внешние утилиты.

## Age

Одна из таких утилит — это Age, идущая на смену PGP.
Она проще в использовании и совместима с *Nix way*.

В действительности нам практически не придётся работать с Age, посколько SOPS берёт все тяготы шифрования на себя.

Нам потребуется вызывать утилиту age-keygen для генерации ключей.
Это всё.

## Начальная настройка

Перед тем, как приступить к конкретным задачам, подготовим плацдарм.
Нам предстоит:

1. Установить модуль sops-nix, чтобы NixOS вызывала sops для расшифровки
   настроечных файлов при сборке конфигурации.
2. Установить пакеты sops и age, чтобы получить доступ к утилитам sops и age-keygen.
3. Сгенерировать ключ.
4. Настроить SOPS, чтобы он использовал Age для шифровки и расшифровки.

### Установка sops-nix

Модуль sops-nix находится в канале **https://github.com/Mic92/sops-nix/archive/master.tar.gz**:

```bash
sudo nix-channel --add https://github.com/Mic92/sops-nix/archive/master.tar.gz sops-nix
sudo nix-channel --update
```

После этого в начале файла **/etc/nixos/configuration.nix** добавьте импорт `<sops-nix/modules/sops>`:

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

Для наших целей нам достаточно одного.
Чтобы настраивать системную конфигурацию, ключ должен быть доступен пользователю **root**, так что он не может быть ключом пользователя.

Ключ хоста традиционно размещают в каталоге **/etc/age** в файле **identity.key**:

```bash
sudo mkdir -p /etc/age
sudo age-keygen -o /etc/age/identity.key
```

Сохраните копию файла в надёжном месте.
Если вы его потеряете, секреты останутся зашифрованными навечно.

Чтобы «подсмотреть» открытый ключ, нужный для шифрования, запустите команду:

```bash
sudo age-keygen -y /etc/age/identity.key 
```

Программа напечатате строку вида «age1...».
Скопируйте её в буфер обмена, она потребуется для настройки SOPS.

### Настройка SOPS

Перейдите в каталог **/etc/nixos**.
В любимом редакторе создайте файл **.sops.yaml** следующего вида:

```yaml
keys:
  - &host_local age1...dla3
creation_rules:
  - path_regex: secrets/secrets.yaml$
    key_groups:
    - age:
      - *host_local
```

Вместо `host_local` можно использовать любое имя.
Для пользователя eve рекомендуют что-то вроде user_eve, а для хоста webserver — что-то вроде host_webserver.

Строка `age1...dla3` — открытый ключ, который вы скопировали в буфер обмена на предыдущем шаге.

В файле **secrets/secrets.yaml** (полное имя **/etc/nixos/secrets/secrets.yaml**) будут храниться наши пароли в зашифрованном виде.
Его пока не существует, но нам и нечего там хранить.

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

Для настройки OpenVPN мы:

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

Но, прежде чем предаться графомании, поговрим о структуре файла.
Можно хранить все пароли в общем списке, а можно разбить их на логические блоки.

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

Мы помним, что `xxx` — это название компании.
Если мне потребуется добавить несколько серверов, я добавлю их в раздел `openvpn`, каждый со своим именем.

Значения параметров `ca`, `cert`, `key` и `tls-crypt` — многострочные.
Как их записать в YAML?
Если вы специалист по YAML, то знаете, что это совсем несложно, но немного громоздко:

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

Если всё работает так, как задумывалось, вы увидите YAML с той же структорой и некоторыми новыми полями, дописанными программой sops.
Главное — значения полей окажутся зашифрованными!

Такой файл уже не страшно доверить гитхабу.

### Описание значений

Для начала добавим в **configuration.nix** раздел `sops`.
Содержимое этого раздела обрабатывается модулем sops-nix:

```nix
sops = {
  defaultSopsFile = ./secrets/secrets.yaml;
  
  # Не файл, а строка, значит, не попадёт в /nix/store!
  age.keyFile = "/etc/age/identity.key";
};
```

Здесь мы подсказываем sops-nix, где искать секреты и что использовать для расшифровки.

Опишем каждый ключ из файла **secrets.yaml**.
Ключ `ca` из подраздела `xxx`, раздела `openvpn` получает имя `"openvpn/xxx/ca"`:

```nix
sops = {
  defaultSopsFile = ./secrets/secrets.yaml;
  age.keyFile = "/etc/age/identity.key";

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

Добавим в конфигурационный файл оставшиеся параметры:

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
Осталось настроить OpenVPN.

### Найстройка OpenVPN

Нам потребуется файл с расширением **.ovpn**, где хранятся настройки OpenVPN.
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

Кажется, всё здесь достаточно очевидно.
Напомню, что `"openvpn/xxx/ca"` — это не строка, а набор атрибутов.
Атрибут `path` содержит полное имя файла.
Таким образом, `${config.sops.secrets."openvpn/xxx/ca".path}` — путь к файлу сертификата `ca`.

Осталось сделать два важнейших шага.
Для начала пересоберём систему:

```bash
sudo nixos-rebuild switch
```

Исправив неизбежные опечатки, получим в результате работающий корпоративный VPN.

Наконец, сохраним изменения в git:

```bash
sudo git add .
sudo git commit -m "Настроил OpenVPN 😎😎😎"
sudo git push
```

Напомню, что в репозитории появились новые файлы **.sops.yaml** и **secrets/secrets.yaml**.

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

Однако, не всё так безнадёжно.
Напомню, что sops-nix, собирая конфигурацию, помещает расшифрованные файлы в каталог **/run/secrets**.

У каждого файла есть владелец, поэтому мы не можем «заглянуть» в чужие файлы, только в свои.
Этого достаточно для безопасной инициализации `ANTHROPIC_AUTH_TOKEN`.

Нам потребуется:

1. Добавить новый секрет в **secrets.yaml**.
2. Описать его в разделе `sops.secrets` файла **configuration.nix**.
   Пересобрать систему, чтобы в каталоге **/run/secrets** появился файл с токеном.
3. Проинициализировать `ANTHROPIC_AUTH_TOKEN` содержимым этого файла в **shell.nix**.

### secrets.yaml

```bash
cd /etc/nixos
sops edit secrets/secrets.yaml
```

В нашем любимом редакторе мы увидим расшифрованные секреты.
Добавим токен, сохраним изменения и закроем редактор:

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