---
title: 社内ISUCON振り返り
date: 2018-12-18T23:00:14+09:00
draft: false
---

# 背景・目的

今年の4月に所属会社が主催するISUCONに参加しましたが、結果、エンジニアを名乗るのが恥ずかしいくらいボロクソで、それ以来「パソコン得意なお兄さん」を名乗っていました。そんなお兄さんにISUCON再挑戦の機会が訪れ、なんとかエンジニアとして恥ずかしくないスコアを出せたので、今後のために今回の取り組みをまとめます。

# パソコン得意なお兄さんの失敗

ポイントは、開発環境の整備など、その後の効率化に繋がるようなことをやらなかった点と、世の中にある便利なツールを活用しきれなかった点が大きいと考えています。

例：

- 計測方法が非効率：例えば、[kataribe](https://github.com/matsuu/kataribe) すら知りませんでした。わざわざPythonスクリプトを書いて集計しました。
- アプリケーションのインスペクション方法が非効率：例えば、[MySQL Workbench](https://www.mysql.com/jp/products/workbench/) や [SchemaSpy](http://schemaspy.org/) を使えば簡単にER図を出力できますが、最後までテキストベースのDDLしか見ませんでした。
- 「ローカル開発環境？そんなものはない。」： ローカルでソースを編集 → git push → サーバでgit pull → ビルド → 再起動 → 動かない…  最後はsshでログインして、vimでJavaのソースを編集してました。そしてcommitし忘れて、ローカルでもまた同じ修正をしたり…

ああ、恥ずかしい…


# 今回のISUCON振り返り

## 準備

### ローカル開発環境

高速に試行錯誤を重ねることができ、アプリケーションのスピードアップに関係のないことには極力頭を使わなくていい状態を目指しました。そのためには、以下のようなものがあったら良いなと考えて準備しました。

1. 本番と同じ設定でミドルウェア / アプリケーションが動作すること
2. 1アクションでアプリケーションをデプロイできること

1つ目に関しては、docker-composeでnginxやmysqlを動かせるようにしておいて、当日はサーバ上の設定ファイルをローカルに持ってくるだけ、という状態にしました。
※ もちろん、ディレクトリ構成の違いなど、環境依存の部分がどうしても存在するので、envsubstで置き換えられる状態にしておきました。

```yaml
version: '3'
services:
  nginx:
    image: nginx:1.15.7-alpine
    command: >
      /bin/sh -c '
        envsubst "$$(cat /etc/nginx/nginx.conf.template.vars)" < /etc/nginx/nginx.conf.template > /etc/nginx/nginx.conf \
        && cat /etc/nginx/nginx.conf \
        && exec nginx -g "daemon off;"
      '
    environment:
      NGINX_USER: nginx
      APP_SERVER: go:3000
    ports:
      - 8080:80
    volumes:
      - ./etc/nginx/nginx.conf.template:/etc/nginx/nginx.conf.template
      - ./etc/nginx/nginx.conf.template.vars:/etc/nginx/nginx.conf.template.vars
      - ./log/nginx:/var/log/nginx
...
```

2つ目に関しては、サーバ上でgit pullするだけでミドルウェア・アプリケーションが自動再起動するように、[goemon](https://github.com/mattn/goemon) の設定を用意しました。

```yaml
tasks:

- match: :RestartNginx
  commands:
    - echo 'TODO log rotate'
    - echo 'TODO restart nginx'

# templateに変更があったら設定を更新してnginxを再起動する。
- match: './etc/nginx/nginx.conf.template'
  commands:
    - env NGINX_USER=TODO APP_SERVER=TODO envsubst "$(cat ./etc/nginx/nginx.conf.template.vars)" < ./etc/nginx/nginx.conf.template > ./etc/nginx/nginx.conf # /etc/nginx/nginx.conf はこのファイルへのシンボリックリンク
    - head -50 ./etc/nginx/nginx.conf
    - :event :RestartNginx
...
```


### 各種ツールの使い方調査

簡単にですが、今回はこれらに関して事前に使い方を調べておきました。

- netdata: cpuやメモリの使用率など
- kataribe: nginxのレスポンス時間の集計
- mysql workbench: ER図の出力と、実行計画の確認
- net/http/pprof (golang): go言語のプロファイラ


## ISUCON中

WIP



## 今後に向けての改善ポイント

- キャッシュの扱い：今回、アプリケーションのメモリ上にデータのキャッシュを持たせてしまったが、これではスケールしない。
- サーバ構成：サーバを3台与えられたたが、1台だけしか使わなかった。もっと構成で工夫できたはず。


# まとめ

私はエンジニアです。どうぞよろしく。
