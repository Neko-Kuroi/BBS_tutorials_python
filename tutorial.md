# FastAPI + SQLite で作る掲示板チュートリアル

## このチュートリアルについて

このチュートリアルでは、FastAPIとSQLiteを使ったシンプルな掲示板アプリの設計と実装を解説します。

外部スクリプトからWebhookでニュース記事をスレッドとして投稿し、ユーザーがコメントを書き込めるスレッドフロート型の掲示板です。ログイン機能のないシンプルな構成を出発点に、全文検索・閲覧数・up/down評価・管理者機能を持つ設計になっています。

**対象読者**

- PythonでWebアプリを作ったことがある
- FastAPIは初めて
- SQLiteのFTS全文検索は初めて

**この設計の方針**

- **匿名の入力は信用しない**: 表示するときは必ずエスケープし、入力の長さにも上限を設けます
- **レス番号は固定する**: コメントを非表示・削除しても、他のコメントの番号は変わりません
- **管理操作は論理削除が基本**: `status` を変えるだけで、元に戻せます

---

## 目次

- [このチュートリアルについて](#このチュートリアルについて)
- [動かしてみよう](#動かしてみよう)
- [技術スタック](#技術スタック)
- [ファイル構成](#ファイル構成)
- [ページ構成](#ページ構成)
- [DB設計の全体像](#db設計の全体像)
- [共通部品](#共通部品)
- [起動ファイル](#起動ファイル)
- [01. データベースの設計と初期化](#01-データベースの設計と初期化)
- [02. Webhookでスレッドを作成する](#02-webhookでスレッドを作成する)
- [03. ユーザーの書き込み処理](#03-ユーザーの書き込み処理)
- [04. スレッドの表示処理](#04-スレッドの表示処理)
- [05. 全文検索](#05-全文検索)
- [06. スレッド一覧とソート](#06-スレッド一覧とソート)
- [07. 投票機能](#07-投票機能)
- [08. 管理者機能](#08-管理者機能)
- [付録1. レス番号と削除の仕様](#付録1-レス番号と削除の仕様)
- [付録2. 既存DBからの移行](#付録2-既存dbからの移行)
- [付録3. PostgreSQLへ移植するときの注意](#付録3-postgresqlへ移植するときの注意)
- [付録4. 運用メモ](#付録4-運用メモ)

---

## 動かしてみよう

### 1. 動作環境

- Python 3.9 以上
- SQLite 3.34.0 以上（FTS5の`trigram`トークナイザに必要です）

SQLiteのバージョンとtrigramの対応は、起動前に確認できます。

```python
import sqlite3

print("SQLite", sqlite3.sqlite_version)
conn = sqlite3.connect(":memory:")
try:
    conn.execute("CREATE VIRTUAL TABLE t USING fts5(a, tokenize='trigram')")
    print("✅ trigram 利用可能")
except sqlite3.OperationalError as e:
    print(f"❌ trigram 非対応: {e}")
```

### 2. インストール

```bash
pip install fastapi uvicorn python-multipart
```

`python-multipart` は `Form()` でHTMLフォームを受け取るために**必須**です。入れ忘れると、起動時に `Form data requires "python-multipart" to be installed` というエラーになります。

### 3. 起動

```bash
export BBS_API_KEY=your-api-key       # Webhook用のキー
export BBS_ADMIN_KEY=your-admin-key   # 管理者用のキー（Webhook用とは別にする）
python app.py                         # → http://localhost:8000
```

### 4. 動作確認

```bash
# サーバーの確認
curl http://localhost:8000/webhook/test

# スレッドを作る
curl -X POST http://localhost:8000/webhook/news \
  -H "X-API-Key: your-api-key" -H "Content-Type: application/json" \
  -d '{"title": "テストニュース", "body": "本文です"}'

# コメントを書き込む
curl -X POST http://localhost:8000/thread/1/post -d "name=taro" -d "content=最初のコメント"
```

ブラウザで `http://localhost:8000/` を開くと、作ったスレッドが見えます。

---

## 技術スタック

```
FastAPI           Webフレームワーク（Python）
SQLite            データベース（FTS5全文検索を使用）
uvicorn           ASGIサーバー
Pydantic          リクエストの型チェック
python-multipart  HTMLフォームの受け取り
```

---

## ファイル構成

```
app.py          起動ファイル・ルーター統合
config.py       定数（キー・ページ件数・DBパス）
db.py           DB接続とトランザクションの管理
utils.py        エスケープ・日時・ページネーション・共通レイアウト
auth.py         APIキー認証
init_db.py      DB初期化
webhook.py      Webhookエンドポイント
post.py         書き込み処理
thread.py       スレッド表示
search.py       全文検索
index.py        スレッド一覧
vote.py         up/down評価
admin.py        管理者機能
ascii_art.py    （任意）装飾用のアスキーアート。本文では扱いません
```

---

## ページ構成

```
GET   /                           スレッド一覧（is_visible=1のみ）
GET   /thread/{id}                スレッド表示（webhook別枠 + コメント一覧）
POST  /thread/{id}/post           コメントの書き込み
POST  /post/{id}/vote             up/down評価
GET   /search                     全文検索（FTS5 trigram + LIKEハイブリッド）
POST  /webhook/news               スレッド自動作成（APIキー認証）
PATCH /admin/{target}/{id}/status 状態の変更（管理者キー認証）
```

---

## DB設計の全体像

```
threads             スレッド（post_count = これまでに振った最大のレス番号）
  ├── webhook_posts    Webhookニュース本文（1件）
  ├── user_posts       ユーザーコメント（res_no = 固定のレス番号）
  │     └── votes      up/down評価（voter_id で1人1票）
  ├── posts_fts        全文検索インデックス（仮想テーブル）
  └── all_posts        webhook_posts と user_posts を束ねたビュー
```

---

## 共通部品

各章のコードが共通で使う部品です。先にここを作っておきます。

### config.py

```python
# file: config.py
import os

DB_PATH = os.environ.get("BBS_DB_PATH", "bbs.db")
API_KEY = os.environ.get("BBS_API_KEY", "")
ADMIN_KEY = os.environ.get("BBS_ADMIN_KEY", "")

POSTS_PER_PAGE = 20
THREADS_PER_PAGE = 20
SEARCH_RESULTS_PER_PAGE = 20
```

キーはコードに直書きせず、環境変数から読みます。Gitにキーを載せてしまう事故を防げます。未設定のときは空文字になり、`auth.py` 側で「空のキーは常に拒否」とします。

### db.py（接続とトランザクション）

```python
# file: db.py
import sqlite3
from contextlib import closing, contextmanager

from config import DB_PATH


@contextmanager
def get_conn(write: bool = False):
    """接続を開き、トランザクション付きで渡し、終わったら必ず閉じる。"""
    with closing(sqlite3.connect(DB_PATH, timeout=10, isolation_level=None)) as conn:
        conn.execute("PRAGMA foreign_keys = ON")
        conn.execute("BEGIN IMMEDIATE" if write else "BEGIN")
        try:
            yield conn
            conn.execute("COMMIT")
        except BaseException:
            conn.execute("ROLLBACK")
            raise
```

使い方は次の2通りです。

```python
with get_conn() as conn:              # 読み取りだけ
    rows = conn.execute("SELECT ...").fetchall()

with get_conn(write=True) as conn:    # 書き込みを含む（読んでから書く処理もこちら）
    conn.execute("UPDATE ...")
```

それぞれの設定には理由があります。

| 設定 | 理由 |
|---|---|
| `closing(...)` | `with sqlite3.connect() as conn:` は**コミット／ロールバックをするだけで、接続は閉じません**。`closing` を組み合わせて確実に閉じます |
| `PRAGMA foreign_keys = ON` | SQLiteは**接続ごとに**この設定をしないと外部キー制約が無効です。これがないと `ON DELETE CASCADE` が動きません |
| `isolation_level=None` | Pythonの `sqlite3` が裏で行う自動的な `BEGIN` を止めます。トランザクションの開始位置を自分で決められます |
| `BEGIN IMMEDIATE` | 開始時に書き込みロックを取ります。「読んでから書く」処理（レス番号の採番など）が同時に走っても、順番に実行されます |
| `timeout=10` | 他の接続がロックを持っているとき、最大10秒待ってからエラーにします |
| `except BaseException` | `HTTPException` が途中で発生しても `ROLLBACK` されるので、中途半端なデータが残りません |

### utils.py（エスケープ・日時・ページネーション・レイアウト）

```python
# file: utils.py
from datetime import datetime, timedelta, timezone
from html import escape
from typing import Callable, Optional

JST = timezone(timedelta(hours=9))


def utc_now() -> str:
    """DBに保存する日時。常にUTCで、SQLiteのdatetime('now')と同じ書式にそろえる。"""
    return datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M:%S")


def fmt_time(s: Optional[str]) -> str:
    """表示用にUTC→JSTへ変換する。"""
    if not s:
        return ""
    try:
        dt = datetime.strptime(s, "%Y-%m-%d %H:%M:%S").replace(tzinfo=timezone.utc)
        return dt.astimezone(JST).strftime("%Y-%m-%d %H:%M")
    except ValueError:
        return s


def esc(s) -> str:
    """HTMLに埋め込む文字列は必ずこれを通す。"""
    return escape("" if s is None else str(s))


def render_body(text: str) -> str:
    """本文用: ①エスケープ → ②改行を<br>に。順序を逆にすると<br>まで潰れる。"""
    return esc(text).replace("\r\n", "\n").replace("\n", "<br>")


def safe_url(url: Optional[str]) -> Optional[str]:
    """http(s)以外（javascript:など）はリンクにしない。"""
    if url and url.lower().startswith(("http://", "https://")):
        return url
    return None


def render_pagination(current: int, total: int, href: Callable[[int], str]) -> str:
    """現在のページを中心に、最大10ページ分のリンクを作る。"""
    if total <= 1:
        return ""
    start = max(1, min(current - 5, total - 9))
    end = min(total, start + 9)
    parts = []
    for p in range(start, end + 1):
        if p == current:
            parts.append(f'<span class="current">{p}</span>')
        else:
            parts.append(f'<a href="{esc(href(p))}">{p}</a>')
    return '<div class="pagination">' + " ".join(parts) + "</div>"


def layout(title: str, body: str) -> str:
    return f"""<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>{esc(title)}</title>
<style>
  body {{ font-family: sans-serif; max-width: 860px; margin: 0 auto; padding: 12px; line-height: 1.6; }}
  header {{ display: flex; gap: 12px; align-items: center; margin-bottom: 16px; }}
  .thread-item, .post, .webhook-post {{ border: 1px solid #ccc; border-radius: 6px; padding: 8px 12px; margin: 8px 0; }}
  .webhook-post {{ background: #f3f7ff; }}
  .meta, .post-head {{ color: #666; font-size: 0.85em; }}
  .no {{ font-weight: bold; margin-right: 6px; }}
  .pagination a, .pagination .current {{ margin: 0 3px; }}
  .current {{ font-weight: bold; }}
  textarea {{ width: 100%; height: 6em; }}
  form.vote {{ display: inline; }}
</style>
</head>
<body>
<header>
  <a href="/">📋 掲示板</a>
  <form action="/search" method="get"><input name="q" maxlength="100" placeholder="検索"><button>検索</button></form>
</header>
{body}
</body>
</html>"""
```

押さえておきたい点は次の3つです。

**1. エスケープ（XSS対策）**
匿名で書き込める掲示板では、`<script>` を含むコメントが投稿される前提で作ります。DB由来の文字列（タイトル・名前・本文・日時）は、HTMLに埋め込む前に必ず `esc()` か `render_body()` を通します。

**2. 日時はUTCで保存し、表示のときにJSTへ変換**
SQLiteの `datetime('now')` はUTCです。アプリ側がローカル時刻で保存すると、時刻の比較（重複チェックなど）がずれます。保存はUTC、表示だけ変換、と分けます。

**3. `source_url` のスキーム検証**
`javascript:alert(1)` のようなURLをリンクにされると、クリックしただけでスクリプトが実行されます。`safe_url()` で `http(s)` だけを通します。

### auth.py（APIキー認証）

```python
# file: auth.py
import secrets

from fastapi import Header, HTTPException

from config import ADMIN_KEY, API_KEY


def _check(given: str, expected: str) -> None:
    # 空のキー（環境変数の設定漏れ）は常に拒否する
    if not expected or not secrets.compare_digest(given.encode(), expected.encode()):
        raise HTTPException(status_code=403, detail="Invalid API Key")


def verify_api_key(x_api_key: str = Header(...)) -> str:
    _check(x_api_key, API_KEY)
    return x_api_key


def verify_admin_key(x_api_key: str = Header(...)) -> str:
    _check(x_api_key, ADMIN_KEY)
    return x_api_key
```

- `secrets.compare_digest` は、通常の `==` と違い、処理時間が一致位置に依存しません。処理時間の差からキーを推測する**タイミング攻撃**を防げます。
- `.encode()` で `bytes` にして渡しているのは、非ASCII文字を含む `str` を渡すと `TypeError`（500エラー）になるためです。
- ヘッダー自体が無いリクエストは、`Header(...)`（必須）の検証で **422** になります。**403になるのは、ヘッダーはあるがキーが違うとき**です。
- Webhook用（`API_KEY`）と管理者用（`ADMIN_KEY`）は**別のキー**にします。Webhook用のキーが漏れても、管理操作はされません。

---

## 起動ファイル

各機能は独立したファイルに分かれており、`app.py` で一本に組み上げます。

```python
# file: app.py
from contextlib import asynccontextmanager

from fastapi import FastAPI

from init_db import init_db
import index, webhook, thread, post, search, vote, admin


@asynccontextmanager
async def lifespan(app: FastAPI):
    init_db()   # 起動時にDBを初期化
    yield


app = FastAPI(lifespan=lifespan)

app.include_router(index.router)
app.include_router(webhook.router)
app.include_router(thread.router)
app.include_router(post.router)
app.include_router(search.router)
app.include_router(vote.router)
app.include_router(admin.router)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000, log_level="warning")
```

`lifespan` はFastAPIの起動・終了時の処理を定義する仕組みです。`yield` の前が起動時、後が終了時の処理です。ここでは起動時に `init_db()` を呼んでDBを準備しています。

`include_router` で各ファイルのルーターを登録します。各ファイルは `APIRouter()` を使って独立したルートを定義しており、`app.py` はそれを束ねるだけです。

### `async def` と `def` の使い分け

このチュートリアルのエンドポイントは、すべて `async def` ではなく **`def`** で書きます。

```python
# ❌ SQLiteの処理を待つあいだ、イベントループ全体が止まる
@router.get("/")
async def index(): 
    with get_conn() as conn: ...

# ✅ FastAPIが自動でスレッドプールで実行する。他のリクエストを止めない
@router.get("/")
def index():
    with get_conn() as conn: ...
```

`async def` は、中で `await` を使うときのためのものです。`sqlite3` は同期ライブラリなので、`async def` の中で呼ぶと、DBの処理中は他のリクエストが全員待たされます。`def` にすれば、FastAPIが別スレッドで実行してくれます。接続はリクエストごとに `get_conn()` で作って閉じるので、スレッド間で共有されず安全です。

---

## 01. データベースの設計と初期化

対象ファイル: `init_db.py`

### init_db.py の役割

アプリが起動するときに一番最初に呼ばれる関数 `init_db()` を定義しています。SQLiteのデータベースファイル（`bbs.db`）を作成し、必要なテーブル・トリガー・インデックスをすべて準備します。

### テーブル構成の全体像

```
threads          スレッド（話題のまとまり）
webhook_posts    Webhookで投稿されたニュース本文（スレッドに1件）
user_posts       ユーザーが書き込んだコメント（スレッドに複数）
votes            user_postsへのup/down評価
posts_fts        全文検索用インデックス（仮想テーブル）
all_posts        webhook_posts と user_posts を束ねたビュー
```

### init_db.py の全体

```python
# file: init_db.py
import sqlite3
from contextlib import closing

from config import DB_PATH
from db import get_conn


def init_db():
    # WALモードはトランザクションの外で、1回だけ切り替える（DBファイルに保存される設定）
    with closing(sqlite3.connect(DB_PATH)) as conn:
        conn.execute("PRAGMA journal_mode = WAL")

    with get_conn(write=True) as conn:
        _create_tables(conn)
        _create_search(conn)
        _create_triggers(conn)
        _create_indexes(conn)


def _create_tables(conn):
    conn.execute('''
        CREATE TABLE IF NOT EXISTS threads (
            id           INTEGER PRIMARY KEY AUTOINCREMENT,
            title        TEXT NOT NULL,
            created_at   TEXT NOT NULL,
            last_post_at TEXT,
            views        INTEGER NOT NULL DEFAULT 0,
            post_count   INTEGER NOT NULL DEFAULT 0,
            is_visible   INTEGER NOT NULL DEFAULT 1,
            status       TEXT NOT NULL DEFAULT 'public'
        )
    ''')

    conn.execute('''
        CREATE TABLE IF NOT EXISTS webhook_posts (
            id         INTEGER PRIMARY KEY AUTOINCREMENT,
            thread_id  INTEGER NOT NULL,
            author     TEXT DEFAULT 'ニュースBot',
            body       TEXT NOT NULL,
            source_url TEXT,
            created_at TEXT NOT NULL,
            is_visible INTEGER NOT NULL DEFAULT 1,
            status     TEXT NOT NULL DEFAULT 'public',
            FOREIGN KEY (thread_id) REFERENCES threads (id) ON DELETE CASCADE
        )
    ''')

    conn.execute('''
        CREATE TABLE IF NOT EXISTS user_posts (
            id         INTEGER PRIMARY KEY AUTOINCREMENT,
            thread_id  INTEGER NOT NULL,
            res_no     INTEGER NOT NULL,
            name       TEXT DEFAULT 'anonymous',
            content    TEXT NOT NULL,
            created_at TEXT NOT NULL,
            is_visible INTEGER NOT NULL DEFAULT 1,
            status     TEXT NOT NULL DEFAULT 'public',
            FOREIGN KEY (thread_id) REFERENCES threads (id) ON DELETE CASCADE,
            UNIQUE (thread_id, res_no)
        )
    ''')

    conn.execute('''
        CREATE TABLE IF NOT EXISTS votes (
            id         INTEGER PRIMARY KEY AUTOINCREMENT,
            post_id    INTEGER NOT NULL,
            voter_id   TEXT NOT NULL,
            vote_type  TEXT NOT NULL CHECK (vote_type IN ('up', 'down')),
            created_at TEXT NOT NULL,
            FOREIGN KEY (post_id) REFERENCES user_posts (id) ON DELETE CASCADE,
            UNIQUE (post_id, voter_id)
        )
    ''')


def _create_search(conn):
    conn.execute('''
        CREATE VIRTUAL TABLE IF NOT EXISTS posts_fts USING fts5(
            content,
            source_type UNINDEXED,
            post_id UNINDEXED,
            thread_title,
            author_name,
            tokenize='trigram'
        )
    ''')

    # 2つのテーブルを「1つの投稿テーブル」として扱うためのビュー
    conn.execute('''
        CREATE VIEW IF NOT EXISTS all_posts AS
            SELECT 'user'    AS source_type, id AS post_id, thread_id,
                   content AS content, is_visible
            FROM user_posts
            UNION ALL
            SELECT 'webhook' AS source_type, id AS post_id, thread_id,
                   body AS content, is_visible
            FROM webhook_posts
    ''')


def _create_triggers(conn):
    # status を変えたら is_visible を自動で連動させる
    for table in ('threads', 'webhook_posts', 'user_posts'):
        conn.execute(f'''
            CREATE TRIGGER IF NOT EXISTS {table}_status_sync
            AFTER UPDATE OF status ON {table}
            BEGIN
                UPDATE {table}
                SET is_visible = CASE WHEN NEW.status = 'public' THEN 1 ELSE 0 END
                WHERE id = NEW.id;
            END
        ''')

    # コメント追加時: 検索インデックスに登録 + スレッドの最終更新日時を更新
    conn.execute('''
        CREATE TRIGGER IF NOT EXISTS user_posts_after_insert
        AFTER INSERT ON user_posts
        BEGIN
            INSERT INTO posts_fts(content, source_type, post_id, thread_title, author_name)
            SELECT NEW.content, 'user', NEW.id, t.title, NEW.name
            FROM threads t WHERE t.id = NEW.thread_id;

            UPDATE threads SET last_post_at = NEW.created_at WHERE id = NEW.thread_id;
        END
    ''')

    # ニュース本文追加時: 検索インデックスに登録
    conn.execute('''
        CREATE TRIGGER IF NOT EXISTS webhook_posts_after_insert
        AFTER INSERT ON webhook_posts
        BEGIN
            INSERT INTO posts_fts(content, source_type, post_id, thread_title, author_name)
            SELECT NEW.body, 'webhook', NEW.id, t.title, NEW.author
            FROM threads t WHERE t.id = NEW.thread_id;
        END
    ''')

    # 物理削除時: 検索インデックスからも消す（CASCADEではFTSの行は消えない）
    conn.execute('''
        CREATE TRIGGER IF NOT EXISTS user_posts_after_delete
        AFTER DELETE ON user_posts
        BEGIN
            DELETE FROM posts_fts WHERE source_type = 'user' AND post_id = OLD.id;
        END
    ''')
    conn.execute('''
        CREATE TRIGGER IF NOT EXISTS webhook_posts_after_delete
        AFTER DELETE ON webhook_posts
        BEGIN
            DELETE FROM posts_fts WHERE source_type = 'webhook' AND post_id = OLD.id;
        END
    ''')


def _create_indexes(conn):
    conn.execute('CREATE INDEX IF NOT EXISTS idx_webhook_posts_thread_id ON webhook_posts(thread_id)')
    conn.execute('CREATE INDEX IF NOT EXISTS idx_threads_last_post ON threads(last_post_at)')
    conn.execute('CREATE INDEX IF NOT EXISTS idx_threads_views ON threads(views)')
```

以降は、この中の要点を順に解説します。

### 各テーブルの詳細

#### threads（スレッド）

| カラム | 説明 |
|--------|------|
| `id` | スレッドの一意なID。自動採番 |
| `title` | スレッドのタイトル |
| `created_at` | スレッド作成日時（UTC） |
| `last_post_at` | 最後にコメントされた日時。更新順ソートに使う |
| `views` | 閲覧数。スレッドページを開くたびに+1 |
| `post_count` | **これまでに振った最大のレス番号**。次のコメントは `post_count + 1` 番になる（03章） |
| `is_visible` | 表示フラグ。`1`=表示、`0`=非表示 |
| `status` | 管理用の状態。`public` / `hidden_temp` / `hidden_perm` / `spam` |

`is_visible` と `status` の使い分けは次の通りです。

```
is_visible → 表示・非表示の高速判定に使う（WHERE is_visible = 1）
status     → 管理者が非表示にした理由や状態を把握するために使う

連動ルール
  status = 'public'      → is_visible = 1
  status = 'hidden_temp' → is_visible = 0（一時非表示、戻す予定あり）
  status = 'hidden_perm' → is_visible = 0（恒久非表示、削除候補）
  status = 'spam'        → is_visible = 0（スパム判定）
```

#### webhook_posts（Webhookニュース本文）

外部スクリプトからWebhookでスレッドを作成したときの本文が入ります。1スレッドにつき1件だけ存在します。スレッドページでは別枠として表示され、コメントのレス番号には含まれません。

`FOREIGN KEY ... ON DELETE CASCADE` は、親のスレッド（threads）が削除されたとき、このテーブルの関連レコードも自動で削除される設定です。ただしSQLiteでは、**接続ごとに `PRAGMA foreign_keys = ON` を実行しないと外部キー制約そのものが無効**です。`db.py` の `get_conn()` がそれを毎回実行しています。

物理削除すると連鎖的に全データが消えるため、管理者画面では `status` を変更する論理削除をデフォルトにします（08章）。

#### user_posts（ユーザーコメント）

ユーザーがフォームから書き込んだコメントが入ります。

**`res_no`（レス番号）はこのテーブルに固定で保存します。** スレッドごとに1, 2, 3…と振り、一度振った番号は変わりません。コメントを非表示にしても削除しても、他のコメントの番号はずれないので、`>>12` のような参照や `#post-12` のアンカーが壊れません。非表示にしたコメントの番号は「欠番」になります。

`UNIQUE (thread_id, res_no)` は、同じスレッドで同じ番号が重複しないようにするDB側の安全網です。同時に書き込みが来ても、片方はエラーになるので、番号が重複したデータは残りません。このUNIQUE制約はインデックスとしても働くため、スレッドごとのコメント取得や、番号の範囲検索に使われます。

#### votes（評価）

`user_posts` の各コメントへのup/down評価を記録します。`CHECK (vote_type IN ('up', 'down'))` で、想定外の値が入らないようにSQLite側で制約をかけています。

`UNIQUE (post_id, voter_id)` が「1人1コメントにつき1票」の本体です。`voter_id` はログインの代わりにCookieで発行する匿名IDです（07章）。将来ログイン機能を追加したときは、`voter_id` を `user_id` に置き換えるだけで済みます。

### FTS5 全文検索テーブル（posts_fts）

FTS5（Full Text Search 5）はSQLiteに組み込まれた全文検索エンジンです。通常の `LIKE` 検索とは異なり、専用のインデックスを使って高速に検索できます。

#### VIRTUAL TABLEとは

通常のテーブルとは異なり、実データを持たずインデックス情報だけを管理する特殊なテーブルです。`USING fts5` でFTS5エンジンを使うことを指定します。

#### tokenize='trigram' とは

テキストを3文字ずつ区切ってインデックスを作る方式です。日本語は英語と違って単語の境界（スペース）がないため、英語向けの `porter` などのtokenizerでは日本語の検索ができません。trigramは文字単位で区切るため日本語に適しています。

```
例：「東京で大雪」をtrigramで分割すると
→「東京で」「京で大」「で大雪」という3文字の組み合わせでインデックス化
→「東京で」を含む検索でヒットする
```

ただしtrigramは3文字以上のキーワードにしか対応できません。2文字以下のキーワードは `LIKE` 検索にフォールバックします（05章）。`tokenize='trigram'` には **SQLite 3.34.0以降**が必要です。

#### UNINDEXED とは

`source_type UNINDEXED` と `post_id UNINDEXED` は、これらのカラムを検索対象にしない（インデックスを作らない）という指定です。この2つは、検索結果からもとのレコードを特定するためだけに使うので、インデックス化する必要がありません。

```sql
-- ✅ JOINには使える（もとのレコードを特定する）
SELECT ... FROM posts_fts JOIN all_posts ap
  ON ap.source_type = posts_fts.source_type AND ap.post_id = posts_fts.post_id

-- ⚠️ WHERE検索には向かない（インデックスが効かず、全件を走査する）
SELECT * FROM posts_fts WHERE post_id = 123
```

### ビュー all_posts と「IDの衝突」

`user_posts` と `webhook_posts` は**別々のテーブルで、別々に `AUTOINCREMENT` が振られます**。そのため、どちらにも `id = 1` の行が存在します。

```
user_posts      id=1, 2, 3 ...
webhook_posts   id=1, 2, 3 ...    ← 同じ番号が両方に存在する
```

`post_id` だけで結合すると、webhookの `id=5` のヒットが、user_postsの `id=5` にも結びついてしまい、ヒット数が水増しされたり、別の投稿の本文が表示されたりします。

そこで、`posts_fts` には `source_type`（`'user'` か `'webhook'`）も一緒に保存し、**結合は必ず `(source_type, post_id)` の組で行います**。`all_posts` ビューは、その組を持った「2つのテーブルを縦に束ねた仮想の1テーブル」です。

`UNION ALL` は結果をそのまま縦に連結します（`UNION` は重複を除去する分だけ遅い）。`source_type` が違えば同じ行にならないので、`UNION ALL` で問題ありません。

### トリガー

トリガーとは、テーブルへのINSERT/UPDATE/DELETEをきっかけに自動で別の処理を実行する仕組みです。

| トリガー | きっかけ | 自動でやること |
|---|---|---|
| `*_status_sync`（3つ） | `status` の更新 | `is_visible` を連動させる |
| `user_posts_after_insert` | コメントのINSERT | `posts_fts` に登録、`threads.last_post_at` を更新 |
| `webhook_posts_after_insert` | ニュース本文のINSERT | `posts_fts` に登録 |
| `*_after_delete`（2つ） | 物理削除 | `posts_fts` から削除 |

`NEW` は今INSERT／UPDATEされた後のレコード、`OLD` は削除される前のレコードを指します。

`status` の同期トリガーは、管理者が `status` を変更したとき `is_visible` が自動で連動するので、アプリケーション側でこの同期を書く必要がありません。

削除トリガーが必要な理由は、`ON DELETE CASCADE` が消してくれるのは**外部キーでつながった行だけ**で、`posts_fts`（仮想テーブル）の行は消えないからです。物理削除したコメントが検索インデックスに残らないよう、トリガーで掃除します。`post_id` は `UNINDEXED` なので全件走査になりますが、物理削除は稀な操作という前提で許容します。

### インデックス

| インデックス | 高速化される操作 |
|---|---|
| `UNIQUE (thread_id, res_no)`（`user_posts`） | スレッドのコメント取得、レス番号の範囲検索 |
| `UNIQUE (post_id, voter_id)`（`votes`） | 投票数の集計、投票の重複チェック |
| `idx_webhook_posts_thread_id` | スレッドのニュース本文取得 |
| `idx_threads_last_post` | 更新順ソート |
| `idx_threads_views` | 閲覧数順ソート |

`UNIQUE` 制約は、自動でインデックスも作ります。

`is_visible` 単体のインデックスは作りません。値が0か1しかなく、ほとんどの行が1なので、インデックスで絞り込める量がほぼなく、効果がないためです。また、スレッド一覧の `new` / `old` ソートは `created_at` ではなく `id` で並べるので、`created_at` のインデックスも不要です。

### CREATE ... IF NOT EXISTS について

テーブル・ビュー・トリガー・インデックスのすべてに `IF NOT EXISTS` をつけています。すでに存在する場合はエラーにならずスキップするので、アプリを再起動するたびに `init_db()` が呼ばれても、既存のデータが消えることはありません。

トリガーに `IF NOT EXISTS` をつけ忘れると、2回目の起動で `trigger ... already exists` というエラーになります。

### init_db.py のまとめ

```
init_db() がやっていること

1. WALモードに切り替える（読み取りと書き込みが競合しにくくなる）
2. threads / webhook_posts / user_posts / votes テーブルを作成
3. posts_fts 仮想テーブル（FTS5 trigram）と all_posts ビューを作成
4. トリガーを作成（status連動・FTS自動更新・削除時の掃除）
5. インデックスを作成
```

WALモード（Write-Ahead Logging）にすると、書き込み中でも読み取りがブロックされにくくなります。スレッドを開くたびに `views` を更新するこのアプリには向いた設定です。DBファイルの横に `bbs.db-wal` と `bbs.db-shm` というファイルができますが、正常な動作です。

---
