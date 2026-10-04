# S2J Video Publisher Service - サービス仕様 (ドラフト)

採用・合意前の設計メモとして `docs_mod/` に置き、確定後は `docs/SERVICE_SPEC.md` に移行します。状態はドラフトです。記録日は2026-10-02です。

## 概要

本ドキュメントは、S2J Video Publisher Service の初期設計における、サービス全体の統合の見取り図を定義します。

呼び出す WordPress プラグインは **S2J Video Publisher** です。リポジトリは [s2j-video-publisher](https://github.com/stein2nd/s2j-video-publisher) です。

本ライブラリは **WordPress 非依存** です。WP フック、設定画面、OAuth トークンの保存、HTTP の実行、動画バイト列の転送は扱いません。

## 背景

案内する相手を限定した、期間限定の動画を YouTube に載せる依頼は、従前から届きます。YouTube Studio では、公開開始の予約はできるが、公開終了の予約はできません。

Studio の予約公開は、非公開の動画を指定時刻に **公開 (public)** にする機能です。限定公開 (unlisted) の開始時刻は予約できません。時刻をそろえるために予約公開を使うと、その時刻にチャンネルに公開され、その後で手作業により「限定公開」に変え、終了時刻にまた手作業で非公開に戻す運用になります。

本プロダクトが埋めるのは、その穴です。

```text
アップロード (非公開のまま)
        ↓
公開開始の日時
        ↓
限定公開 (unlisted)
        ↓
公開終了の日時
        ↓
非公開 (private)
```

## 目的

自身が運営する YouTube チャンネルについて、動画ごとの **公開期間** を管理します。

公開期間の開始では `privacyStatus` を `unlisted` にし、終了では `private` にします。開始時刻も終了時刻も、分単位で指定した時刻にそろえます。

初版のユーザーは、そのチャンネルの運営者です。複数サイトに配る OAuth クライアントは、初版の前提にしません。

## 非目標 (初版)

* YouTube の `status.publishAt` による予約公開。これは時刻到来で `public` になる。
* 公開開始の到達状態を `public` にすること。
* 公開終了で動画を削除すること。終了後の状態は `private` である。
* 公開期間中に Studio 側の手動変更を検知して戻す、継続的な強制。
* WordPress の投稿や固定ページの公開状態との連動。
* ライブ放送、プレミア公開、コミュニティ投稿、サムネイル、字幕、再生リスト。
* 複数チャンネルの同時接続。
* WordPress.org への掲載。

## YouTube API との対応

使うメソッドは、次の2つです。

| メソッド | 役割 | クォータの置き場 (2026-10時点の公式) |
| --- | --- | --- |
| `videos.insert` | 動画ファイルとメタデータのアップロード | Video Uploads バケット。デフォルト100コール/日。1コールあたり1unit |
| `videos.update` | `status.privacyStatus` の変更 | その他のメソッドと共有の10,000units/日。`videos.update` は50units |
| `videos.list` | 処理状況と公開状態の確認 | 共有バケット。1unit |

`videos.insert` と `search.list` は、2026-06-01以降、共有バケットとは別の枠です。共有バケットの残量があっても、アップロード枠が尽きた `videos.insert` は失敗します。

`status.publishAt` は、`privacyStatus` が `private` であり、かつ一度も公開されていない動画にだけ設定できます。設定した時刻が来ると `public` になります。過去の時刻を入れると、ただちに `public` になるのと同じです。限定公開の開始には使いません。

`privacyStatus` に指定できる値は `public` / `unlisted` / `private` です。

公式の記載は、下記を参照しました。

* [Videos: insert](https://developers.google.com/youtube/v3/docs/videos/insert)
* [Videos](https://developers.google.com/youtube/v3/docs/videos)
* [Quota and Compliance Audits](https://developers.google.com/youtube/v3/guides/quota_and_compliance_audits)
* [Quota Calculator](https://developers.google.com/youtube/v3/determine_quota_cost)
* [Revision History](https://developers.google.com/youtube/v3/revision_history)
* [Schedule video publish time](https://support.google.com/youtube/answer/1270709)

2025-12-04の改訂で、アップロードのコストは約1,600units から約100units へ変わっています。2026-06-01以降の現行ドキュメントでは、`videos.insert` は専用バケットの1unit / コールです。Quota Calculator の表には旧コストが残っている箇所があります。実装時は Cloud Console の当該プロジェクトの表示を正とします。

本プロダクトの本数 (1日に数本、状態変更は開始と終了の2回) は、デフォルト枠の中に収まります。処理待ちの確認は、`processing` の動画だけを `videos.list` で見ます。公開期間中の動画を毎分ポーリングしません。

## 監査と OAuth

製品化の前に、Google 側の門が2つあります。別の手続きです。

### アップロードした動画が「非公開」に固定される門

2020-07-28以降に作られた未監査の API プロジェクトから `videos.insert` した動画は、private viewing mode に制限されます。リクエストで `unlisted` や `public` を指定しても、動画は非公開のままロックされます。ロックされた動画の解除申請はできず、監査済みクライアントか YouTube のサイトから上げ直す必要があります。

したがって、**監査が通るまで `videos.insert` は製品の機能にしません。**

監査の前に作るのは、台帳とスケジューラです。YouTube Studio で、予約公開を付けずに非公開でアップロードした動画の ID を登録します。開始時刻に `videos.update` で `unlisted` にし、終了時刻に `private` に戻します。

この `videos.update` が、Studio から上げた動画に対して監査なしで通るかは、公式のロック文が `videos.insert` 経由の動画に限定していることから可能と読みます。実装の前に、動画1本で確認します。確認が失敗した場合、開始と終了の自動化は、監査完了まで保留します。

### トークンが7日で切れる門

OAuth 同意画面が Testing のアプリでは、リフレッシュトークンが7日で失効します。無人のスケジューラには、本番 (In production) の同意画面と、失効しないリフレッシュトークンがいります。YouTube のコンプライアンス監査とは別です。

スコープは次の2つです。`https://www.googleapis.com/auth/youtube.upload` は `videos.insert` と `videos.update` に使います。`https://www.googleapis.com/auth/youtube.readonly` は、プラグインが接続中のチャンネル名を `channels.list` (`mine=true`、`part=snippet`) で取るために使います。`youtube.upload` だけでは `channels.list` は呼べません。チャンネル全体を扱う `youtube` スコープは要求しません。

クライアント ID とクライアントシークレットは、プラグインに同梱しません。サイトの設定として渡します。リポジトリにコミットしません。

## Composer ライブラリの理由

見た目は WordPress の管理画面ですが、層は次に分かれます。

| 層 | 中身 | 置き場 |
| --- | --- | --- |
| 計算 | 公開期間の検証、状態の写像、次にたたく API 操作の決定、`videos.insert` / `videos.update` のリクエスト材料 | **本ライブラリ** |
| 副作用 | OAuth、HTTP、ブラウザからの resumable upload、cron、管理画面、動画 ID の保存 | **S2J Video Publisher** (プラグイン) |

プラグイン一本に判断を置くと、時刻と公開状態の組み合わせをユニットテストするたびに WordPress と YouTube が必要になります。

## 本ライブラリの責務

入力は、動画レコードと「今」の時刻です。出力は、次の操作と、操作後のレコードです。HTTP レスポンスの解釈も、渡されたステータスとボディから結果レコードを作るところまで、です。

| 責務 | 内容 |
| --- | --- |
| 検証 | `publish_at` は `unpublish_at` より前。両方ともタイムゾーン付きの時刻。タイトルは空でない |
| 次の操作 | 下記の状態遷移に従い、`none` / `set_unlisted` / `set_private` / `insert_private` のいずれかを返す |
| リクエスト材料 | `privacyStatus` だけを変える `videos.update` のボディ。アップロード時は `privacyStatus=private` かつ `publishAt` なしの `videos.insert` メタデータ |
| 結果の写像 | API の成功で適用時刻を埋める。失敗では適用時刻を空のままにし、エラー文を残す。トークンと動画ファイルの中身は残さない |

アップロード用メタデータに含める項目は、次のとおりです。

* `snippet.title`
* `snippet.description`
* `snippet.categoryId`。プラグインのコンボボックスが渡す。未変更時は `22` (People & Blogs)
* `status.privacyStatus` = `private`
* `status.selfDeclaredMadeForKids`。プラグインが `true` か `false` を渡す。未選択のままでは insert しない
* `status.containsSyntheticMedia`。プラグインが `true` か `false` を渡す。未選択のままでは insert しない

`status.publishAt` は材料に含めません。

## 状態と次の操作

レコードが持つ時刻は UTC の瞬間です。画面の日付と時刻は、WordPress「設定 > 一般」のタイムゾーン、日付形式、時刻形式で出します。ライブラリは表示文字列を作りません。

```text
video_id                 未アップロードなら空
title
publish_at
unpublish_at
phase                    processing | scheduled | live | ended | failed
observed_privacy         private | unlisted | public | unknown
observed_processing      pending | processing | succeeded | failed | unknown
publish_applied_at       空、または開始の API が成功した時刻
unpublish_applied_at     空、または終了の API が成功した時刻
last_error               空、または直近の失敗。トークンは入れない
```

`phase` の意味は、次のとおりです。

| phase | 意味 |
| --- | --- |
| `processing` | YouTube がファイルを処理している。公開状態は `private` |
| `scheduled` | 処理が終わり、開始時刻を待っている。公開状態は `private` |
| `live` | 公開期間の中にあり、公開状態は `unlisted` |
| `ended` | 終了時刻を過ぎ、公開状態は `private`。または、処理が終わる前に期間が閉じた |
| `failed` | 処理失敗、またはこれ以上リトライしない API エラー |

時刻 `now` を受けたとき、次の操作は1つです。

1. `observed_processing` が `failed` なら `failed`。操作は `none`。
2. 終了時刻を過ぎ、`unpublish_applied_at` が空で、一度でも `live` になっている、または `observed_privacy` が `unlisted` なら、操作は `set_private`。
3. 終了時刻を過ぎ、まだ `live` になっておらず、観測上も `private` のままなら、API は呼ばず `ended`。開始しない。
4. 処理が `succeeded` であり、開始時刻を過ぎ、終了時刻より前で、`publish_applied_at` が空なら、操作は `set_unlisted`。
5. 処理が未完了で、開始時刻を過ぎ、終了時刻より前なら、操作は `none`。`phase` は `processing` のまま。画面には、処理待ちのため開始が遅れている、と出す。
6. 処理が `succeeded` で、開始時刻より前なら、`scheduled`。操作は `none`。
7. `publish_applied_at` と `unpublish_applied_at` がすでに埋まっていれば、操作は `none`。同じ遷移を繰り返さない。
8. 開始の適用時刻が埋まっていて、終了時刻より前なら、操作は `none`。`phase` は `live`。公開期間の途中を毎分更新しません。

成功した応答を受けたときだけ、対応する `*_applied_at` を `now` で埋めます。失敗では空のままにし、次の実行で同じ操作を返します。待避時間はプラグインが決めます。

観測した `privacyStatus` が、これから行う操作の到達値とすでに同じなら、API を呼ばずに `*_applied_at` を埋めます。

処理が開始時刻の後に `succeeded` になり、まだ終了前であれば、遅れて `set_unlisted` します。`publish_applied_at` が実際の適用時刻になります。

## プラグインの責務 (境界。詳細はプラグイン仕様)

プラグイン仕様の詳細は [S2J Video Publisher の docs_mod/specs.md](https://github.com/stein2nd/s2j-video-publisher/blob/main/docs_mod/specs.md) です。ここには境界だけを置きます。

| 責務 | 内容 |
| --- | --- |
| 接続 | 管理画面の「YouTube と接続」。認可コードをリフレッシュトークンに換え、サイト設定へ保存する |
| 台帳 | 動画1行が、上記レコードである。保存先はプラグインが決める |
| 実行 | 分の粒度で本ライブラリを呼ぶ。返った操作だけを YouTube API に送る |
| アップロード | 監査後。セッション開始はプラグインが OAuth 付きで行い、バイト列はブラウザから YouTube の resumable 先へ直接送る。WordPress のディスクと PHP のメモリにファイルを載せない |
| 表示 | 見出しは「動画の公開期間」。列はタイトル、状態、開始、終了、直近のエラー |

実行の契機はプラグインが持ちます。処理は WP-Cron の1分間隔に置き、最終実行の時刻の遅れで、訪問なしに進んでいるかを画面が判断します。ライブラリは契機を知りません。`now` を受け取るだけです。

監査前の登録画面は、動画 ID、タイトル、開始日時、終了日時です。ファイル入力は、監査完了を設定で示すまで出しません。

## 設計方針

本ライブラリは [kis-wordpress エコシステム仕様](https://github.com/stein2nd/kis-wordpress/blob/main/docs_mod/specs.md) と同じく、FOP + Clean Coding を基本とします。Clean Architecture の定型分割は採用しません。

データの中身は、純粋関数と不変レコードに閉じます。

| 借用する原則 | 本ライブラリでの意味 |
| --- | --- |
| 依存の向きは外 → 内 | 純関数は WordPress / HTTP / Google SDK を知らない |
| 内側はビジネスルール | 公開期間の検証と、次の `privacyStatus` の決定 |
| 外側は詳細 | プラグインが OAuth、HTTP、cron、画面を持つ |

パッケージ名は `s2j/video-publisher-service` とします。PHP の名前空間は `S2J\VideoPublisherService\` とします。ライセンスは GPL-3.0-or-later とします。プラグインも同じです。

## 関連リポジトリ

| 名称 | 種別 | 役割 |
| --- | --- | --- |
| **本ライブラリ** | Composer | 公開期間の検証、次の操作、API リクエスト材料 |
| **S2J Video Publisher** | WP プラグイン | 接続、台帳、cron、resumable upload のセッション、管理画面 |
| [kis-wordpress](https://github.com/stein2nd/kis-wordpress.git) | モノレポ | サイト専用プラグイン群。本機能は kis-core に抱え込まない |

KIS のサイトは、このプラグインのユーザーの一つです。チャンネルの OAuth クライアントは KIS 専用であり、ライブラリには埋めません。

## 実装順

1. 本ドラフトの合意。
2. Studio で非公開 (予約公開なし) にした動画1本に対し、監査前のプロジェクトから `videos.update` で `unlisted` と `private` を往復できるかを確認する。
3. 本 repo でスケルトンと純関数の初版 (PHPUnit、WordPress なし、HTTP なし)。
4. プラグインが Composer で require し、台帳と `videos.update` のスケジューラをつなぐ。
5. OAuth 同意画面を Production にし、リフレッシュトークンが7日で切れないことを確認する。
6. YouTube API のコンプライアンス監査の後に、ブラウザから YouTube へ直接送る `videos.insert` を足す。挿入時の公開状態は `private`、`publishAt` は付けない。処理が `succeeded` になってから、上記の開始・終了に乗る。

## 本ドラフトの提案

合意前の提案です。

* 製品のコアは、限定公開の公開期間である。開始で `unlisted`、終了で `private`。
* 開始時刻は本ライブラリのスケジュールが担当する。`status.publishAt` はリクエストに含めない。
* 初版の到達状態は `unlisted` のみ、終了状態は `private` のみ。
* 同じ遷移は1回だけ行う。適用後に Studio で変えた公開状態は、初版では戻さない。
* 処理完了前に終了時刻を過ぎた動画は、限定公開にしない。
* 監査が通るまでアップロード機能は出さない。その間は動画 ID の登録と、開始・終了の `videos.update` だけを作る。
* 動画バイト列は WordPress サーバーに置かない。
* OAuth クライアントはサイト設定であり、配布物に含めない。
* ライセンスは、プラグインとライブラリの両方で GPL-3.0-or-later。
* パッケージ名は `s2j/video-publisher-service`。プラグインのスラッグは `s2j-video-publisher`。
* 管理画面の見出しは「動画の公開期間」。

## 未決事項

* 監査前の `videos.update` が、Studio から上げた非公開動画に通るか。確認のやり方は決まっている。動画1本を、監査前の API プロジェクトから `unlisted` と `private` に往復させ、人が結果を見る。「今すぐ実行」は、その1行について cron と同じ手順をその場で走らせる。自動テストの項目ではない。未決なのは結果で、通れば監査前に開始と終了の自動化を作り、失敗すれば監査完了まで保留する。

## 改訂履歴

| 日付 | 内容 |
| --- | --- |
| 2026-10-02 | 初版ドラフト。限定公開の公開期間、`publishAt` を使わないこと、監査前は台帳と `videos.update` だけにすること、ライブラリとプラグインの境界を記録 |
| 2026-10-04 | プラグイン仕様を [s2j-video-publisher の docs_mod/specs.md](https://github.com/stein2nd/s2j-video-publisher/blob/main/docs_mod/specs.md) に書いた。公開期間の途中は操作 `none`、と補足 |
| 2026-10-05 | insert の `categoryId` は未変更時 `22`。`selfDeclaredMadeForKids` はプラグインが選んだ `true` か `false` だけを受ける、と記録 |
| 2026-10-05 | `containsSyntheticMedia` はプラグインが選んだ `true` か `false` だけを受ける。未選択のままでは insert しない、と記録 |
| 2026-10-05 | 接続中のチャンネル名のため、スコープに `youtube.readonly` を足す。`channels.list` は `youtube.upload` だけでは呼べない、と記録 |
| 2026-10-05 | 実行契機はプラグインの WP-Cron に置く。ライブラリは `now` を受け取るだけ、と記録 |
| 2026-10-05 | 監査前の `videos.update` の未決を、プラグイン仕様と同じく、人が動画1本で往復する結果である、とそろえた |
