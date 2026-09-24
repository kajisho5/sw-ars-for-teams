# sw-ars for Microsoft Teams

[PULSE](https://sw-ars.com)（`kajisho5/sw-ars`）のイベント（ルーム）に接続し、ライブ投票・Q&A・ワードクラウドの結果を Microsoft Teams のタブ（個人タブ／チームタブ／会議中タブ）にリアルタイムで表示する Teams アプリです。

PULSE 本体（イベント作成・投影スクリーン・参加者ページなど）はすでに sw-ars.com 上で稼働しているため、このリポジトリは**そのイベントに繋いで結果を見るだけの、薄い読み取り専用ビューア**です。イベントの作成や進行操作自体は引き続き PULSE の管理画面（`https://sw-ars.com/admin?...`）で行います。PowerPoint やドキュメントと違いスライド等への「挿入」に相当する操作は無いため、`sw-ars-for-powerpoint` / `sw-ars-for-googleslides` とは異なりビューア専用です。

## 構成

```
manifest.json       Teamsアプリマニフェスト（静的個人タブ + チーム/会議用の構成可能タブ。ホスト先は YOUR-HOSTING-DOMAIN のまま）
manifest.prod.json  本番用マニフェスト（ホスト先を sw-ars.com/integrations/teams/ にしたもの）
index.html          タブ本体（接続フォーム・3秒間隔ポーリング・結果描画）
config.html         チーム/会議にタブを追加する時の設定画面（ルーム名・鍵を先に入力しておける）
icons/              color.png(192x192)・outline.png(32x32) のアプリアイコン（PULSE のブランドアイコン）
```

### 本番の配信元について

本番（`https://sw-ars.com/integrations/teams/`）で配信しているのは、このリポジトリのファイルそのものではなく、PULSE 本体（`kajisho5/sw-ars`）の `src/pages/integration-assets.js` に文字列として埋め込んだ複製です。

| このリポジトリ | PULSE 本体の定数 |
| --- | --- |
| `index.html` | `TEAMS_INDEX_HTML` |
| `config.html` | `TEAMS_CONFIG_HTML` |
| `manifest.prod.json` | `TEAMS_MANIFEST_JSON`（`TEAMS_APP_ZIP_B64` の zip 内の `manifest.json` も同じ内容） |
| `icons/color.png`・`icons/outline.png` | `TEAMS_ICON_COLOR_B64`・`TEAMS_ICON_OUTLINE_B64`（zip 内の画像も同じ） |

**変更するときは、このリポジトリと PULSE 本体の埋め込みの両方に同じ差分を当ててください。** 片方だけを直すと、あとでもう片方の内容で丸ごと差し替えたときに改修が消えます。

```
PULSE (sw-ars.com, Cloudflare Workers + Durable Objects)
        ▲  GET /api/admin/state?r=ルーム名&key=閲覧用の鍵（3秒間隔でポーリング、タブ側から直接fetch）
        │
Teams タブ（個人 / チーム / 会議中）── 表示のみ（Teams-JS SDKでテーマ追従・会議中コンテキスト取得）
```

### 機能

- **接続**: PULSE の「閲覧専用リンク」（`/view?r=ルーム名&key=viewKey`）から取得した *ルーム名* と *鍵* を入力するだけ。バックエンド側の変更は不要（PULSE の既存の `viewKey` 権限をそのまま利用）。
- **個人タブ**: `index.html` を開いて手動で接続。接続情報はブラウザの `localStorage` に保存され次回も自動再接続。
- **チーム／グループチャット／会議タブ**: `config.html` でタブ追加時にルーム名・鍵を先に入力しておけば、追加後は自動接続した状態でタブが開く（`contentUrl` にクエリパラメータとして引き継ぐ）。
- **ライブ表示**: 現在出題中の設問の結果（選択式は棒グラフ、自由記述はワードクラウド）と Q&A 一覧を、3 秒間隔のポーリングでリアルタイムに表示。
- **Teams テーマ追従**: ライト/ダーク/ハイコントラストに自動追従（Teams JS SDK の `registerOnThemeChangeHandler`）。

## セットアップ（開発・お試し）

Teams アプリはマニフェストが参照する `contentUrl` を HTTPS で実際にホストする必要があります。以下はローカル確認の一例です。

```bash
npx http-server . -p 3000 --ssl -c-1
# または Teams Toolkit / ngrok 等で任意のHTTPS URLを用意する
```

1. `manifest.json` の `YOUR-HOSTING-DOMAIN` を、実際にホストしたドメイン（例: Cloudflare Pages, GitHub Pages, ngrok の一時ドメイン等）に置き換える。
2. `manifest.json` と `icons/*.png` を zip にまとめる（`zip pulse-teams-app.zip manifest.json icons/color.png icons/outline.png`）。
3. Teams の **アプリ → アプリを管理 → カスタムアプリをアップロード** から zip をアップロードして動作確認。

## 使い方

1. PULSE（`https://sw-ars.com`）でイベント（ルーム）を作成・進行する。
2. 管理画面から対象イベントの「閲覧専用リンク」を開き、URL の `r=` の値（ルーム名）と `key=` の値（viewKey）を控える。
3. Teams でこのアプリのタブを開く（個人タブなら直接、チーム/会議タブなら追加時の設定画面で）ルーム名と鍵を入力。
4. 出題中の設問の結果や Q&A が自動更新される。

## 本番運用・組織配布に向けて

- `manifest.json` の `id` は仮の GUID を割り当て済みです。組織固有のものに差し替える場合は新しい GUID を発行してください。
- `icons/color.png` / `icons/outline.png` は PULSE のブランドアイコンです。差し替える場合、outline は透明背景に白いシルエットである必要があります。
- Microsoft Teams Store（AppSource）への公開を目指す場合は、[Teams アプリの公開手順](https://learn.microsoft.com/microsoftteams/platform/concepts/deploy-and-publish/appsource/publish) と審査要件（アクセシビリティ・プライバシーポリシー等）を確認してください。
- 現状は読み取り専用ビューアですが、Bot Framework と組み合わせて「チャットに結果を投稿する」「会議中に自動でリマインドする」等の拡張も可能です。
