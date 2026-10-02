# S2J Video Publisher Service - CHANGELOG

## unreleased

### Changed

* `package.json` の `description` を、YouTube 動画の公開期間を決めるライブラリ (WordPress 非依存) に更新
* `.vscode/settings.json` で `json.schemaDownload.enable` を有効にし、`package.json` のスキーマを取得できるようにした
* `.vscode/settings.json` から `npm.enableScriptExplorer` を削除
* textlint 拡張の導入後にマージする設定例を `.vscode/textlint.settings.jsonc.example` に追加

## 0.0.1 - 2026-10-02

### Added

* 確定前のサービス仕様を `docs_mod/service_spec.md` に記載
    * 公開期間の開始で `unlisted`、終了で `private`。`publishAt` は使わない
    * 監査が通るまでアップロードは出さない。その間は動画 ID の登録と `videos.update` のみ
    * 本ライブラリは WordPress 非依存。OAuth、HTTP、cron、管理画面はプラグイン側
* `docs_mod/specs.md` からサービス仕様への参照を追加
* ドキュメント lint を追加 (`@s2j/docs-linter` ^1.0.25、`npm run lint:docs`、GitHub Actions)
* npm v12以降向けに `.npmrc` の `allow-git=all` と `package.json` の `allowScripts` を追加

### Changed

* README に、YouTube 動画の公開期間を決める Composer ライブラリであることを記載
* `.gitignore` を Composer、Node、テスト成果物向けに拡張
