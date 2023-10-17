# PythonとDockerについて学習する

## 学習元
    https://docs.docker.jp/language/python/index.html

## 学習内容
- DockerfileでPython-appのイメージ構築
- イメージをコンテナとして実行
- ボリュームとコンテナのセットアップ
- Composeを用いてコンテナをおーけストレート
- コンテナを用いたアプリケーション開発
- Github Actionsを用いてCI/CDパイプラインの形成
- アプリケーションをクラウドにデプロイ

## Pythonイメージの構築

### BuildKit有効化
- Docker-Desktopユーザであるため省略

### サンプルアプリケーションの構築
```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, Docker!'
```

### アプリケーションのテスト
```control
python -m flask run
```

### Python用のDockerfileを作成


Dockerfileのファイル名について  
- Dockerfile.hogeやhoge.Dockerfileにすることも可能だが、その場合はfオプションが必要  
- docker build -f Dockerfile.hoge  
- プロジェクトで主となるDockerfileはデフォルトが推奨される  


学んだこと
- syntaxはあらゆる記述より前に書く(コメントよりも)
- docker/dockerfile:1とすればBuildKitが自動的に構文を確認し、直近の現行バージョンを使えるようにする
- Dockerイメージはベースイメージを継承するので、公式のものを使えばいい

Dockerイメージの選び方について
- versionだけのものは公式
- Alpineはpythonと一緒に使うと速度が遅くなる可能性がある
- Debianのv10がbuster,v11がbullseye
- Slimがついていると使用頻度の低いパッケージを内包していない
- distrolessについて後ほど検討する

### 現状のDockerfile
```dockerfile
#syntax=docker/dockerfile:1

# hostのpython-docker/で実行する
# ubuntuで実行することを前提で書いている。cmdの場合は以下
# pip3 -> pip, python3 -> python

FROM python:3.8-slim-buster
WORKDIR /app
#currentのrequirements.txtをDockerの/app/requirements.txtにcopy
# COPY <src> <dest>
COPY requirements.txt requirements.txt
#pip install
RUN pip3 install -r requirements.txt
#sourceコードをimageの中に追加
COPY . .
#image実行時に実行するコマンド。(appをコンテナ外から見たいので0.0.0.0)
CMD ["python3", "-m", "flask", "run", "--host:0.0.0.0"]
```

### 現状のディレクトリ構成
```control
$ tree -I ".pyc|__pycache__"
.
├── Dockerfile
├── app.py
└── requirements.txt
```

### イメージ構築

docker buildでイメージ構築
- `docker build --tag python-docker .`
- Dockerfileと指定したパスに置かれているファイル群からイメージを構築する

### イメージにタグ付け

タグの作成
- `$ docker tag python-docker:latest python-docker:v1.0.0`
- 同じイメージを別の方法で参照しているだけ

タグの消去
- `$ docker rmi python-docker:v1.0.0`
- rmiコマンド=イメージ削除

## コンテナとしてイメージを実行

### 概要
イメージを実行する
`--publish [host port]:[container port]`
`$ docker run --publish 8000:5000 python-docker`

バックグラウンドとして実行
`$ docker run -dp 8000:5000 python-docker`

実行中のコンテナ一覧(停止中も含めるなら-a)
`$ docker ps`

コンテナをストップ
`$ docker stop <container_name or container_id>`

コンテナを再起動
`$ docker restart <container_name or container_id>`

コンテナを削除
`$ docker rm <container_name or container_id>`

コンテナに名前を付ける
`$ docker run -dp 8000:5000 --name rest-server python-docker`

## 開発にコンテナを使う

### ローカルデータベースとコンテナ

### Mysqlのセットアップ

- MySQLデータベースの代わりに、MySQL用のDocker公式イメージを用いる
- MySQLのデータ用volumeと設定ファイル用のvolumeを作成する
```
docker volume create mysql
docker volume create mysql-config
```
- アプリケーションとデータベースを繋ぐネットワークを作成
`$ docker network create mysqlnet`
- MongoDBをコンテナとして実行し、上記のボリュームとネットワークに接続する
```
$ docker run --rm -d -v mysql:/var/lib/mysql \
  -v mysql_config:/etc/mysql -p 3306:3306 \
  --network mysqlnet \
  --name mysqldb \
  -e MYSQL_ROOT_PASSWORD=<password> \
  mysql
```
補足
- --rm コンテナ終了時に自動的に削除する
- mysql: Docker-hubからイメージを取得し、localで実行している

MySQLデータベースに接続
`$ docker exec -it mysqldb mysql -u root -p`

補足
- docker exec: Dockerコンテナ内でコマンドを実行
- mysqldb: 実行対象のコンテナ名
- mysql -u root -p: コンテナ内で実行するコマンド。
- mysqlクライアントを起動し、-uでユーザをrootで指定。-pでパスワードを指定している

###　アプリケーションをデータベースに接続

- 上節では、mysqldbコンテナに対してmysqlコマンドを実行し、MySQLデータベースにログインした

- appをMysqlに接続するために、中身を追加する 

```python
import mysql.connector
import json
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, Docker!'

#widgetsの取得
@app.route('/widgets')
def get_widgets():
    #database名をinventoryにした
    mydb = mysql.connector.connect(
        host="mysqldb",
        user="root",
        password="<password>",
        database="inventory"
    )
    # 起動
    cursor = mydb.cursor()

    cursor.execute("SELECT * FROM widgets")
    # cursour.description -> tuple x[0]=column_name
    row_headers=[x[0] for x in cursor.description] #this will extract row headers

    results = cursor.fetchall()
    json_data=[]
    
    """
    row_headers = ["name", "age"]
    results = [["dog", 12], ["cat", 7]]
    
    return ex) [
                {"animal": "cat", "age": 3}, 
                {"animal": "dog", "age": 12}
                ]
    """ 
    for result in results:
        json_data.append(dict(zip(row_headers,result)))

    cursor.close()

    return json.dumps(json_data)

#初期化
@app.route('/initdb')
def db_init():
    mydb = mysql.connector.connect(
        host="mysqldb",
        user="root",
        password="<password>"
    )
    cursor = mydb.cursor()

    cursor.execute("DROP DATABASE IF EXISTS inventory")
    cursor.execute("CREATE DATABASE inventory")
    cursor.close()

    mydb = mysql.connector.connect(
        host="mysqldb",
        user="root",
        password="<password>",
        database="inventory"
    )
    cursor = mydb.cursor()

    cursor.execute("DROP TABLE IF EXISTS widgets")
    cursor.execute("CREATE TABLE widgets (name VARCHAR(255), description VARCHAR(255))")
    cursor.close()

    return 'init database'

if __name__ == "__main__":
    app.run(host ='0.0.0.0')
```

- 必要なモジュールをインストールする
```
$ pip3 install mysql-connector-python
$ pip3 freeze | grep mysql-connector-python >> requirements.txt
```

- イメージを構築する
- . = currentのDockerfileを使用
- docker build --tag {image-name} {path}
```
$ docker build --tag python-docker-dev .
```

- イメージを実行する
```
$ docker run \
-rm -d \
--network mysqlnet \
--name rest-server \
-p 8000:5000 \
python-docker-dev
```

- アプリケーションがdbに接続しているか確認
```
$ curl http://localhost:8000/initdb
$ curl http://localhost:8000/widgets
```

### Composeを用いて開発する
- python-dockerとMySQLdbを1つのコマンドで起動する
- 自動的にサービス名がaliasになるので便利

- ★必要なのはymlとdockerfileのみ

```yml
version: '3.8'

services:
 web:
  #buildには現在階層のdockerfileを用いる
  build:
   context: .
  ports:
  - 8000:5000
  volumes:
  - ./:/app

 mysqldb:
  image: mysql
  ports:
  - 3306:3306
  environment:
  - MYSQL_ROOT_PASSWORD=<password>
  volumes:
  - mysql:/var/lib/mysql
  - mysql_config:/etc/mysql

volumes:
  mysql:
  mysql_config:
```

- アプリケーションを起動する
```
# --build コンパイルしたのち、イメージを起動
$ docker-compose -f docker-compose.dev.yml up --build
```

## GitHub Actionsを使って、CI/CDパイプラインをセットアップ

### 手順
1. サンプルDockerプロジェクトを例に、GItHub Actionsを設定する
2. GitHub Actionsワークフローをセットアップする
3. pullリクエストを減らす方向で、ワークフローを最適化
4. 特定のバージョンのみDockerHubに送信する

### Dockerプロジェクトのセットアップ
- ワークフローがDocker hubにアクセスできるようにする
```
1. Settings -> Secrets and variables -> Actions -> New Secrets
2. DOCKER_HUB_USERNAME : <MyDockerID>
3. https://hub.docker.com/settings/security　にアクセスし、New Access TokenでPersonal Access Tokenを入手
4. Settings -> Secrets and variables -> Actions -> New Secrets
5. DOCKER_HUB_ACCESS_TOKEN : <Personal Access Token>
```

### GitHub Actions ワークフローのセットアップ
1. .github/workflowsに移動
2. ymlファイルを作成
```yml
name: CI to Docker Hub
# push時に、Hubに送信
on:
  push:
    branches: [ main ]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # $GITHUB_WORKSPACE以下のrepoを調べ、ワークフローがアクセスできるようにする
      - name: Check Out Repo
        uses: actions/checkout@v2

      #キャッシュの作成と復元をサポート
      # path: キャッシュを保存するパス
      # key: キャッシュを識別 osの種類-buildx-Githubのハッシュ値
      # restore-keys: キャッシュを復元するキーのパターンマッチ
      # マッチしていればイメージをダウンロードする必要もDockerHubから取得する回数も減る
      - name: Cache Docker layers
        uses: actions/cache@v2
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-

      # DockerHubにログイン
      - name: Login to Docker Hub
        uses: docker/login-action@v1
        # secretsから入手
        with:
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}
        
      # Build -> push(構築cacheを利用)
      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v2
        with:
          context: ./python-docker/
          file: ./python-docker/Dockerfile
          push: true
          tags: ${{ secrets.DOCKER_HUB_USERNAME }}/simplewhale:latest
          build-args: BUILDX_BUILDER=image-builder

      - name: Image digest
        run: echo ${{ steps.docker_build.outputs.digest }}
```

### タグ付けされたバージョンのみ送信する
- 指定したversionのみ、DockerHubに送信する
- 通常のpushの場合、ghcr.ioに反映する
- Docker-hubへのpullは制限があるということなので
```yml
name: CI to Docker Hub
# リリース用にデプロイされるバージョンのタグがプッシュされたときに実行
# ex) git push origin v1.0.2
  push:
    tags:
      - "v1.*.*"
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # $GITHUB_WORKSPACE以下のrepoを調べ、ワークフローがアクセスできるようにする
      - name: Check Out Repo
        uses: actions/checkout@v2

      #キャッシュの作成と復元をサポート
      # path: キャッシュを保存するパス
      # key: キャッシュを識別 osの種類-buildx-Githubのハッシュ値
      # restore-keys: キャッシュを復元するキーのパターンマッチ
      # マッチしていればイメージをダウンロードする必要もDockerHubから取得する回数も減る
      - name: Cache Docker layers
        uses: actions/cache@v2
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-

      # DockerHubにログイン
      - name: Login to Docker Hub
        uses: docker/login-action@v1
        # secretsから入手
        with:
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}
        
      # Build -> push(構築cacheを利用)
      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v2
        with:
          context: ./python-docker/
          file: ./python-docker/Dockerfile
          push: true
          tags: ${{ secrets.DOCKER_HUB_USERNAME }}/simplewhale:latest
          build-args: BUILDX_BUILDER=image-builder

      - name: Image digest
        run: echo ${{ steps.docker_build.outputs.digest }}
```

```yml
# 主に通常のpushイベントに対応(ただしpull_requestは除く)
name: Github Trigger
on:
  push:
    branches: [ main ]
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      packages: write
    steps:
      # $GITHUB_WORKSPACE以下のrepoを調べ、ワークフローがアクセスできるようにする
      - name: Check Out Repo
        uses: actions/checkout@v2

      #キャッシュの作成と復元をサポート
      # path: キャッシュを保存するパス
      # key: キャッシュを識別 osの種類-buildx-Githubのハッシュ値
      # restore-keys: キャッシュを復元するキーのパターンマッチ
      # マッチしていればイメージをダウンロードする必要もDockerHubから取得する回数も減る
      - name: Cache Docker layers
        uses: actions/cache@v2
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-

      # Github container registryにログイン
      - name: Login to Github container registry
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v1
        # secretsから入手
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # lowercase
      - name: set lower case owner name
        run: |
          echo "OWNER_LC=${OWNER,,}" >>${GITHUB_ENV}
        env:
          OWNER: '${{ github.repository_owner }}'
      
      # Build -> push(構築cacheを利用)
      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v2
        with:
          context: ./python-docker/
          file: ./python-docker/Dockerfile
          push: true
          tags: ghcr.io/${{ env.OWNER_LC }}/simplewhale:latest
          build-args: BUILDX_BUILDER=image-builder
      - name: Image digest
        run: echo ${{ steps.docker_build.outputs.digest }}
          
    

```
サンプル
```control
$ git tag -a v1.0.0
$ git push origin v1.0.0
```


