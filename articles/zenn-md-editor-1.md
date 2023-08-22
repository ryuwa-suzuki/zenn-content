---
title: "Electron ✖️React ✖️TypeScriptでZenn用マークダウンエディタを作ってみる"
emoji: "✏️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react, typescript, electron, javascript, マークダウン]
published: true

---

みなさんはじめまして。

普段はバックエンドエンジニアとして、ひっそりとphp等を書くことでご飯を食べております。
この度は流行りのZennに投稿するということで、大人気の**React**と**TypeScript**を学習しながらアプリを作ってみる記事にすることにしました。

学習するにしてもTodoアプリなどではモチベーションを保てず、ゼルダやピクミンに時間を奪われがちなので、**俺の俺による俺のための**エディタを作るんだっていう気持ちで挑むことにします。

Electronって何？って人もいるかもしれないので説明しておくと、Windows、macOS、Linuxのデスクトップアプリケーションを作成するためのフレームワークです。
自分のアプリ作って使うってちょっと興奮しますよね。

# はじめに

以下のgifのようにGit連携したZennの記事や本を編集できるマークエディタを目指します。
![](/images/ezgif.com-video-to-gif.gif)

このアプリを作ろうと思ったきっかけは、
- ZennのWebエディタにMarkdown記法を挿入するツールバーがないこと
- プレビューをしながら編集できないこと
- github連携すると画像のアップロードがめんどくさいこと
- ちょうどTypeScriptとReactを勉強しようと思ってたこと

です。
この課題を全て解決するアプリにしようと思ってます。

## 完成物
以下に置いております。Macで開発したので、Mac用のアプリしか作成してません。
Windowsは使ったことほとんどないので、Mac前提で書いてます。すいません🙇‍♂️
https://github.com/ryuwa-suzuki/zenn-md-editor/releases

ソースコードはこちら
https://github.com/ryuwa-suzuki/zenn-md-editor

## 前提条件
- [ZennとのGitHubリポジトリ連携](https://zenn.dev/zenn/articles/connect-to-github)が完了していること

## 実装する機能
最低限、以下の機能を実装します。
- Zennとのファイル同期（ローカルにあるZennのディレクトリから記事や本のファイルを取得）
- Zennファイルへの自動保存
- 画像のアップロード（Zennのディレクトリ/imagesにアップロードしたファイルをコピー）
- 画像のプレビューZennのディレクトリ/imagesにアップロードされたファイルを参照する）
- コードブロックの言語別ハイライト
- コードブロックのファイル名を表示

## 利用するツール・ライブラリ
- [ElectronForge](https://www.electronforge.io/)
	electronをいい感じで簡単に使えるようにしてくれる
- [react-simplemde-editor](https://github.com/RIP21/react-simplemde-editor)
 いい感じのマークダウンエディタ
- [marked](https://github.com/markedjs/marked)
マークダウン形式の文字列をHTMLに変換してくれる
- [highlight.js](https://github.com/highlightjs/highlight.js)
  コードブロックにハイライトつけてくれる
- [mantine](https://mantine.dev/)
  ReactのUIライブラリ
	
# プロジェクトの作成
早速[ドキュメント](https://www.electronforge.io/templates/typescript-+-webpack-template)に従い、以下のコマンドでWebpack,TypeScriptをtemplateに選択して、Electron アプリを作成します。

```
$ npm init electron-app@latest zenn-md-editor -- --template=webpack-typescript
```
以上のコマンドを実行するとzenn-md-editorフォルダができあがるので、とりあえず動くか確認します。

```
$ cd zenn-md-editor
$ npm run start
```

できました！
![](/images/screen-shot1.png)

続いて、tsconfig.jsonにJSX変換の指定をします。
```json:tsconfig.json
{
  "compilerOptions": {
    "target": "ES6",
    "allowJs": true,
    "module": "commonjs",
    "skipLibCheck": true,
    "esModuleInterop": true,
    "noImplicitAny": true,
    "sourceMap": true,
    "baseUrl": ".",
    "outDir": "dist",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "paths": {
      "*": ["node_modules/*"]
    },
    "jsx": "react-jsx", // 追加
  },
  "include": ["src/**/*"]
}
```
これで開発準備が整ったといったところでしょうか。
簡単でした。ありがとうございます😭
# マークダウンエディタの作成
## 初期画面の表示

まず必要なライブラリのインストールを行います。
```
$ npm install --save react react-dom react-simplemde-editor easymde highlight.js marked
$ npm install --save-dev @types/react @types/react-dom @types/marked
```

続いてsrcディレクトリにroot.tsxとApp.tsxファイルを新規作成してApp.tsxを初期画面として表示します。
- 🗂src/index.htmlに初期画面表示のためidを追記
```diff html:src/index.html
  <head>
    <meta charset="UTF-8" />
    <title>Hello World!</title>
  </head>
  <body>
-	<h1>💖 Hello World!</h1>
-  <p>Welcome to your Electron application.</p>

+    <div id="app"></div>
  </body>
```

- 🗂root.tsでAppを表示
```tsx:src/root.tsx
import { createRoot } from "react-dom/client";

import App from "./App";

function render() {
  const root = createRoot(document.getElementById("app"));
  root.render(<App />);
}

render();
```

- 🗂renderer.tsでroot.tsxをインポート。このファイルはWebpackで自動的にロードされ、"renderer" contextで実行されます。レンダラーやメインプロセスについては後ほど。
```diff ts:renderer.ts
import './index.css';
+ import './root';

console.log('👋 This message is being logged by "renderer.js", included via webpack');
```

- 🗂App.tsx編集
```tsx:src/App.tsx
const App: React.FC = () => {
  return (
    <div>Hello World!</div>
  )
}

export default App;
```
中身は説明するまでもないので省きます。
表示できました！

![](/images/screen-shot2.png)

## マークダウンエディタのComponent作成
いよいよ本番！と言っても、ライブラリを使用するだけですが、、

srcディレクトリ配下にcomponentsディレクトリを作成し、Home.tsxとeditor/MarkdownEditor.tsxファイルを作成します。
Home.tsxは各コンポーネントの呼び出し、MarkdownEditor.tsxはマークダウンエディタ自体です。
ディレクトリ構成は以下。
```
src
├─ components
│   ├─ editor
│   │   └─ MarkdownEditor.tsx
│   └─ Home.tsx
└─ App.tsx
``` 
- 🗂App.tsx編集
```diff tsx:src/App.tsx
+ import Home from './components/Home';

const App: React.FC = () => {
  return (
-    <div>Hello World!</div>
+   <Home/>
  )
}

export default App;
```

- 🗂Home.tsx作成
```tsx: src/components/Home.tsx
import MarkdownEditor from '../components/editor/MarkdownEditor';

const Home: React.FC = () => {
  return (
    <>
      <MarkdownEditor/>
    </>
  )
}

export default Home;
```

- 🗂MarkdownEditor.tsx作成
```tsx:src/components/editor/MarkdownEditor.tsx
import SimpleMdeReact from "react-simplemde-editor";
import "easymde/dist/easymde.min.css";

const MarkdownEditor: React.FC = () => {
  return (
    <SimpleMdeReact id="simple-mde"/>
  );
};

export default MarkdownEditor;
```

ここまでで一旦表示してみると、動作はしていますがツールバーのアイコンが表示されません。

![](/images/screen-shot3.png)

エラーを見ると各モジュールのCDNがCSPで拒否されているようです。

超簡単に言うと、Webサイトが攻撃されないようにセキュリティ設定してあるので、外部読み込みで許可するものは明示的に許可してってことですかね。
※ 詳しくは[コチラ](https://developer.mozilla.org/ja/docs/Web/HTTP/CSP)

[ドキュメント](https://www.electronjs.org/ja/docs/latest/tutorial/security)に詳しく載ってますが、src/index.tsを編集してエラーが出ている該当ドメインを許可します。

```diff ts:src/index.ts
-import { app, BrowserWindow } from 'electron';
+import { app, BrowserWindow, session } from 'electron';

...

- app.on('ready', createWindow);

+// セッションが準備できた後にCSPを設定
+app.whenReady().then(() => {
+  session.defaultSession.webRequest.onHeadersReceived((details, callback) => {
+    const cspHeader = "default-src 'self' 'unsafe-eval' 'unsafe-inline' file: data: https://maxcdn.bootstrapcdn.com https://cdn.jsdelivr.net/;";
+
+    callback({
+      responseHeaders: {
+        ...details.responseHeaders,
+        'Content-Security-Policy': [cspHeader]
+      }
+    });
+  });
+
+  createWindow(); // 最初のウィンドウを作成
+});
```

これでツールバーが問題なく表示されました！

# エディタのカスタマイズ
## ツールバーのカスタマイズ
このアプリを作ろうと思ったキッカケの一つに、Markdown記法を挿入するツールバーが欲しいという気持ちがあったので、ツールバーのカスタマイズからいきます。

SimpleMdeコンポーネントにはEasyMDEのオプションをpropsとして渡すことができ、ツールバーをカスタマイズすることができます。

[https://github.com/Ionaru/easy-markdown-editor#toolbar-icons](https://github.com/Ionaru/easy-markdown-editor#toolbar-icons)にデフォルトで用意されているツールバーの一覧が見れるので、ここから好きなものを追加します。

- 🗂MarkdownEditor.tsxを編集
```diff tsx:src/components/editor/MarkdownEditor.tsx
+import { useMemo } from "react";
 import SimpleMdeReact from "react-simplemde-editor";
+import SimpleMDE from "easymde";
 import "easymde/dist/easymde.min.css";
+const toolbar: SimpleMDE.Options["toolbar"] = [
+  'bold',
+  'italic',
+  'quote',
+  'unordered-list',
+  'ordered-list',
+  'link',
+  'image',
+  'strikethrough',
+  'code',
+  'table',
+  'redo',
+  'heading',
+  'undo',
+  'clean-block',
+  'horizontal-rule',
+  'preview',
+  'side-by-side',
+  'fullscreen'
+];
 
 const MarkdownEditor: React.FC = () => {
+  const mdeOptions: SimpleMDE.Options = useMemo(() => {
+    return {
+      width: 'auto',
+      spellChecker: false,
+      toolbar
+    };
+  }, []);
+
   return (
-    <SimpleMdeReact id="simple-mde"/>
+    <SimpleMdeReact
+      id="simple-mde"
+      options={mdeOptions}/>
   );
 };
```
## オートセーブ機能の追加

SimpleMDEオプションにオートセーブ機能があるので、それを利用します。
Zennファイルとの実装時には編集中のファイルに内容を書き込みますが、同期していない場合も使えるようにしたいので。

このオートセーブ機能はローカルストレージにsmde_{指定したid}というKeyで保存してくれます。
編集1秒後に保存されるように設定します。
- 🗂MarkdownEditor.tsxを編集
```diff tsx:src/components/editor/MarkdownEditor.tsx
...

+const delay = 1000;// 1秒後に保存されるように設定
 
 const MarkdownEditor: React.FC = () => {
+  const value = localStorage.getItem('smde_saved_content') || "";// リロード時にローカルストレージの値をバリューにセット
   const mdeOptions: SimpleMDE.Options = useMemo(() => {
     return {
       width: 'auto',
       spellChecker: false,
-      toolbar
+      toolbar,
+      autosave: {
+        enabled: true,
+        uniqueId: "saved_content",
+        delay
+      },
     };
   }, []);
 
   return (
     <SimpleMdeReact
       id="simple-mde"
-      options={mdeOptions}/>
+      options={mdeOptions}
+      value={value}/>
   );
 };
```

これでリロードしても値が保存されるようになりました！

## コードブロックのカスタマイズ
まずはhighlight.js, marked, highlight.jsで自分の好みのcssをインポートします。
https://highlightjs.org/examples

- 🗂MarkdownEditor.tsxを編集
```diff tsx:src/components/editor/MarkdownEditor.tsx
+ import { marked } from "marked";
+ import hljs from 'highlight.js';
+ import "highlight.js/styles/base16/bright.css"; //👉 https://highlightjs.org/examples
```

続いてコードブロックでtsx:index.tsxのように{言語}:{ファイル名}で入力できるようにしたいので、SimpleMDE.OptionsのpreviewRenderを使ってカスタマイズしていきます。

previewRenderはプレーンテキストの Markdown を解析して HTMLを返すカスタム関数でユーザーがプレビューするときに使用されます。

- 🗂MarkdownEditor.tsxを編集

```diff tsx:src/components/editor/MarkdownEditor.tsx
...

+  const previewRender = (value: string): string => {
+    const renderer = new marked.Renderer();
+    renderer.code = (code, codeInfo) => {
+      const codeInfoSplit = codeInfo.split(':');
+      const lang = codeInfoSplit[0]
+      const fileName = codeInfoSplit[1];
+      const langClass = hljs.getLanguage(lang) ? lang : 'plaintext';
+      const highlightedCode = hljs.highlight(langClass, code).value;
+      const codeBlockClass = fileName === undefined ? 'code-block-no-info' : 'code-block';
+
+      let codeBlock = `<code class="hljs ${codeBlockClass} language-${langClass}">${highlightedCode}</code>`
+      if (fileName !== undefined) {
+        codeBlock = `<div class="code-info"><span>${fileName}</span></div>` + codeBlock
+      }
+      return `<pre>${codeBlock}</pre>`
+    };
+
+    return marked(value, { renderer });
+  };
+
   const mdeOptions: SimpleMDE.Options = useMemo(() => {
     return {
       width: 'auto',
       spellChecker: false,
       toolbar,
+      previewRender,
       autosave: {
         enabled: true,
         uniqueId: "saved_content",
         delay
       },
     };
-  }, []);
+  }, [previewRender]);
```

コードブロックの見た目も整えたいのと、エディタが全画面に表示されて欲しいので、cssを編集します。
- 🗂index.cssを編集
```css:src/index.css
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica,
    Arial, sans-serif;
  margin: auto;
  max-width: auto;
}

.code-block{
  border-radius: 0px 0px 10px 10px
}

.code-block-no-info{
  border-radius: 10px
}

code{
  font-size: 1rem;
}

.code-info {
  display: flex;
  justify-content: space-between;
  color: #e6e6ee;
  background: #484747;
  padding: 12px;
  font-size: 12px;
  border-radius: 10px 10px 0px 0px;
}
```

まぁまぁいい感じ。
![](/images/screen-shot4.png)

## ここまでのMarkdownEditor.tsxのコード全体
```tsx:src/components/editor/MarkdownEditor.tsx
import { useMemo } from "react";
import SimpleMdeReact from "react-simplemde-editor";
import SimpleMDE from "easymde";
import { marked } from "marked";
import hljs from 'highlight.js';
import "easymde/dist/easymde.min.css";
import "highlight.js/styles/base16/bright.css"; //👉 https://highlightjs.org/examples

const toolbar: SimpleMDE.Options["toolbar"] = [
  'bold',
  'italic',
  'quote',
  'unordered-list',
  'ordered-list',
  'link',
  'image',
  'strikethrough',
  'code',
  'table',
  'redo',
  'heading',
  'undo',
  'clean-block',
  'horizontal-rule',
  'preview',
  'side-by-side',
  'fullscreen'
];

const delay = 1000; // 1秒後に保存されるように設定

const MarkdownEditor: React.FC = () => {
  const value = localStorage.getItem('smde_saved_content') || ""; // リロード時にローカルストレージの値をSimpleMdeReactコンポーネントのvalueとしてセット

  const previewRender = (value: string): string => {
    const renderer = new marked.Renderer();
    renderer.code = (code, codeInfo) => {
      const codeInfoSplit = codeInfo.split(':');
      const lang = codeInfoSplit[0]
      const fileName = codeInfoSplit[1];
      const langClass = hljs.getLanguage(lang) ? lang : 'plaintext';
      const highlightedCode = hljs.highlight(langClass, code).value;
      const codeBlockClass = fileName === undefined ? 'code-block-no-info' : 'code-block';

      let codeBlock = `<code class="hljs ${codeBlockClass} language-${langClass}">${highlightedCode}</code>`
      if (fileName !== undefined) {
        codeBlock = `<div class="code-info"><span>${fileName}</span></div>` + codeBlock
      }
      return `<pre>${codeBlock}</pre>`
    };

    return marked(value, { renderer });
  };

  const mdeOptions: SimpleMDE.Options = useMemo(() => {
    return {
      width: 'auto',
      spellChecker: false,
      toolbar,
      previewRender,
      autosave: {
        enabled: true,
        uniqueId: "saved_content",
        delay
      },
    };
  }, [previewRender]);

  return (
    <SimpleMdeReact
      id="simple-mde"
      options={mdeOptions}
      value={value}/>
  );
};

export default MarkdownEditor;
```

# Zennとの連携機能概要
ここまででマークダウンエディタ作成することはできたので、Zennとの連携について考えたいと思います。
## GitHub連携したZennのディレクトリ構成の確認
まずGitHub連携したZennのディレクトリ構成を見ていきます。

```
zennのルートディレクトリ
├─ articles
│   └─ 記事
├─ books
│   └─ 本のスラッグ
│        └─ 本のチャプタ
└─images
``` 

記事はarticles配下に、booksはarticlesよりも１階層深く、books配下に本のスラッグ→本のチャプタとなってました。

画像に関してはimagesフォルダに配置してくれとのことでした。
/imagesというパスで参照できるようです。

## 機能の詳細
ディレクトリ構成は想定通りなので、機能の詳細について考えます。
- zennディレクトリのフルパスの登録
アプリからzennの情報を取得するために、ディレクトリのフルパスをローカルストレージに登録します。
ローカルストレージにzennディレクトリのフルパス情報がなければモーダルを表示し、そこから登録できるようにします。また、ツールバーの設定アイコンからも登録できるようにします。
- ナビゲーションバーの表示
  articlesとbooksのmdファイルを取得し、ナビゲーションバーに表示します。
	再度開いた時に前回編集していたページを表示したいので、編集中に中に選択しているファイル名やスラッグ名もローカルストレージに保存します。
	イメージはこんな感じ

![](/images/screen-shot5.png =250x)

- マークダウンファイルの読み込みと保存
フルパスと選択したファイルさえわかれば、node.jsでファイルの書き込みや読み込みは可能なので、簡単かと思います。

- 画像のアップロード
画像ファイルはimagesフォルダに配置してとのことなので、アップロードされたファイルをそのままimagesフォルダにコピーして配置するのみとします。
- 画像のプレビュー/images/xxxと入力されたパスをプレビュー時にfile://フルパス/imagesに書き換えて表示したいと思います。
セキュリティ的に問題ありそうですが、自分で使うだけなので。

# Zennとの連携機能の実装
長い記事になりそうなので、詳細はなるべく割愛しながら進めたいと思います。。

### mantineのインストール
モーダルやサイドバーを一から作るのは面倒なので、ReactのUIライブラリであるmantineを利用したいと思います。
とても使いやすかったので、オススメです！
https://mantine.dev/

```
$ npm install @mantine/core @mantine/hooks @emotion/react
$ npm install @tabler/icons-react
```

### ipcMainとipcRendererについて
electronはメインプロセスとレンダラープロセスという二つのプロセスに分かれています。
メインプロセスはアプリケーションの制御、レンダラープロセスは画面を担当するプロセスです。

ざっくり説明ですいませんが、今まで書いてきたのはレンダラープロセスで、node.jsが使えません。
node.jsでファイルの読み込みや書き込みをしたいわけなので、レンダラープロセスからメインプロセスにアクセスする処理を実装していきます。
API通信みたいなもんですかね。

## Zennコンテンツのファイル一覧を取得する
ナビゲーションバーに編集するファイルを表示したいので、ファイル一覧を取得する処理を実装します。

### preloadスクリプトにAPIを公開
まずpreloadスクリプトでレンダラープロセスとメインプロセスの間で非同期の通信を行うAPIを公開します。
preloadスクリプトは、レンダラープロセス内で実行されpreloadスクリプト内で定義された関数やオブジェクトをレンダラープロセスのコンテキストに安全に公開することができます。
https://www.electronjs.org/docs/latest/tutorial/process-model#preload-scripts

- 🗂src/preload.tsを編集します。
```ts:src/preload.ts
// See the Electron documentation for details on how to use preload scripts:
// https://www.electronjs.org/docs/latest/tutorial/process-model#preload-scripts
import { contextBridge, ipcRenderer } from 'electron';
contextBridge.exposeInMainWorld(
  'api', {
    syncWithZenn: (zennDirPath: string) => ipcRenderer.invoke('sync-with-zenn', {zennDirPath}),
  }
)
```

apiという、`api`という名前のオブジェクトを作成し、`syncWithZenn`という関数をプロパティとして持たせました。
これで、レンダラープロセスからwindow.api.syncWithZenn(zennのディレクトリパス)でメインプロセスに通信できます。

### 実際にファイルを取得する処理
以下のディレクトリ構成で、実際にファイルを取得する関数の実装をします。
backgroundフォルダにindex.tsとzenn.tsファイルを作成します。

```
src
└ background
	├─ index.ts
	└─ zenn.ts
``` 

- 🗂zenn.ts作成
syncWithZenn関数の中身を実装します。
ipcMain.handleでレンダラープロセスからのリクエストを受け取ります。
zennディレクトリのフルパスを引数に受け取りfsでファイルの読み込みを行なってます。
```ts:src/background/zenn.ts
import fs from 'fs';
import path from 'path';

export function syncWithZenn (ipcMain: Electron.IpcMain) {
  ipcMain.handle('sync-with-zenn', async (e, {zennDirPath}) => {
    if (!fs.existsSync(zennDirPath)) throw new Error('no-content');

    let articles = {};
    if (fs.existsSync(path.join(zennDirPath, 'articles'))) {
      articles = fs.readdirSync(path.join(zennDirPath, 'articles')).filter(file => file.endsWith('.md'));
    }

    let books = {};
    if (fs.existsSync(path.join(zennDirPath, 'books'))) {
      const booksDirs = fs.readdirSync(path.join(zennDirPath, 'books')).filter(file => {
        const filePath = path.join(zennDirPath, 'books', file);
        const stats = fs.statSync(filePath);

        return stats.isDirectory();
      });

      books = booksDirs.map((dirName) => {
        const bookfiles = fs.readdirSync(path.join(zennDirPath, 'books', dirName)).filter(file => file.endsWith('.md'));
        return {
          bookName: dirName,
          files: bookfiles
        }
      })
    }

    if (!Object.keys(articles).length && !Object.keys(books).length) {
      throw new Error('no');
    }

    return {
      articles: articles,
      books: books
    }
  })
}
```

- 🗂index.ts作成
こちらは先ほど作ったメソッドを呼び出しているだけです。
```ts:src/background/index.ts
import { ipcMain } from 'electron'
import { syncWithZenn } from './zenn'
export function initIpcMain () {
  syncWithZenn(ipcMain)
}
```

- 🗂型を管理するzennTypes.tsの作成
型を定義するファイルがあった方がいいなと思ったので作ったんですが、どうなんですかね？TypeScriptもほとんど触ったことないので、アドバイスいただけたら嬉しいです。。
```ts:src/types/zennTypes.ts
type Book = {
  bookName: string;
  files: Array<string>;
};
export type zennType = {
  zennDirPath: string;
  fileNames: {
    articles: Array<string>,
    books: Array<Book>
  };
  setZennDirPath: (zennDirPath: string) => void;
};
```
- 🗂src/index.ts編集
src/background/index.tsで定義したいinitIpcMainをcreateWindow内で呼び出し、IPC通信を可能にします。
ただ、これだとメインプロセス側を編集した際に、毎回立ち上げ直さないと変更が反映されません。
皆さんどうしてるんでしょうか？
とりあえず突き進みます。
```diff ts:src/index.ts
import { app, BrowserWindow, session } from 'electron';
+ import { initIpcMain } from './background/index';
+ import { zennType } from './types/zennTypes';
// This allows TypeScript to pick up the magic constants that's auto-generated by Forge's Webpack
// plugin that tells the Electron app where to look for the Webpack-bundled app code (depending on
// whether you're running in development or production).
declare const MAIN_WINDOW_WEBPACK_ENTRY: string;
declare const MAIN_WINDOW_PRELOAD_WEBPACK_ENTRY: string;
+ declare global {
+  interface Window {
+     api: {
+     syncWithZenn: (zennDirPath: string) => Promise<zennType['fileNames']>;
+     };
+  }
+ }


// Handle creating/removing shortcuts on Windows when installing/uninstalling.
if (require('electron-squirrel-startup')) {
  app.quit();
}

const createWindow = (): void => {
  // Create the browser window.
  const mainWindow = new BrowserWindow({
    height: 800,
    width: 1200,
    webPreferences: {
      preload: MAIN_WINDOW_PRELOAD_WEBPACK_ENTRY,
    },
  });

  // and load the index.html of the app.
  mainWindow.loadURL(MAIN_WINDOW_WEBPACK_ENTRY);

  // Open the DevTools.
  mainWindow.webContents.openDevTools();
+ initIpcMain();
};
```

### Contextの作成
ファイルを実際に取得する処理は実装できたので、レンダラープロセス側から呼び出していきたいと思います。
ファイルのデータや状態はコンポーネント間で共有したいので、Contextを作成します。
ReactのContextはコンポーネント間でデータ・状態管理を共有できる仕組みみたいです。
ReactやVueを勉強していると、よく親コンポーネントとか子コンポーネントとか出てきますが、形見のように子や孫にデータを渡さなくても、一族みんなでデータを共有できるということですね。

では早速Contextファイルを作成します。
src/contexts/ZennContext.tsxファイルを作成して、状態管理を任せようと思います。

※ const [zennDirPath, setZennDirPath] = useState<zennType['zennDirPath']>('{Zennディレクトリのフルパス}');
の部分は自分のZennのディレクトリのフルパスを指定します。
```tsx:src/contexts/ZennContext.tsx
import { createContext, useContext, useState, useEffect } from 'react';
import { zennType } from '../types/zennTypes';

const ZennContentContext = createContext<zennType | undefined>(undefined);

export const useZennContentContext = () => {
  return useContext(ZennContentContext);
}

export const ZennContentProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [fileNames, setFileNames] = useState({ articles: [], books: [] });

  const [zennDirPath, setZennDirPath] = useState<zennType['zennDirPath']>('{Zennディレクトリのフルパス}');

  const syncWithZenn = async () => {
    try {
      const files = await window.api.syncWithZenn(zennDirPath);
      setFileNames(files);
    } catch (error) {
      alert('エラーが発生しました')
    }
  };

  useEffect(() => {
    syncWithZenn();
  }, []);

  return (
    <ZennContentContext.Provider value={{
      fileNames,
      zennDirPath,
      setZennDirPath
      }}>
      {children}
    </ZennContentContext.Provider>
  );
}
```
context.Provider以下で呼び出されるコンポーネントは全てuseContextを使ってデータを共有することができるみたい。
便利ですね。
全コンポーネントで共有したいので、App.tsxでHomeコンポーネントをZennContentProviderで囲みます。

```tsx:src/App.tsx
import Home from './components/Home';
import { ZennContentProvider } from './contexts/ZennContext';

const App: React.FC = () => {
  return (
    <ZennContentProvider>
      <Home/>
    </ZennContentProvider>
  )
}

export default App;
```

## ナビゲーションバーの作成
ナビゲーションバーを作成します。
UIに関してはほとんど以下をコピーしてます。
https://ui.mantine.dev/category/navbars#navbar-nested
components配下にnavigationフォルダを作成します。

```
src
└ components
  └ navigation
	  ├─ LinksGroup.tsx
	  └─ Navbar.tsx
``` 

- 🗂Navbar.tsxの作成
```tsx:src/components/Navbar.tsx
import { Navbar, ScrollArea, createStyles } from '@mantine/core';
import { IconNotes, IconBook } from '@tabler/icons-react';
import LinksGroup from './LinksGroup';
import { useZennContentContext } from '../../contexts/ZennContext';

const useStyles = createStyles((theme) => ({
  navbar: {
    backgroundColor: theme.colorScheme === 'dark' ? theme.colors.dark[6] : theme.white,
    paddingBottom: 0,
  },

  links: {
    marginLeft: `calc(${theme.spacing.md} * -1)`,
    marginRight: `calc(${theme.spacing.md} * -1)`,
  },

  linksInner: {
    paddingTop: theme.spacing.xl,
    paddingBottom: theme.spacing.xl,
  },
}));

const NavbarNested: React.FC = () =>  {
  const { fileNames } = useZennContentContext();
  const { classes } = useStyles();

  const articleLinks = fileNames.articles.map((article) => {
    return (
      {label: article}
    )
  })

  const booksMockData = fileNames.books.map((book) => {
    const bookLinks = book.files.map((file) => {
      return ({label: file})
    })
    return (
      {
        label: book.bookName,
        icon: IconBook,
        initiallyOpened: false,
        links: bookLinks
      }
    )
  })
  const mockdata = [
    {
      label: 'articles',
      icon: IconNotes,
      initiallyOpened: true,
      links: articleLinks
    },
    ...booksMockData
  ]

  const links = mockdata.map((item) => <LinksGroup {...item} key={item.label} />);

  return (
    <Navbar height={800} p="md" className={classes.navbar}>
      <Navbar.Section grow className={classes.links} component={ScrollArea}>
        <div className={classes.linksInner}>{links}</div>
      </Navbar.Section>
    </Navbar>
  );
}

export default NavbarNested;
```

- 🗂LinksGroup.tsxの作成
UI上で選択したファイル名は、ファイル内容を取得する際に使うので、selectedFileとしてContextにセットしてます。
また、リロードなど再度開いた時に前回開いていたファイルを表示したいので、ローカルストレージにも登録してます。
```tsx:src/components/LinksGroup.tsx
import { useState } from 'react';
import {
  Group,
  Box,
  Collapse,
  ThemeIcon,
  Text,
  UnstyledButton,
  createStyles,
  rem,
} from '@mantine/core';
import { TablerIconsProps, IconChevronLeft, IconChevronRight } from '@tabler/icons-react';
import { useZennContentContext } from '../../contexts/ZennContext';

const useStyles = createStyles((theme) => ({
  control: {
    fontWeight: 500,
    display: 'block',
    width: '100%',
    padding: `${theme.spacing.xs} ${theme.spacing.md}`,
    color: theme.colorScheme === 'dark' ? theme.colors.dark[0] : theme.black,
    fontSize: theme.fontSizes.sm,

    '&:hover': {
      backgroundColor: theme.colorScheme === 'dark' ? theme.colors.dark[7] : theme.colors.gray[0],
      color: theme.colorScheme === 'dark' ? theme.white : theme.black,
    },
  },

  link: {
    fontWeight: 500,
    display: 'block',
    textDecoration: 'none',
    padding: `${theme.spacing.xs} ${theme.spacing.md}`,
    paddingLeft: rem(31),
    marginLeft: rem(30),
    fontSize: theme.fontSizes.sm,
    color: theme.colorScheme === 'dark' ? theme.colors.dark[0] : theme.colors.gray[7],
    borderLeft: `${rem(1)} solid ${
      theme.colorScheme === 'dark' ? theme.colors.dark[4] : theme.colors.gray[3]
    }`,

    '&:hover': {
      backgroundColor: theme.colorScheme === 'dark' ? theme.colors.dark[7] : theme.colors.gray[0],
      color: theme.colorScheme === 'dark' ? theme.white : theme.black,
      cursor: 'pointer'
    },
  },
  selectedLink: {
    backgroundColor: theme.colorScheme === 'dark' ? theme.colors.dark[7] : theme.colors.gray[0],
    color: theme.fn.variant({ variant: 'light', color: theme.primaryColor }).color,
    cursor: 'pointer',
  },
  resetLink: {
    backgroundColor: 'transparent',
    color: theme.colorScheme === 'dark' ? theme.colors.gray[7] : theme.colors.gray[7],
    cursor: 'pointer',
  },
  chevron: {
    transition: 'transform 200ms ease',
  },
}));

type LinksGroupProps = {
  icon: React.FC<TablerIconsProps>;
  label: string;
  initiallyOpened?: boolean;
  links?: { label: string;}[];
};

const LinksGroup: React.FC<LinksGroupProps> = ({ icon: Icon, initiallyOpened, label, links }) => {
  const { classes, theme } = useStyles();
  const { selectedFile, setSelectedFile } = useZennContentContext();
  const hasLinks = Array.isArray(links);

  let isOpend = false;
  (hasLinks ? links : []).forEach((link) => {
    if (link.label === selectedFile.file && label === selectedFile.label) {
      isOpend = true;
    }
  })

  const [opened, setOpened] = useState(isOpend || initiallyOpened);

  const ChevronIcon = theme.dir === 'ltr' ? IconChevronRight : IconChevronLeft;

  const selectFile = (label: string, file: string) => {
    setSelectedFile({label, file});
    localStorage.setItem('selected_file', file)
    localStorage.setItem('selected_label', label)
  }

  const items = (hasLinks ? links : []).map((link) => (
    <Text
      className={`${classes.link} ${(link.label === selectedFile.file && label === selectedFile.label) ? classes.selectedLink : ''}`}
      key={link.label}
      onClick={() => {
        selectFile(label, link.label)
      }}
    >
      {link.label}
    </Text>
  ));

  return (
    <>
      <UnstyledButton onClick={() => setOpened((o) => !o)} className={classes.control}>
        <Group position="apart" spacing={0}>
          <Box sx={{ display: 'flex', alignItems: 'center' }}>
            <ThemeIcon variant="light" size={30}>
              <Icon size="1.1rem" />
            </ThemeIcon>
            <Box ml="md">{label}</Box>
          </Box>
          {hasLinks && (
            <ChevronIcon
              className={classes.chevron}
              size="1rem"
              stroke={1.5}
              style={{
                transform: opened ? `rotate(${theme.dir === 'rtl' ? -90 : 90}deg)` : 'none',
              }}
            />
          )}
        </Group>
      </UnstyledButton>
      {hasLinks ? <Collapse in={opened}>{items}</Collapse> : null}
    </>
  );
}

export default LinksGroup;
```
- 🗂zennTypes.tsの編集
```diff ts:src/types/zennTypes.ts
type Book = {
  bookName: string;
  files: Array<string>;
};
export type zennType = {
  zennDirPath: string;
  fileNames: {
    articles: Array<string>,
    books: Array<Book>
  };
+  selectedFile: {
+    label: string;
+    file: string;
+  };
+  setSelectedFile: (selected: { label: string, file: string }) => void;
  setZennDirPath: (zennDirPath: string) => void;
};
```
- 🗂ZennContext.tsxの編集
```diff tsx:src/contexts/ZennContext.tsx
...
  const [fileNames, setFileNames] = useState({ articles: [], books: [] });

+ const [selectedFile, setSelectedFile] = useState<zennType['selectedFile']>({
+    label: localStorage.getItem('selected_label') || '',
+    file: localStorage.getItem('selected_file') || '',
+  });

...
return (
    <ZennContentContext.Provider value={{
      fileNames,
      zennDirPath,
+     selectedFile,
      setZennDirPath,
+     setSelectedFile
      }}>
      {children}
    </ZennContentContext.Provider>
```

- 🗂Home.tsxの編集
```diff tsx:src/components/Home.tsx
 import MarkdownEditor from '../components/editor/MarkdownEditor';
+import NavbarNested from '../components/navigation/Navbar';
 
 const Home: React.FC = () => {
   return (
     <>
-      <MarkdownEditor/>
+      <div style={{display: 'flex'}}>
+        <div style={{width: '25%'}}>
+          <NavbarNested/>
+        </div>
+        <div style={{width: '75%'}}>
+          <MarkdownEditor/>
+        </div>
+      </div>
     </>
   )
 }
```
## Zennのディレクトリパスを登録するモーダルの作成
参照するディレクトリのフルパスを登録するモーダルを作成します。
アプリを開いた際にローカルストレージにzennのディレクトリのパスがなければモーダルを表示し、そのモーダルから登録できるようにします。

```
src
└ components
   └ modal
       └─ InitModal.tsx
```
	
- 🗂InitModal.tsxの作成
```tsx:src/components/modal/InitModal.tsx
import { Input, Button, Group, Space } from '@mantine/core';
import { useZennContentContext } from '../../contexts/ZennContext';

interface InitModalProps {
  closeModal: () => void;
}

const InitModal: React.FC<InitModalProps> = ({closeModal}) => {
  const { zennDirPath, syncWithZenn, setZennDirPath, setIsZennSynced } = useZennContentContext();
  const sync = async () => {
    localStorage.setItem('zenn_dir_path', zennDirPath);
    setZennDirPath(zennDirPath);
    syncWithZenn();
    closeModal();
  }
  const  notSync = () => {
    localStorage.setItem('zenn_dir_path', '');
    setZennDirPath('');
    setIsZennSynced(false);
    closeModal();
  }
  return (
    <>
      <Input.Wrapper
        id="input-demo"
        label="Zennのルートパスを入力"
      >
        <Input
          placeholder="/Users/zenn"
          value={zennDirPath}
          onChange={(event: React.ChangeEvent<HTMLInputElement>) => setZennDirPath(event.target.value)}
        />
      </Input.Wrapper>
      <Space h="xl" />
      <Group position="center" spacing="xl">
        <Button onClick={sync}>
          Zennと同期する
        </Button>
        <Button onClick={notSync} variant="outline">
          同期しないで始める
        </Button>
      </Group>
    </>
  )
}

export default InitModal;
```

- 🗂zennTypes.tsの編集
```diff ts:src/types/zennTypes.ts
  type Book = {
  bookName: string;
  files: Array<string>;
};
export type zennType = {
  zennDirPath: string;
  fileNames: {
    articles: Array<string>,
    books: Array<Book>
  };
  selectedFile: {
    label: string;
    file: string;
  };
+  isZennSynced: boolean;
  setSelectedFile: (selected: { label: string, file: string }) => void;
  setZennDirPath: (zennDirPath: string) => void;
+  syncWithZenn: () => Promise<void>;
+  setIsZennSynced: (value: boolean) => void;
};
```

- 🗂ZennContext.tsxの編集
```diff tsx:src/contexts/ZennContext.tsx
 ...
 
 export const ZennContentProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
   const [fileNames, setFileNames] = useState({ articles: [], books: [] });
-
-  const [zennDirPath, setZennDirPath] = useState<zennType['zennDirPath']>('{Zennディレクトリのフルパス}');
+  const [isZennSynced, setIsZennSynced] = useState<zennType['isZennSynced']>(localStorage.getItem('zenn_dir_path') ? true : false);
+  const [zennDirPath, setZennDirPath] = useState<zennType['zennDirPath']>(localStorage.getItem('zenn_dir_path') || '');
   const [selectedFile, setSelectedFile] = useState<zennType['selectedFile']>({
     label: localStorage.getItem('selected_label') || '',
     file: localStorage.getItem('selected_file') || '',
   });
 
   const syncWithZenn = async () => {
+    if (!zennDirPath) {
+      setIsZennSynced(false);
+      return;
+    }
+
     try {
       const files = await window.api.syncWithZenn(zennDirPath);
+      setIsZennSynced(true);
       setFileNames(files);
     } catch (error) {
       alert('エラーが発生しました')
+      setIsZennSynced(false);
     }
   };

...

       fileNames,
       zennDirPath,
       selectedFile,
+      isZennSynced,
       setZennDirPath,
-      setSelectedFile
+      setSelectedFile,
+      syncWithZenn,
+      setIsZennSynced
       }}>
       {children}
     </ZennContentContext.Provider>
```

- 🗂Home.tsxの編集
```diff tsx:src/components/Home.tsx
 import MarkdownEditor from '../components/editor/MarkdownEditor';
 import NavbarNested from '../components/navigation/Navbar';
+import { useZennContentContext } from '../contexts/ZennContext';
+import { Modal } from '@mantine/core';
+import { useDisclosure } from '@mantine/hooks';
+import InitModal from '../components/modal/InitModal';
+
+const Home = () => {
+  const { zennDirPath, isZennSynced } = useZennContentContext();
+  const ModalOpen = !zennDirPath;
+  const [opened, { close }] = useDisclosure(ModalOpen);
+  const editorWidth = isZennSynced ? {width: '75%'} : {width: '100%'}
 
-const Home: React.FC = () => {
   return (
     <>
+      <Modal opened={opened} onClose={close}>
+        <InitModal closeModal={close} />
+      </Modal>
       <div style={{display: 'flex'}}>
+        {isZennSynced &&
         <div style={{width: '25%'}}>
           <NavbarNested/>
         </div>
-        <div style={{width: '75%'}}>
+        }
+        <div style={editorWidth}>
           <MarkdownEditor/>
         </div>
       </div>
```
zennと同期したかどうかもcontextで状態管理してます。

## エディタとファイル内容の同期と画像のアップロード
以上までで新規に作成するファイルは全て作成したので、一気に実装していきます。
内容の説明は割愛しますが、編集したファイルだけ載せときます。
すみません、、

※src/index.tsでwebSecurity: falseとしてますが、画像プレビューの際にfile://がセキュリティエラーになってしまうため、追加してます。

```diff ts:src/index.ts
...
declare global {
  interface Window {
    api: {
      syncWithZenn: (zennDirPath: string) => Promise<zennType['fileNames']>;
+      getZennContent: (zennDirPath: string, label: string, file: string) => Promise<string>;
+      saveZennFile: (zennDirPath: string, label: string, file: string, content: string) => Promise<void>;
+      uploadImage: (zennDirPath: string, imagePath: string) => Promise<string>;
    };
  }
}

...

const createWindow = (): void => {
  // Create the browser window.
  const mainWindow = new BrowserWindow({
    height: 800,
    width: 1200,
    webPreferences: {
      preload: MAIN_WINDOW_PRELOAD_WEBPACK_ENTRY,
+     webSecurity: false
    },
  });

  // and load the index.html of the app.
  mainWindow.loadURL(MAIN_WINDOW_WEBPACK_ENTRY);

  // Open the DevTools.
  mainWindow.webContents.openDevTools();
  initIpcMain();
};
```

```diff ts:src/preload.ts
...
contextBridge.exposeInMainWorld(
  'api', {
    syncWithZenn: (zennDirPath: string) => ipcRenderer.invoke('sync-with-zenn', {zennDirPath}),
+   getZennContent: (zennDirPath: string, label: string, file: string) => ipcRenderer.invoke('get-zenn-content', {zennDirPath, label, file}),
+   saveZennFile: (zennDirPath: string, label: string, file: string, content: string) => ipcRenderer.invoke('save-zenn-file', {zennDirPath, label, file, content}),
+   uploadImage: (zennDirPath: string, imagePath: string) => ipcRenderer.invoke('upload-image', {zennDirPath, imagePath})
  }
)
```

```diff ts:src/background/index.ts
 import { ipcMain } from 'electron'
-import { syncWithZenn } from './zenn'
+import { getZennContent, saveZennFile, syncWithZenn, uploadImage } from './zenn'
 export function initIpcMain () {
   syncWithZenn(ipcMain)
+  getZennContent(ipcMain)
+  saveZennFile(ipcMain)
+  uploadImage(ipcMain)
 }
```

```ts:src/background/zenn.ts
// 追加
export function getZennContent (ipcMain: Electron.IpcMain) {
  ipcMain.handle('get-zenn-content', async (e, {zennDirPath, label, file}) => {
    let fileDirPath = '';
    if (label === 'articles') {
      fileDirPath = path.join(zennDirPath, 'articles');
    } else {
      fileDirPath = path.join(zennDirPath, 'books', label);
    }

    if (fs.existsSync(path.join(fileDirPath, file))) {
      return fs.readFileSync(path.join(fileDirPath, file), 'utf8').toString()
    }

    throw new Error('Failed to read file');
  })
}

export function saveZennFile(ipcMain: Electron.IpcMain) {
  ipcMain.handle('save-zenn-file', async (e, {zennDirPath, label, file, content }) => {
    let fileDirPath = '';
    if (label === 'articles') {
      fileDirPath = path.join(zennDirPath, 'articles');
    } else {
      fileDirPath = path.join(zennDirPath, 'books', label);
    }

    try {
      fs.writeFileSync(path.join(fileDirPath, file), content, 'utf8');
      return;
    } catch (error) {
      throw new Error('Failed to save file');
    }
  });
}

export function uploadImage(ipcMain: Electron.IpcMain) {
  ipcMain.handle('upload-image', async (e, {zennDirPath, imagePath }) => {
    const distPath = path.join(zennDirPath, 'images');
    const imageName = path.basename(imagePath);
    const uploadImagePath = path.join(distPath, imageName)

    try {
      if (!fs.existsSync(distPath)) {
        fs.mkdirSync(distPath);
      }

      fs.copyFileSync(imagePath, uploadImagePath)

      return imageName;
    } catch (error) {
      throw new Error(error);
    }
  });
}
```

```tsx:src/components/editor/MarkdownEditor.tsx
import { useMemo } from "react";
import SimpleMdeReact from "react-simplemde-editor";
import SimpleMDE from "easymde";
import { marked } from "marked";
import hljs from 'highlight.js';
import "easymde/dist/easymde.min.css";
import "highlight.js/styles/base16/bright.css";
import { Modal } from '@mantine/core';
import { useDisclosure } from '@mantine/hooks';
import { useZennContentContext } from '../../contexts/ZennContext';
import InitModal from '../modal/InitModal';

const MarkdownEditor = () => {
  const [opened, { open, close }] = useDisclosure(false);
  const modalOpen = () => {
    open();
  }
  const toolbar: SimpleMDE.Options["toolbar"] = [
    'bold',
    'italic',
    'quote',
    'unordered-list',
    'ordered-list',
    'link',
    'image',
    'strikethrough',
    'code',
    'table',
    'redo',
    'heading',
    'undo',
    'clean-block',
    'horizontal-rule',
    'preview',
    'side-by-side',
    'fullscreen',
    '|',
    {
      name: "settings",
      action: modalOpen,
      className: "fa fa-cog",
      title: "settings"
    },
    {
      name: "image",
      action: () => {
        const input = document.getElementById("imageFileInput");
        if (input) {
          input.click();
          input.onchange = async () => {
            const imgFile = (input as HTMLInputElement).files?.[0];
            if (imgFile) {
              imageUploadFunction(imgFile)
            }
          };
        }
      },
      className: "fa fa-upload",
      title: "Image Upload",
    },
  ];

  const { zennData, isZennSynced, setZennData } = useZennContentContext();
  const value = localStorage.getItem('smde_saved_value') || "";

  marked.setOptions({breaks : true});

  const previewRender = (value: string): string => {
    const renderer = new marked.Renderer();
    renderer.code = (code, codeInfo) => {
      const codeInfoSplit = codeInfo.split(':');
      const lang = codeInfoSplit[0]
      const fileName = codeInfoSplit[1];
      const langClass = hljs.getLanguage(lang) ? lang : 'plaintext';
      const highlightedCode = hljs.highlight(langClass, code).value;
      const codeBlockClass = fileName === undefined ? 'code-block-no-info' : 'code-block';

      let codeBlock = `<code class="hljs ${codeBlockClass} language-${langClass}">${highlightedCode}</code>`
      if (fileName !== undefined) {
        codeBlock = `<div class="code-info"><span>${fileName}</span></div>` + codeBlock
      }
      return `<pre>${codeBlock}</pre>`
    };

    renderer.image = (href, title, text) => {
      let newHref = href;
      if (!newHref.startsWith('http://') && !newHref.startsWith('https://')) {
        newHref = `file://${zennDirPath}${newHref}`;
      }
      return `<img style="margin: 1.5rem auto;display: table;max-width: 100%; height: auto;" src="${newHref}" alt="${text}" title="${title || text}">`;
    };

    return marked(value, { renderer });
  };

  const imageUploadFunction = async (image:File) => {
    if (!isZennSynced) {
      alert('アップロードできません');
      return;
    }

    const imageName = await window.api.uploadImage(zennDirPath, image.path);

    const newContent = localStorage.getItem('smde_saved_value') + '![](/images/'+ imageName +')';

    setZennData({...zennData, content: newContent});
    saveFile(newContent)
  }

  const mdeOptions: SimpleMDE.Options = useMemo(() => {
    const delay = 1000;

    return {
      breaks: true,
      width: 'auto',
      spellChecker: false,
      toolbar,
      uploadImage: true,
      previewRender,
      imageUploadFunction,
      autosave: {
        enabled: true,
        uniqueId: "saved_value",
        delay,
      },
    };
  }, [previewRender]);

  let saveTimeout: NodeJS.Timeout | undefined;
  const zennDirPath = localStorage.getItem('zenn_dir_path');
  const saveFile = async (newContent: string) => {
    if (saveTimeout) {
      clearTimeout(saveTimeout);
    }

    if (newContent !== zennData.content) {
      saveTimeout = setTimeout(async () => {
        try {
          await window.api.saveZennFile(zennDirPath, zennData.label, zennData.file, newContent);
        } catch (error) {
          alert('エラーが発生しました');
        }
      }, 2000);
    }
  }

  return (
    <>
      <input type="file" id="imageFileInput" style={{ display: "none" }} />
      <Modal opened={opened} onClose={close}>
        <InitModal closeModal={close}/>
      </Modal>
      <SimpleMdeReact
        id="simple-mde"
        onChange={isZennSynced ? saveFile : null}
        value={isZennSynced ? zennData.content : value}
        options={mdeOptions} />
    </>
  );
};

export default MarkdownEditor;
```

```tsx:src/contexts/ZennContext.tsx
import { createContext, useContext, useState, useEffect } from 'react';
import { zennType } from '../types/zennTypes';

const ZennContentContext = createContext<zennType | undefined>(undefined);

export const useZennContentContext = () => {
  return useContext(ZennContentContext);
}

export const ZennContentProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [fileNames, setFileNames] = useState({ articles: [], books: [] });
  const [isZennSynced, setIsZennSynced] = useState<zennType['isZennSynced']>(localStorage.getItem('zenn_dir_path') ? true : false);
  const [zennData, setZennData] = useState<zennType['zennData']>({
    content: '',
    label: '',
    file: '',
  });
  const [selectedFile, setSelectedFile] = useState<zennType['selectedFile']>({
    label: localStorage.getItem('selected_label') || '',
    file: localStorage.getItem('selected_file') || '',
  });

  const [zennDirPath, setZennDirPath] = useState<zennType['zennDirPath']>(localStorage.getItem('zenn_dir_path') || '');

  const syncWithZenn = async () => {
    if (!zennDirPath) {
      setIsZennSynced(false);
      return;
    }

    try {
      const files = await window.api.syncWithZenn(zennDirPath);
      setIsZennSynced(true);
      setFileNames(files);
    } catch (error) {
      alert('エラーが発生しました')
      setIsZennSynced(false);
    }
  };

  const getZennContent = async () => {
    try {
      const content = await window.api.getZennContent(zennDirPath, selectedFile.label, selectedFile.file);

      setZennData({
        content,
        label: selectedFile.label,
        file: selectedFile.file
      });
      localStorage.setItem('smde_saved_value', content);
    } catch (error) {
      alert('エラーが発生しました');
    }
  };

  useEffect(() => {
    syncWithZenn();
    if(selectedFile.label === '' || selectedFile.file === '' || !isZennSynced) return;

    getZennContent();
  }, [selectedFile]);

  return (
    <ZennContentContext.Provider value={{
      zennData,
      fileNames,
      isZennSynced,
      zennDirPath,
      selectedFile,
      setZennData,
      setZennDirPath,
      setSelectedFile,
      syncWithZenn,
      getZennContent,
      setIsZennSynced
      }}>
      {children}
    </ZennContentContext.Provider>
  );
}
```

```ts:src/types/zennTypes.ts
type Book = {
  bookName: string;
  files: Array<string>;
};
export type zennType = {
  zennData: {
    content: string;
    label: string;
    file: string;
  };
  isZennSynced: boolean;
  fileNames: {
    articles: Array<string>,
    books: Array<Book>
  };
  zennDirPath: string;
  selectedFile: {
    label: string;
    file: string;
  };
  setZennData: (zennData: {content: string, label: string, file: string}) => void;
  setZennDirPath: (zennDirPath: string) => void;
  setSelectedFile: (selected: { label: string, file: string }) => void;
  syncWithZenn: () => Promise<void>;
  getZennContent: () => Promise<void>;
  setIsZennSynced: (value: boolean) => void;
};
```

# おわり
まだまだバグもあるし、追加したい機能もありますが、学習目的としては非常に良かったと思ってます。
やっぱ何か作ってみると勉強になりますね。

最後まで読んでいただき、大変ありがとうございました🙇‍♂️
