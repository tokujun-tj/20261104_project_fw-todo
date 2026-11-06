# DB設計書：やること管理アプリ

## 1. 文書の目的

この文書は、やること管理アプリが使うデータベース（データを保存する仕組み）の構造を定めるものです。

対象は、MySQL（データベース製品）の `todos` テーブル（表）1つです。1件のやることを、`todos` の1行に保存します。

画面の配置や画面遷移は、別の画面設計書で定めます。

## 2. 設計方針

- 利用者は1人で、ログイン情報や利用者情報は保存しません。
- 作成するテーブルは `todos` だけです。
- `id` は、1件を見分けるための番号です。主キー（重複しない識別番号）として使います。
- `created_at` と `updated_at` は、画面から入力せず、MySQLが自動で設定します。
- `category` と `priority` は、決められた値だけを保存できるようにします。
- 完了日時は保存しません。要件で対象外とされているためです。

## 3. テーブル一覧

| テーブル名 | 役割 | 行の単位 |
|---|---|---|
| `todos` | やること、期限、完了状態などを保存する | 1件のやることにつき1行 |

## 4. `todos` テーブル定義

### 4.1 カラム（列）一覧

| カラム名 | 型 | NULL（値なし） | 初期値 | 制約・内容 |
|---|---|---:|---|---|
| `id` | `BIGINT` | 不可 | 自動採番 | 主キー。1から1ずつ増える、1件を見分ける番号 |
| `title` | `VARCHAR(255)` | 不可 | なし | やること。255文字以内。空白だけの値はアプリ側で拒否 |
| `detail` | `VARCHAR(255)` | 可 | `NULL` | メモ。255文字以内 |
| `category` | `VARCHAR(255)` | 不可 | なし | ジャンル。次の5種類だけ：デザイン、マーケティング、プログラミング、資格、就職活動 |
| `priority` | `INT` | 不可 | `2` | 優先度。`1`＝高、`2`＝中、`3`＝低 |
| `due_date` | `DATE` | 可 | `NULL` | 期限。未入力を許可 |
| `completed` | `BOOLEAN` | 不可 | `FALSE` | 完了状態。`TRUE`＝完了、`FALSE`＝未完了 |
| `created_at` | `DATETIME` | 不可 | 現在日時 | 登録日時。MySQLが登録時に自動設定 |
| `updated_at` | `DATETIME` | 不可 | 現在日時 | 最終更新日時。MySQLが登録時と更新時に自動設定 |

### 4.2 ジャンルの許可値

`category` は自由入力ではなく、次の5種類のいずれかです。

- `デザイン`
- `マーケティング`
- `プログラミング`
- `資格`
- `就職活動`

### 4.3 画面表示との対応

データベースに保存する値と、画面に表示する値が異なる項目は次のとおりです。

| カラム | 保存値 | 画面表示 |
|---|---|---|
| `priority` | `1` / `2` / `3` | 高 / 中 / 低 |
| `completed` | `TRUE` / `FALSE` | 完了 / 未完了 |

## 5. 制約（保存時に守る決まり）

### 5.1 データベースで守る制約

- `id` は主キーにし、同じ番号を保存できないようにします。
- `title`、`category`、`priority`、`completed`、`created_at`、`updated_at` は必須です。
- `detail` と `due_date` は未入力を許可します。
- `priority` は `1`、`2`、`3` のみ許可します。
- `category` は指定された5種類のみ許可します。
- `completed` の初期値は未完了（`FALSE`）にします。
- `priority` の初期値は中（`2`）にします。

### 5.2 アプリケーション（画面や処理）で守る制約

データベースだけでは判断しにくい次の内容は、保存前にアプリケーションでチェックします。

- `title` の空文字と、半角・全角の空白だけの入力を拒否します。
- `title` と `detail` が255文字を超えた入力を拒否します。
- `category` と `priority` が未選択の入力を拒否します。
- `due_date` は未入力を許可します。

## 6. 検索・並び替えでの利用

一覧画面の処理では、次のようにカラムを使います。

| 画面の機能 | 使うカラム | 条件 |
|---|---|---|
| 名前で絞り込み | `title` | 検索語を一部でも含む行を表示（部分一致） |
| ジャンルで絞り込み | `category` | 選択した値と完全に一致する行を表示 |
| 期限が近い順 | `due_date` | 期限の昇順（小さい日付から並べる） |
| 期限が遠い順 | `due_date` | 期限の降順（大きい日付から並べる） |
| 完了状態の表示 | `completed` | `TRUE` は完了、`FALSE` は未完了として表示 |

名前とジャンルの条件がどちらもない場合は、全件を表示します。初回の一覧表示は期限が近い順です。

## 7. 操作ログの扱い

要件では、登録・編集・削除の実行時に、操作の種類と対象の `id` を1行ずつ記録します。`title` と `detail` は記録しません。

ただし、前提によりデータベースのテーブルは `todos` だけとします。そのため、操作ログ用のテーブルは作成せず、アプリケーションのログ（処理の記録）へ次の形式で1行ずつ出力します。

```text
操作の種類,対象id
```

例：

```text
登録,12
編集,12
削除,12
```

このログは `todos` の保存データには含めません。

## 8. CREATE TABLE文（DDL）

DDL（テーブルを作るためのSQL文）は次のとおりです。MySQL 8.0.16以降で実行する想定です。

```sql
CREATE TABLE todos (
    id BIGINT NOT NULL AUTO_INCREMENT,
    title VARCHAR(255) NOT NULL,
    detail VARCHAR(255) NULL,
    category VARCHAR(255) NOT NULL,
    priority INT NOT NULL DEFAULT 2,
    due_date DATE NULL,
    completed BOOLEAN NOT NULL DEFAULT FALSE,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    CONSTRAINT chk_todos_category
        CHECK (category IN (
            'デザイン',
            'マーケティング',
            'プログラミング',
            '資格',
            '就職活動'
        )),
    CONSTRAINT chk_todos_priority
        CHECK (priority IN (1, 2, 3))
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_unicode_ci;
```

`utf8mb4` は日本語などの文字を保存するための文字コード（文字の保存方法）です。`InnoDB` はMySQLで一般的に使われる保存エンジン（データの管理方式）です。

## 9. 要件とカラムの対応表

| 要件番号 | 要件の内容 | 対応するカラム・設計 |
|---|---|---|
| F-01 | やることを一覧表示する | `id`、`title`、`category`、`priority`、`due_date`、`completed` |
| F-02 | 新しいやることを登録する | `title`、`detail`、`category`、`priority`、`due_date`、`completed` |
| F-03 | 登録済みのやることを編集する | `id` で対象を特定し、編集対象の各カラムを更新 |
| F-04 | 登録済みのやることを削除する | `id` で削除対象を特定 |
| F-05 | 名前で部分一致検索する | `title` |
| F-06 | ジャンルで完全一致検索する | `category` |
| F-07 | 期限の近い順・遠い順に並べる | `due_date` |
| F-08 | 完了状態を変更する | `completed` |
| F-09 | 操作結果を一覧上部に表示する | DBの保存値ではなく、アプリケーションの処理結果表示。対象データの特定には `id` を使用 |
| F-10 | 登録・編集・削除の操作ログを残す | `id` と操作種類をアプリケーションログへ記録。ログ用テーブルは作成しない |
| 入力チェック | 必須、255文字、空白だけの入力を確認する | 必須制約は `title`・`category`・`priority`、文字数と空白だけの判定はアプリケーション側 |
| 対象外 | 完了日時を記録しない | 完了日時用カラムを作成しない |
| 対象外 | ログイン・利用者管理・共有を行わない | 利用者用カラムや利用者用テーブルを作成しない |
| 対象外 | 写真・ファイルを添付しない | 添付ファイル用カラムやテーブルを作成しない |

## 10. 想定する主なデータ操作

| 操作 | 対象カラム | 備考 |
|---|---|---|
| 登録 | `title`、`detail`、`category`、`priority`、`due_date`、必要に応じて `completed` | `id`、`created_at`、`updated_at` は自動設定 |
| 編集 | `title`、`detail`、`category`、`priority`、`due_date`、`completed` | `updated_at` はMySQLが自動更新 |
| 削除 | `id` | 対象の1行を削除 |
| 一覧取得 | 全カラムのうち表示に必要なもの | `title`、`category`、`priority`、`due_date`、`completed` を表示 |

