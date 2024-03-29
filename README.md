# Qithub PortalHost

Docker 内のネットワークにあるコンテナの Web サービスポートを外部に簡易公開するだけのコンテナです。

## 用途

Docker で作成した Web サービスを外部に公開したい場合、このコンテナを同じ Docker ネットワークに参加させるだけで以下の設定が不要になります。

1. ホスト側にコンテナのポートを解放する（`-p 8888:80` などの）設定
2. 自宅のルーターをホストに向けるなどの設定

Docker ネットワーク内のクローズドな環境で完結できるため、意図的にホスト側（LAN 側）に解放しない限り、安全で簡単にコンテナの Web サービスの確認および公開ができます。

## 使い方

例えば、日本語形態素解析器 [`kagome`](https://qiita.com/KEINOS/items/8b5e3a251430db89de3f) の Web API サービス・コンテナを外部に簡易公開したい場合は以下のようになります。（リポジトリの `/sample` ディレクトリでお試しください）

### TL;DR

```bash
# サービス・コンテナの起動
docker-compose up -d
# 公開 URL の取得
url_srv=$(docker-compose logs portalhost | tail -1 | awk -F' ' '{print $NF}' | grep localhost.run)
# 公開 URL の確認
echo $url_srv
# 公開 URL で Kagome Web API を叩いてみる
curl -s -XPUT "${url_srv%$'\r'}/a" -d'{"sentence":"すもももももももものうち", "mode":"normal"}' | jq .
```

### TS;DR

`docker-compose.yml` の中身は以下になります。

```yaml
version: "3.7"
services:
  portalhost:
    container_name: portalhost
    build: .
    image: portalhost:local
    tty: true
    stdin_open: true
    init: true
    env_file: config.env
    depends_on:
      - kagome
  kagome:
    container_name: kagome
    image: ikawaha/kagome:latest
    command: server -http=":80"
```

```shellsession
$ # サービス・コンテナをバックグラウンドで起動
$ docker-compose up -d
...
Successfully built 9315b40db3b5
Successfully tagged portalhost:latest
WARNING: Image for service portalhost was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Creating kagome ... done
Creating portalhost ... done
```

```shellsession
$ # コンテナの起動を確認（Ports でポートがホスト側に公開されていないことに注目）
$ docker-compose ps
   Name              Command           State   Ports
----------------------------------------------------
kagome       kagome server -http=:80   Up
portalhost   /entrypoint.sh            Up
```

```shellsession
$ # 外部公開された URL を確認
$ docker-compose logs portalhost
Attaching to portalhost
portalhost    | ssh.localhost.run の接続を kagome:80 にポートフォワーディングしています。
portalhost    | Warning: Permanently added 'ssh.localhost.run,35.193.161.204' (RSA) to the list of known hosts.
portalhost    | Connect to http://qithub-tq63.localhost.run or https://qithub-tq63.localhost.run
$ # ---------------------------------------------------------------------------
$ # 上記の場合、https://qithub-tq63.localhost.run/ へのアクセスが、この Docker
$ # ネットワーク内の http://kagome:80/ へポートフォワーディングされます。
$ # ---------------------------------------------------------------------------
```

```shellsession
$ # ホストから kagome の API を叩いてみる
$ # https://qithub-tq63.localhost.run/a がエンドポイントです。
$ curl -s -XPUT https://qithub-tq63.localhost.run/a \
  -d'{"sentence":"すもももももももものうち", "mode":"normal"}' | jq .
```

以下のようなレスポンスが返ってきます。

```json
{
  "status": true,
  "tokens": [
    {
      "id": 36163,
      "start": 0,
      "end": 3,
      "surface": "すもも",
      "class": "KNOWN",
      "features": [
        "名詞",
        "一般",
        "*",
        "*",
        "*",
        "*",
        "すもも",
        "スモモ",
        "スモモ"
      ]
    },
    ...（中略）...
    {
      "id": 8027,
      "start": 10,
      "end": 12,
      "surface": "うち",
      "class": "KNOWN",
      "features": [
        "名詞",
        "非自立",
        "副詞可能",
        "*",
        "*",
        "*",
        "うち",
        "ウチ",
        "ウチ"
      ]
    }
  ]
}
```

## 仕組みの概要

基本的には、このコンテナの通信を SSH サーバーに [SSH ポートフォワーディング](https://www.google.com/search?q=site:qiita.com+ssh%E3%83%9D%E3%83%BC%E3%83%88%E3%83%95%E3%82%A9%E3%83%AF%E3%83%BC%E3%83%87%E3%82%A3%E3%83%B3%E3%82%B0)しているだけです。

この時、SSH サーバーに [localhost.run](https://localhost.run/) のサービスを利用しています。このサービスは、SSH でポートフォワーディングされた接続を HTTP（80 番ポート） もしくは SSL（443 番ポート）で外部に公開します。

## 参考文献

- [ngrok大好きな私がServeoで感動した話](https://qiita.com/sskmy1024y/items/8395c85d09931d7dea27) @ Qiita
- [taichunmin/docker-serveo](https://github.com/taichunmin/docker-serveo) @ GitHub
- [jacobtomlinson/docker-serveo](https://github.com/jacobtomlinson/docker-serveo) @ GitHub
