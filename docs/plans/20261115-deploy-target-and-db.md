# デプロイ先と DB の決定

## Context

当初の計画は「Cloudflare へのデプロイを学ぶ」を目的にしていた。DB とデプロイ先の決定はステップ 4 まで保留にしていた。学習者から「クラウドにデプロイできるならどこでもいい」「無料枠で使えるものがよい」「無料枠を超えても課金されない仕組みがほしい」と要望があり、デプロイ先と DB を見直して今決めた。

インタビューで決まったこと:
- バックエンド（Go）は Google Cloud Run の東京リージョン（`asia-northeast1`）に置く。無料枠があり、Docker コンテナをそのまま動かせる。請求先アカウントの登録にクレジットカードが必要になる
- DB は TiDB Cloud Starter（MySQL 互換）の東京リージョン（`aws-ap-northeast-1`）にする。Spending Limit は 0 のままにする
- ローカルの DB は、Docker Compose で公式の `pingcap/tidb` イメージを立てる
- フロントは Cloudflare Workers または Pages に置く（ステップ 6）
- Cloud Run の課金の歯止めとして、`--max-instances=1` と予算アラートを必ず入れる。Spend cap 型の予算は、ステップ 5 で使えるかを確かめ、使えれば入れる。Pub/Sub で課金を無効にする方法は入れない

調べて分かった前提（2026 年 11 月時点）:
- TiDB Cloud Starter は、Spending Limit が 0 なら課金されない。無料枠を超えると読み書きが制限されるか、新しい接続を拒否される。無料枠は毎月リセットされる
- Neon は課金されない点では同じだが、東京リージョンがない。Cloud Run と DB の距離が離れると、1 リクエストの中の DB との往復ごとに遅延が積み重なるので、東京に置ける TiDB を選んだ
- Cloud Run の予算アラートは通知を送るだけで、課金は止まらない。Spend cap が使えない場合、Cloud Run には課金の可能性が残る（`--max-instances=1` でも、1 台が動き続ければ無料枠を超えうる）。この残るリスクは学習者に伝えてあり、ロードマップと ADR にも書く
- Cloudflare D1 は、Go から使うには Worker か REST API を経由する必要があり、バックエンドを Cloud Run に置くと構成が複雑になるので、候補から外した
- フロントの置き場所は DB の選択に影響しない。ブラウザが呼ぶのは Cloud Run の API だけで、DB に接続するのはバックエンドだけ

`LEARNING_PLAN.md` は元の要望の記録なので変えない。

## 変更するファイル

### `docs/plans/20261115-go-todo-learning-roadmap.md`

編集の前に、元のファイルを scratchpad にコピーしておく（このファイルはまだ git に登録されていないため、`git diff` では比べられない）。

1. Context: 「Cloudflare へのデプロイ」を「クラウドへのデプロイ」に変える
2. 設計の方針: 「DB とデプロイ先は保留のままとし、ステップ 4 に入るときに決める」を、次の内容に置き換える
   - バックエンドは Cloud Run（東京）、DB は TiDB Cloud Starter（東京）、フロントは Cloudflare Workers または Pages に置く。理由は `docs/adr/0001-tidb-and-cloud-run.md` に書く
3. 進め方のルールの「AI に任せていいところ」: 「wrangler の設定、Cloudflare との接続部分」を「wrangler の設定、Cloud Run と gcloud の設定」に変える
4. ステップ 4 の見出しを「TiDB を導入する」に変え、本文を案 A（PostgreSQL）と案 B（D1）の分岐から、次の内容に書き換える。DB を先に決めたので、ステップ 4 だけは方針とヒントまで先に書いたと注記する
   - Docker はここで初めて導入する。Docker Compose で `pingcap/tidb` を立て、ローカルの DB にする。イメージのタグは TiDB Cloud のバージョンに合わせて固定する
   - テーブルは SQL ファイル（例: `migrations/001_create_todos.sql`）を手で流して作る。ローカルと TiDB Cloud の両方に同じファイルを使う。マイグレーションツールは使わない
   - Go からは `database/sql` と `go-sql-driver/mysql` で接続する。リポジトリの TiDB 実装を追加して、メモリ実装と差し替えられるようにする
   - DB 実装のテストは、ローカルの TiDB に対して書く
   - 用語解説の候補: `database/sql`、DSN（接続文字列）、プレースホルダー（`?`）と SQL インジェクション、マイグレーション
   - ヒントの候補:
     - `time.Time` として読むには、DSN に `parseTime=true` が必要。タイムゾーンは `loc=UTC` に揃える。マイクロ秒まで保つなら列を `DATETIME(6)` にする
     - TiDB の `AUTO_INCREMENT` は一意だが、飛び番になる。既定のままでは、TiDB ノードが複数あると後から作った行の id が小さくなることもあり、仕様の「同じ日時なら id の降順」の意味がずれる。単調に増やしたいなら、テーブルに `AUTO_ID_CACHE=1` を付ける（それでも飛び番は残る）。Starter で使えるかは、ステップ 4 に入るときに制限一覧で確かめる
     - DB 実装のテストは、接続先を環境変数（例: `TEST_DSN`）で受け取り、未設定なら `t.Skip` する。TiDB を起動していなくても `go test ./...` が通るようにする
     - 本番の TiDB Cloud には TLS で接続する（DSN に `tls=true`）
     - ローカルの `pingcap/tidb` はポート `4000` で待ち受け、`root` にパスワードがない。TiDB Cloud Starter のユーザー名には `xxxx.root` のような接頭辞が付く
     - TiDB Cloud が使われていない接続を切る可能性があるので、`db.SetConnMaxLifetime` を短めにしておく
5. ステップ 5 を「Cloud Run へデプロイ」に書き換える
   - Dockerfile をマルチステージビルドで作り、Artifact Registry に置いて `gcloud run deploy` する。リージョンは `asia-northeast1`
   - ビルドは `CGO_ENABLED=0 go build` で行い、libc に依存しないバイナリにする。実行用のイメージには CA 証明書が入っているもの（例: distroless の `static`）を使う。入っていないと TiDB Cloud への TLS 接続に失敗する
   - Artifact Registry の無料枠は約 0.5 GB なので、古いイメージを消すクリーンアップポリシーを設定する
   - 待ち受けるポートは環境変数 `PORT` から読み、未設定なら `8080` にする（Cloud Run は `PORT` を設定する）。ステップ 1 で `:8080` を直接書いた箇所を直す
   - DB の接続文字列は Secret Manager に入れ、コードやイメージには含めない。Cloud Run には `--set-secrets` で環境変数として渡し、実行するサービスアカウントに `roles/secretmanager.secretAccessor` を付ける
   - 課金の歯止め: 予算アラート（例: 1 ドル）と `--max-instances=1` を入れる。`min-instances` は 0（既定）、課金はリクエスト単位（既定）のままにし、`--cpu` と `--memory` は小さくする。予算アラートは通知を送るだけで、課金は止まらない。Spend cap 型の予算が使えるかを確かめ、使えれば入れる。使えない場合は、Cloud Run に課金の可能性が残ることを書いておく
   - TiDB Cloud の Spending Limit が 0 のままになっていることを確かめる
   - 「コンテナのディスクは永続化されない前提」の注意は残す
   - Dockerfile と gcloud の設定は、頼まれたら Claude が書く。学習者は 1 行ずつ説明できるまで読む
6. ステップ 6: フロントを Cloudflare Workers または Pages に置くことを書く。CORS で許可するオリジンは環境変数で渡し、ローカル（例: `http://localhost:5173`）と本番のフロントのドメインを切り替える

### `docs/adr/0001-tidb-and-cloud-run.md`（新規）

デプロイ先を Cloud Run、DB を TiDB Cloud Starter（どちらも東京）にした理由を ADR として書く。後から読んだ人が「なぜ当初の Cloudflare ではないのか」「なぜ PostgreSQL ではないのか」と疑問に思う決定で、変えるには DB 実装の作り直しが必要になるため残す。比べた候補（Cloudflare Containers、Workers + Wasm、Render、Neon、D1）と、選ばなかった理由を書く。Spend cap が使えない場合に Cloud Run に課金の可能性が残ることも、受け入れたリスクとして書く。

### `GLOSSARY.md`

冒頭の説明文の「Cloudflare へのデプロイ」を「クラウドへのデプロイ」に変える。用語の追加や変更はない。

### このプランファイル

承認後に `docs/plans/$(date +%Y%m%d)-deploy-target-and-db.md` へリネームする。

## 検証

- `grep -n "Cloudflare" docs/plans/20261115-go-todo-learning-roadmap.md` で、Cloudflare の記述が「設計の方針」とステップ 6 だけに残ることを確かめる
- `grep -n "D1\|Fly.io\|Containers\|保留\|案 B\|PostgreSQL" docs/plans/20261115-go-todo-learning-roadmap.md` で、何も出ないことを確かめる
- scratchpad に取った元のファイルと `diff` で比べ、ステップ 1〜3 と API 仕様が変わっていないことを確かめる
- `grep -n "Cloudflare" GLOSSARY.md` で何も出ないことを確かめる
- `docs/adr/0001-tidb-and-cloud-run.md` があり、比べた 5 つの候補と受け入れたリスクが書かれていることを確かめる
