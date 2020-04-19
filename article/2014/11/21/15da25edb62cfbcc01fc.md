---
title: Cygwinインストール後のWindows環境変数設定
tags: Cygwin:2.4 Windows 環境変数
author: harry0000
slide: false
---
# 前置き
Cygwinインストールの度に、設定すべき基本的なWindows環境変数をググって消耗したくないのでまとめました。
`ntsec`など、いくつかの設定は既に廃止されたようです（情弱）

ここに書かれている設定を使用して発生した問題につきまして、当方は一切責任を負いません。（テンプレ免責事項）

特に断りのない限り、下記の設定でCygwinを導入したものとして説明を行います。

* インストールディレクトリ

```
C:\cygwin64
```

* ユーザー名

```
harry
```

* ログインシェル

```
/bin/bash
```

# 環境変数

Environment Variables
https://cygwin.com/cygwin-ug-net/setup-env.html

基本的にWindowsに設定されている環境変数はCygwin起動時にインポートされます。

> All Windows environment variables are imported when Cygwin starts.

## PATH

* `/etc/profile`[^1]により、以下のパスが自動的にPATHへ追加されます
  * `/usr/local/bin`
  * `/usr/bin`
  * Windowsの環境変数に設定されている`PATH`[^2]
     * Windows形式のパスは自動的にCygwinのパスに変換されます  
  (例: `C:\Program Files\` -> `/cygdrive/c/Program Files/`)

* `/bin`はパスが通っている状態のため、PATHに追加する必要はありません

Cygwinの`PATH`に追加したいパスがある場合、以下の2つの方法があります。

* Windowsの環境変数`PATH`に追加する
* `/home/harry/.bash_profile`[^3]に追加する

例: `.bash_profile`にgit bashのPathを追加

```
PATH="${PATH}:/cygdrive/c/users/harry/AppData/Local/GitHub/PortableGit_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx/mingw32/bin"
```

## HOME

Windowsの環境変数として設定すると、そのディレクトリがホームディレクトリとして使用されます。

ただし、Windowsの環境変数として`HOME`を設定することは**推奨されていません。**

> However, it's usually set in the shell profile scripts in the /etc directory, and it's **not** recommended to set the variable in your Windows environment.

## LD_LIBRARY_PATH

共有ライブラリ検索パス。
Windowsの環境変数として設定するか、または`.bash_profile`に追加します。

* Windowsの環境変数の場合
  * 変数名： `LD_LIBRARY_PATH`
  * 値： `C:\cygwin64\lib;C:\cygwin64\usr\local\lib;`

* `.bash_profile`の場合

```
LD_LIBRARY_PATH=/lib:/usr/local/lib
export LD_LIBRARY_PATH
```

## TMP / TEMP

これらの環境変数がWindowsで設定されていても、`/env/profile`により`/tmp`へ上書きされます。

* /env/profile

> ```
>   # TMP and TEMP as defined in the Windows environment
>   # can have unexpected consequences for cygwin apps, so we define
>   # our own to match GNU/Linux behaviour.
>   unset TMP TEMP
>   TMP="/tmp"
>   TEMP="/tmp"
> ```

## CYGWIN

The CYGWIN environment variable
https://cygwin.com/cygwin-ug-net/using-cygwinenv.html

Cygwinランタイムシステムに対する設定です。
Windowsの環境変数として設定しておきます。

* 環境変数名： `CYGWIN`
* 設定例： `glob nodosfilewarning`

-----
[^1]: `C:\cygwin64\etc\profile`
[^2]: 正確には、`CYGWIN_NOWINPATH`という環境変数が未設定、または`addwinpath`の場合に追加されます
[^3]: `C:\cygwin64\home\harry\.bash_profile`
