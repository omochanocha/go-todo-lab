---
status: superseded by ADR-0002
---

# バックエンドは Cloud Run、DB は TiDB Cloud Starter（どちらも東京）にする

当初は Cloudflare へのデプロイを学ぶ計画だったが、「クラウドにデプロイできるならどこでもよい」「無料枠で使いたい」「無料枠を超えても課金されない仕組みがほしい」という要望で見直した。バックエンドは Google Cloud Run の `asia-northeast1`、DB は TiDB Cloud Starter の `aws-ap-northeast-1` に置き、Spending Limit は 0 のままにする。フロントは Cloudflare Workers または Pages に置く。

## Considered Options

- Cloudflare Containers: Workers 有料プランが必要で、最低でも月 5 ドルかかる
- Go を Wasm にして Cloudflare Workers で動かす: 無料だが制約が多く、Go 以外のところで詰まりやすい
- Render: カード不要で無料だが、15 分アクセスがないとスリープし、起きるまで最大 60 秒ほどかかる
- Neon（PostgreSQL）: 課金されない点は同じだが、東京リージョンがない。Cloud Run と DB の距離が離れると、1 リクエスト内の DB との往復ごとに遅延が積み重なる
- Supabase（PostgreSQL）: 課金されない点は同じだが、Free プランは DB へのアクセスが 1 週間ほどないとプロジェクトが一時停止され、復帰させるまで API が DB につながらない。学習のペースが空くと本番の DB が止まる。認証などの強みは API を Go で自作するこの計画では使わず、Cloud Run からの接続も共有プーラー（IPv4 用）を経由する分だけ設定が増える。東京リージョンの有無は理由にしていない
- Cloudflare D1: Go から使うには Worker か REST API を経由する必要があり、バックエンドを Cloud Run に置くと構成が複雑になる

## Consequences

- DB は MySQL 互換になる。リポジトリの DB 実装は `go-sql-driver/mysql` を使う。PostgreSQL 向けの記事やライブラリは、そのままでは使えない
- ローカルは Docker Compose で `pingcap/tidb` を立てるので、本番と同じ製品でテストできる
- Spend cap 型の予算が使えない場合、Cloud Run には課金の可能性が残る。予算アラートは通知を送るだけで、課金は止まらない。`--max-instances=1` を付けても、1 台が動き続ければ無料枠を超えうる。このリスクを受け入れる
- Cloud Run の請求先アカウントの登録に、クレジットカードが必要になる
