---
title: "wsl上のdockerコンテナ上で動作するnextjsアプリケーションにngrokを使う"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [wsl, docker, nextjs, ngrok]
published: true
---

# はじめに

nodeは比較的環境構築が容易だと思いますが、WSL上に開発用のDockerコンテナを立てて開発する場面があるかと思います。
その時、WindowsのブラウザからはWSLのフォワーディングを利用して動作を確認できます。
しかし、スマホから動作を確認したい場合はWindowsのポートを開け、それをWSLのポートへと繋ぎ…といった作業が発生し煩雑です。
そこで今回はngrokを利用してどこからでもnextjsアプリケーションをスマホから動作確認したいと思います。

# 対象読者
- Windows利用者でWSL内のdockerコンテナ上で動作するアプリケーションを外部から動作確認したい人
- Dockerに関する知識がある程度ある人

# 方法
ngrokが提供するdockerイメージを使ってngrok用のコンテナを立てます。そのコンテナからアプリケーション(今回はnextjs)へアクセスさせ、ngrokのエージェント経由でアプリケーションにアクセスします。

# 手順
## 前提条件
ngrokでアカウントを作成し、Authtokenを取得しておきます。
そしてWSL上で環境変数`NGROK_AUTHTOKEN`にそれをセットしておきます。

## docker-compose
アプリケーションのDockerfile, docker-compose.ymlはすでに作成している前提で書きます。
`docker-compose.yml`に以下を追記します。

```docker-compose.yml
  ngrok:
    container_name: ngrok
    image: ngrok/ngrok:latest
    restart: unless-stopped
    command: ["http", "app:3000"]
    ports:
      - 4040:4040
    depends_on:
      - app
    environment:
      - NGROK_AUTHTOKEN
```

nextjsのアプリケーション用のコンテナのサービス名`app`としています。適宜読み替えてください。

## 実際にアクセスする
`docker-compose up -d`した後、[localhost:4040](http://localhost:4040)に接続するとこんな画面になるので表示されるURLに接続します。

![](/images/9ccc9fe2fa91d5/2023-06-13-22-46-16.png)

## スマホでアクセスする
Chromeだと飛んだページで右クリックを押して、`このページのQRコードを作成`を押すとQRコードが作れます。
URLを手入力するのも大変なのでカメラで読みましょう。

# まとめ
ngrokを用いてスマホから簡単にWSLのdockerコンテナ上で動作するWebアプリケーションを確認する方法を紹介しました。
似たような言葉で探すと`wernight/ngrok`(ngrokのバージョンが**2**)のイメージを利用した方法もあるにはあるんですが、イメージの更新が2021年で止まってたので公式が配ってるイメージを利用した方法でやりました。簡単にスマホでアプリケーションを確認できるので便利です。
