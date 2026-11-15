---
status: accepted
---

# DB は Neon（PostgreSQL）にする。バックエンドは Cloud Run の東京のままにする

ADR 0001 で TiDB Cloud Starter を選んだが、`AUTO_INCREMENT` が既定で 30,000 個単位で確保されて id が大きく飛ぶため、学習用の連番 id と相性が悪く、Neon の Free プランに変えた。Neon はシンガポール（`aws-ap-southeast-1`）に置き、Cloud Run は東京（`asia-northeast1`）のままにする。ADR 0001 は Neon を「東京リージョンがない」ことを理由に外していたが、この API は 1 リクエストにつき SQL が 1〜2 回なので、距離による遅延は受け入れる。

PostgreSQL の `IDENTITY` も、ロールバックやクラッシュからの復旧で値が飛ぶ。ただし飛ぶ幅は小さく、単一のノードなので順序は逆転しない。

## Considered Options

- TiDB Cloud Starter: 東京にあり、Spending Limit 0 で課金されない。id が大きく飛ぶ点が決め手で外した
- Supabase: Free プランは 1 週間ほどアクセスがないとプロジェクトが一時停止され、復帰させるまで DB につながらない。API を Go で自作するこの計画では、認証などの強みも使わない

## Consequences

- Cloud Run（東京）と Neon（シンガポール）の間で、SQL 1 回あたり 80 ms 前後の遅延が増える（目安で、実測していない）
- Free プランの上限を超えると、コンピュートが停止するか書き込みが失敗する。課金はされず、課金が始まるのは有料プランにアップグレードしたときだけ
- Neon のコンピュートは使われないと停止し、次の接続が遅くなる
- Free プランでクレジットカードが必要かどうかは確認していない
- DB は PostgreSQL になる。ドライバーは `database/sql` と `pgx` の stdlib（`github.com/jackc/pgx/v5/stdlib`）を使う。ローカルは Docker Compose の公式 `postgres` イメージを使う
