---
title: "electron ✖️React ✖️TypeScriptでZenn用マークダウンエディタを作ってみる"
emoji: "✏️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react, typescript, electron, javascript, マークダウン]
published: false

---

みなさんはじめまして。

普段はバックエンドエンジニアとして、ひっそりとphp等を書くことでご飯を食べております。
この度は流行りのZennに投稿するということで、大人気の**React**と**TypeScript**を学習する記事にすることにしました。

学習するにしてもTodoアプリなどではモチベーションを保てずゼルダやピクミンに時間を奪われがちなので、**俺の俺による俺のための**エディタを作るんだっていう気持ちで挑むことにします。


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
型を定義するファイルがあった方がいいなと思ったので作ったんですが、どうなんですかね？TypeScriptもほとんど触ったことないので、アドバイスいただけたら嬉しいですが。。
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
とりあえず突き進みますが、、
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

