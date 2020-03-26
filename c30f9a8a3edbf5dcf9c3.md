---
title: TypeScript2の私的環境の作り方
tags: TypeScript
author: Mizunashi_Mana
slide: false
---
## TypeScript2.1リリースめでたい

* https://github.com/Microsoft/TypeScript/releases/tag/v2.1.4
* いや、まだひと月も経ってないからね？対応しきれないからね？
* そもそも2対応してないプロジェクトとか普通にあるからね？
* というわけで、今日は2.1の話はしません

## はじめに

いまやTypeScriptもメジャーバージョン2になりました。結構初リリースから色々あったんじゃないんですかね？多分感慨深い人には感慨深いんでしょう。まあ、今年だけでも結構色々ありました。VSCode 1.0は出るわ、typingsが作られるわ、es2016が本格的に降ってくるわ、Angular2は出るわ、yarnは出るわ。

先日2.1が出て全く実感が湧かないんですが、まだTypeScript2に移行できてない人とかもいたりするんじゃないんですかね？というわけで、先日TypeScript2の環境の作り方という発表を某場所でしました。今日は、まあどうせならそれ整形してMarkdownにでもまとめるかって感じで、やらせてもらいます。

主に僕の環境の作り方なので、気に入らない点は補完してもらえればなと思います。あと、結構僕の作り方そのまま紹介する感じなんで、あんまり入門向けではないかもしれません。コメントや編集リクエストも随時受け付けていますので、もし間違いなどを見つけたら報告してもらえれば幸いです。分からないところなどがあれば、Twitterやコメントで聞いてもらえれば答えます。よろしくお願いします。

Yeomanテンプレなどは作るのめんどいんで、提供する気はないです。これ元に自由に作って公開してもらっても大丈夫です。ただ、間違いなどはあるかもしれないので、そこは気をつけてくださいね。

## 基本構成

まずは、基本のTypeScriptソースをビルドして実行できるまでの環境を作っていきましょう。


### Nodeのインストール

僕はanyenv使って入れてます。node-buildとかデフォルトで入ってくれて便利です。え、便利じゃない？まあ、そういうこともあるかもしれない。

```bash
git clone https://github.com/riywo/anyenv ~/.anyenv
echo 'export PATH="$HOME/.anyenv/bin:$PATH"' >> ~/.your_profile
echo 'eval "$(anyenv init -)"' >> ~/.your_profile
exec $SHELL -l
anyenv install nodenv
nodenv install $(nodenv install -l | grep -e '^\s*[0-9]' | tail -1)
nodenv global $(nodenv install -l | grep -e '^\s*[0-9]' | tail -1)
```

nodenvは、nodeやglobalのnpmパッケージをホームに置きます。さらに、それぞれのnodeバージョンで別個にnpmパッケージを管理します。また、もしあなたがglobalに実行ファイルを持つパッケージをインストールした場合、その実行ファイルにリンクをしたファイルを共通のbinディレクトリに置かなければならないため、`nodenv rehash`という呪文を唱える必要があります。とにかく、何かで詰まったら、`nodenv rehash`と`npm ls -g`を実行してみることです。


### npmパッケージ作成

```bash
mkdir some-project
cd some-project
npm init
```


### TypeScriptのインストール

```bash
npm install --save-dev typescript @types/node
```

`@types/node`に関しては後ほど説明します。いちよ説明しておくとNodeの型定義です。Nodeのアプリ作ろうが作らないだろうが、テストとかで有用なのでいちよ入れておいた方が便利です。まあ、ただNodeのオブジェクトが使えるようになってしまうので、完全にブラウザオンリーなら入れない方がいい場合もありますね。


### TypeScriptの環境設定

```bash
$ cat > tsconfig.json
{
  "compilerOptions": {
    "module": "commonjs",
    "moduleResolution": "node",
    "target": "es5",
    "noFallthroughCasesInSwitch": true,
    "noImplicitAny": true,
    "noImplicitReturns": true,
    "noImplicitThis": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "experimentalDecorators": true,
    "forceConsistentCasingInFileNames": true,
    "preserveConstEnums": true,
    "stripInternal": true,
    "skipDefaultLibCheck": true,
    "skipLibCheck": false,
    "strictNullChecks": true,
    "traceResolution": false,
    "sourceMap": true,
    "declaration": true,
    "newLine": "LF",
    "inlineSources": true,
    "outDir": "dist"
  },
  "include": [
    "lib/**/*.ts",
    "test/**/*.ts"
  ],
  "exclude": [
    "dist",
    "coverage",
    "node_modules"
  ]
}
```


### TypeScriptのビルド設定

これでインストールは`npm install`でビルドまで、テストは`npm test`でって感じですね。

```bash
$ editor package.json
   "scripts": {
+    "prepublish": "npm run build",
+    "build": "tsc --project tsconfig.json",
-    "test": "echo \"Error: no test specified\" && exit 1"
+    "test": "node ./dist/test/index.js"
   }
```


### Gitの設定

```bash
$ git init
$ cat > .gitignore
# Project git ignore file
#
# Notice:
# You should specify environment ignored files at global ignore e.g.
#  * `*~` (vim swap file)
#  *`.DS_Store` (Mac OS Store File)
#  * `Thumb.db` (Windows Store File)
#  * etc.
#

# dists
/dist/
/coverage/

# for Node
node_modules/
npm-debug.log*
isolate-*-v8.log
```


### Sampleファイル

```bash
$ mkdir lib test
$ cat > lib/index.ts
console.log('Hello, World!');
$ cat > test/index.ts
console.error('not implemented!');
process.exit(1);
$ npm run build
$ node ./dist/lib/index.ts
Hello, World!
$ npm test
...
not implemented!
npm ERR! Test failed. ...
```

ここまでの一連の構成の最終形態は、[ここ](https://github.com/mizunashi-mana/typescript-example/tree/ts2.0-basic)にあります。

## 型定義ファイル

さて、TypeScriptで大切なのが型定義ファイルです。これの簡単な説明と扱い方を説明しておきます。


### TypeScript事情

TypeScriptはStatic Types + JavaScriptソースです。

TypeScriptソースはコンパイルすると、TypeScriptの型定義とJavaScriptソースになります。この型定義とJavaScriptソースはファイル名で結びついていて、このセットをユーザーに提供することで、JavaScriptからもTypeScriptからも使えるようになります。逆に言えばJavaScriptだけが提供されているライブラリでも、それに合わせた型定義を用意してあげることでTypeScriptから何の問題もなく使えるようになります。

TypeScriptではTypeScriptソースか型定義をimportすることができます。importするのが型定義なら、それに紐付いているJavaScriptソースが実行時には読み込まれるようコンパイルが行われます。例えば以下のような感じです。

```bash
$ ls lib/
index.ts    js-lib.d.ts js-lib.js
$ cat lib/index.ts
import * as jsLib from './js-lib';

jsLib.testFunc();

$ npm run build
$ cat dist/lib/index.js
"use strict";
var jsLib = require("./js-lib");
jsLib.testFunc();
//# sourceMappingURL=index.js.map
```

TypeScript上では型定義が、コンパイル後はJavaScriptソースがそれぞれ読み込まれるようになります。


### TypeScript2事情

さて、世の中のほとんどのライブラリはJavaScript製で、もちろんTypeScriptの型定義など置いてる酔狂なプロジェクトはとても少数です。そこで、TypeScriptユーザーによって書かれた、JavaScriptライブラリに対応した型定義が[Definitely Typed](https://github.com/DefinitelyTyped/DefinitelyTyped/)というリポジトリに置かれています。このリポジトリを利用することで、私たちは特に何もせずにjavaScriptライブラリをTypeScriptから使えるようになっています。

今まではtsdやtypingsという型定義を管理するための独自ツールが用意されてきたんですが、まあ色々問題があって、TypeScript2からnpmjsレジストリの[@typesグループ](https://www.npmjs.com/~types)を使用することになりました[^notice_types_registry]。

なお、@typesの内容は、DefinitelyTypedのtypes-2.0ブランチからMicrosoftの力で自動反映されています。単純にnpmjsのグループなので、npmコマンドから利用することが可能です。`@types/`というプレフィックスをつけます。上での`@types/node`がそれです。

types-2.0ブランチにPR送ってマージされると、バージョンがインクリメントされて即反映されるようになってます。さすがMicrosoftさん、惚れるわ。

[^notice_types_registry]: 管理体制があんまりちゃんと決まっていなくて、結構不安定。ま、TypeScriptユーザーだしなんとかなるだろwとか思われてる疑惑がある


### サードパーティライブラリ使う場合

では、ライブラリ利用の手順をまとめておきます。

1. ライブラリが型定義提供？
    
    希少ですが、ライブラリで型定義が提供されている場合があります。momentとかがそれです。そういう場合は
    
    ```bash
    npm install --save (ライブラリ)
    ```
    
    これだけで、TypeScriptから利用ができます。ほんとに何もせずに`import '(ライブラリ)'`すれば大丈夫です。
    
    ~~提供されているかは、package.jsonのtypingsという欄が書かれているかを見れば分かります。~~ どうやら、typingsのシノニムとしてtypesで大丈夫になったようです。それと、index.d.tsがある場合、特に記載がなくても利用出来るそうです。なので、index.d.tsが、なければpackage.jsonのtypings/typesを見るとよいですね。自分でライブラリを公開する場合は、TypeScript1系を考慮に入れるなら、typingsという名前で型定義を提供するのがいいかもしれません[^note_thanks_falsandtru]。

1. DefinitelyTypedが型定義提供？
    
    ほとんどの場合はこのパターンです。`npm search @types/(ライブラリ)`して出てくれば提供されています。そういう場合は
    
    ```bash
    npm install --save (ライブラリ)
    npm install --save-dev @types/(ライブラリ)
    ```
    
    とすれば、TypeScriptから利用ができます。`import '(ライブラリ)'`すれば大丈夫です。

[^note_thanks_falsandtru]: http://qiita.com/Mizunashi_Mana/items/c30f9a8a3edbf5dcf9c3#comment-8a899755d01ff2044b6f

### 誰も型を提供していない場合

悲しいかな、たまに誰も型を提供していないJavaScriptライブラリを使う羽目になったりします。そういう場合の手順は以下のようになります。

1. 自分で型定義書く?

    自分で型定義書く気力があれば、DefinitelyTyped#types-2.0にPull Requestを送れば、マージされた後上記の手順で利用ができるようになります。

1. 型書くのちょっと面倒い

    まあ、大体はそうなるでしょう。そういう場合は、
    
    ```bash
    $ cat > mytype.d.ts
    declare module '(ライブラリ)' {
      ...
    }
    ```
    
    って感じで使う部分だけ型を適当に書いた型定義を書くと良いです。そこから、`import './mytype'; import '(ライブラリ)'`とかすればライブラリが使えるようになります。


## リント

プログラムの質を保つにはリンテーションは大事です。リンテーションを基本構成に加えてみます。


### tslint

* https://palantir.github.io/tslint/
* TypeScript2にはちょっと未対応な部分も
    * Bug: https://github.com/palantir/tslint/issues/1769
* でも、大体対応してるので使える
* DefinitelyTypedも使ってる謹製ツールです


### インストール

```bash
$ npm install --save-dev tslint
$ editor package.json
   "scripts": {
     ...
+    "lint": "tslint lib/**/*.ts test/**/*.ts",
-    "test": "node ./dist/test/index.js"
+    "testonly": "node ./dist/test/index.js",
+    "pretest": "npm run lint",
+    "test": "npm run testonly"
   }
```


### リントルール作成

```bash
$(npm bin)/tslint --init
```

Ruleの詳細は、https://palantir.github.io/tslint/rules/


### 僕の使ってるルール

なお、trailing-commaは件のバグのため・・・:pray: まあ、PRマージされたし、次期リリースでは治ってるはず・・・！

```bash
$ cat > tslint.json
{
  "rules": {
    "adjacent-overload-signatures": true,
    "align": [
      true,
      "parameters",
      "statements"
    ],
    "array-type": [
      true,
      "array"
    ],
    "arrow-parens": false,
    "class-name": true,
    "comment-format": [
      true,
      "check-space"
    ],
    "curly": true,
    "cyclomatic-complexity": [
      true
    ],
    "eofline": true,
    "forin": true,
    "indent": [
      true,
      "spaces"
    ],
    "interface-name": [
      true,
      "never-prefix"
    ],
    "jsdoc-format": true,
    "label-position": true,
    "linebreak-style": [
      true,
      "LF"
    ],
    "max-classes-per-file": [
      true,
      5
    ],
    "max-line-length": [
      true,
      120
    ],
    "member-access": true,
    "new-parens": true,
    "no-any": false,
    "no-arg": true,
    "no-bitwise": true,
    "no-conditional-assignment": true,
    "no-consecutive-blank-lines": [
      true
    ],
    "no-console": [
      true,
      "debug",
      "info",
      "time",
      "timeEnd",
      "trace"
    ],
    "no-construct": true,
    "no-debugger": true,
    "no-empty": true,
    "no-eval": true,
    "no-inferrable-types": [
      true
    ],
    "no-internal-module": true,
    "no-invalid-this": true,
    "no-null-keyword": true,
    "no-reference": true,
    "no-require-imports": true,
    "no-shadowed-variable": true,
    "no-string-literal": false,
    "no-switch-case-fall-through": true,
    "no-trailing-whitespace": true,
    "no-unsafe-finally": true,
    "no-unused-expression": true,
    "no-unused-new": true,
    "no-use-before-declare": true,
    "no-var-keyword": true,
    "no-var-requires": true,
    "object-literal-key-quotes": [
      true,
      "as-needed"
    ],
    "object-literal-shorthand": true,
    "object-literal-sort-keys": false,
    "one-line": [
      true,
      "check-open-brace",
      "check-catch",
      "check-else",
      "check-finally",
      "check-whitespace"
    ],
    "one-variable-per-declaration": [
      true,
      "ignore-for-loop"
    ],
    "only-arrow-functions": [
      true,
      "allow-declarations"
    ],
    "ordered-imports": [
      true,
      {
        "import-sources-order": "any",
        "named-imports-order": "any"
      }
    ],
    "prefer-for-of": true,
    "quotemark": [
      true,
      "single",
      "avoid-escape"
    ],
    "radix": true,
    "semicolon": [
      true,
      "always"
    ],
    "switch-default": true,
    "trailing-comma": [
      false,
      {
        "singleline": "never",
        "multiline": "always"
      }
    ],
    "triple-equals": [
      true,
      "allow-null-check"
    ],
    "typedef-whitespace": [
      true,
      {
        "call-signature": "nospace",
        "index-signature": "nospace",
        "parameter": "nospace",
        "property-declaration": "nospace",
        "variable-declaration": "nospace"
      }
    ],
    "use-isnan": true,
    "variable-name": [
      true,
      "ban-keywords",
      "check-format",
      "allow-pascal-case"
    ],
    "whitespace": [
      true,
      "check-branch",
      "check-decl",
      "check-operator",
      "check-separator",
      "check-type"
    ]
  }
}
$ npm run lint
```


## テスト・カバレッジ

テストやカバレッジは、プログラムの質を保証し、保守管理をするために必須ですね。というわけで、さらにテスト・カバレッジタスクを加えてみます。


### Karmaは省略ね

* ブラウザ側まで僕はテストは回らないので
* 大体e2eテストで済ましてるんで
* 大体e2eテストフレームワーク優秀なんで
* もう、なんかめんどい、Karma
* すーぐAPI変わるし、もうほんと、しんどい
* PhantomJSすぐ動かなくなるし、もうほんと(ry


### Mochaのインストール

```bash
$ npm install --save-dev \
  mocha @types/mocha \
  ts-node
```

ts-nodeはコンパイルせずmochaテストをTypeScriptで書くための小技のために必要です。


### Mochaの設定

```bash
$ editor package.json
   "scripts": {
     ...
-    "testonly": "node ./dist/test/index.js"
+    "testonly": "mocha --compilers ts:ts-node/register,tsx:ts-node/register test/**/*.test.ts"
     ...
   }
```


### Sampleファイル

```bash
$ cat > test/sample.test.ts
import * as assert from 'assert';
import '../lib';

describe('sample target', () => {

  it('should be valid', () => {
    assert(true, 'sample test');
  });

});
$ npm run testonly
```


## TypeScriptでのカバレッジ

ところで、カバレッジと簡単に言いましたが、実はTypeScriptのカバレッジなど存在しません。なぜなら、そもそもTypeScriptには実行という概念がないからです。上記で述べた通り、TypeScript = Static Types + JavaScriptです。実行されるのはJavaScriptです。

しかし、それでは不便です。そんな変に厳密な解釈講義など必要ないし、私たちに必要なのは開発時間の短縮です。さて、TypeScriptに実行の概念はないわけですが、JavaScriptのカバレッジ結果をTypeScriptソースの該当箇所にマップすることは可能です。ソースマップの出番ですね。

一般にTypeScriptプロジェクトのカバレッジタスクは、以下の流れをとります。

1. TypeScriptからJavaScriptへコンパイル
2. JavaScriptソースのカバレッジを取る
3. JavaScriptカバレッジ情報を、TypeScriptのソースへソースマップでマッピングし直す

では具体的にIstanbulでタスクを追加してみます。


### Istanbulのインストール

```bash
$ npm install --save-dev \
  istanbul \
  remap-istanbul \
  source-map-support
```

remap-istanbulがまさに今回の肝で、ソースマップからIstanbulのカバレッジ情報をリマッピングしてくれるツールです。

それから、source-map-supportはテスト途中でエラーが起きた時、StackTraceをTypeScriptソースの箇所として表示させるための小技です。単純にテストだけの時は、これをts-nodeがやってくれたんですが、今回はコンパイルをしなければいけないので、source-map-supportの力を借りねばなりません。


### Istanbulの設定

```bash
$ editor
   "scripts": {
     ...
+    "precoverage": "npm run build",
+    "coverage": "istanbul cover _mocha -- --require source-map-support/register ./dist/test/**/*.test.js"
+    "postcoverage": "remap-istanbul -i coverage/coverage.json -o coverage/ts-lcov-report -t html"
   }
$ npm run coverage
...
```

あとは、`coverage/ts-lcov-report/index.html`にレポートが出るので、それを見るだけです。やったね。

今までの最終形態は、[ここ](https://github.com/mizunashi-mana/typescript-example/tree/ts2.0-fullset)に置いてあります。


## ビルドツール


### ビルドツールの必要性

今までnpm-scriptsでタスクを書いてきました。もちろんそれでも可ですが・・・・・・。

npm-scriptsの問題点には以下のような問題点があります。

* なんしろ書きにくい
* 何も考えずに書くとWindowsで実行できない
    * Windowsのデフォルトはcmd.exeなんですよね・・・
    * &&とか使えないです
* 並列性が・・・
    * タスクは並列化できる場合が多いです
    * でもワンライナーで並列化なんて書きたくないですよね
* Globパターンが使えないことが多い
    * 今回は優秀なツールが多いですが、引数をGlobパターンで見てくれるツールなんてあまりないです
    * そういう時はGlobパターン対応したライブラリが使いたくなるわけです


NodeのアプリなんだからNodeだけあれば、環境が整うのがいいですよね。Nothing Shell/Nothing Build-In Commandsの精神で生きていきましょう。


### ビルドツールのインストール

Gulp使います

Grunt？Jake？Webpack？Fly？知らない子ですねえ・・・

まあGulpも最近開発停滞気味なんで、そろそろ潮時かもですね

```bash
$ npm install --save-dev gulp \
  gulp-load-plugins \
  gulp-typescript gulp-tslint gulp-sourcemaps \
  gulp-mocha gulp-istanbul \
  coffee-script \
  del merge2 yargs run-sequence
```

Coffee ScriptがないとGulpfile書けない体になってしまっての・・・

TypeScriptで書きたい人は適当に型定義付け加えてインストールしましょう。


### Gulpfileを書く

`gulpfile.coffee`を順次書いていきます。

まず諸々プラグインを使えるようにします。

```coffee
gulp = require 'gulp'
$ = do require 'gulp-load-plugins'

$.remapIstanbul = require 'remap-istanbul/lib/gulpRemapIstanbul'
$.del = require 'del'
$.merge = require 'merge2'
$.runSequence = require 'run-sequence'
  .use gulp
```

雑にオプションパースしておきます。

```coffee
argv = require 'yargs'
  .argv

if argv.production
  argv.mode = 'production'
else if argv.development
  argv.mode = 'development'
else if process.env.NODE_ENV isnt undefined
  argv.mode = process.env.NODE_ENV
else
  argv.mode = 'development'
```

TypeScriptのプロジェクトを作っておくと、watchとかで便利なので作っておきます。

```coffee
tsProject = $.typescript.createProject 'tsconfig.json',
  typescript: require 'typescript'
```

Mochaのためのオプションと、registerを読み込んでおきます。registerを読み込んでおくと、TypeScriptのrequireとかをいい感じにしてくれます。mochaライブラリ内のrequireにももちろん適用されるのでいい感じになります。それぞれテストタスク用・カバレッジタスク用なのは変わらないです。雑にいきましょ。どうせタスクファイルなんだから。

```coffee
# for mocha
require 'ts-node/register'
require 'source-map-support/register'

mochaOptions = (reporterType) ->
  reporter: process.env.MOCHA_REPORTER || reporterType
  timeout: 5000
```

まずは、ビルドタスクです。よく分かんないんですが、型定義ファイルはコンパイル時dist内にはコピーされないんで、それもコピーしてます。テストの方の型定義はいらないんで特にコピーしてないです。

```coffee
gulp.task 'build:tsc', ->
  tsResult = gulp.src [
    'lib/**/*.ts'
    'test/**/*.ts'
  ]
    .pipe $.sourcemaps.init()
    .pipe tsProject $.typescript.reporter.defaultReporter()

  $.merge [
    tsResult.js
    tsResult.dts
  ]
    .pipe $.sourcemaps.write()
    .pipe gulp.dest 'dist'

gulp.task 'build:dts', ->
  gulp.src [
    'lib/**/*.d.ts'
  ]
    .pipe gulp.dest 'dist/lib'

gulp.task 'build:ts', [
  'build:tsc'
  'build:dts'
]

gulp.task 'build', [
  'build:ts'
]
```

お次はリンテーションタスクです。リンテーションとかテストとかは並列で動かすと、表示が酷くなるので基本runSequenceとかで同期実行しとくのが良いですね。

```coffee
tslint = require 'tslint'

gulp.task 'lint:ts', ->
  gulp.src [
    'lib/**/*.ts'
    'test/**/*.ts'
  ]
    .pipe $.tslint
      tslint: tslint
    .pipe $.tslint.report()

gulp.task 'lint', (cb) ->
  $.runSequence 'lint:ts'
    , cb
```

続いてテストタスクです。Mochaのレポーターはnyanが好きです。特に意味はありません。

```coffee
gulp.task 'test:ts', ->
  gulp.src [
    'test/**/*.test.ts'
  ], { read: false }
    .pipe $.mocha mochaOptions {
      production: 'spec'
      development: 'nyan'
    }[argv.mode]

gulp.task 'test', (cb) ->
  $.runSequence 'test:ts'
    , cb
```

では次がカバレッジです。カバレッジは少々複雑です。remap-istanbulのgulpプラグインのAPIがちょっと酷くって、まあいつもいつもPR送ろうって思いながら送ってないです。いつか送ろう。

```coffee
gulp.task 'coverage:ts-pre', [
  'build:ts'
], ->
  gulp.src [
    'dist/lib/**/*.js'
  ]
    .pipe $.istanbul()
    .pipe $.istanbul.hookRequire()

gulp.task 'coverage:ts-trans', [
  'coverage:ts-pre'
], ->
  gulp.src [
    'dist/test/**/*.test.js'
  ], { read: false }
    .pipe $.mocha mochaOptions {
      production: 'progress'
      development: 'nyan'
    }[argv.mode]
    .pipe $.istanbul.writeReports
      reporters: ['lcov', 'json']

gulp.task 'coverage:ts', [
  'coverage:ts-trans'
], ->
  gulp.src [
    'coverage/coverage-final.json'
  ]
    .pipe $.remapIstanbul
      reports:
        'text': undefined
        'text-summary': undefined
        'lcovonly': 'coverage/ts-lcov.info'
        'json': 'coverage/ts-coverage-final.json'
        'html': 'coverage/ts-lcov-report'
      reportOpts:
        log: console.log

gulp.task 'coverage', (cb) ->
  $.runSequence 'coverage:ts'
    , cb
```

後はせっかくなんでcleanタスクとかも加えておきます。watchとかは僕はほぼエディタ任せなんですが、CUIユーザーはもっと増やしてもいいかもですね。

```coffee
gulp.task 'clean', (cb) ->
  $.del [
    'dist'
    'coverage'
  ], cb

gulp.task 'watch', ->
  gulp.watch [
    'lib/**/*.ts'
    'test/**/*.ts'
  ], [
    'build:ts'
  ]
```

### npm-scriptsを書く

Gulpに合わせて、npm-scriptsも変えておきます。

```json
  "scripts": {
    "prepublish": "npm run build",
    "build": "gulp build",
    "watch": "gulp watch",
    "clean": "gulp clean",
    "lint": "gulp lint",
    "testonly": "gulp test --production",
    "pretest": "npm run lint",
    "test": "gulp test",
    "coverage": "gulp coverage"
  }
```

なお、今までの最終形態は[ここ](https://github.com/mizunashi-mana/typescript-example/tree/ts2.0-myproject)に置いてあります。

## 最後に

はい、というわけでTypeScript2の作り方を見ていきました。

こんな感じで僕は作ってるんで、宜しくお願いします。

## 補足

この記事結構前に書いたやつで予約投稿してたんですけど、今読んでみて思ったことを幾つか補足しておきます。

まず、tslintの4.1.1がきました( https://github.com/palantir/tslint/pull/1875 )。 ~~trailing_commaのバグはなおっていそうです（まだ試してない）。~~ これは僕の勘違いで、修正自体は4.0.2で入っていたようです。しかしながら、幾つかの未修正のケースが残っていたようです。

それから、型定義がない場合の話ですが、TypeScript 2.1.4から、[untyped importsが許可されるようになった](https://github.com/Microsoft/TypeScript/pull/11889)ようですね。ということで、noImplicitAnyを許可する雑多な環境なら、特に何もせずとも型定義がなくともimportができるようになったみたいです。まあ、せめて自分たちが使う関数ぐらいは、型書いとくのが無難だと思いますけどね。

