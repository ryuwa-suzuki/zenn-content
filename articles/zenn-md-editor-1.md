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

## 前提条件
- [ZennとのGitHubリポジトリ連携が完了していること](https://zenn.dev/zenn/articles/connect-to-github)

## 実装する機能
最低限、以下の機能を実装します。
- Zennとのファイル同期
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

以下のような画面が立ち上がれば成功です。
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

# 