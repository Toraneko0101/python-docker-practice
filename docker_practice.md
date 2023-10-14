# Dockerについて学習する

## 概要
```control

用語:
    Docker: 
        ・ホストのカーネルを利用し、プロセスやユーザを隔離することで、別のマシンが動いているように見せかけるもの
        ・仮想マシンではないので、軽量
        ・ミドルウェアのインストールや、各種環境設定をコード化して管理できる(IaC)
        ・同じ環境をすぐに作れる。配布も容易(サーバの各種設定やインストールなどをコードで共有できるので、)
        ・よって、環境構築等の無駄な時間をなくせる
        ・コンテナ内での、アプリのパッケージ化と、実行機能を提供する
        ・クラスタ構成が容易(コンテナ名を変えるだけ)
        ・コード化したものをCI/CDツールで毎日実行すれば、リリースサイクルも容易になる

    Dockerデーモン(dockerd):
        Docker APIリクエストを受け付け、イメージやコンテナなどのDockerオブジェクトを管理する
        他のデーモンとも通信を行う

    Dockerクライアント(docker)
        docker run等のコマンドを通じ、dockerdに命令を伝える
        複数のデーモンと通信可能

    Dockerデスクトップ
        コンテナ化したアプリケーションと、マイクロサービスを構築、共有可能
        dockerd, docker, Docker Compose, Docker Content Trust, Kubernetes, CredentialHelperが含まれる
    
    Dockerレジストリ
        Dockerイメージを保管する(GHCR等もこの一種)
        デフォルトでは、DockerHubのイメージを探す
        docker pull -> 設定されたレジストリからイメージを取得
        docker push -> イメージを指定したレジストリに送信

    Dockerイメージ
        ・Dockerコンテナを作成する命令が入っている読み込み専用のテンプレート
        ・他のイメージ(たとえばubuntu)をベースとして、カスタマイズしたイメージを利用する
        ・例) ubuntuイメージをベースにapacheをインストールし、その設定を加えたもの
        
        イメージを作成する場合
            Dockerfileを作成
            ※Dockerfileを書き換えた場合、イメージは再構築されるが、その時は変更されたレイヤのみを再生成するので、ほかの仮想化技術に比べて軽量になっている

    コンテナ
        ・Dockerイメージが実行状態となったインスタンス
        ・生成、開始、停止などを行いたい場合、Docker APIやCLIを用いる
        ・複数のネットワークへの接続やストレージ追加、現時点の状態をもとにした新たなイメージの生成が可能。
        ・デフォルトではコンテナ同士は分離される
        ・コンテナを削除すると永続的なストレージに保存されていないものは消失する

    docker runコマンドの例
        docker run -it ubuntu /bin/bash

    上記の説明
        0. docker run = docker pull + docker create + docker startのようなもの
        1. docker run ubuntu
            ubuntuイメージがlocalになければ設定察れているレジストリからイメージ取得=docker pull ubuntu
        2. Dockerは新しいコンテナを取得=docker create
        3. Dockerはコンテナに対し、読み書き可能なファイルシステムを最後のレイヤとして割り当てる。=実行中のコンテナは、ファイルの変更が可能
        4. Dockerはネットワークインターフェースを生成し、コンテナを接続する。コンテナに対するIPアドレスの割り当てもなされる
        5. Dockerはコンテナを起動し、/bin/bashを実行する。
            -i 標準入力を受け付ける
            -t 標準入出力となっている端末デバイスを指定
            上記のオプションがあるので対話的に行える
        6. exitで/bin/bashコマンドは終了する(停止状態であり、削除は行われない)
    
    なぜコンテナが隔離された作業空間を準備できるのか
        namespacesを用いているため


    
具体例:
    1. 作業環境を共有するために、開発段階で用いる
    2. テストも、テスト環境のDockerで行う
    3. バグがあったら開発環境に戻し、テスト環境に再デプロイする
    4. ユーザに修正版を配布する

仕組み:
    ・クライアントはdocker build等の命令を通して、デーモンにDockerコンテナの配布、構築、実行などを行わせる
    ・デーモンとクライアントは必ずしも同一システム上にある必要はない
    ・Dockerクライアントとデーモンの間の通信にはREST APIが用いられる

```
## Docker Desktopのインストールに際して
```
- WSL2が有効化されている
- 何らかのLinuxディストリビューションがsetupされている

windows11なら以下のコマンドを打ち込むだけでいい
    wsl --install
```

## Getting Started
```
docker run -dp 80:80 docker/getting-started

    -d : コンテナをバックグラウンドで実行
    -p 80:80 : ホストの80番ポートをコンテナの80番ポートに割り当てる
    docker/getting-started : 使用するdockerイメージ

-> 実行すると、Docker DesktopのDashBoardに情報が追加された
-> Desktop上から使用ポートを確認したり、停止が可能
```

## 用語補足
```
sandbox: 
    通常利用する領域から隔離され、保護された空間に構築された仮想環境

隔離をなぜ実現できるのか
    カーネルのnamespaceとcgroupの活用
    コンテナ: 単一のホスト上で実行されるプロセスの分離されたグループ

    cgroup:
        ・プロセスをグループ化し、リソース制御を行う
        ・CPU,メモリ,ディスクI/O,ネットワーク帯域幅などのリソースを制御し、各プロセスグループに割り当てる
        ・Dockerコンテナはcgroupsを使用して、各コンテナのリソース制御を管理する。これによりコンテナ間でのリソース競合を防ぐ
    Namespaces:
        ・プロセスやリソースの隔離を実現するためのLinuxカーネル機能
        ・異なるコンテナ間でプロセスやファイルシステム、ネットワークなどの名前空間を分離
        ・Dockerコンテナは異なる名前空間で実行されるので、各コンテナは独自のプロセスID,ファイルシステム、ホスト名などを持つ。よって互いの影響を受けずに実行可能
        ・例) PID Namespaces->プロセスIDを隔離する

    cgroups + Namespaces:
        Dockerは名前空間でプロセスグループを隔離し、cgroupsでプロセスグループ(コンテナ)にリソースを制御、割り当てている。
        それによってリソースの競合はおきず、独自の環境が保たれる

    

Docker利点まとめ
    local,仮想マシン上で実行可能
    クラウドにもデプロイ可能
    多くのOSで実行可能
    imageには、依存関係、設定ファイル、スクリプト、バイナリ、環境変数、メタデータ等が含まれる


```

## アプリケーションのコンテナ化

```
制作物
    Node.jsで動作するシンプルなTodoリスト

注意点
    ・Git Bashで動かす際は、\記号をエスケープする必要がある
    ・docker buildの実行はUnix環境なので、行末はrnではなく、nとする
```

```powershell
手順

1. package.jsonがあるディレクトリ内でDockerfileを作成

2. Dockerfileに以下のように記述

#Docker構文の場所を定義している
#最新の 1.x.x マイナー および パッチ・リリースが更新され続ける(docker/dockerfile:1.2の場合、1.3.xがリリースされると更新停止となる)
# syntax=docker/dockerfile:1
# ベースとなるDockerイメージを指定
FROM node:18-alpine
# それ以降の操作を指定したディレクトリで行う
WORKDIR /app
# コンテナへファイルをコピー(ADDの場合、tarの展開まで行う)
# COPY <コピー元> <Dockerイメージ内の展開先>
COPY . .
# OSのコマンドを実行する際に使用
# package.jsonとyarn.lockをもとにyarn installして状態を再現する
# yarnの場合、ソース管理はpackage.jsonとyarn.lockで行うのでnode_modulesが不要となる
# --productionをつけると、本番用のインストールが実行され、開発用パッケージを含めなくなる
RUN yarn install --production
# コンテナ起動時に実行するコマンド
CMD ["node", "src/index.js"]
# Dockerコンテナのポートを公開する
# 使いどころ:docker run -PでEXPOSEしているすべてのポートを公開するなど
EXPOSE 3000

補足.
    package.jsonについて
        ・yarn installでpackage.jsonファイルに記載された依存パッケージをインストール。手作業でversionを変えてyarn installを行うと環境が変わる。
        ・yarn add [パッケージ名]@[バージョン指定]で新しいパッケージを追加可能
        ・例) reactのバージョン^16.3.1をinstall -> yarn add 'react@^16.13.1'
        ・^は以上という意味を持つ -> 16.3.1以上のversionをinstall
        ・追加すると、package.jsonが更新される
        ・実際にインストールしたバージョンはyarn.lockで確認
        ・パッケージをさらに増やした場合、パッケージマネージャはパッケージ同士の依存関係を解消しながらinstallを行う。
        ・なお、npm installはyarn installとyarn addを1つのコマンドでこなしている
        ・しかし、yarnは高速なパッケージ追加やセキュリティ面に強みがある。
        ・uninstallの際、yarn removeであることに注意
        ・yarn add --devコマンドの場合、開発時に必要なパッケージとして追加される。--productionを指定すると、installされなくなる。

        つまり....
            package.jsonとyarn.lockがあれば、yarn installすることで、同じ状態を再現可能。
        
        依存関係を新しくしたい場合
            yarn upgrade(lockファイルのみ変化)
            yarn upgrade --latest(package.jsonも変化)

        自分のプログラムからinstallしたものを読み込みたい場合
            importを用いる
        
        scriptsについて
            コマンドを登録できる
            たとえば今回の場合、prettifyという記載があるので、yarn prettifyで実行可能

    CMDとENTRYPOINTについて
        CMD: コンテナ実行時のデフォルトを指定->Optionを指定すれば上書きされる
             今回の場合、デフォルトでnode src/index.jsが行われる
        ENTRYPOINT:コンテナ実行時に必ず実行される

3. コンテナイメージを構築する
$ docker build -t getting-started .

docker build : 
    Dockerfileを使い、新しいコンテナimageを構築
    Dockerfile内の構文が解釈される（上記補足参照）
-t:
    imageにタグをつける。今回はgetting-started
    今後はこのimage名でコンテナを実行可能

.:
    Dockerに対して、現在のディレクトリ内にあるDockerfileを探せと命令
    WORKDIRの影響を受けていることに注意

4. コンテナを起動する
$ docker run -dp 127.0.0.1:3000:3000 getting-started

docker run
    1. 指定されたimage(getting-started)に、書き込み可能なコンテナをcreateする
    2. 指定されたコマンドを使ってstartする
    補足1. 二回目以降は、docker startで再起動可能
    補足2. 全てのコンテナはdocker ps -aで確認可能
        (実行中のコンテナのみ表示したい場合は、aオプションを付けない)
-d
    コンテナをバックグラウンドで実行
-p
    ホストとコンテナ間でポートを関連付ける
    HOST:CONTAINERなので、今回はコンテナのポート3000番を、、ホスト上の127.0.0.1:3000へ割り当てた(公開した)
    ポート割り当てを行わなければ、ホスト上からアプリケーションへは接続できない

getting-started
    起動するimageをタグ名を用いて指定している

```

## アプリケーションの更新
```
1. ソースコードを更新する

2. イメージを更新し、これをbuildする
$ docker build -t getting-started .

3. 古いコンテナが3000番を使用しているので停止して削除する
    3-1. docker psでコンテナのIDを調べる
    3-2. docker stop <コンテナID> で停止
    3-3. docker rm <コンテナID>で削除
    3-4. Dockerダッシュボードで削除してもいい

4. docker runで更新したアプリを起動
$ docker run -dp 127.0.0.1:3000:3000 getting-started

問題点:
    todoリストに追加していたアイテムが消えてしまう
    -> 再構築を必要としないコードの編集方法
    -> 変更するたびに新しくコンテナを起動する方法については以下で学ぶ
```

## アプリケーションの共有
```
概要
    ・Dockerイメージを共有するにはDockerレジストリを使う
    ・GHCRやDockerHub等にリポジトリを作成しよう
    ⇒今回はDockerHubを用いる

リポジトリ作成
    1. DockerHubにサインイン
    2. Create Repository
    3. Repository-nameを設定
    4. create

imageを送信
    1. docker login -u <USERNAME>でログイン
    2. docker tag <local_image> <repository_tag>で新しい名前を付ける
    3. docker push <USERNAME>/getting-startedで新しい名前を付けたイメージをレジストリに送信
        ※tagnameは省略した場合latest(最新)となる
        ※docker-hubのリポジトリ名とイメージ名は同一なのか?
        ※つまりpushする際には、リモートとローカルのリポジトリ名が同一でないといけなそう。だからtagをつけるのか

新しいイメージを実行
    1. https://labs.play-with-docker.com/を用いる
    2. 新しいインスタンスを立ち上げる
    3. $docker run -dp 0.0.0.0:3000:3000 <USERNAME>/getting-started

    補足:
        ・0.0.0.0はホスト上のすべてのインターフェースでlistenする
        ・ つまり、同一ネットワーク上の別ホストからアクセス可能
        ・127.0.0.1は他ホストからはアクセス不可。コンテナのlocalhostと、別のコンテナのlocalhostは異なるので
        ・今回ホストはDockerコンテナなので、ローカルマシン（コンテナの外）からアクセスしたい場合、0.0.0.0の方がいい。
        ・なお、0.0.0.0が宛先となっていた場合、これは書き換えられる。

Next:
    再起動してもデータを保持できる方法について学ぶ
```

## データベースの保持
```
コンテナのファイルシステム
    ・コンテナはファイルの作成、更新、削除するための権限を持っている
    ・同じイメージを使っていたとしても、コンテナ内のファイルシステムに対する変更は、他のコンテナからは閲覧できない

実際に確認する
    1. ubuntuコンテナを起動し、/data.txtという名前のファイルを作成。1~10000までのランダムな数を入れる
    $ docker run -d ubuntu bash -c "shuf -i 1-10000 -n 1 -o /data.txt && tail -f /dev/null"
        ※bashシェルを開始する
        ※-c <string>: コマンドをstringから読み込み、bashで実行
        ※shuf:入力行をシャッフルする
        ※-i 1-10000: 範囲
        ※-n 1: 行数(1行)
        ※ -o /data.txt: 結果を/data.txtに書きだす
        ※tail -f : ファイルが変更されてもtailをし続ける
        ※tail -f /dev/null: 終了されないのでコンテナが起動し続ける
    
    2. コマンドラインからコンテナにアクセスする
    docker exec <CONTAINER ID> cat /data.txt

    3. 他のubuntuコンテナ(同じイメージ)を起動しても同じファイルは見えない
    $ docker run -it ubuntu ls /
    ⇒ファイルを書きだしたのは1つ目のコンテナのスクラッチ領域なので

コンテナのボリューム
    ・つまり、コンテナを削除したら、それらの変更は消失する
    ・そのためボリュームを使う
    ・ボリューム：コンテナ内で指定したファイルシステムのパスをホストマシン上へと接続できる機能を備えたもの
    ・コンテナ内にディレクトリをマウントすれば、ディレクトリに対する変更がホストマシンから閲覧できる。
    ・また同一ディレクトリをマウントすれば再起動後も同じファイルが見える

todoデータを保持するためには？
    ・今回の場合、デフォルトで/etc/todo/todb.dbに保存される
    ・ホスト上のボリュームにtodo.dbを置いておき、再起動した際に、データを保管するディレクトリに取り付ける（マウントする）

ボリュームの作成とコンテナの起動
    1. docker volume create <volume_name> でボリュームを作成

    2. todoアプリのコンテナを作り直す(現在は保存するボリュームを使わずに起動しているので)

    3. todoアプリのコンテナを起動する際に、ボリュームのマウントを指定する--mountオプションを追加する
    $ docker run -dp 127.0.0.1:3000:3000 --mount type=volume,src=todo-db,target=/etc/todos getting-started
        # --mount : マウントを行う
        # type: マウントのタイプ。今回はvolume
        # src: マウント元(ボリューム名のみの指定でOK)
        # target: マウント先

    4. 実際にコンテナを削除、起動し保存されていることを確認する
    -> OK

    ボリュームの格納位置に関する補足
        ・WSL2を用いている場合、注意が必要
        docker volume inspect todo-dbで格納位置を尋ねたところ、こう返答があった
        [
            {
                "CreatedAt": "2023-10-13T07:30:33Z",
                "Driver": "local",
                "Labels": null,
                "Mountpoint": "/var/lib/docker/volumes/todo-db/_data",
                "Name": "todo-db",
                "Options": null,
                "Scope": "local"
            }
        ]
        しかし、実際には格納先は以下の通りであった
        "\\wsl.localhost\docker-desktop-data\data\docker\volumes\todo-db\_data\todo.db"

Next
    現在は変更を加えるたび、再構築をしている
    これを改善する
```

## バインドマウントを使う
```
バインドマウント
    ・ホスト上のファイルシステムをコンテナ内と直接共有する
    ・バインドマウントを用いてソースコードをコンテナ内にマウントした場合、動作中でもアプリの変更が直ちに反映される
    ・ホスト上の場所は自分で決められる
    ・変更するたびにイメージの再構築をする必要はない

バインドマウントを試す
    1. ubuntuコンテナでbashを実行
    $ docker run -it --mount type=bind,src="$(pwd)",target=/src ubuntu bash
    2. pwd=getting-started/appなので、ホストのgetting-started/appとDockerの/srcが結びつく
    3. そのため、/srcの中にmyfile.txtをtouchすると、ホストのgetting-started/app配下に、myfile.txtが生成される。逆もしかり。
    -> ホストとコンテナ間で、ファイルを共有していることが確認できた。肝はtype=bind

コンテナのデプロイ
    ・バインドマウントの利点は、開発マシン上に全ての構築ツールをインストールする必要がないこと
    ・docker runを実行するだけで依存関係とツールを取得可能

    1. 以下のコマンドを実行
        $ docker run -dp 127.0.0.1:3000:3000 -w /app --mount type=bind,src="$(pwd)",target=/app node:18-alpine sh -c "yarn install & yarn run dev"
    説明:
        -w /app : コンテナ上の作業ディレクトリを指定
        --mount type=bind,src="$(pwd)",target=/app: ホスト上のcurrentを、コンテナ内の/appディレクトリにバインドマウント
        node:18-alpine: 使用image
        sh -c "string": コマンドをstringで読み込みshで実行
        yarn install && yarn run dev: パッケージinstall -> 開発用サーバを開始
    
    補足:
        package.jsonのscriptsには以下の記述がある
        "dev": "nodemon src/index.js"
        したがって、yarn run devで、devスクリプトはnodemonを起動する
    
    2. docker logs <container-id>でログを観察する
        $ docker logs -f <container-id>
        yarn install v1.22.19
        [1/4] Resolving packages...
        success Already up-to-date.
        Done in 0.32s.
        yarn run v1.22.19
        $ nodemon src/index.js
        [nodemon] 2.0.20
        [nodemon] to restart at any time, enter `rs`
        [nodemon] watching path(s): *.*
        [nodemon] watching extensions: js,mjs,json
        [nodemon] starting `node src/index.js`
        Using sqlite database at /etc/todos/todo.db
        Listening on port 3000
```

## ホストマシン上のアプリを更新し、コンテナに変更を反映する
```
1. sourceを更新
2. リロードするだけで変更が反映された

問題点:
    DBを指定していないので、またデータが消えている

Next:
    他のRDB(MySQL)を使えるようにする
    コンテナ間で通信を行わせる
```
![bind_mount](./images/bindmount.png)

## 複数コンテナのアプリ
```
方針
    1つのコンテナでは1つのことを実行する
理由
    ・バージョンを分離する
    ・マイクロサービスの方がいい
    ・複数のプロセスを実行する場合、コンテナは1つのプロセスのみを起動するので、プロセスマネージャが必要になり、コンテナの起動、停止が複雑になる

コンテナのネットワーク機能
    ・通常は各コンテナ間の情報を得られない
    ・通信を可能にするため、ネットワーク機能を使う

コンテナをネットワークに加えるには
    ・コンテナの起動時にネットワークを割り当てる
    ・既に実行しているコンテナをネットワークに接続する

MySQLの起動
    ・今回はネットワークの作成->起動時に接続の手順を踏む

    1.　ネットワークを作成する
        $ docker network create todo-app

    2. MySQLコンテナを起動し、ネットワークに接続する
        ・この時、環境変数も併せて定義する
        ・環境変数を設定するのは、データベースの初期化のため

        $ docker run -d \
            --network todo-app --network-alias mysql \
            -v todo-mysql-data:/val/lib/mysql \
            -e MYSQL_ROOT_PASSWORD=<PASSWORD> \
            -e MYSQL_DATABASE=todos \
            mysql:8.0

        補足:
            --network todo-app: todo-appネットワークに接続
                ※networkの一覧はdocker network lsで
            --network-alias <aliad>　別名で名前解決
            -v ボリュームの設定
            -v -v todo-mysql-data:/val/lib/mysql: ホストのtodo-mysql-dataボリュームを、/var/lib/mysqlにバインドマウント
                ※docker volume createをしていないが、名前付きボリュームを使う場合、Dockerが自動的にボリュームを作成してくれる
                ※ --mount type=bindとの相違点は、--mountが自動的にボリュームを作成しないところ。一般的には後者の方が安全面でよさそう
            -e 環境変数の設定
            mysql:8.0 mysqlのversion

    
    3. データベースに接続し、接続されているか確認する
        $ docker exec -it <CONTAINERID> mysql -u root -p

        mysql> show databases;
        +--------------------+
        | Database           |
        +--------------------+
        | information_schema |
        | mysql              |
        | performance_schema |
        | sys                |
        | todos              |
        +--------------------+  

        mysql> exit 


(MySQLへと接続)どのようにして他のコンテナはMySQLのコンテナを見つけるのか
    ポイント: コンテナは自身のIPアドレスを持つ

    1. nicolaka/netshootでdigコマンドを用い、DNS解決の一端を見る
        $ docker run -it --network todo-app nicolaka/netshoot
                                    dP            dP                      
            dP
                            88            88                      
            88
        88d888b. .d8888b. d8888P .d8888b. 88d888b. .d8888b. .d8888b. d8888P
        88'  `88 88ooood8   88   Y8ooooo. 88'  `88 88'  `88 88'  `88   88
        88    88 88.  ...   88         88 88    88 88.  .88 88.  .88   88
        dP    dP `88888P'   dP   `88888P' dP    dP `88888P' `88888P'   dP
                                                                

        Welcome to Netshoot! (github.com/nicolaka/netshoot)       
        Version: 0.11

        $ dig mysql
        ; <<>> DiG 9.18.13 <<>> mysql
        ;; global options: +cmd
        ;; Got answer:
        ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 40736 
        ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

        ;; QUESTION SECTION:
        ;mysql.                         IN      A

        ;; ANSWER SECTION:
        mysql.                  600     IN      A       <IP_ADDRESS>
        ;; Query time: 10 msec
        ;; SERVER: 127.0.0.11#53(127.0.0.11) (UDP)
        ;; WHEN: Sat Oct 14 07:35:09 UTC 2023
        ;; MSG SIZE  rcvd: 44

    2.  上記より、mysqlと打つだけで、名前解決が行われることがわかる

    補足：
        ・これが可能なのは、--network-aliasフラグを用いているため
        ・DockerはコンテナのIPアドレスをネットワークエイリアスで調べられる

MySQLとアプリを動かす
    ※環境変数を用いた接続設定は本番環境では推奨されない

    1. コンテナをアプリのネットワークに接続する(pwdを使うので、getting-started/appに居ること！)
        $ docker run -dp 127.0.0.1:3000:3000 \
            -w /app -v "$(pwd):/app" \
            --network todo-app \
            -e MYSQL_HOST=mysql \
            -e MYSQL_USER=root \
            -e MYSQL_PASSWORD=<PASSWORD> \
            -e MYSQL_DB=todos \
            node:18-alpine \
            sh -c "yarn install && yarn run dev"

        補足:
            MYSQL_HOST:MySQLサーバを実行中のホスト名
            MYSQL_USER: 接続に使うユーザ名
            MYSQL_PASSWORD: 接続に使うパスワード
            MYSQL_DB: 接続先として使うデータベース

    2. コンテナのlogにmysqlの使用を示すメッセージが現れたのを確認する
        yarn install v1.22.19
        [1/4] Resolving packages...
        success Already up-to-date.
        Done in 0.38s.
        yarn run v1.22.19
        $ nodemon src/index.js
        [nodemon] 2.0.20
        [nodemon] to restart at any time, enter `rs`
        [nodemon] watching path(s): *.*
        [nodemon] watching extensions: js,mjs,json
        [nodemon] starting `node src/index.js`
        Waiting for mysql:3306.
        Connected!
        Connected to mysql db at host mysql
        Listening on port 3000
    
    3. todoリストにいくつかアイテムを追加する

    4. mysqlデータベースで確認する
        docker exec -it <MYSQL_CONTAINER_ID> mysql -p todos

        Reading table information for completion of table and column names
        You can turn off this feature to get a quicker startup with -A

        Welcome to the MySQL monitor.  Commands end with ; or \g.
        Your MySQL connection id is 11
        Server version: 8.0.34 MySQL Community Server - GPL

        Copyright (c) 2000, 2023, Oracle and/or its affiliates.

        Oracle is a registered trademark of Oracle Corporation and/or its
        affiliates. Other names may be trademarks of their respective
        owners.

        Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

        mysql>
    
    4. mysqlシェルで確認
        mysql> select * from todo_items;
        +--------------------------------------+------------------------+-----------+
        | id                                   | name                   | completed |
        +--------------------------------------+------------------------+-----------+
        | b89b9c06-443d-4b3a-8558-c5a924f89c2b | ??????????5????        |         0 |
        | 65b91af8-7328-4875-8aa8-328b33a42291 | docker?image?????????? |         0 |
        | 58448611-f365-4f15-83ab-06866e181202 | ??????????????         |         0 |
        | 20ba828a-c7df-403f-8175-e2e249a15c91 | only english           |         0 |
        +--------------------------------------+------------------------+-----------+

        問題点: 日本語が表示されない?
            ・my.cnfやdocker-compose.ymlを修正した方がよさそう？
        場当たり的な解決策:
            SET character_set_results=utf8mb4

        mysql> select * from todo_items;
        +--------------------------------------+----------------------------------------------+-----------+
        | id                                   | name                                         | completed |
        +--------------------------------------+----------------------------------------------+-----------+
        | b89b9c06-443d-4b3a-8558-c5a924f89c2b | 昼ご飯を買いに行く（5時まで）  |         0 |
        | 65b91af8-7328-4875-8aa8-328b33a42291 | dockerのimageの制限について調べる |         0 |
        | 58448611-f365-4f15-83ab-06866e181202 | とっととゲーム開発に移行する   |         0 |
        | 20ba828a-c7df-403f-8175-e2e249a15c91 | only english                                 |         0 |
        +--------------------------------------+----------------------------------------------+-----------+

        問題点:
            ・表示が汚い
        場当たり的な解決策：
            以下の通り　-> 横並びの方が見やすいので何とかしたい

        mysql> select * from todo_items\G
        *************************** 1. row ***************************
            id: b89b9c06-443d-4b3a-8558-c5a924f89c2b
            name: 昼ご飯を買いに行く（5時まで）
        completed: 0
        *************************** 2. row ***************************
            id: 65b91af8-7328-4875-8aa8-328b33a42291
            name: dockerのimageの制限について調べる
        completed: 0
        *************************** 3. row ***************************
            id: 58448611-f365-4f15-83ab-06866e181202
            name: とっととゲーム開発に移行する
        completed: 0
        *************************** 4. row ***************************
            id: 20ba828a-c7df-403f-8175-e2e249a15c91
            name: only english
        completed: 0
    
    5. 保管されているのが確認できればOK

        問題点：
            ・todosデータベースの中にtodo_itemsというテーブルができていた
            ・作った覚えがないのだが、デフォルトなのだろうか？
    
Next
    ・Docker Composeを使って、より簡単な方法でアプリを立ち上げる
    ・ネットワークの構成,コンテナ起動,環境変数指定,ポート公開などを簡単に済ませる
```

## 環境変数を用いた開発設定が推奨されない理由
```
    ・環境変数は子プロセスに渡されるので、意図しないアクセスが可能となる
    ・誤って公開される可能性が高くなる
```

## Docker Composeを使う
```
Docker Compose
    ・複数コンテナのアプリケーションを定義、共有するために開発されたツール
    ・サービスを定義するYAMLファイルを作成->コマンド実行で立ち上げ、削除可能

Composeの利点
    ・アプリケーションスタックをファイルに定義し、リポジトリの一番上における
    ・つまり、リポジトリのClone->起動->貢献の流れが容易になる

Composeのinstall
    ・Docker Desktooをinstall済みなら既に入っている
    ・version確認
        $ docker compose version

```

Composeファイルの作成
```
1. getting-started/appでdocker-compose.ymlを作成
```

```
2. サービスを定義
```
```yml
#実行したいサービス(コンテナ)の一覧を定義する
#名前は自動的にネットワークエイリアス(--network-alias)となる

services:
  app:
    image: node:18-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 127.0.0.1:3000:3000
    working_dir: /app
    # source:target:mode
    # 長い形式ならtype等も指定可能
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos
  
  mysql:
    image: mysql:8.0
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos

#Composeは自動的に名前付きボリュームを作成しないので
#volumesセクションでボリュームを定義する
#ボリューム名のみを指定するとデフォルトのオプションが使われる
volumes:
  todo-mysql-data:
```
```
3. アプリケーションスタックを起動
    $ docker compose up -d

    Creating network "app_default" with the default driver
    Creating volume "app_todo-mysql-data" with default driver
    Creating app_app_1   ... done
    Creating app_mysql_1 ... done

4. docker compose logs -fでサービスのログを相互に閲覧
    ※fオプションで追跡し続ける
    ※特定のサービスのみ追跡したい場合は-fの後ろで指定
    app-mysql-1  | 2023-10-14 10:01:06+00:00 [Note] [Entrypoint]: MySQL init process done. Ready for start up.
    app-app-1    | Listening on port 3000

5. docker-desktopのダッシュボードでスタックを確認

6. docker-compose-upで削除する

Next:
    イメージ構築のベストプラクティスを学ぶ


```

## イメージ構築のベストプラクティス
```
docker image history <image>
    イメージ内の各レイヤが作成時に使われたコマンドを表示可能
    最新のレイヤが一番上にある


依存関係のキャッシュをサポートさせる(Dockerfileで)
    理由：
        imageを変更するたびにyarnの依存関係がインストールされるのは非効率なので
    手段：
        1. Nodeを用いたアプリの依存関係がpackage.jsonで定義されていることを思い出す
        2. package.jsonをcopyし、依存関係をインストール。
        3. そののちに、他の必要なすべてをコピーする
        4. これにより、package.jsonを変更したときだけ、yarnの依存関係が再作成されるようになる

        # syntax=docker/dockerfile:1
        FROM node:18-alpine
        WORKDIR /app
        COPY package.json yarn.lock ./
        RUN yarn install --production
        COPY . .
        CMD ["node", "src/index.js"]

        5. .dockerignoreを作成し、node_modulesと記載
        6. これによりnode_modulesはRUNステップ中の命令で作成されるファイルの影響を受けなくなる
        7. docker build -t getting-started . でbuild
        8. staticファイルを修正
        9. 再度、docker build -t getting-started .
        10. cacheが使われ、早くなったことが確認できる(1.7sくらいになった)
    
```

## マルチステージビルド
```
利点
    ・構築時の依存関係と、実行時の依存関係を分離できる
    ・必要なものだけコンテナに送るので、imageの容量を削減可能

Javaの例
    ・構築時はJDK,Gradleなどが必要
    ・最終的なイメージには（コンパイル後は）要らない
    ・そのため以下のようにする

    #syntax=docker/dockerfile:1
    FROM maven AS build
    WORKDIR /app
    COPY . .
    RUN mvn package

    FROM tomcat
    COPY --from=build /app/target/file.war /usr/local/tomcat/webapps

    ・上では、buildでJavaの構築をMavenで行っている
    ・その後、FROM tomcat以後で、buildステージからファイルをコピーし、最終的なイメージには最後のステージに作成されたものだけを利用する。

Reactの例
    開発環境でNodeを用い、
    本番環境ではnginxを使う

    # syntax=docker/dockerfile:1
    FROM node:18 AS build
    WORKDIR /app
    COPY package* yarn.lock ./
    RUN yarn install
    COPY public ./public
    COPY src ./src
    RUN yarn run build

    FROM nginx:alpine
    COPY --from=build /app/build /usr/share/nginx/html

まとめ
    ・共通しているのはコンパイルした出力を、サーバのコンテナにコピーしているということ
    ・マルチステージビルドで、イメージ全体の容量を減らせる
```

## その他
```
コンテナオーケストレーション
    
```

# 便利コマンド
```
docker image prune
    dangling状態のimageのみ削除する

docker rm -f <CONTAINER ID>
    コンテナを削除
```