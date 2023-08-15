---
title: "electron ✖️React ✖️TypeScriptで俺の俺による俺のためのZenn用マークダウンエディタを作ってみる①"
emoji: "😘"
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

- 🗂renderer.tsでroot.tsxをインポート。このファイルはWebpackで自動的にロードされ、"renderer" contextで実行されます。レンダラーやメインプロセスについては後日まとめたいと思ってます。
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
├── components
│   ├── editor
│   │   └── MarkdownEditor.tsx
│   └── Home.tsx
└── App.tsx
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

[https://github.com/Ionaru/easy-markdown-editor#toolbar-icons](https://github.com/Ionaru/easy-markdown-editor#toolbar-icons)にデフォルトで用意されているツールバー一覧が見れるので、ここから好きなものを追加します。

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
- サイドバーの表示
  articlesとbooksのmdファイルを取得し、サイドバーに表示します。
	再度開いた時に前回編集していたページを表示したいので、編集中に中に選択しているファイル名やスラッグ名もローカルストレージに保存します。
	イメージはこんな感じ

![](/images/screen-shot5.png)

- マークダウンファイルの読み込みと保存
フルパスと選択したファイルさえわかれば、node.jsでファイルの書き込みや読み込みは可能なので、簡単かと思います。

- 画像のアップロード
画像ファイルはimagesフォルダに配置してとのことなので、アップロードされたファイルをそのままimagesフォルダにコピーして配置するのみとします。
- 画像のプレビュー/images/xxxと入力されたパスをプレビュー時にfile://フルパス/imagesに書き換えて表示したいと思います。
セキュリティ的に問題ありそうですが、自分で使うだけなので。

# Zennとの連携機能の実装
非常に長い記事になりそうなので、詳細はなるべく割愛しながら進めたいと思います。
## zennディレクトリのフルパスを登録
### mantineのインストール
モーダルやサイドバーを一から作るのは面倒なので、ReactのUIライブラリであるmantineを利用したいと思います。
とても使いやすかったので、オススメです！
https://mantine.dev/

```
$ npm install @mantine/core @mantine/hooks @emotion/react
```
### Contextの作成
ReactのContextはコンポーネント間でデータ・状態管理を共有できる仕組みみたいです。
ReactやVueを勉強していると、よく親コンポーネントとか子コンポーネントとか出てきますが、形見のように子や孫にデータを渡さなくても、一族みんなでデータを共有できるということですね。

では早速Contextファイルを作成します。





