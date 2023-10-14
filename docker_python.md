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

> **Note**
> 概要: `Dockerfileのファイル名`について 
> Dockerfile.hogeやhoge.Dockerfileにすることも可能だが、その場合はfオプションが必要
> docker build -f Dockerfile.hoge
> プロジェクトで主となるDockerfileはデフォルトが推奨される

> **Note**
> こんな感じに書き込むと強調表示される
> a

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



