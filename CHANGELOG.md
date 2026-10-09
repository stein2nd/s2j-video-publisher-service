# S2J Video Publisher Service - CHANGELOG

## unreleased

## 0.0.1 - 2026-10-09

### Changed

* `docs_mod/service_spec.md` の表記をドキュメント lint に合わせた (`リフレッシュトークン` を `リフレッシュ・トークン`、`クライアントシークレット` を `クライアント・シークレット`)

## 0.0.1 - 2026-10-08

### Changed

* `docs_mod/service_spec.md` の表記をドキュメント lint に合わせた (列挙の `次の` を `下記`、文中の参照を `右記`、`受けたとき` を `受けた場合`)

## 0.0.1 - 2026-10-05

### Changed

* `docs_mod/service_spec.md` の insert で、`categoryId` の未変更時は `22` とした
* `selfDeclaredMadeForKids` と `containsSyntheticMedia` は、プラグインが選んだ `true` か `false` だけを受ける。未選択では insert しない
* 接続中のチャンネル名のため、スコープに `youtube.readonly` を足した。`channels.list` は `youtube.upload` だけでは呼べない
* 実行契機はプラグインの WP-Cron に置く。ライブラリは `now` を受け取るだけ、と記録した
* 監査前の `videos.update` の確認は、人が動画1本で見る結果であり、自動テストの項目ではない、と補足

## 0.0.1 - 2026-10-04

### Changed

* ドキュメント lint の `@s2j/docs-linter` を ^1.0.27に更新
* `docs_mod/service_spec.md` に、公開期間の途中は操作 `none`、`phase` は `live` と補足した
* プラグイン仕様の参照先を s2j-video-publisher の `docs_mod/specs.md` にした
* `docs_mod/service_spec.md` から、kis-wordpress の `docs_mod/specs.md` に1行を足す未決事項を外した

## 0.0.1 - 2026-10-03

### Added

* `composer.json` を追加 (`s2j/video-publisher-service`、名前空間 `S2J\VideoPublisherService\`、PHP 8.0以上、開発依存に PHPUnit / PHPStan / PHPCS)

### Changed

* ドキュメント lint の `@s2j/docs-linter` を ^1.0.26に更新
* README と `docs_mod/service_spec.md` の表記をドキュメント lint に合わせた (半角括弧、数字前後の空白、`そろえ`、`デフォルト` など)

* `package.json` の `description` を、YouTube 動画の公開期間を決めるライブラリ (WordPress 非依存) に更新
* `composer.json` と `package.json` の `description` を、純粋な PHP ライブラリである旨に更新
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
