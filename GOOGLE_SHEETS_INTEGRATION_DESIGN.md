# 非公開 Google Sheets 連携の設計方針

## 目的

旅行ごとに作成する非公開 Google スプレッドシートを正本とし、PC またはスマートフォンのサイトから最新の行程とスポットを閲覧する。GitHub、GitHub Pages、ブラウザの永続ストレージには旅行本文を保存しない。

## 採用する構成

```text
Google Drive Picker で旅行 Sheet を選択
  -> この端末に旅行名と Spreadsheet ID だけを保存
  -> Google OAuth で一時アクセストークンを取得
  -> Google Sheets API で「行程」「スポット」を読み取り表示
```

スプレッドシートを編集した後、サイトの更新操作で最新データを再取得する。GitHub へのコミットやデプロイは不要。

## 境界と保存方針

| 項目 | 方針 |
| --- | --- |
| 正本 | 旅行ごとの非公開 Google Sheets |
| 端末に保存するもの | 旅行名、Spreadsheet ID、Google Drive URL、追加日時、最終取得日時 |
| 端末に保存しないもの | 行程本文、スポット本文、OAuth アクセストークン |
| GitHub に保存しないもの | Sheet ID を含む利用者の旅行一覧、旅行本文、トークン |
| 閲覧時の本文 | メモリ上にだけ保持し、再読み込み・ログアウト時に破棄 |

機種変更時は、Google ログイン後に旅行 Sheet を再選択して登録する。将来必要になれば、本文を含まない接続一覧の JSON バックアップ／復元を追加する。

## Google Cloud と認可

Google Cloud プロジェクトで Google Sheets API と Google Picker API を有効化する。Web アプリ用 OAuth クライアントID、Picker 用ブラウザ API キー、プロジェクト番号を設定する。

OAuth の要求スコープは `https://www.googleapis.com/auth/drive.file` だけにする。Google Picker で利用者が選んだファイルへのアクセスに限定し、広範な Drive 閲覧権限や `spreadsheets.readonly` は要求しない。アプリは Sheets API の値読み取りエンドポイントだけを呼び、書き込み API は呼ばない。

OAuth クライアントID、ブラウザ API キー、プロジェクト番号は静的サイトへ設定する。これらは秘密情報ではないが、API キーは `https://manimani830.github.io/*` と Google Picker の iframe 用 `https://docs.google.com/*` のリファラ、および Google Picker API に制限する。クライアントシークレット、アクセストークン、リフレッシュトークンはコードやストレージに置かない。

## データ契約

選択する各 Spreadsheet は次の2タブを持つ。

| タブ | 読み取り範囲 | 必須ヘッダー |
| --- | --- | --- |
| 行程 | `A:E` | 日付 / 時間 / 種別 / 行動 / 移動・備考 |
| スポット | `A:E` | カテゴリ / 採用状況 / スポット名 / アクセス / 特徴・魅力（歴史的背景や見どころの解説） |

値は `FORMATTED_VALUE` として取得し、既存表示ロジックと互換のTSV文字列へ変換する。これにより、現在のスプレッドシート書式を変えずに移行できる。

## UI 方針

- 初回は旅行一覧を空にし、「Google スプレッドシートから追加」を表示する。
- 旅行追加時は Google Drive Picker を開き、Spreadsheet だけを選択可能にする。
- 同じ Spreadsheet を再選択した場合は重複登録せず、その旅行へ切り替える。
- Google Sheet 連携旅行では、ヘッダーの編集操作を「Sheetを開く」に変更する。
- 更新ボタンは連携旅行にだけ表示し、データを再取得する。
- 旅行削除はこの端末の接続情報だけを削除し、Google Drive 上の Sheet は変更しない。
- 既存の手動貼り付け旅行はローカル旅行として残し、勝手に削除・上書きしない。

## エラーと安全性

- 未ログイン、Picker のキャンセル、権限不足、削除済みSheet、ネットワーク失敗を区別して表示する。
- タブまたはヘッダーが不足している場合、必要な名前と列を案内する。
- 更新失敗時は現在表示中のデータを消さない。
- 貼り付け値・Sheet値はHTMLとして解釈せず、必ず文字列として表示する。
- エラー表示やログへSheetの本文を出力しない。

## 実装ステップ

1. Google API の遅延読み込み、OAuth、Google Picker を追加する。
2. Spreadsheet選択、端末内の接続情報保存、旅行切替を実装する。
3. Sheets API の `values:batchGet` で2タブを取得し、既存ビューへ渡す。
4. 更新状態、エラー表示、Sheetを開く導線を追加する。
5. 既存のローカル旅行を残したまま移行できることを確認する。
6. Google Cloud の実際のクライアントID等を設定し、PC・スマホの実機で認証と再読み込みを確認する。

## 完了条件

- 非公開の新しい旅行SheetをPickerから追加できる。
- Sheet編集後、サイトの更新操作だけで内容が反映される。
- 権限のないGoogleアカウントでは本文を取得できない。
- GitHubのコミット、GitHub Pages、localStorageに旅行本文やトークンが残らない。
- 既存のローカル旅行は削除されずに閲覧・編集できる。
