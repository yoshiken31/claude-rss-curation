# claude-rss-curation

Inoreaderのフォルダ記事をGeminiで選別し、コメント付きRSSとして公開する仕組みです。
Claude活用の実践事例（MDファイル運用・トークン節約・アプリ開発・DX業務自動化）を毎日自動で拾います。
Gemini無料枠とGitHub無料枠だけで動き、Claudeの使用量・課金は発生しません。

公開フィード: https://yoshiken31.github.io/claude-rss-curation/feed.xml

## 仕組み
1. 毎日 06:00 JST（cron `0 21 * * *` UTC）に GitHub Actions が `scripts/curate.py` を実行します。手動実行は Actions の `curate` → Run workflow です。
2. Secret `INOREADER_FEED_URL` のRSSを取得します。処理済み・重複（同じ元URL、ほぼ同じタイトル）を除きます。Googleアラートの中継URLは元記事のURLに変換します。
3. 初回は直近3日分だけが対象で、それより古い記事は処理済み扱いで取り込みません。
4. 1次選別: 25件ずつまとめてGeminiに渡し、分野・実践事例か・1〜5点を判定します。
5. 2次確認: しきい値以上の記事だけ本文を取得（10秒タイムアウト、失敗時はタイトルと要約で代用）し、日本語コメントを付けて再評価します。
6. 採用記事を `docs/feed.xml` に追加します（件数上限なし、60日で削除）。guidは元記事URLです。タイトルの先頭に分野マーク（📝MD運用 🪙トークン節約 📱アプリ開発 ⚙️DX自動化）が付きます。
7. `state/seen.json` と `docs/feed.xml` をワークフローがコミットします（変更がなければコミットしません）。

## 設定の変更
- `config/criteria.md`: 分野の定義、実践事例の条件、除外条件、点数の目安。文章を編集するだけで反映されます。
- `config/settings.json`: 数値設定。
  - `model`: 使うGeminiモデル。
  - `stage1_threshold` / `final_threshold`: 1次・最終の点数しきい値（既定3）。
  - `stage1_batch_size`: 1回に判定する件数。
  - `feed_retention_days`: フィードの保持日数（既定60）。
  - `max_items_per_run` / `max_api_calls_per_run` / `min_seconds_between_calls`: 1回の処理量とAPI呼び出し間隔（無料枠の保護）。
- Secrets: `INOREADER_FEED_URL`、`GEMINI_API_KEY`（リポジトリの Settings → Secrets and variables → Actions）。

## 注意点
- **Geminiの無料枠の上限**: 公式ドキュメント（rate-limits / models / pricing）には、無料枠のRPM・RPD数値が載っていません。数値は Google AI Studio の Rate limit ページでのみ確認できます。既定モデルは公式一覧で無料と明記された安定版 `gemini-3.5-flash-lite` です。上限（429など）に当たると、待って3回まで再試行し、それでも失敗したらその日は中断します。未判定の記事は処理済みにせず翌日に回します。
- **課金**: 課金設定は行っていません。APIキーは無料枠のプロジェクトのものを使い、課金を有効にしないでください。無料枠の入力は製品改善に使われる場合があります（公式pricingページの記載）。
- **ログ**: 件数の要約だけを出力し、記事URLの全文やフィードURLは出しません。
- **本文取得**: サイトによっては拒否されます。その場合はタイトルと要約からコメントを作ります。
- **定期実行の自動停止**: 公開リポジトリでは、60日間リポジトリの動きがないと定期実行が自動で無効になります。記事が採用されなくても `state/seen.json` は通常毎日更新されコミットされるため、動きが止まりにくい設計です。ただし、ワークフロー自身（GITHUB_TOKEN）のコミットが「動き」に数えられるかは、GitHub公式文書で確認できていません。無効化された場合は、Actions タブで `curate` を再度有効にしてください。確実にしたい場合は、定期的に手動実行するか、手動でコミットしてください。
- フィードに載る記事数は、Inoreaderの出力フィードが返す件数に依存します。
