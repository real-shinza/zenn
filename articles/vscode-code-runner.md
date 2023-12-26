---
title: "VSCodeにてショートカットキーでコードを実行しよう！ Code Runnerの初期設定の方法について"
emoji: "👨‍💻"
type: "tech"
topics: ["vscode", "visualstudiocode", "vscode拡張機能", "coderunner"]
published: true
published_at: "2023-07-17 19:02"
---

# はじめに
VSCodeのコマンドラインでコードを実行したいけど、毎回コマンドを打つのが面倒臭いと感じることはありませんか？
そんな時は拡張機能の「Code Runner」を使ってみましょう！
「Code Runner」を使えばショートカットキーで簡単にコードを実行できます。
※各プログラム言語の実行環境は別に設定しておく必要があります。

### Visual Studio Codeとは
恐らく説明は不要だと思いますが、一応「Visual Studio Code」について簡単に説明します。
「Visual Studio Code（VSCode）」は開発者に人気があるMicrosoft社が提供するコードエディターです。
軽量かつ高速であり、拡張機能が豊富なのが特徴です。

### Code Runnerとは
「Code Runner」は「VSCode」で簡単にコードを実行できる拡張機能になります。
具体的にはコードの言語に応じてコマンドを実行してくれるという機能を持っています。
更に対応しているプログラム言語も多いです。以下が対応言語です。
> C, C++, Java, JS, PHP, Python, Perl, Ruby, Go, Lua, Groovy, PowerShell, CMD, BASH, F#, C#, VBScript, TypeScript, CoffeeScript, Scala, Swift, Julia, Crystal, OCaml, R, AppleScript, Elixir, VB.NET, Clojure, Haxe, Obj-C, Rust, Racket, Scheme, AutoHotkey, AutoIt, Kotlin, Dart, Pascal, Haskell, Nim,

# インストール手順
1. VSCode の右側にあるアクティビティバーから拡張機能のアイコンをクリック、もしくは[Ctrl] + [Shift] + [X]で拡張機能ビューを開く。（Mac の場合、[Command] + [Shift] + [X]）
2. 検索窓に「code runner」と入力。
3. 「インストール」ボタンを押下。
![](/images/vscode-code-runner/install.png)

たったこれだけです。以上でインストールが完了です。

# カスタマイズ
まずカスタマイズをするには以下の手順で設定を開いて下さい。
1. 「アンインストール」の右にある歯車マークをクリック。
2. 「拡張機能の設定」をクリックし設定を開く。（英語の場合、「Extension Settings」）
![](/images/vscode-code-runner/customize.png)

### Run In Terminal
「Run In Terminal」はコードの実行結果をターミナルに出力できるようにする設定です。
設定方法は設定画面の「Run In Terminal」にチェックを付けるだけです。
![](/images/vscode-code-runner/run-in-terminal.png)
※「settings.json」にて以下記述を追加するでも設定可能です。
```json:settings.json
"code-runner.runInTerminal": true
```

### Executor Map
「Executor Map」は各プログラム言語ごとに実行コマンドを設定できます。
設定方法は設定画面の「Executor Map」にある「settings.json で編集」をクリックするだけです。
「settings.json」が開かれますが、何も編集せず閉じても問題ないです。
もし実行されるコマンドを変えたい場合は、直接jsonファイルを書き換えてみて下さい。
![](/images/vscode-code-runner/executor-map-1.png)
![](/images/vscode-code-runner/executor-map-2.png)
※「settings.json」にて以下記述を追加するでも設定可能です。
※不要な記述があれば行ごと削除しても問題ないです。
:::details settings.json
```json:settings.json
"code-runner.executorMap": {
    "javascript": "node",
    "java": "cd $dir && javac $fileName && java $fileNameWithoutExt",
    "c": "cd $dir && gcc $fileName -o $fileNameWithoutExt && $dir$fileNameWithoutExt",
    "zig": "zig run",
    "cpp": "cd $dir && g++ $fileName -o $fileNameWithoutExt && $dir$fileNameWithoutExt",
    "objective-c": "cd $dir && gcc -framework Cocoa $fileName -o $fileNameWithoutExt && $dir$fileNameWithoutExt",
    "php": "php",
    "python": "python -u",
    "perl": "perl",
    "perl6": "perl6",
    "ruby": "ruby",
    "go": "go run",
    "lua": "lua",
    "groovy": "groovy",
    "powershell": "powershell -ExecutionPolicy ByPass -File",
    "bat": "cmd /c",
    "shellscript": "bash",
    "fsharp": "fsi",
    "csharp": "scriptcs",
    "vbscript": "cscript //Nologo",
    "typescript": "ts-node",
    "coffeescript": "coffee",
    "scala": "scala",
    "swift": "swift",
    "julia": "julia",
    "crystal": "crystal",
    "ocaml": "ocaml",
    "r": "Rscript",
    "applescript": "osascript",
    "clojure": "lein exec",
    "haxe": "haxe --cwd $dirWithoutTrailingSlash --run $fileNameWithoutExt",
    "rust": "cd $dir && rustc $fileName && $dir$fileNameWithoutExt",
    "racket": "racket",
    "scheme": "csi -script",
    "ahk": "autohotkey",
    "autoit": "autoit3",
    "dart": "dart",
    "pascal": "cd $dir && fpc $fileName && $dir$fileNameWithoutExt",
    "d": "cd $dir && dmd $fileName && $dir$fileNameWithoutExt",
    "haskell": "runghc",
    "nim": "nim compile --verbosity:0 --hints:off --run",
    "lisp": "sbcl --script",
    "kit": "kitc --run",
    "v": "v run",
    "sass": "sass --style expanded",
    "scss": "scss --style expanded",
    "less": "cd $dir && lessc $fileName $fileNameWithoutExt.css",
    "FortranFreeForm": "cd $dir && gfortran $fileName -o $fileNameWithoutExt && $dir$fileNameWithoutExt",
    "fortran-modern": "cd $dir && gfortran $fileName -o $fileNameWithoutExt && $dir$fileNameWithoutExt",
    "fortran_fixed-form": "cd $dir && gfortran $fileName -o $fileNameWithoutExt && $dir$fileNameWithoutExt",
    "fortran": "cd $dir && gfortran $fileName -o $fileNameWithoutExt && $dir$fileNameWithoutExt",
    "sml": "cd $dir && sml $fileName"
}
```
:::

# Code Runnerを使ってコード実行してみます。
今まで実行コマンドは自分で入力していたと思いますが、
[Ctrl] + [Alt] + [N]のショートカットキーで自動実行ができます。（Mac の場合、[Control] + [Option] + [N]）
該当のソースコードを右クリックし「Run Code」を選択して実行することも可能です。
※この時、ターミナルが開かれている必要はありません。開いていても閉じていても大丈夫です。

今回は試しに以下のC言語のコードを実行してみました。
```c
#include <stdio.h>

void main()
{
    printf("Hello World!");
}
```
![](/images/vscode-code-runner/run-code.png)

実行できましたね。
動画じゃないので伝わらないと思いますが、コマンドは一切入力していません。

何気に便利なので皆さんも是非Code Runnerの設定をしてみてはいかがでしょうか。
