---
title: "VS Code в NixOS"
subtitle: Можно ли настроить VS Code декларативно?
---

Одной из проблем классических дистрибутивов Linux, да и операционных систем в целом, является сложность переноса с компютера на компьютер.

Меняете вы, скажем, рабочий ноутбук и точно знаете, что вас ждёт двухдневная установка и настройка программ.

Настройки можно, конечно, сохранить.
Но они разбросаны по всей системе, не все вспомнишь.

В Nix *все* настройки находятся в каталоге **/etc/nixos**, который можно сохранить в репозиторий Git и восстановить за несколько секунд на любой машине.

Казалось бы — вот оно, счастье.

Однако, далеко не все программы следуют *Nix way*.
Скажем, для VS Code расширения — не только строчки кофигурации, но и модули исходного кода.

Программист скачивает и устанавливает их непосредственно из текстового редактора.

Возможно, VS Code — один из тех инструментов, которые невозможно настроить через **configuration.nix**?

Почти.

На наше счастье, разработчки Nix умудряются находить остроумные обходные пути для настройки программ, которые не соответствуют духу Nix.

VS Code можно установить, как программу, общую для всех пользователей, перечисли её в списке `environment.systemPackages`.

Но, скажем прямо, не всем любителям Vim и Emacs это понравится.
Правильнее устанавливать VS Code через home manager — только тем пользователям, которым он действительно нужен.

```nix
home-manager.users.mark = { pkgs, ... }: {
  home = {
    username = "mark";
    homeDirectory = "/home/mark";
    stateVersion = "25.11";
  };

  programs.vscode = {
    enable = true;
  };
};
```

Большую часть расширений мы можем перечислить в списке `profiles.default.extensions`:

```nix
programs.vscode = {
  enable = true;
  profiles.default = {
    extensions = with pkgs.vscode-extensions; [
      bbenoist.nix
      docker.docker
      golang.go
      # ...
    ];
  };
};
```

Однако, некоторых пакетов, например **brunnerh.insert-unicode** в nixpkgs нет.

Их придётся добавлять вручную.

```nix
profiles.default = {
  extensions = with pkgs.vscode-extensions; [
    bbenoist.nix
    docker.docker
    golang.go
    # ...
  ] ++ pkgs.vscode-utils.extensionsFromVscodeMarketplace [
    {
      name = "insert-unicode";
      publisher = "brunnerh";
      version = "0.15.1";
      sha256 = "sha256-RHsq7JmlC+4zGSbDdovCZpjpSW+DvcmYnuz9f6F/N4g=";
    }
    # ...
  ];
};
```

Полное имя пакета состоит из имени издателя, в данном случае это **brunnerh** и имени расширения — **insert-unicode**.

Их мы записываем в поля `publisher` и `name`.

Затем по имени пакета находим через Google [репозиторий проекта](https://github.com/brunnerh/insert-unicode) и [подсматриваем](https://github.com/brunnerh/insert-unicode/blob/master/CHANGELOG.md) там номер версии, которую записываем в поле `version`.

Поле `sha256` традиционно заполняем в два этапа.
Сначала записываем туда пустую строку, запускаем команду **sudo nixos-rebuild switch** и получаем ошибку компиляции.

Ищем среди сообщений об ошибках ожидаемое значение поля `sha256`, компируем и вставляем.

Вот и всё.

Некоторые дополнительные настройки мы можем сохранить с помощью атрибута `userSettings`:

```nix
profiles.default = {
  extensions = with pkgs.vscode-extensions; [
    bbenoist.nix
    docker.docker
    golang.go
    # ...
  ] ++ pkgs.vscode-utils.extensionsFromVscodeMarketplace [
    {
      name = "insert-unicode";
      publisher = "brunnerh";
      version = "0.15.1";
      sha256 = "sha256-RHsq7JmlC+4zGSbDdovCZpjpSW+DvcmYnuz9f6F/N4g=";
    }
    # ...
  ];
  userSettings = {
    "editor.fontSize" = 14;
    "editor.bracketPairColorization.enabled" = false;
    "git.blame.editorDecoration.enabled" = true;
    "git.confirmSync" = false;
    "git.enableSmartCommit" = true;
    "git.mergeEditor" = true;
    "gopls" = {
      "ui.semanticTokens" = true;
    };
    "window.zoomLevel" = 1.4;
    "workbench.editor.enablePreview" = false;
    "workbench.colorTheme" = "Visual Studio Dark";
  };
};
```
