---
title: "Docker Compose Watchを触ってみる"
emoji: "👀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Docker]
published: true
---

# はじめに
こんにちは、おそらく犬です。

今回はつい先日リリースされたDocker Compose Version 2.22以降で利用可能になったDocker Compose Watchを触ってみようと思います。

# 対象読者
- Dockerについてある程度知識がある人
- Docker Compose Watchについて気になっている人

# 前置き: コンテナ開発環境
フロントエンド開発等ではソースコードに対して行った変更をフレームワーク側で監視し、変更が保存されるとその差分部分をリビルドしたりしするホットリロードの機能があります。バックエンド開発等でもNode.jsで言えばnodemonも変更を監視し、変更が保存されると再起動してくれたりしますし、そういった機能は主要なフレームワークには搭載されていると思います。

Dockerコンテナとして動作するアプリケーションの場合、開発中にローカルで行った変更をコンテナ側に反映する必要があります。これは今まではDockerのバインドマウントを使って実現されていたことが多いように感じます。

例えば今までのdocker-compose.ymlはこんな感じでした。(現在はcompose.ymlにするのが推奨です)

```docker-compose.yaml
services:
  app:
    build:
      context: .
    container_name: app
    ports:
      - 3000:3000
    command: ['sh', '-c', 'npm run dev']
    volumes:
      - ./src:/home/node/app/src
      - ./public:/home/node/app/public
```

バインドマウントで`./src`以下と`./public`以下を`/home/node/app`に同期するというものです。Node.jsはちょっと特殊ですが、他の言語では大抵リポジトリのディレクトリを丸ごとバインドマウントしてしまうこともあります。

ところが、バインドマウントによる開発環境構築には2つの弱点があります。

1. 本番環境と違うDockerfileを作成し、バインドマウントする必要がある
2. Node.jsなどの同じディレクトリ配下にランタイムがある場合、Volume Trickと呼ばれるモノをしなければならない

1番の弱点ですが、これは出来れば開発環境だろうが本番環境だろうが同じDockerfileを使って一貫性を担保したいという気持ちに反する行為です。
もちろんバインドマウントを使わずDockerfile内でCOPYしてもいいんですが、開発時にソースコードの変更のたびにビルドし直すのは面倒です。

2番の弱点はNode.jsなどが抱える問題で、`./`をバインドマウントしてしまうと`node_modules`などのランタイムも同期されてしまいます。
それに対して以下のようなVolume Trickという対処をしないといけないのですが、これも完璧ではなく`package.json`を更新した時、手動でボリュームを削除しないといけなかったりでとにかく大変です。
根本の原因はバインドマウントがコピーではなく、本当に文字通りホスト-コンテナ間のバインドを行うこと、そしてそのバインドに対しての除外を行えるのがボリュームしかないがそのボリュームも以前から残っているものがある場合には同様にマウントされてしまう(=コンテナ内のファイルが無視されて上書きされてしまう)ことです。。
詳しくはVolume Trickという言葉で検索してみてください。こちら様の記事などが詳しいです。

https://zenn.dev/yumemi_inc/articles/3d327557af3554

特に2番が非常に厄介なのですが、これに対する解決策が今回の**Docker Compose Watch**になります。

devcontainerとか使えば解決するのかもしれないですが、そっちは詳しくないです、ｽﾐﾏｾﾝ。

# Docker Compose Watch
Docker Compose Watchは一言で言ってしまえばDockerコンテナのホットリロードです。公式の説明は以下です。

> Use watch to automatically update and preview your running Compose services as you edit and save your code.
> For many projects, this allows for a hands-off development workflow once Compose is running, as services automatically update themselves when you save your work.

https://docs.docker.com/compose/file-watch/

要約すると、コードを監視して変更を検知するとComposeで動いてるコンテナ(or Dockerイメージ)を更新するのでむっちゃ楽だよという感じです。(間違ってたらｽﾐﾏｾﾝ)

## 使い方
後で述べる設定をdocker-compose.ymlにした後、`docker compose watch`

## 2つの機能
Docker Compose Watchには以下2つの機能があります。

- Sync
- Rebuild

### Sync
監視対象のディレクトリやファイルを指定し、それらが更新されるとコンテナ内の指定のディレクトリと同期を取る機能です。これにより、一度Dockerイメージをビルドした後でもリビルドすることなくホスト上でのコードの変更をコンテナ内に反映させることが出来ます。

```compose.yml
services:
  web:
    build: .
    command: npm start
    develop:
      watch:
        - action: sync # <- ここ
          path: ./web 
          target: /src/web
          ignore:
            - node_modules/
        - action: rebuild
          path: package.json
```

この例ではホストの`./web`に変更があった時、その変更内容をコンテナ内の`/src/web`に同期するということをします。(あくまでホスト -> コンテナ内の同期で、逆はされません。)

### Rebuild
監視対象のディレクトリやファイルを指定し、それらが更新されるとDockerイメージからリビルドし、コンテナを新しくする機能です。これにより、例えば依存関係(package.json)の更新などをホスト側で行ったときに自動でリビルドしてくれます。

```compose.yml
services:
  web:
    build: .
    command: npm start
    develop:
      watch:
        - action: sync
          path: ./web
          target: /src/web
          ignore:
            - node_modules/
        - action: rebuild # <- ここ
          path: package.json
```

# 実演
自分のポートフォリオのリポジトリで試してみました。

compose.ymlは以下のように設定しました。Next.jsの動作するappコンテナ以外にStorybook,ngrokの2コンテナが動作します。

```compose.yml
services:
  app:
    build:
      context: .
    container_name: app
    ports:
      - 3000:3000
    command: ['sh', '-c', 'npm run dev']
    develop:
      watch:
        - action: sync
          path: ./
          target: /home/node/app/
          ignore:
            - node_modules/
        - action: rebuild
          path: package.json
  storybook:
    build:
      context: .
    container_name: storybook
    ports:
      - 6006:6006
    command: ['sh', '-c', 'npm run storybook']
    develop:
      watch:
        - action: sync
          path: ./
          target: /home/node/app/
          ignore:
            - node_modules/
        - action: rebuild
          path: package.json
  ngrok:
    container_name: ngrok
    image: ngrok/ngrok:latest
    restart: unless-stopped
    command: ['http', 'app:3000']
    ports:
      - 4040:4040
    depends_on:
      - app
    environment:
      - NGROK_AUTHTOKEN
```

`docker compose watch`で起動します。

ターミナルに`watching [hogehoge]`と表示されます。

```
$ docker compose watch
[+] Building 0.0s (0/0)                                                                                                                                                                    docker:default
[+] Running 3/0
 ✔ Container storybook  Running                                                                                                                                                                      0.0s 
 ✔ Container app        Running                                                                                                                                                                      0.0s 
 ✔ Container ngrok      Running                                                                                                                                                                      0.0s 
watching [/home/maybe_dog/dev/maybe-dog-portfolio /home/maybe_dog/dev/maybe-dog-portfolio/package.json]
watching [/home/maybe_dog/dev/maybe-dog-portfolio /home/maybe_dog/dev/maybe-dog-portfolio/package.json]
```

適当にファイルを変更してみます。すると以下のように変更が検知され、コンテナ内へと反映されます。

![](/images/bfeeee3b6650a1/2023-10-10-22-36-30.png)

```
Syncing app after changes were detected:
  - /home/maybe_dog/dev/maybe-dog-portfolio/src/components/layouts/Header.tsx
Syncing storybook after changes were detected:
  - /home/maybe_dog/dev/maybe-dog-portfolio/src/components/layouts/Header.tsx
```

コンテナ内に反映された結果、Next.jsの機能でホットリロードされています。

![](/images/bfeeee3b6650a1/2023-10-10-22-37-21.png)

次は依存関係を更新してみます。試しに適当な依存を更新してみます。

```
Rebuilding app after changes were detected:
  - /home/maybe_dog/dev/maybe-dog-portfolio/package.json
Rebuilding storybook after changes were detected:
  - /home/maybe_dog/dev/maybe-dog-portfolio/package.json
[+] Building 0.0s (0/0)                                                                                                                                                                    docker:default
[+] Building 0.1s (2/2)                                                                                                                                                                    docker:default
 => [storybook internal] load build definition from Dockerfile                                                                                                                                       0.1s
 => => transferring dockerfile: 169B                                                                                                                                                                 0.0s
 => [storybook internal] load .dockerignore 
```

自動的にリビルドが走りました🎉

# まとめ
Docker版ホットリロードのようなDocker Compose Watchを紹介しました。
開発時のバインドマウントの弊害に頭を悩ませていた自分にはとって非常に助かる新機能でした。もうこれなしには多分compose.ymlを書けません。

分かりにくいところ等あったらコメント頂けると嬉しいです！
