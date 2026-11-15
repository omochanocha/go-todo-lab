# Go Todo API 学習計画

## Context

`LEARNING_PLAN.md` をもとに、Go のバックエンド・テスト・クラウドへのデプロイを学ぶための実装計画を作る。学習者はフロントエンドエンジニア（React / TypeScript）で、Go とバックエンドは初めて扱う。完成させることより、自分の言葉で説明できる理解を優先する。

リポジトリにはまだ Go のコードがない。mise（go, golangci-lint, node, pnpm）、lefthook と commitlint（Conventional Commits）、golangci-lint（gofumpt, goimports, modernize）の設定だけがある。

この計画は全体のロードマップとし、ステップ 1〜2 だけを詳しく書く。ステップ 3 以降は、そのステップに入るときに詳細を追記する。

## 決定事項（API 仕様）

### Todo のデータ

リクエスト・レスポンス・保存のすべてで同じ形を使う。JSON のキーは camelCase にする。

| フィールド | 型 | 内容 |
|---|---|---|
| `id` | int64 | 連番。サーバーが採番する |
| `title` | string | 必須。前後の空白を除いて空なら 400。最大文字数は設けない |
| `status` | string の列挙値 | `"todo"`（未着手）/ `"in_progress"`（進行中）/ `"done"`（完了） |
| `priority` | string の列挙値 | `"low"` / `"medium"` / `"high"` |
| `createdAt` | 日時（RFC 3339） | 作成時にサーバーが設定する |
| `updatedAt` | 日時（RFC 3339） | 作成時と更新時にサーバーが設定する |

- Go では `type Status string` と `type Priority string` を定数とあわせて定義する。列挙にない値は 400 にする
- 状態の遷移に制約は設けない（どの状態からどの状態へも変更できる）
- サーバーが設定するフィールド（`id`, `createdAt`, `updatedAt`）は、リクエストに含まれていても無視する
- POST では、`status`・`priority` の空文字も省略と同じに扱う。PUT では空文字は列挙にない値なので 400 になる

### エンドポイント

ルーティングは Go 1.22 以降の標準 `net/http.ServeMux` だけで行う（`"GET /todos/{id}"` の形で書き、`r.PathValue` で値を取る）。

| メソッドとパス | 成功時 | 内容 |
|---|---|---|
| `POST /todos` | 201 と作成した Todo | `title` は必須。`status` を省略すると `"todo"`、`priority` を省略すると `"medium"` |
| `GET /todos` | 200 と配列 | `createdAt` の降順。同じ日時なら `id` の降順。0 件なら `null` ではなく `[]` を返す |
| `PUT /todos/{id}` | 200 と更新後の Todo | 全体を置き換える。`title`・`status`・`priority` はすべて必須。`updatedAt` だけを更新し、`createdAt` は変えない |
| `DELETE /todos/{id}` | 204 | |

- 存在しない id は 404 を返す。PUT でも新規作成はしない。`0` や `-1` も数字として解釈できるので、存在しなければ 404 になる
- id が数字でないとき、JSON が不正なとき、バリデーションに失敗したときは 400 を返す
- エラーの本文は `{"error": "メッセージ"}` に統一する。統一するのはハンドラーが返すエラーだけで、登録していないパスの 404 や許可していないメソッドの 405 は、ServeMux が返す既定の `text/plain` のままにする
- `PATCH`、状態遷移の制約、優先度による並べ替えや絞り込みは扱わない。余力があれば発展課題にする

### 設計の方針

- ステップ 1〜2 はリポジトリ直下の `package main` で書く（`main.go`, `handler.go` などのファイル分割だけ）。ステップ 3 で `internal/todo` に移す
- 現在時刻は `now func() time.Time` として外から渡す。テストでは固定時刻を返す関数に差し替える
- ステップ 2 から、ハンドラーはストアを持つ struct のメソッドとして書く。ストアをパッケージ変数にはしない。テストごとに新しいストアを作り、テストどうしで状態を共有しない
- テストは標準の `testing` と `net/http/httptest` だけで書く。モックライブラリは使わない
- 「メモリ実装」と「テスト用スタブ」を分ける
  - メモリ実装: 本番コードとして動く実装。正常系のテストにも使う
  - テスト用スタブ: テストファイル内で手書きする小さな struct。保存処理がエラーを返したときに 500 になるか、といったメモリ実装では起こせない異常系を再現する
- バックエンドは Cloud Run（東京）、DB は Neon（PostgreSQL、シンガポール）、フロントは Cloudflare Workers または Pages に置く。理由は `docs/adr/0001-tidb-and-cloud-run.md` と `docs/adr/0002-neon-postgres.md` に書く

## 進め方のルール

- 「自分で手で書くところ」（ハンドラー、ルーティング、JSON の扱い、`TodoRepository` とその実装、テスト）は、Claude は頼まれない限りコードを書かない。代わりにヒント、方針、参考ドキュメントを示す
- 「AI に任せていいところ」（Dockerfile、Docker Compose、wrangler の設定、Cloud Run と gcloud の設定、2 つ目以降の似たコード）も、頼まれたときだけ書く。設定ファイルを書く前には一言確認する
- 学習者は、まず 15〜30 分は自分で書く。詰まったら、最初はヒントだけをもらう
- AI が書いたコードは、学習者が 1 行ずつ「なぜこう書くのか」を説明できるまで次に進まない。AI が書いたコードには、学習者が自分でテストを書く
- 時間を置いてから、何も見ずに書き直してみる
- 各ステップの前に、新しく出てくる用語を 1 語あたり 3 行程度で解説する
- 各ステップの終わりに理解度チェックをする。Claude が出した問いに学習者が自分の言葉で答え、Claude が添削する。問い・回答・添削は `docs/learning/stepN.md` に残してコミットする
- 1 ステップずつ確認しながら進める。次のステップには、理解度チェックが終わってから入る
- ステップごとに `step-N-<topic>` ブランチを切り、終わったら PR を作って学習者自身がマージする
- コミットメッセージは Conventional Commits に従う（commitlint が検査する）。Claude 経由のコミットには末尾に `Co-Authored-By: Claude <noreply@anthropic.com>` を付ける

## 準備（ステップ 1 の前）

- Claude が `GLOSSARY.md` を作り、ドメイン用語を書く: Todo、状態（未着手・進行中・完了）、優先度（低・中・高）、メモリ実装、テスト用スタブ
- `go mod init github.com/omochanocha/go-todo-lab` は、ステップ 1 の最初に学習者が自分で実行する（Claude は手順だけ示す）
- `go test -race` には C コンパイラが必要だが、この環境には gcc が入っていない。ステップ 2 の確認までに、学習者が `sudo apt install build-essential` で入れる。ステップ 1 は `-race` を付けずに実行する
- PR は `gh pr create` で作る（リモート `origin` と `gh` は使える）

## ステップ 1: 最小の HTTP サーバー（詳細）

ブランチ: `step-1-http-server`

用語解説: package / module（`go.mod`）、`net/http`、`http.Handler` と `http.HandlerFunc`、`ServeMux`、struct と JSON タグ、`encoding/json`、`httptest.NewRecorder` と `httptest.NewRequest`、`go test`

学習者が書くもの:
1. `go mod init` でモジュールを作る
2. `Todo` struct を上の仕様どおりに定義する（JSON タグを含む）
3. `GET /todos` で固定の Todo 配列を JSON で返すハンドラーを書き、`main` で `ServeMux` に登録して `:8080` で起動する
4. `handler_test.go` に、ステータス 200、`Content-Type: application/json`、本文の内容を確かめるテストを 1 本書く

Claude が示すもの: 公式ドキュメントへのリンク（`net/http`, `encoding/json`, `net/http/httptest`, Go 1.22 のルーティング強化の記事）と、詰まったときのヒント。ヒントの候補は次のとおり。
- `.golangci.yml` の errcheck は `check-blank: true` なので、`Encode` や `ListenAndServe` が返す error を無視すると lint エラーになる。`_ =` で捨てることも禁止されている。`log.Fatal(http.ListenAndServe(...))` の形や、エラーをログに出す方法を検討する
- `w.Header().Set("Content-Type", ...)` は、`WriteHeader` や `Write` より先に呼ばないと効かない
- modernize は新しい書き方（`interface{}` の代わりに `any` など）を勧めてくる。古い記事と違っても、指摘に従ってよい

確認: `go test ./...` と `golangci-lint run` が通る。`go run .` で起動し、`curl localhost:8080/todos` で JSON が返る

理解度チェックの例: 「`http.ResponseWriter` と `*http.Request` はそれぞれ何か」「`httptest.NewRecorder` を使うと、なぜ実際にサーバーを起動しなくてもテストできるのか」

## ステップ 2: CRUD を 1 つずつ追加（詳細）

ブランチ: `step-2-crud`

用語解説: ポインタとメソッドレシーバー、`sync.Mutex`（並行アクセスとデータ競合）、nil スライスと空スライスの違い、map の反復順序がランダムなこと、`sort.Slice` / `slices.SortFunc`、`json.Decoder`、`strconv.ParseInt`、`r.PathValue`、テーブル駆動テスト、`time.Time` と関数を値として渡すこと

1 機能ずつ、実装とテストをセットで進める。順番は次のとおりにする。

1. 保存場所: メモリ上のストアを作る（`sync.Mutex`、次に振る ID、Todo の集まり、`now func() time.Time` を持つ struct）。ここではまだインターフェースにしない
2. 作成: `POST /todos`。既定値、`title` のバリデーション、`createdAt`/`updatedAt` の設定、201 を返す
3. 一覧: `GET /todos` を固定 JSON からストアの内容に置き換える。ステップ 1 のテストも、ストアを使う形に書き換える。0 件で `[]` が返ること、並び順が仕様どおりであることをテストで確かめる
   - 日時が異なるケース: 呼ぶたびに 1 秒ずつ進む `now` を渡す
   - 同じ日時のケース: 固定時刻の `now` を渡すと全件が同じ日時になるので、`id` の降順になることを確かめる
   - ヒント: 並べ替えには `slices.SortFunc` と `time.Time.Compare` を使える。時刻の比較は `==` ではなく `Equal` を使う
4. 更新: `PUT /todos/{id}`。3 フィールドが必須であること、404、id の 400、`createdAt` が変わらないこと
5. 削除: `DELETE /todos/{id}`。204（本文は書かない）と 404

- 最初の作成と一覧は、実装もテストもすべて学習者が書く。更新と削除は、学習者が頼めば Claude が書いてよい。その場合は、学習者が 1 行ずつ説明できるまで次に進まず、テストは学習者が自分で書く
- 異常系（400、404）はテーブル駆動テストでまとめる
- 発展課題（任意）: `PATCH /todos/{id}` を `*string` などのポインタで実装し、「送られなかった」と「空の値が送られた」を区別する

確認: `go test -race ./...` と `golangci-lint run` が通る。curl で作成 → 一覧 → 更新 → 削除の流れを一通り試す

## ステップ 3〜6: ロードマップ（各ステップの開始時に詳細を追記）

### ステップ 3: リポジトリパターン

- `TodoRepository` インターフェースを定義し、ステップ 2 のストアをメモリ実装に作り替える。ハンドラーはインターフェースだけに依存させる
- コードを `internal/todo` に移し、`main.go` は組み立てだけを行う
- テスト用スタブでリポジトリのエラーを再現し、500 のテストを書く
- エラーの見分け方（`errors.Is` と `ErrNotFound` のような目印のエラー値）を扱う

### ステップ 4: Neon（PostgreSQL）を導入する

DB を先に決めたので、このステップだけは方針とヒントまで先に書いている。

- Docker はここで初めて導入する。Docker Compose で公式の `postgres` イメージを立て、ローカルの DB にする。イメージのタグは、Neon で選ぶ PostgreSQL のメジャーバージョンに合わせて固定する
- テーブルは SQL ファイル（例: `migrations/001_create_todos.sql`）を手で流して作る。ローカルと Neon の両方に同じファイルを使う。マイグレーションツールは使わない。`id` は `BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY`、日時の列は `TIMESTAMPTZ` にする
- Go からは `database/sql` と `github.com/jackc/pgx/v5/stdlib` で接続する。リポジトリの PostgreSQL 実装を追加して、メモリ実装と差し替えられるようにする
- DB 実装のテストは、ローカルの PostgreSQL に対して書く
- 用語解説の候補: `database/sql`、DSN（接続文字列）、プレースホルダー（`$1`）と SQL インジェクション、マイグレーション、IDENTITY 列
- ヒントの候補:
  - プレースホルダーは `?` ではなく `$1`, `$2` と書く
  - `pgx` の stdlib では `LastInsertId` が使えない。作成した行の `id` は、`INSERT ... RETURNING id` で受け取る
  - `time.Time` は `TIMESTAMPTZ` と対応する。PostgreSQL はマイクロ秒までなので、メモリ実装と値をそろえるために、保存前に、外から渡す `now` 関数が返した値を `Truncate(time.Microsecond)` で丸める。時刻の比較は `Equal` で行い、`==` は使わない。DB から返ってきた時刻のタイムゾーンはローカルになっていることがあるので、必要なら `.UTC()` に揃える
  - `IDENTITY` の値は増えていくが、ロールバックやクラッシュからの復旧では飛ぶ。仕様の「連番」は「増えていく一意な番号」の意味で読む
  - `GENERATED ALWAYS` の列には、id を明示して INSERT できない。テストの後始末で連番を戻したいときは、`TRUNCATE ... RESTART IDENTITY` を使う
  - DB 実装のテストは、接続先を環境変数（例: `TEST_DSN`）で受け取り、未設定なら `t.Skip` する。PostgreSQL を起動していなくても `go test ./...` が通るようにする
  - Neon には TLS で接続する。Neon の画面に出る接続文字列をそのまま使い、`pgx` で接続できるかを確かめる（`channel_binding=require` が付くことがあり、対応は未確認）。`sslmode=require` は証明書を検証しないので、可能なら `sslmode=verify-full` にする
  - 接続文字列には、直接つなぐものと pooler 経由のものの 2 種類がある。pooler は PgBouncer の transaction mode で、`pgx` の prepared statement のキャッシュと相性が悪い可能性がある。Cloud Run からどちらを使うかは、ステップ 4 に入るときに公式ドキュメントで確かめる
  - Neon のコンピュートは、使われないと停止し、次の接続が遅くなる。`db.SetConnMaxIdleTime` と `db.SetConnMaxLifetime` を短めにして、切れた接続を使い続けないようにする。開いたままの接続が自動停止を妨げるかどうかも、このとき確かめる
  - ローカルの `postgres` イメージは `POSTGRES_PASSWORD` の指定が必要で、ポートは `5432`

### ステップ 5: Cloud Run へデプロイ

- Dockerfile をマルチステージビルドで作り、Artifact Registry に置いて `gcloud run deploy` する。リージョンは `asia-northeast1`
- ビルドは `CGO_ENABLED=0 go build` で行い、libc に依存しないバイナリにする。実行用のイメージには CA 証明書が入っているもの（例: distroless の `static`）を使う。入っていないと Neon への TLS 接続に失敗する
- Artifact Registry の無料枠は約 0.5 GB なので、古いイメージを消すクリーンアップポリシーを設定する
- 待ち受けるポートは環境変数 `PORT` から読み、未設定なら `8080` にする（Cloud Run は `PORT` を設定する）。ステップ 1 で `:8080` を直接書いた箇所を直す
- DB の接続文字列は Secret Manager に入れ、コードやイメージには含めない。Cloud Run には `--set-secrets` で環境変数として渡し、実行するサービスアカウントに `roles/secretmanager.secretAccessor` を付ける
- 課金の歯止め: 予算アラート（例: 1 ドル）と `--max-instances=1` を入れる。`min-instances` は 0（既定）、課金はリクエスト単位（既定）のままにし、`--cpu` と `--memory` は小さくする。予算アラートは通知を送るだけで、課金は止まらない。Spend cap 型の予算が使えるかを確かめ、使えれば入れる。使えない場合は、Cloud Run に課金の可能性が残ることを書いておく
- Neon が Free プランのままで、有料プランにアップグレードしていないことを確かめる
- コンテナのディスクは永続化されない前提で、コンテナ内に保存したファイルを本番のデータにはしない
- Dockerfile と gcloud の設定は、頼まれたら Claude が書く。学習者は 1 行ずつ説明できるまで読む

### ステップ 6: フロントをつなぐ（任意）

- React などで画面を作り、API を呼ぶ。フロントは Cloudflare Workers または Pages に置く
- CORS で許可するオリジンは環境変数で渡し、ローカル（例: `http://localhost:5173`）と本番のフロントのドメインを切り替える

## 検証

- 各ステップの終わりに `go test -race ./...` と `golangci-lint run` が通ること（ステップ 1 だけは `-race` なしでよい）
- `go run .` で起動し、そのステップで追加したエンドポイントを curl で確かめること
- `docs/learning/stepN.md` に理解度チェックの記録があること
- ステップのブランチから PR を作り、差分を見返してからマージすること
