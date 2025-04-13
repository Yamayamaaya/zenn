---
title: "Flask を使った Todo リスト＆LINE Bot 開発ハンズオン"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flask", "LINE Bot", "Python", "API", "SQLite"]
published: true
---

Flask を使った Todo リスト＆LINE Bot 開発ハンズオン

本ハンズオンでは、Flask の構造理解を目的に、SQLite を用いた簡易 Todo リスト API と LINE Messaging API 連携 Bot を作成します。Mac 環境で仮想環境を構築し、Flask の MVC 的なプロジェクト構成を体験します。最終的に、**Flask のアプリケーション構造（アプリケーションファクトリ、Blueprint、テンプレート構成、サービス層の分離など）**を理解することがゴールです。所要時間は約 4 時間を想定しています。

このハンズオンで実装する内容は以下のとおりです：

- Python 仮想環境の構築（Mac 環境）
- Flask プロジェクトの構成（アプリケーションファクトリと Blueprint、モデルの分離）
- SQLite データベースの初期化スクリプトと操作方法
- Todo リストの REST API（CRUD）実装と Flask における MVC 構造の解説
- LINE Messaging API のセットアップ手順（チャネル作成、Webhook 設定、アクセストークン取得）
- Flask アプリで LINE の Webhook を受信し、署名検証とメッセージ内容に応じた Todo 操作（追加・一覧表示・削除）
- 各ステップのコード例と動作確認方法（curl や Postman による API テスト、実際の LINE Bot テスト）

目次：

1. 環境準備と仮想環境構築
2. Flask プロジェクト構成の作成
3. SQLite データベースの準備
4. Todo モデルと CRUD API の実装
5. LINE Messaging API のセットアップ
6. Flask で LINE Webhook を処理して Todo 操作
7. アプリの起動と動作確認
8. まとめ：Flask アプリ構造のポイント

⸻

環境準備と仮想環境構築

前提条件：Mac に Python 3 系がインストールされていることを前提とします（ターミナルで `python3 --version` で確認可能）。また、SQLite は Python に標準モジュールとして含まれているため追加インストール不要です。

まず、開発用のディレクトリを作成し、その中で Python の仮想環境を構築します。仮想環境を使うことで、他のプロジェクトと切り離して必要なパッケージを管理できます。以下の手順を実行してください：

1. ターミナルでプロジェクト用ディレクトリを作成して移動します。例：

```bash
$ mkdir flask_todo_line_bot && cd flask_todo_line_bot
```

2. 仮想環境を作成します（ここでは Python 標準の venv モジュールを使用）。例：

```bash
$ python3 -m venv venv
```

すると venv という仮想環境ディレクトリが作成されます。

3. 仮想環境を有効化します：

```bash
$ source venv/bin/activate
```

プロンプトに(venv)と表示されれば仮想環境の有効化完了です。

4. 開発に必要な Python パッケージをインストールします：

```bash
(venv) $ pip install Flask requests
```

- Flask：軽量な Web フレームワーク。
- requests：LINE の API に HTTP リクエストを送るために使用（標準ライブラリでも可能ですが、利便性のため利用）。

※ LINE 公式の SDK（line-bot-sdk-python）を利用する方法もありますが、本ハンズオンでは Flask の学習を優先し、SDK を使わずに実装してみます。

以上で環境準備は完了です。次に、Flask アプリケーションのプロジェクト構成を作成していきます。

Flask プロジェクト構成の作成

Flask ではアプリをシンプルに 1 ファイルで書くこともできますが、規模が大きくなるほどディレクトリを分割して Blueprint を用いることでコードを整理するメリットが大きいです ￼。本ハンズオンではアプリを複数のモジュールに分割し、アプリケーションファクトリと Blueprint を活用した構成にします。この構成により、後から機能追加しやすくなり、Flask の MVC 的な役割分担も理解しやすくなります。

まず、以下のようなディレクトリ構成を用意します：

flask_todo_line_bot/ (プロジェクトのルートディレクトリ)
├── venv/ (仮想環境ディレクトリ)
├── app/ (Flask アプリケーションのパッケージ)
│ ├── **init**.py (アプリケーションファクトリや Blueprint 登録を行う)
│ ├── models.py (Todo モデルとデータベース操作関連)
│ ├── todo_service.py (Todo のビジネスロジック層：データ操作関数を定義)
│ ├── routes/ (ルーティング用モジュールを格納するディレクトリ)
│ │ ├── **init**.py
│ │ ├── todo_routes.py (Todo CRUD 用の API ルート定義)
│ │ └── line_routes.py (LINE Webhook 用のルート定義)
│ └── templates/ (HTML テンプレートを置くディレクトリ：今回は API のみのため未使用)
├── schema.sql (SQLite データベース初期化用 SQL スクリプト)
├── init_db.py (データベース初期化用の Python スクリプト)
└── run.py (アプリケーションを起動するスクリプト)

ポイント：上記のように分割することで、役割ごとにコードを整理できます。たとえば、routes/以下に Blueprint ごとのルーティングを記述し、models.py にデータモデルと DB 操作を記述、todo_service.py でビジネスロジックを記述する構成にしています。これらを組み合わせるのが**init**.py に定義するアプリケーションファクトリ関数です。

アプリケーションファクトリと Blueprint 登録

まず、app/**init**.py に Flask アプリケーションを生成するファクトリ関数を定義します。アプリケーションファクトリとは、Flask アプリ（WSGI アプリ）を生成して設定を行う関数のことです ￼。これにより、複数の構成やテスト用設定でアプリインスタンスを作成でき、柔軟性やテスト容易性が向上します ￼ ￼。

app/**init**.py を以下のように作成します：

# app/**init**.py

from flask import Flask

def create_app():
"""Flask アプリケーションの生成（アプリケーションファクトリ）"""
app = Flask(**name**)
app.config['DB_PATH'] = 'todo.db' # SQLite データベースファイルのパスを設定

    # Blueprintのインポートと登録
    from app.routes import todo_routes, line_routes
    app.register_blueprint(todo_routes.bp)        # Todo API用Blueprintを登録
    app.register_blueprint(line_routes.bp)        # LINE Webhook用Blueprintを登録

    return app

解説：
• Flask(**name**) で Flask アプリのインスタンスを生成します。**name**を渡すことで Flask が静的ファイルやテンプレートを探す基準になります。
• app.config['DB_PATH'] はアプリの設定にデータベースファイル名を登録しています（簡易的に使用）。必要に応じて他の設定（例えばデバッグモードや SECRET_KEY など）もここで設定できます。
• Blueprint 登録: app.register_blueprint(...) で事前に定義した Blueprint をアプリに組み込みます。Blueprint には URL プレフィックスを付けることもできますが、ここでは各 Blueprint 側でルートパスを定義するので指定していません（必要に応じて url_prefix='/todo'のように付与できます ￼）。Blueprint を登録すると、その Blueprint 内で定義されたルーティングやエラーハンドラなどが一括してアプリに追加されます ￼ ￼。

次に、Blueprint 自体の定義を行います。Blueprint とは関連するビューやルートをグループ化する仕組みで、アプリケーションの機能ごとにモジュールを分割できます ￼。これにより共通の URL プレフィックスで纏めたり再利用したりでき、アプリが大規模になっても管理しやすくなります ￼ ￼。

それでは、Todo CRUD 用の Blueprint と、LINE Webhook 用の Blueprint を順に実装していきましょう。

SQLite データベースの準備

Todo を保存するデータストアとして組み込みの SQLite データベースを使用します。Flask 本体に ORM は含まれていないため、今回はシンプルに sqlite3 モジュールで操作します（小規模なので直接 SQL を使いますが、規模拡大時は SQLAlchemy など ORM の導入を検討できます）。

まず、データベースのスキーマを定義しましょう。プロジェクトルートに schema.sql というファイルを作成し、Todo 項目のテーブルを作成する SQL 文を書きます：

-- schema.sql （Todo テーブルのスキーマ定義）
CREATE TABLE IF NOT EXISTS todo (
id INTEGER PRIMARY KEY AUTOINCREMENT,
content TEXT NOT NULL,
done BOOLEAN NOT NULL DEFAULT 0,
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

解説：
• id: 自動採番される主キー。
• content: ToDo 項目のテキスト内容。
• done: 完了したかどうかのフラグ（未完了=0、完了=1）。
• created_at: 登録日時（デフォルトで現在時刻が入る）。
この程度のシンプルな構成とします。必要に応じて updated_at 列を追加して更新日時を管理しても良いですが、今回は省略しています。

次に、このスキーマを使ってデータベースを初期化するスクリプトを作成します。init_db.py というファイルをプロジェクトルートに作成し、以下のように記述します：

# init_db.py

import sqlite3

# Flask の config に設定した DB_PATH を利用

DB_PATH = 'todo.db' # （create_app で app.config['DB_PATH'] に同じパスを設定）

conn = sqlite3.connect(DB_PATH)
cur = conn.cursor()

# スキーマ SQL ファイルを読み込み実行

with open('schema.sql', 'r') as f:
cur.executescript(f.read())
conn.commit()
conn.close()
print(f"Initialized the database at {DB_PATH}.")

このスクリプトを実行すると（仮想環境を有効にした状態で(venv)$ python init_db.py）、同じディレクトリに todo.db という SQLite ファイルが作成され、Todo テーブルが準備されます。
※ メモ：Flask では flask shell やカスタムコマンドを使って init_db する方法もありますが、簡便のためスクリプトを直接実行する形にしています。また、app.config['DB_PATH']を直接使うために DB_PATH を重複定義していますが、実際のアプリでは Flask.app_context()内で current_app.config から参照する、もしくは設定を config.py などに切り出す方法もあります。

データベースが用意できたら、Todo データを扱うモデル／サービス層を実装します。ここでは ORM は使わず、必要なデータ操作を行う関数群を app/todo_service.py に用意します。

Todo モデル／サービス層の実装

app/todo_service.py にデータベースを操作する関数を実装します。このモジュールがサービス層として、実際のビジネスロジック（Todo の追加・取得・削除など）を担います。ルートハンドラからこれらの関数を呼び出すことで、Web の処理（リクエスト/レスポンス）とデータ処理を分離できます。では、以下のコードを記述してください：

# app/todo_service.py

import sqlite3
from flask import current_app

def get_all_todos():
"""Todo テーブルの全レコードを取得してリストで返す"""
conn = sqlite3.connect(current_app.config['DB_PATH'])
conn.row_factory = sqlite3.Row # 行を辞書のように扱えるようにする
cur = conn.cursor()
cur.execute("SELECT \* FROM todo")
rows = cur.fetchall()
conn.close() # sqlite3.Row を dict に変換
return [dict(row) for row in rows]

def get_todo(todo_id):
"""指定した ID の Todo を取得。存在しなければ None を返す。"""
conn = sqlite3.connect(current_app.config['DB_PATH'])
conn.row_factory = sqlite3.Row
cur = conn.cursor()
cur.execute("SELECT \* FROM todo WHERE id=?;", (todo_id,))
row = cur.fetchone()
conn.close()
return dict(row) if row else None

def create_todo(content):
"""新しい Todo を追加し、作成されたレコードを返す。"""
conn = sqlite3.connect(current_app.config['DB_PATH'])
cur = conn.cursor()
cur.execute("INSERT INTO todo (content, done) VALUES (?, ?);", (content, 0))
conn.commit()
new_id = cur.lastrowid # 挿入された行の ID # 新規レコードを取得して返す
cur.execute("SELECT \* FROM todo WHERE id=?;", (new_id,))
row = cur.fetchone()
conn.close()
return dict(row) if row else None

def update_todo(todo_id, content=None, done=None):
"""指定 ID の Todo を更新。更新後のレコードを返す。存在しない場合 None。"""
if content is None and done is None:
return None # 更新すべき内容がない場合
conn = sqlite3.connect(current_app.config['DB_PATH'])
cur = conn.cursor() # 動的に SET 句を構成
fields = []
params = []
if content is not None:
fields.append("content = ?")
params.append(content)
if done is not None:
fields.append("done = ?")
params.append(done)
params.append(todo_id)
sql = f"UPDATE todo SET {', '.join(fields)}, updated_at=CURRENT_TIMESTAMP WHERE id=?;"
cur.execute(sql, tuple(params))
conn.commit() # 該当レコードが無ければ変更件数 0
if cur.rowcount == 0:
conn.close()
return None # 更新後のレコード取得
cur.execute("SELECT \* FROM todo WHERE id=?;", (todo_id,))
row = cur.fetchone()
conn.close()
return dict(row)

def delete_todo(todo_id):
"""指定 ID の Todo を削除。削除できた場合 True を返す。"""
conn = sqlite3.connect(current_app.config['DB_PATH'])
cur = conn.cursor()
cur.execute("DELETE FROM todo WHERE id=?;", (todo_id,))
conn.commit()
deleted = (cur.rowcount > 0)
conn.close()
return deleted

解説：上記の関数群では都度データベースへの接続を開いていますが、シンプルさを優先しています。本来はアプリ起動時に sqlite3.connect してグローバルに保持したり、Flask の g オブジェクトやアプリコンテキストを利用してコネクションを使い回す方法もあります。ただ、SQLite はファイルベースで軽量なため、この規模なら都度接続でも問題ありません。また、current_app.config['DB_PATH']でアプリの設定から DB ファイルパスを取得しています。Blueprint 内の関数からでも current_app でアプリ設定にアクセスできます。

では次に、これらサービス関数を使って実際の API ルーティングを実装しましょう。

Todo モデルと CRUD API の実装

続いて、Todo リストの RESTful API エンドポイントを実装します。Flask の Blueprint を使い、エンドポイントごとにビュー関数を定義します。app/routes/todo_routes.py を作成し、以下の内容を記述してください：

# app/routes/todo_routes.py

from flask import Blueprint, request, jsonify, abort
from app import todo_service

# Todo 用の Blueprint を作成

bp = Blueprint('todo', **name**, url_prefix='/todos')

@bp.route('/', methods=['GET'])
def list_todos():
"""全ての Todo を取得する（LIST）"""
todos = todo_service.get_all_todos()
return jsonify(todos), 200 # JSON の配列として返す

@bp.route('/<int:todo_id>/', methods=['GET'])
def get_todo_detail(todo_id):
"""特定 ID の Todo を取得（READ）"""
todo = todo_service.get_todo(todo_id)
if todo is None:
abort(404, description="Todo not found")
return jsonify(todo), 200

@bp.route('/', methods=['POST'])
def create_todo_item():
"""新規 Todo を追加する（CREATE）"""
data = request.get_json()
if not data or 'content' not in data:
abort(400, description="Request JSON must include 'content'")
content = data['content']
new_todo = todo_service.create_todo(content) # 作成成功時、201 Created を返す
return jsonify(new_todo), 201

@bp.route('/<int:todo_id>/', methods=['PUT'])
def update_todo_item(todo_id):
"""既存 Todo を更新する（UPDATE）"""
data = request.get_json()
if not data:
abort(400, description="Request JSON required") # リクエスト JSON から内容取得（無くても None になる）
content = data.get('content')
done = data.get('done')
updated = todo_service.update_todo(todo_id, content=content, done=done)
if updated is None:
abort(404, description="Todo not found or no update performed")
return jsonify(updated), 200

@bp.route('/<int:todo_id>/', methods=['DELETE'])
def delete_todo_item(todo_id):
"""特定 ID の Todo を削除する（DELETE）"""
deleted = todo_service.delete_todo(todo_id)
if not deleted:
abort(404, description="Todo not found") # 通常、DELETE 成功時は 204 No Content を返すことも多いが、ここではメッセージ付きで 200 を返す
return jsonify({"result": "deleted", "id": todo_id}), 200

実装内容の説明：
• Blueprint 設定：bp = Blueprint('todo', **name**, url_prefix='/todos')
Blueprint 名を todo とし、url_prefix に/todos を設定しました。これにより、この Blueprint に属する全てのルートは自動的に/todos で始まる URL になります ￼。例えば list_todos 関数は GET /todos/でアクセスされます。
• リクエストとレスポンス：Flask では request オブジェクトから JSON データを取得し、jsonify 関数や辞書/リストを返すことで JSON レスポンスを返却できます。各関数で HTTP メソッドに対応する CRUD 操作を行い、結果を JSON で返しています。
• list_todos: Todo 全件をリストで取得し返す。空の場合は[]が返ります。
• get_todo_detail: パスパラメータ<int:todo_id>で指定された ID の Todo を取得します。見つからなければ 404 を返します（Flask の abort(404)を利用）。
• create_todo_item: リクエストボディ(JSON)から content を取り出し、新しい Todo を作成します。成功時には 201 ステータスで作成した Todo オブジェクトを返しています。
• update_todo_item: 指定 ID の Todo を更新します。リクエスト JSON に content や done（完了フラグ）が含まれていれば更新し、更新後のデータを返します。対象が無ければ 404 になります。
• delete_todo_item: 指定 ID の Todo を削除します。削除結果に応じて「削除成功」JSON または 404 を返します。

各エンドポイントは REST の慣習に従って CRUD 操作を実現しています。これらを curl コマンド等で試すことで API 単体が正しく動作するか確認できます。動作確認方法は後述します。

ここまでで、Flask アプリケーションとしては Todo CRUD API が完成しました。次は、このアプリに LINE Messaging API からの Webhook を連携させてみましょう。

LINE Messaging API のセットアップ

アプリを LINE と連携させるには、LINE Developers コンソールで Messaging API チャネル（LINE 公式アカウント）を作成し、Webhook の設定やトークンの取得を行います。以下は LINE 側で行う設定手順です： 1. LINE Developers でプロバイダーとチャネルの作成：まず LINE Developers コンソールにログインし、新規プロバイダー（任意の名前の開発グループのようなもの）を作成します。その中で「Messaging API」タイプのチャネルを作成します。チャネル作成時に表示される項目（チャネル名や説明、アイコンなど）は適宜入力してください。 2. チャネルアクセストークンの発行：チャネルが作成できたら、コンソールの「Messaging API 設定」タブで「チャネルアクセストークン」を発行します ￼。これが後で Bot から LINE プラットフォームへメッセージを返信する際の認証に必要です。発行したアクセストークンはメモしておきましょう（後から再表示も可能ですが、見えなくなる場合もあるので注意）。 3. チャネルシークレットの確認：同じ設定画面に「チャネルシークレット」が表示されています。これは Webhook リクエストの署名検証に使います。こちらも後ほど Flask アプリの設定に組み込みます。 4. Webhook URL の設定：Flask アプリで受信する Webhook のエンドポイント URL を設定します ￼。たとえば開発中は ngrok などを使って localhost を公開し、https://<ランダム ID>.ngrok.io/line/webhook のような URL を取得して設定します。コンソールの「Webhook 設定」欄で該当 URL を入力し、「更新」ボタンをクリックしてください。
• Webhook の有効化：Webhook URL を設定したら「Webhook の利用」を有効にします ￼。あわせて「接待メッセージ/応答メッセージ」の自動返信設定は無効にしておきます（デフォルトで LINE 公式アカウントが自動返信する設定が有効な場合、それをオフにしないと Bot の応答と競合するため）。
• 動作確認：Webhook URL 設定後、「検証」ボタンで接続確認ができます ￼。Flask アプリを先に起動していれば「成功」と表示されます（起動前の場合は後で再度検証可能）。 5. 友だち追加：スマートフォンの LINE アプリで、先ほど作成した LINE 公式アカウントを友だち追加します。コンソール上部に QR コードや招待 URL があるので、それを利用してください。これで、私たちの Bot（Flask アプリ）にメッセージを送信できるようになります。

以上で LINE 側の準備は完了です。必要な情報（チャネルアクセストークンとチャネルシークレット）はメモしたでしょうか？ 次に、Flask アプリ側で Webhook 受信エンドポイントを実装し、これらの情報を使って LINE からのイベントを処理します。

Flask で LINE Webhook を処理して Todo 操作

LINE プラットフォームからの Webhook リクエストを受け取るには、Flask アプリに対応するルート（エンドポイント）を用意します。前述の構成では app/routes/line_routes.py にこの処理を書き、Blueprint として登録します。ではコードを示します：

# app/routes/line_routes.py

import os, hmac, hashlib, base64, json, requests
from flask import Blueprint, request, abort, current_app

bp = Blueprint('line_bot', **name**, url_prefix='/line')

# LINE Developers コンソールで取得したチャネルシークレットとアクセストークンを環境変数から取得

CHANNEL_SECRET = os.getenv("LINE_CHANNEL_SECRET", default="YOUR_LINE_CHANNEL_SECRET")
CHANNEL_ACCESS_TOKEN = os.getenv("LINE_CHANNEL_ACCESS_TOKEN", default="YOUR_LINE_CHANNEL_ACCESS_TOKEN")

@bp.route('/webhook', methods=['POST'])
def line_webhook(): # 1. リクエストヘッダから署名取得
signature = request.headers.get('X-Line-Signature', '')
body = request.get_data(as_text=True)
if not signature:
abort(400, description="Missing signature")

    # 2. 署名の検証
    hash = hmac.new(CHANNEL_SECRET.encode('utf-8'), body.encode('utf-8'), hashlib.sha256).digest()
    computed_signature = base64.b64encode(hash).decode('utf-8')
    if signature != computed_signature:
        abort(400, description="Invalid signature")  # LINEからのリクエストでない可能性

    # 3. Webhookイベントの処理
    data = json.loads(body)
    events = data.get("events", [])
    for event in events:
        # メッセージイベントの場合
        if event.get("type") == "message" and event.get("message"):
            text = event["message"].get("text", "")
            reply_token = event.get("replyToken")
            user_id = event["source"].get("userId")  # ユーザーID（使用しない場合もあり）
            # シンプルなコマンド判定（大文字小文字を無視して判定）
            command = text.strip()
            response_msg = ""
            if command.lower().startswith("add "):
                # "ADD <内容>" -> Todo追加
                todo_content = command[4:]  # ADDの後の文字列
                if todo_content:
                    new_todo = current_app.todo_service.create_todo(todo_content)  # サービス関数でDB登録
                    response_msg = f"Todo追加: (id={new_todo['id']}) {new_todo['content']}"
                else:
                    response_msg = "追加するTodoの内容が空です。"
            elif command.lower().startswith("delete "):
                # "DELETE <id>" -> Todo削除
                try:
                    del_id = int(command.split()[1])
                except (IndexError, ValueError):
                    response_msg = "削除コマンドの形式: DELETE <数字ID>"
                else:
                    deleted = current_app.todo_service.delete_todo(del_id)
                    response_msg = f"Todo id={del_id} を削除しました。" if deleted else f"Todo id={del_id} が見つかりません。"
            elif command.lower() == "list":
                # "LIST" -> Todo一覧
                todos = current_app.todo_service.get_all_todos()
                if not todos:
                    response_msg = "登録されているTodoはありません。"
                else:
                    # 一覧を整形
                    lines = ["[Todo一覧]"]
                    for todo in todos:
                        status = "✅" if todo['done'] else "☐"
                        lines.append(f"{status} (id={todo['id']}) {todo['content']}")
                    response_msg = "\n".join(lines)
            else:
                # コマンドに合致しない場合はヘルプメッセージを返す
                response_msg = ("コマンドを入力してください:\n"
                                "・Todo追加: `ADD <内容>`\n"
                                "・Todo削除: `DELETE <ID>`\n"
                                "・Todo一覧: `LIST`")

            # Replyメッセージ送信
            if reply_token:
                headers = {
                    "Content-Type": "application/json",
                    "Authorization": f"Bearer {CHANNEL_ACCESS_TOKEN}"
                }
                reply_body = {
                    "replyToken": reply_token,
                    "messages": [{"type": "text", "text": response_msg}]
                }
                requests.post("https://api.line.me/v2/bot/message/reply",
                              headers=headers, json=reply_body)
    # LINEサーバーへHTTP 200を返す
    return "OK", 200

上記コードの流れを順を追って説明します。

1. 署名の取得：LINE の Webhook リクエストには HTTP ヘッダ X-Line-Signature が含まれており、ボディの内容に対する HMAC-SHA256 署名が付与されています ￼。まずは request.headers.get('X-Line-Signature')でこの署名文字列を取得します。また request.get_data(as_text=True)で生のリクエストボディを文字列として取り出します。署名が無ければ不正リクエストなので 400 を返します。

2. 署名の検証：取得したボディと、事前に控えた CHANNEL_SECRET（チャネルシークレット）を用いて HMAC-SHA256 ハッシュを計算し、base64 エンコードしたものが LINE 送信元の署名と一致するか検証します ￼。一致しなければこのリクエストは LINE から送信されたものではない可能性があるため、以降の処理をせず 400 エラーを返します。
   ※この署名検証はセキュリティ上必須です ￼。必ず行ってください。

3. イベント処理：LINE の Webhook では 1 回の POST リクエスト中に複数のイベントが含まれる場合があります。そのため、JSON ボディをパースして events 配列を取得し、ループで処理します。各イベントは種類（type）を持ちますが、今回はテキストメッセージの受信（message イベント）にのみ対応します。event["type"] == "message"で判定し、メッセージ内容 text と返信用トークン replyToken を取得します。
   • コマンド判定：ユーザーが送ったメッセージテキストに応じて Todo 操作を行います。単純化のため、以下のコマンドをサポートします（大文字小文字区別しません）:
   • "ADD <内容>": <内容>を新規 Todo として追加します。追加後、Todo 追加: (id=... ) 内容...という確認メッセージを返信します。
   • "DELETE <ID>": 指定された数値 ID の Todo を削除します。削除の成否に応じて結果を返信します。
   • "LIST": 現在登録されている Todo 一覧を返信します。ID と内容、および未完/完了を ☐/✅ 記号で表示します。もし Todo が無ければその旨を伝えます。
   • 上記以外の入力には、使い方を案内する簡単なヘルプメッセージを返信します。
   • コマンドに応じて、先ほど作成した todo_service 内の関数を呼び出しています。Blueprint 内の関数でも current_app.todo_service のようにしてアプリのグローバルリソースにアクセスできます（create_app で組み込んだ場合は app.todo_service = todo_service のようにセットする方法もありますが、ここでは current_app を介しています）。例えば todo_service.create_todo()で DB に挿入し、新規 Todo を取得しています。

4. LINE への返信：ユーザーからのメッセージには応答メッセージをできるだけ早く返信する必要があります。返信には、受信したイベントに含まれる replyToken を使い、LINE Messaging API の Reply API エンドポイントに対して HTTP POST リクエストを送ります。上記コードでは、Python の requests ライブラリを用いてhttps://api.line.me/v2/bot/message/replyに対し、ヘッダにチャネルアクセストークンを付与し、返信メッセージをJSONで送信しています。
   （チャネルアクセストークンは Authorization: Bearer <token>ヘッダで認証に使用します。）

最後に、処理が完了したら LINE プラットフォームに HTTP 200 でレスポンスを返します。これで LINE 側では「Webhook を正常に処理した」と判断し、ユーザーにもボットからの返信が届きます。

これで Flask アプリケーションの全体が完成です！Todo CRUD API と LINE Webhook 処理という二つの側面を持つアプリになりました。それでは、実際にアプリを起動して動作を確認してみましょう。

アプリの起動と動作確認

まず、Flask アプリを起動する準備として、エントリーポイントのスクリプトを用意します。run.py というファイルをプロジェクトルートに作成し、以下を記述します：

from app import create_app, todo_service

# アプリケーションファクトリから Flask アプリを生成

app = create_app()

# グローバルにサービスをアプリに持たせておく（LINE 処理用に利用）

app.todo_service = todo_service

if **name** == "**main**": # Flask 開発サーバーを起動
app.run(host="0.0.0.0", port=5000, debug=True)

ここでは create_app()を呼び出して Flask アプリを生成し、必要に応じてアプリに属性をセットしています（app.todo_service としてサービス関数群を登録）。**name** == "**main**"ブロックで app.run()を呼び、デバッグモード有効で起動しています。

それでは仮想環境下で以下のコマンドを実行し、Flask アプリを起動しましょう：

(venv) $ python run.py

- Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
- Restarting with stat
- Debugger is active!

アプリが起動したら、まず Todo リスト API の動作を確認します。別のターミナル（または同じターミナルで curl コマンド）から、Flask が動いているローカルサーバーに対して HTTP リクエストを送ります。

例として、以下の手順で CRUD を一通り試せます： 1. （READ）Todo 一覧取得:

$ curl -X GET http://127.0.0.1:5000/todos/

初回はまだ何も登録していないので、[]（空のリスト）が返ってくるはずです。

    2.	（CREATE）Todo追加:

$ curl -X POST -H "Content-Type: application/json" \
 -d '{"content": "牛乳を買う"}' \
 http://127.0.0.1:5000/todos/

Todo が追加されると、201 Created とともに追加された Todo の JSON が返ってきます。例：
{"id": 1, "content": "牛乳を買う", "done": 0, "created_at": "2025-04-11 17:35:00"}

    3.	（UPDATE）Todo更新:

$ curl -X PUT -H "Content-Type: application/json" \
 -d '{"done": 1}' \
 http://127.0.0.1:5000/todos/1/

上記は ID=1 の Todo を完了済みに更新するリクエストです。成功すると更新後の JSON データが返ります。例えば"done": 1 に変わっていることが確認できます。

    4.	（DELETE）Todo削除:

$ curl -X DELETE http://127.0.0.1:5000/todos/1/

成功すると{"result": "deleted", "id": 1}が返ります（もしくは 200 OK）。もう一度 GET で一覧を取得すると空に戻っているはずです。

これらの操作が期待通り動作すれば、Todo CRUD API は正常です。次に、LINE Bot としての動作を確認しましょう。

LINE Developers コンソールで Webhook URL を設定した際に ngrok を使った場合、ngrok を起動してトンネルを張っておきます。例えば：

$ ngrok http 5000

と実行すると、Forwarding 欄にhttps://xxxxxxxx.ngrok.io -> http://localhost:5000 という表示が得られます。このhttps://xxxxxxxx.ngrok.io/line/webhookをWebhook URL に設定している前提です。また、スマホの LINE アプリで Bot（公式アカウント）を友だち追加済みとします。

準備ができたら、スマホの LINE から Bot 宛てにメッセージを送ってみましょう。例えば：
• 「ADD 本を読む」と送信してみてください。→ Flask アプリが Webhook を受け取り、新しい Todo「本を読む」を追加します。そして LINE 返信として「Todo 追加: (id=…) 本を読む」とメッセージが届くはずです。
• 続けて「ADD 宿題をする」と送信。→ 2 つ目の Todo が追加され、確認メッセージが届きます。
• 「LIST」と送信。→ Bot から現在の Todo 一覧が返信されます。追加した 2 件がリスト表示されていることを確認してください。
• 「DELETE 1」と送信。→ id=1 の Todo を削除します。Bot から「Todo id=1 を削除しました。」と返信が来て、実際に削除されたことを確認できます。再度「LIST」を送ると残りの 1 件だけが表示されるでしょう。

このように、LINE のチャットから Todo の追加・一覧・削除が操作できれば成功です。もし動作しない場合は、以下を確認してください：
• Flask のログにエラーが出ていないか（署名不一致やコードの例外など）。
• ngrok のトンネルが正しく動いているか（ngrok のコンソールにリクエスト履歴が出ます）。
• LINE Developers コンソールで Webhook が有効か、チャネルアクセストークン・シークレットを正しくコードに設定したか。

最後に、本ハンズオンで作成した Flask アプリケーションの構造について、学んだポイントを整理します。

まとめ：Flask アプリ構造のポイント

今回のハンズオンでは、Flask を用いた Todo リストアプリと LINE Bot 連携を実装する中で、Flask のプロジェクト構成やアーキテクチャについて多くの点を学びました。
• アプリケーションファクトリ：create_app()関数で Flask アプリを生成し設定するパターンを採用しました。これにより、設定の分離やテストの容易さ、複数インスタンスの生成が可能になります ￼。Flask 公式でも推奨される手法であり、flask run コマンドはデフォルトで create_app を探す機能も持っています ￼。
• Blueprint によるモジュール分割：ビュー関数（ルート処理）を Blueprint 単位でグループ化しました。Todo 管理機能と LINE Webhook 機能を別々の Blueprint(todo_routes と line_routes)にしたことで、各機能のコードを分離して見通し良く管理できます。Blueprint には名前と URL プレフィックスを設定し、register_blueprint でアプリにまとめています。Blueprint は実際のアプリではなく、アプリの「設計図」のようなものとして、関連するビューや静的ファイル、テンプレート等をまとめる仕組みです ￼。大規模アプリでは Blueprint での分割が必須と言えるほど有用です。
• MVC 的な構造：Flask 自体は厳密な MVC フレームワークではありませんが、本ハンズオンでは Model（データ層）、View（テンプレート）、Controller（ルーティング）に相当する責務分担を意識しました。todo_service.py や models.py がデータアクセスやビジネスロジック（Model）、routes/\*.py が HTTP リクエストを受け付けて結果を返すコントローラ（Controller）に当たります。今回はテンプレートを用いた画面表示は行わなかったため View は JSON レスポンスとなりましたが、Flask では templates/ディレクトリに HTML ファイルを置き、render_template することで View を描画できます。Blueprint ごとにテンプレートフォルダを分けることも可能です ￼。
• サービス層の分離：ビュー関数内で直接 DB 操作を書くのではなく、todo_service モジュールに処理を委譲しました。これにより、Web 以外（例えば他のスクリプトやタスクスケジューラ）からも Todo 操作ロジックを再利用しやすくなります。また単体テストを書く場合にも、サービス関数を直接テストできるため有用です。小規模な API であれば直接 SQL を書くこともありますが、ビジネスロジックが複雑になるほどこの分離が効いてきます。
• 設定とシークレット管理：今回は簡略化のためコード中にシークレットキーやパスを直接書きましたが、実際の開発では config.py や環境変数、.env ファイル等で管理するのが望ましいです。特にチャネルアクセストークンやシークレットなど機密情報はソースコードから分離することを推奨します。

以上、Flask の基本的な構造から外部 API 連携まで、一連の流れを体験しました。お疲れさまでした！このハンズオンを通じて、Flask アプリの作り方やディレクトリ構成、API 実装方法、さらに Webhook を用いた外部サービスとの連携方法について理解が深まったと思います。今後はこの基礎をもとに、更に機能を拡張したり、別のデータベースや本番環境デプロイ、認証機能の追加などに挑戦してみてください。
