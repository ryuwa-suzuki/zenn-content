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
こんな感じのものを完成形として目指します。

![](/images/ezgif.com-video-to-gif.gif)

このアプリを作ろうと思ったきっかけは、
- ZennのWebエディタにMarkdown記法を挿入するツールバーがないこと
- プレビューをしながら編集できないこと
- github連携すると画像のアップロードがめんどくさいこと
- ちょうどTypeScriptとReactを勉強しようと思ってたこと

です。
この課題を全て解決するアプリにしようと思ってます。

## 前提条件
- [ZennとのGitHubリポジトリ連携が完了していること](https://zenn.dev/zenn/articles/connect-to-github)

## 実装する機能
最低限、以下の機能を実装します。
- Zennとのファイル同期
- Zennファイルへの自動保存
- 画像のアップロード
- 画像のプレビュー
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

## エディタのカスタマイズ
### ツールバーのカスタマイズ
このアプリを作ろうと思ったキッカケの一つに、Markdown記法を挿入するツールバーが欲しいという気持ちがあったので、ツールバーのカスタマイズからいきます。



