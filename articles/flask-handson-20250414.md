---
title: "Flask を使った Todo リスト＆LINE Bot 開発ハンズオン"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flask", "LINE Bot", "Python", "API", "SQLite"]
published: true
---

# 目次

- [目次](#目次)
- [はじめに](#はじめに)
- [前提条件と準備](#前提条件と準備)
- [1. Flask プロジェクト構成の作成](#1-flask-プロジェクト構成の作成)
- [2. SQLite データベースの準備](#2-sqlite-データベースの準備)
  - [app/**init**.py](#appinitpy)
  - [app/todo_service.py](#apptodo_servicepy)
- [3. Todo リスト API の実装](#3-todo-リスト-api-の実装)
  - [app/routes/todo_routes.py](#approutestodo_routespy)
  - [4. アプリの起動と API 動作確認](#4-アプリの起動と-api-動作確認)
    - [run.py](#runpy)
    - [API 動作確認手順（curl コマンド例）](#api-動作確認手順curl-コマンド例)
- [5. LINE Messaging API のセットアップ](#5-line-messaging-api-のセットアップ)
  - [5.1 LINE Developers でのチャネル設定](#51-line-developers-でのチャネル設定)
  - [5.2 アクセストークンとシークレットの取得](#52-アクセストークンとシークレットの取得)
  - [5.3 ngrok でローカルサーバーを公開](#53-ngrok-でローカルサーバーを公開)
    - [ngrok のインストールと使用方法](#ngrok-のインストールと使用方法)
  - [5.4 Webhook URL の設定](#54-webhook-url-の設定)
  - [5.5 応答設定](#55-応答設定)
  - [5.6 Bot を友だち追加](#56-bot-を友だち追加)
  - [5.7 環境変数の設定（.env ファイル）](#57-環境変数の設定env-ファイル)
    - [python-dotenv のインストール](#python-dotenv-のインストール)
    - [.env ファイルの作成](#env-ファイルの作成)
    - [セキュリティに関する注意](#セキュリティに関する注意)
- [6. Flask で LINE Webhook を処理して Todo 操作](#6-flask-で-line-webhook-を処理して-todo-操作)
  - [app/routes/line_routes.py](#approutesline_routespy)
  - [ポイント:](#ポイント)
  - [動作確認](#動作確認)
- [まとめ：Flask アプリ構造のポイント](#まとめflask-アプリ構造のポイント)
  - [オプション: 発展的な改善・拡張案](#オプション-発展的な改善拡張案)

# はじめに

エンジニアが Flask アプリの構造を学ぶためのハンズオンとして、SQLite を用いた簡易 Todo リスト API と、LINE 公式 SDK（`line-bot-sdk-python`）を使った Messaging API 連携 Bot を構築します。**Next.js/TypeScript が主なバックグラウンドで、Python はスクリプト程度の経験**という想定読者向けに、フロントエンド（Jinja テンプレートや Next.js）は使用せず、LINE 上のメッセージ（Webhook）経由で操作する形に絞っています。ハンズオンを通じて **Flask のプロジェクト構造**（アプリケーションファクトリ、Blueprint、サービス層の分離など）を体験し、最終的に以下の成果物を得ます：

- **Todo リスト API（SQLite 永続化）** – Todo アイテムの追加・一覧・削除を行う REST API（GET, POST, DELETE）
- **LINE ボット** – 「ADD」「DELETE」「LIST」のテキストコマンドで Todo を操作できる LINE Messaging API 対応のボット
- **Flask アプリの構成理解** – Blueprint によるモジュール分割やアプリケーションファクトリパターンを採用したプロジェクト構造

所要時間は約 3 時間を想定しています。各ステップでコード例や実行例を示し、さらに発展的な内容は「オプション」として後述します。

# 前提条件と準備

- **前提環境**：MacOS 上で Python 3 系がインストール済みであること（ターミナルで `python3 --version` コマンドで確認できます）。また、LINE アカウントを持ち、LINE Developers での開発者登録が済んでいることを前提とします（Messaging API 用のチャネル作成に必要）。SQLite は Python に標準搭載されています。

1. **ディレクトリ作成**

   作業用のプロジェクトディレクトリを作成しましょう。例としてターミナルで以下を実行します：

   ```bash
   $ mkdir flask_todo_line_bot && cd flask_todo_line_bot
   ```

2. **Python 仮想環境の構築**
   Python の仮想環境ツール venv を使い、本プロジェクト用の独立した環境を用意します。次のコマンドでプロジェクトフォルダ内に仮想環境を作成・有効化します：

   ```bash
   $ python3 -m venv venv # 仮想環境フォルダを作成（名前は venv としています）
   $ source venv/bin/activate # 仮想環境を有効化する
   ```

   成功すると、ターミナルの先頭に (venv) と表示され、仮想環境下でコマンド実行されるようになります。以降はこの環境で作業してください。

3. **必要パッケージのインストール**
   仮想環境が有効になった状態で、Flask と LINE SDK をインストールします。

   ```bash
   (venv) $ python -m pip install --upgrade pip
   (venv) $ pip install Flask line-bot-sdk python-dotenv
   ```

# 1. Flask プロジェクト構成の作成

Flask ではシンプルなスクリプト 1 ファイルでアプリを作ることも可能ですが、規模が大きくなるほどディレクトリを分割し Blueprint を用いることでコードを整理するメリットが大きいです。ここでは MVC 的な役割分担を意識しつつ、以下の構成を用いてアプリケーションファクトリ+Blueprint を採用します。

> **Blueprint とは**：Flask の Blueprint（ブループリント）は、アプリケーションを機能ごとに分割するための仕組みです。関連するビュー（ルート）、テンプレート、静的ファイルなどをグループ化し、モジュール化された再利用可能なコンポーネントとして定義できます。例えば、ユーザー認証、管理画面、API など、機能ごとに別々の Blueprint として実装することで、コードの管理が容易になります。Blueprint 自体は直接実行できず、Flask アプリケーションに登録して初めて機能します。

> **アプリケーションファクトリとは**：Flask のアプリケーションファクトリとは、Flask アプリケーションオブジェクト（`Flask(__name__)` で作成されるもの）を返す関数のことです。この関数内でアプリの設定、拡張機能の初期化、Blueprint の登録などを行います。このパターンを使うことで、テスト時に異なる設定を適用したり、複数のインスタンスを作成したりと、柔軟にアプリケーションを構成できます。Flask の公式ドキュメントでも推奨されているアプローチで、特に大規模なアプリケーション開発に適しています。

```
flask_todo_line_bot/
├── app/
│   ├── __init__.py ← Flask アプリ生成（ファクトリ）と Blueprint 登録
│   ├── models.py ← Todo モデル定義と DB 操作（データ層）
│   ├── todo_service.py ← Todo のビジネスロジック（サービス層）
│   └── routes/ ← ルーティング用 Blueprint モジュール群（コントローラ層）
│       ├── todo_routes.py ← Todo CRUD 用の API ルート定義
│       └── line_routes.py ← LINE Webhook 用のルート定義
├── run.py ← アプリケーション起動スクリプト
└── requirements.txt ← （必要に応じて）依存パッケージ一覧
```

- **app/**init**.py**: Flask のアプリケーションファクトリ create_app()関数を定義し、アプリ設定や Blueprint 登録を行う。
- **app/models.py/app/todo_service.py**: Todo モデルやビジネスロジックの実装。今回は SQLite アクセスを直接書きます。
- **app/routes/**: Flask の Blueprint を使ってビュー関数（ルーティング）を機能ごとにまとめる。
- **run.py**: アプリを起動するスクリプト。

上記の構成図を再現します。

# 2. SQLite データベースの準備

Todo を永続化するため、組み込みデータベースの SQLite を使用します。Flask に標準 ORM はないため、ここでは Python の sqlite3 モジュールで直接 SQL を実行します。必要になれば SQLAlchemy などを使いましょう。

### app/**init**.py

まず、Flask アプリ生成用のファクトリ関数を定義し、DB ファイル名や初期化処理を設定します。

```python
# app/__init__.py

from flask import Flask
from dotenv import load_dotenv
import os

# .env ファイルから環境変数を読み込む
load_dotenv()

def create_app():
    """Flask アプリ生成用ファクトリ関数"""
    app = Flask(__name__)
    app.config['DB_PATH'] = 'todos.db' # SQLite データベースファイルのパス

    # データベース初期化
    with app.app_context():
        from app import todo_service
        todo_service.init_db()

    # このあとでBlueprintを登録
    from app.routes import todo_routes, line_routes
    app.register_blueprint(todo_routes.bp)
    app.register_blueprint(line_routes.bp)

    return app
```

### app/todo_service.py

DB 初期化関数と、Todo の CRUD 操作を行う関数を実装します。init_db()でテーブルが無ければ作成し、それ以外は Todo の追加・一覧・削除を担当。

```python
# app/todo_service.py

import sqlite3
from flask import current_app

def init_db():
    """データベースの初期化（テーブルがなければ作成）"""
    db_path = current_app.config['DB_PATH']
    conn = sqlite3.connect(db_path)
    conn.execute(
        "CREATE TABLE IF NOT EXISTS todos ("
        "id INTEGER PRIMARY KEY AUTOINCREMENT, "
        "content TEXT NOT NULL)"
    )
    conn.commit()
    conn.close()

def create_todo(content):
    """新しい Todo を追加し、作成されたレコードを返す"""
    conn = sqlite3.connect(current_app.config['DB_PATH'])
    cur = conn.cursor()
    cur.execute("INSERT INTO todos (content) VALUES (?)", (content,))
    conn.commit()
    new_id = cur.lastrowid
    conn.close()
    return {"id": new_id, "content": content}

def get_all_todos():
    """Todo を全件取得してリストで返す"""
    conn = sqlite3.connect(current_app.config['DB_PATH'])
    cur = conn.cursor()
    cur.execute("SELECT id, content FROM todos")
    rows = cur.fetchall()
    conn.close()
    return [{"id": r[0], "content": r[1]} for r in rows]

def delete_todo(todo_id):
    """指定 ID の Todo を削除する。削除できたら True を返す"""
    conn = sqlite3.connect(current_app.config['DB_PATH'])
    cur = conn.cursor()
    cur.execute("DELETE FROM todos WHERE id = ?", (todo_id,))
    conn.commit()
    deleted = (cur.rowcount > 0)
    conn.close()
    return deleted
```

# 3. Todo リスト API の実装

### app/routes/todo_routes.py

Todo CRUD を実行できる API を Flask の Blueprint で定義します。/todos/配下の URL で操作し、JSON を返す形です。

```python
# app/routes/todo_routes.py

from flask import Blueprint, request, jsonify, abort
from app import todo_service

bp = Blueprint('todo_routes', __name__, url_prefix='/todos')

@bp.route('/', methods=['GET'])
def list_todos():
    """全 Todo を一覧取得"""
    todos = todo_service.get_all_todos()
    return jsonify(todos), 200

@bp.route('/', methods=['POST'])
def create_todo_item():
    """新規 Todo を追加"""
    data = request.get_json()
    if not data or 'content' not in data:
        abort(400, description="Request JSON must include 'content'")
    new_todo = todo_service.create_todo(data['content'])
    return jsonify(new_todo), 201

@bp.route('/<int:todo_id>', methods=['DELETE'])
def delete_todo_item(todo_id):
    """指定 ID の Todo を削除"""
    deleted = todo_service.delete_todo(todo_id)
    if not deleted:
        abort(404, description=f"Todo id={todo_id} not found")
    return '', 204
```

## 4. アプリの起動と API 動作確認

### run.py

開発用サーバの起動スクリプトを作成します。

```python
# run.py

from app import create_app, todo_service
from dotenv import load_dotenv

# .env ファイルから環境変数を読み込む
load_dotenv()

app = create_app()
app.todo_service = todo_service # LINE ハンドラから使うために登録

if __name__ == '__main__':
    app.run(debug=True)
```

ターミナルで実行し、curl などで API を呼び出して動作確認します。

```bash
# アプリ起動
(venv) $ python run.py
```

### API 動作確認手順（curl コマンド例）

別のターミナルウィンドウを開き、以下の curl コマンドでテストしてみましょう：

```bash
# Todo 一覧取得 (GET)
$ curl http://localhost:5000/todos/
```

- **Todo 一覧 GET /todos/**：初回は空配列`[]`が返る。

```bash
# Todo 追加 (POST)
$ curl -X POST -H "Content-Type: application/json" -d '{"content":"Buy milk"}' http://localhost:5000/todos/
```

- **Todo 追加 POST /todos/**：`{"content":"Buy milk"}` を送信 → `{"id":1,"content":"Buy milk"}` が返り、201 ステータス。

```bash
# Todo 削除 (DELETE)
$ curl -X DELETE http://localhost:5000/todos/1
```

- **Todo 削除 DELETE /todos/1**：削除成功で 204 が返る。

Postman などの API クライアントツールを使う場合も、同じエンドポイントとメソッドでリクエストを送信してください。

# 5. LINE Messaging API のセットアップ

LINE Bot を作成するには、LINE Developers でのチャネル作成と、Webhook URL の設定が必要です。以下の手順で進めましょう：

## 5.1 LINE Developers でのチャネル設定

1. [LINE Developers コンソール](https://developers.line.biz/console/)にログインする
2. 新規プロバイダーを作成（初めての場合）または既存のプロバイダーを選択
3. 「新規チャネル作成」から「Messaging API」を選択
4. チャネル情報を入力：
   - チャネル名：「Flask Todo Bot」など、ユーザーに表示される名前
   - チャネル説明：簡単な説明
   - 大業種・小業種：適切なものを選択
   - メールアドレス：連絡先
5. 利用規約に同意して「作成」ボタンをクリック

## 5.2 アクセストークンとシークレットの取得

1. 作成したチャネルの「基本設定」タブを開く
   - 「チャネルシークレット」をメモする（Python コードで使用）
2. 「Messaging API 設定」タブを開く
   - 「チャネルアクセストークン」の「発行」ボタンをクリック
   - 発行されたトークンをメモする（Python コードで使用）

## 5.3 ngrok でローカルサーバーを公開

LINE の Webhook は HTTPS の URL が必要です。開発中は ngrok というツールを使って、ローカルの Flask サーバーを一時的にインターネットに公開します。

### ngrok のインストールと使用方法

1. [ngrok の公式サイト](https://ngrok.com/)でアカウント作成・ダウンロード
2. ダウンロードした ngrok を適切な場所に解凍
3. ターミナルで以下のコマンドを実行（Flask は 5000 番ポートで実行中と仮定）

```bash
# ngrok を実行
$ ./ngrok http 5000
```

4. 実行すると以下のような画面が表示され、`https://xxxx.ngrok.io` という URL が生成されます

```
ngrok by @inconshreveable                           (Ctrl+C to quit)

Session Status                online
Session Expires               1 hour, 59 minutes
Version                       2.3.40
Region                        United States (us)
Web Interface                 http://127.0.0.1:4040
Forwarding                    http://xxxx.ngrok.io -> http://localhost:5000
Forwarding                    https://xxxx.ngrok.io -> http://localhost:5000

Connections                   ttl     opn     rt1     rt5     p50     p90
                              0       0       0.00    0.00    0.00    0.00
```

5. 表示された `https://xxxx.ngrok.io` の URL をメモします（xxxx は実行するたびに変わります）

:::message alert
macOS の ControlCenter がデフォルトポート(5000)を既に使用している場合は、8080 番ポートを指定してください。
:::

## 5.4 Webhook URL の設定

1. LINE Developers コンソールの「Messaging API 設定」タブに戻る
2. 「Webhook 設定」セクションで以下の設定を行う：
   - Webhook URL: `https://xxxx.ngrok.io/line/webhook` と入力（xxxx は ngrok で生成された値）
   - 「Webhook の利用」を「有効」に設定
   - 「検証」ボタンをクリックし、「成功」と表示されることを確認
     （Flask アプリが起動していないと失敗します）

## 5.5 応答設定

1. 「応答メッセージ」を「無効」に設定
2. 「Webhook」のみを「有効」に設定

これで LINE Bot からのメッセージは全て Flask アプリで処理されるようになります。

## 5.6 Bot を友だち追加

1. 「Messaging API 設定」タブ内の「QR コード」をスマホで読み取る
2. または「Bot リンク」をスマホで開く
3. 「友だち追加」ボタンをタップして Bot を友だちに追加

これで LINE Bot のセットアップは完了です。次に Flask アプリに LINE Webhook 処理コードを実装しましょう。

## 5.7 環境変数の設定（.env ファイル）

セキュリティのため、LINE のチャネルシークレットやアクセストークンはコードに直接書かず、環境変数として管理するのがベストプラクティスです。Flask で環境変数を扱うために、python-dotenv ライブラリを使用します：

### python-dotenv のインストール

```bash
# 仮想環境内で実行
$ pip install python-dotenv
```

### .env ファイルの作成

プロジェクトのルートディレクトリに `.env` ファイルを作成し、以下の内容を記述します（値は自分のものに置き換えてください）：

```
LINE_CHANNEL_SECRET=ここにチャネルシークレットを入力
LINE_CHANNEL_ACCESS_TOKEN=ここにチャネルアクセストークンを入力
```

### セキュリティに関する注意

`.env` ファイルには機密情報が含まれるため、Git リポジトリにコミットしないよう `.gitignore` ファイルに追加することをお勧めします：

```bash
# .gitignore ファイルを作成し、.env を無視するよう設定
$ echo ".env" >> .gitignore
```

これで環境変数の設定は完了です。次に LINE Webhook を処理する Flask ルートを実装します。

# 6. Flask で LINE Webhook を処理して Todo 操作

### app/routes/line_routes.py

公式 SDK（line-bot-sdk-python）を使うことで、署名検証・返信 API の呼び出しをシンプルに記述できます。

```python
# app/routes/line_routes.py

import os
from flask import Blueprint, request, abort, current_app
from linebot import LineBotApi, WebhookHandler
from linebot.exceptions import InvalidSignatureError
from linebot.models import MessageEvent, TextMessage, TextSendMessage
from dotenv import load_dotenv

# .env ファイルから環境変数を読み込む
load_dotenv()

# 環境変数から LINE の設定を取得
CHANNEL_SECRET = os.getenv("LINE_CHANNEL_SECRET")
CHANNEL_ACCESS_TOKEN = os.getenv("LINE_CHANNEL_ACCESS_TOKEN")

# 環境変数が設定されているか確認（開発時の助けになります）
if not CHANNEL_SECRET or not CHANNEL_ACCESS_TOKEN:
    print("Warning: LINE API の認証情報が設定されていません。.env ファイルを確認してください。")

line_bot_api = LineBotApi(CHANNEL_ACCESS_TOKEN)
handler = WebhookHandler(CHANNEL_SECRET)

bp = Blueprint('line_routes', __name__, url_prefix='/line')

@bp.route('/webhook', methods=['POST'])
def webhook():
    signature = request.headers.get('X-Line-Signature')
    if signature is None:
        abort(400, description="Missing signature")
    body = request.get_data(as_text=True)
    try:
        handler.handle(body, signature)
    except InvalidSignatureError:
        abort(400, description="Invalid signature")
    return 'OK'

@handler.add(MessageEvent, message=TextMessage)
def handle_message(event):
    user_message = event.message.text.strip()
    reply_token = event.reply_token
    service = current_app.todo_service

    # コマンド判定
    if user_message.lower().startswith("add "):
        content = user_message[4:].strip()
        if content:
            new_todo = service.create_todo(content)
            reply_text = f"Todo追加: (id={new_todo['id']}) {new_todo['content']}"
        else:
            reply_text = "追加するTodoの内容を入力してください。"
    elif user_message.lower().startswith("delete "):
        parts = user_message.split(maxsplit=1)
        if len(parts) < 2 or not parts[1].isdigit():
            reply_text = "削除コマンドの形式: DELETE <数字ID>"
        else:
            todo_id = int(parts[1])
            deleted = service.delete_todo(todo_id)
            if deleted:
                reply_text = f"Todo id={todo_id} を削除しました。"
            else:
                reply_text = f"Todo id={todo_id} が見つかりません。"
    elif user_message.lower() == "list":
        todos = service.get_all_todos()
        if not todos:
            reply_text = "登録されているTodoはありません。"
        else:
            lines = ["[Todo一覧]"]
            for t in todos:
                lines.append(f"- {t['id']}: {t['content']}")
            reply_text = "\n".join(lines)
    else:
        reply_text = (
            "コマンドを入力してください:\n"
            "・Todo追加: ADD <内容>\n"
            "・Todo削除: DELETE <ID>\n"
            "・Todo一覧: LIST"
        )

    line_bot_api.reply_message(reply_token, TextSendMessage(text=reply_text))
```

### ポイント:

- **@bp.route('/webhook', methods=['POST'])** で Webhook URL を受け取り、LINE SDK の handler.handle を呼ぶだけで署名検証・イベントパースを実行。
- **イベントハンドラ @handler.add(...)** で TextMessage が届いたときの処理を定義。ユーザーからのメッセージに応じて Todo を操作し、line_bot_api.reply_message で返信メッセージを返す。
- **current_app.todo_service** により Todo 操作関数（create, delete, list など）を呼び出し。

### 動作確認

Flask を再起動し、ngrok を立ち上げ、LINE から Bot に「ADD <内容>」「DELETE <ID>」「LIST」などを送信してみましょう。正常に Webhook が飛び、Todo 操作が行われ、返信が届けば成功です。

# まとめ：Flask アプリ構造のポイント

1. **アプリケーションファクトリ (create_app)**
   - Flask アプリを生成・設定する公式推奨パターン。テストやデプロイ時にも柔軟。
2. **Blueprint によるモジュール分割**
   - Todo 管理機能、LINE 連携機能をそれぞれ todo_routes・line_routes に分けることで可読性 UP。
3. **サービス層の分離**
   - ルート関数から直接 SQL を書くのではなく、todo_service を挟むことでビジネスロジックをまとめる。
4. **LINE 公式 SDK**
   - 署名検証や返信 API 呼び出しを簡潔に書ける。イベントハンドラをデコレータで管理できる。

## オプション: 発展的な改善・拡張案

- **完了フラグ/DONE コマンドの導入**: テーブルに done カラムを追加し、完了・未完了を切り替える。
- **UPDATE(EDIT)コマンド**: 既存 Todo 内容を更新する機能。
- **テストコードの追加**: pytest などを用いてユニットテスト/Flask テストクライアントで API テストを導入。
- **ORM の利用**: SQLAlchemy を導入し、モデル定義をよりリッチに。
- **デプロイ**: Heroku や Railway などの PaaS にデプロイし、環境変数で LINE シークレットを安全に管理。

これで Flask × SQLite × LINE SDK の基本的な Todo アプリが完成です。ぜひ本稿をベースに自分の Bot 機能を拡張してみてください。
