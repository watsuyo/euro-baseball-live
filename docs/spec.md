# Euro Baseball Live — ペライチ仕様

本番 URL: https://euro-baseball-live.vercel.app/
ソース: `index.html` 1 枚（vanilla HTML + inline JS）

## 目的

- 日本人向けに**今日の欧州野球**をワンビューでまとめ、SNS 等にスクリーンショットで共有するペライチ
- 試合開始時刻と配信リンクを即座に把握する
- 日本人選手・指導者が出場する試合を見逃さない

viewer アプリ (`euro-baseball-viewer.vercel.app/live`) の List ビューと UI を揃える。ただしフィルタ / ビュー切替 / 日付ナビゲーションは**持たない**（今日特化）。

## データソース

viewer アプリの公開 JSON をそのまま fetch する。

| URL | 用途 |
|---|---|
| `https://euro-baseball-viewer.vercel.app/data/schedule.json` | 全リーグの試合（date, time, tz, home, away, stream_url?） |
| `https://euro-baseball-viewer.vercel.app/data/jp_players.json` | 日本人選手・指導者一覧 |
| `https://euro-baseball-viewer.vercel.app/data/streams.json` | リーグ単位の配信リンク（web フォールバック用） |

取得に失敗した補助データ（jp_players / streams）は空配列で扱い、schedule のみ必須。

## 日付と時刻

- **表示タイムゾーン**: `Asia/Tokyo`（JST）固定
- **日付境界**: **04:00 JST**（28 時制）
  - 閲覧時刻 `< 04:00` の場合、「今日」は前日扱い
  - 試合時刻（JST）が `< 04:00` の場合、その試合は前日の一覧に入る
- **時刻表示**: **28 時制**
  - `00:00`〜`03:59` → `24:00`〜`27:59`
  - `04:00` 以降はそのまま
  - 並び替えもこの文字列で昇順（`"18:00" < "25:30"` 等）
- **date ラベル**: ISO 形式 `YYYY-MM-DD`（例: `2026-04-18`）

## セクション構成

上から順に:

1. **🇯🇵 日本人選手・指導者の所属チーム**
   - `home` または `away` が日本人選手の所属チームに一致する試合のみ
   - チーム名マッチは小文字化 + 英数字以外除去（`norm(s)`）で緩和
   - `jp_players.json` の `status === "active"` のみ対象
   - 同一試合に複数の日本人選手がいれば名前を `・` で連結し、`.game-row-players` に表示
   - セクションのあとに divider 的な余白
2. **今日**
   - 今日の試合を全件
   - JP セクションと重複あり（冗長だが探しやすさ優先）

フィルタ・検索・絞り込みトグルは**なし**。

## 行レイアウト（GameRow 相当）

HTML 要素順（左 → 右）:

```
time | 🇨🇿 flag | Home vs Away | 📺 stream | 🇯🇵 players
```

- `.game-row-time` — 28 時制、accent 色、太字
- `.game-row-flag` — ISO-2 コードから絵文字生成
- `.game-row-matchup` — `flex: 1` で伸びる
- `.game-row-stream` — `margin-left: auto` で右端固定、stream がある場合のみ
- `.game-row-players` — accent 色、11px。JP セクション以外では非表示（JP 以外の行では省略）

### 配信有無

- `stream` が truthy のとき行は `<a>` 要素（`has-link`）
- なければ `<div>`
- 配信なしでもグレーアウト等の視覚的区別はしない

### JP セクションだけ 2 行許可

`.game-row.jp-team`:
- `flex-wrap: wrap` で複数行化
- `align-content: center` で行グループを縦中央寄せ
- `row-gap: 2px`
- `border-left: 3px solid var(--accent)` で強調
- `.game-row-players` を `flex-basis: 100%; margin-left: 40px` で強制改行 & matchup 位置までインデント
- `white-space: normal; overflow: visible; text-overflow: clip` で**選手名を省略しない**

JP 以外のセクションでは `white-space: nowrap; overflow: hidden` を維持（省略）。

## 配信リンク優先順

1. `g.stream_url`（per-game、WBSC game-video 等の正規ソース）
2. `streams.json` の `leagues[].streams[?type === "web"].url`（リーグの共通スコアボード/スケジュールページ、フォールバック）
3. どちらも無ければ 📺 非表示

## スタイル方針

- モバイルファースト（コンテナ `max-width: 700px`、padding 16px）
- ダーク / ライト自動切替（`prefers-color-scheme: dark`）
- ダーク時のカラー: viewer アプリと CSS 変数を完全一致
  - `--bg: #0f1117`, `--surface: #1a1d27`, `--border: #2d3140`
  - `--text-h: #f9fafb`, `--accent: #60a5fa`, `--tag-bg: #1f2233`
- セクション見出し `.list-section-title` は opacity なしの純白（スクショ映えを優先）
- 行パディング 5px 8px、font-size 13px、gap 6px（viewer List ビューと同寸）

## リフレッシュ

- `setInterval(load, 300000)` で 5 分ごとに再 fetch
- エラー時は "データ取得エラー" を表示して再試行を待つ

## デプロイ

- Vercel auto-deploy（`git push` のみ、`vercel --prod` 不要）
- プロジェクト: `euro-baseball-live`
- `vercel.json` のみでビルドステップなし
- デプロイ反映はハードリロード（Cmd+Shift+R）で確認

## 非実装（意図的に持たない）

| 機能 | 理由 |
|---|---|
| Guide / EPG ビュー | スクショ共有のペライチ用途のため単一リストのみ |
| フィルタ / 日付ナビ / 検索 | 今日特化で不要 |
| 週境界（火曜） | 「今日だけ」表示なので不要 |
| i18n / 英語切替 | 日本人向け固定 |
| PWA / Service Worker | 軽量ペライチのため不要 |
| 配信なしのグレーアウト | 一覧性を優先、視覚ノイズを減らす |
