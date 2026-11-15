# DB を TiDB から Neon（PostgreSQL）に変える

## Context

ADR 0001 では、DB を TiDB Cloud Starter（東京）に決めた。その後、TiDB の `AUTO_INCREMENT` で id が大きく飛ぶという記事を見て、学習者が DB を Neon に変えたいと考えた。

TiDB の公式ドキュメントによると、`AUTO_INCREMENT` の値は TiDB サーバーごとに、既定で 30,000 個単位でまとめて確保される。複数のサーバーがあるときや再起動したときに、使われなかった値が捨てられて飛び番になる。記事にあった「4000 番」という数字は確認できていない。計画に書いた `AUTO_ID_CACHE=1` で単調に増やせるというヒントも、情報源の間で食い違いがあり、確認が取れていない。Neon（PostgreSQL）なら、`id` は `GENERATED ALWAYS AS IDENTITY` で増える。PostgreSQL のシーケンスもロールバックやクラッシュからの復旧で飛ぶが、飛ぶ幅は小さく、単一のノードなので順序は逆転しない。

インタビューで決まったこと:
- DB は Neon（PostgreSQL）の Free プランにする。リージョンはシンガポール（`aws-ap-southeast-1`）
- Cloud Run は東京（`asia-northeast1`）のまま。Neon とは 1 往復あたり 80 ms 前後離れる（一般的な目安で、実測はしていない）。この API は 1 リクエストにつき SQL が 1〜2 回なので、許容範囲と見ている
- Go からは `database/sql` と `pgx` の標準ライブラリ向けドライバー（`github.com/jackc/pgx/v5/stdlib`）で接続する
- ローカルは Docker Compose で公式の `postgres` イメージを立てる
- ADR は新しく 0002 を作り、0001 の `status` を `superseded by ADR-0002` にする。TiDB を選んだ経緯と、変えた理由が履歴に残る
- Cloud Run の課金の歯止め（`--max-instances=1`、予算アラート、使えれば Spend cap）は ADR 0001 のまま変えない

調べて分かった前提（2026 年 11 月時点、Neon の公式 FAQ による）:
- Free プランは月 0 ドルで、超過分は請求されない
- CU 時間（コンピュートの使用時間）かネットワーク転送を使い切ると、翌月の請求期間までコンピュートが停止する。ストレージが 1 プロジェクトあたり 1 GB を超えると、ストレージが増える書き込みが失敗する。どの場合もデータは削除されない
- 課金が始まるのは、自分で有料プランにアップグレードしたときだけ
- Free プランでクレジットカードが必要かどうかは、公式 FAQ には書かれていない（未確認）

## 変更するファイル

### `docs/adr/0001-tidb-and-cloud-run.md`

先頭の `status: accepted` を `status: superseded by ADR-0002` に変える。本文は変えない。

### `docs/adr/0002-neon-postgres.md`（新規）

先頭に ADR 0001 と同じ形式の front matter（`status: accepted`）を付け、次の内容で書く。

- 見出し: 「DB は Neon（PostgreSQL）にする。バックエンドは Cloud Run の東京のままにする」
- 冒頭: ADR 0001 で TiDB Cloud Starter を選んだが、`AUTO_INCREMENT` が既定で 30,000 個単位で確保されて id が大きく飛ぶため、学習用の連番 id と相性が悪く、Neon に変えた。Neon の Free プランはシンガポールに置き、Cloud Run は東京のままにする。ADR 0001 は Neon を「東京リージョンがない」ことを理由に外していたが、この API は 1 リクエストにつき SQL が 1〜2 回なので、距離による遅延は受け入れる
- Considered Options:
  - TiDB Cloud Starter: 東京にあり、Spending Limit 0 で課金されない。id が大きく飛ぶ点が決め手で外した
  - Supabase: Free プランは 1 週間ほどアクセスがないとプロジェクトが一時停止され、復帰させるまで DB につながらない。API を Go で自作するこの計画では、認証などの強みも使わない
- 冒頭に、PostgreSQL の `IDENTITY` も、ロールバックやクラッシュからの復旧で値が飛ぶことを書く。「飛ばない」とは書かず、「飛ぶ幅が小さく、単一のノードなので順序は逆転しない」と書く
- Consequences:
  - Cloud Run（東京）と Neon（シンガポール）の間で、SQL 1 回あたり 80 ms 前後の遅延が増える（目安で、実測していない）
  - Free プランの上限を超えると、コンピュートが停止するか書き込みが失敗する。課金はされず、課金が始まるのは有料プランにアップグレードしたときだけ
  - Neon のコンピュートは使われないと停止し、次の接続が遅くなる
  - Free プランでクレジットカードが必要かどうかは確認していない
  - DB は PostgreSQL になる。ドライバーは `database/sql` と `pgx` の stdlib を使う。ローカルは Docker Compose の `postgres` イメージ

### `docs/plans/20261115-go-todo-learning-roadmap.md`

編集の前に、現在のファイルを scratchpad にコピーしておく。

1. 設計の方針（56 行目）: 「DB は TiDB Cloud Starter（東京）」を「DB は Neon（PostgreSQL、シンガポール）」に変え、「理由は `docs/adr/0001-tidb-and-cloud-run.md`」を「理由は `docs/adr/0001-tidb-and-cloud-run.md` と `docs/adr/0002-neon-postgres.md`」に変える
2. ステップ 4 を、見出し「Neon（PostgreSQL）を導入する」と次の内容に書き換える。先頭の注記（DB を先に決めたので方針とヒントまで先に書いている）は残す
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
3. ステップ 5 は 2 行だけ変える。ほかの項目は変えない
   - 151 行目（distroless の `static` の項目）の「TiDB Cloud への TLS 接続に失敗する」を「Neon への TLS 接続に失敗する」に変える
   - 156 行目の「TiDB Cloud の Spending Limit が 0 のままになっていることを確かめる」を「Neon が Free プランのままで、有料プランにアップグレードしていないことを確かめる」に変える
4. ステップ 3 以前、API 仕様、ステップ 6 は変えない

### `GLOSSARY.md`

変更しない。

### このプランファイル

承認後に `docs/plans/$(date +%Y%m%d)-switch-to-neon.md` へリネームする。

## 検証

- `grep -n "TiDB\|MySQL\|AUTO_ID_CACHE\|parseTime\|4000" docs/plans/20261115-go-todo-learning-roadmap.md` で何も出ないことを確かめる
- `grep -n "Cloudflare" docs/plans/20261115-go-todo-learning-roadmap.md` で、設計の方針とステップ 6 だけに残ることを確かめる
- scratchpad に取った編集前のロードマップと `diff` で比べ、変更が設計の方針、ステップ 4、ステップ 5 の 2 行だけであることを確かめる
- `head -3 docs/adr/0001-tidb-and-cloud-run.md` で `status` が `superseded by ADR-0002` になっていることを確かめる
- `head -3 docs/adr/0002-neon-postgres.md` で `status: accepted` になっていることを確かめる。`grep -n "^## " docs/adr/0002-neon-postgres.md` で、Considered Options と Consequences の見出しがあることを確かめる
