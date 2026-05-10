# Routines (claude.ai cloud Routine 用 prompt 集)

> 2026-05-08 v2 対応版。`docs/data-model-v2.md` および `docs/phase11-secretary-quality.md` §3, §8 に準拠。
> 2026-05-10 から **bootstrap pattern** で自動更新化 (Phase 1 T1)。詳細は本 README §bootstrap pattern。

## 配置済 prompt

| ファイル | cron | 役割 |
|---|---|---|
| `morning-digest.md` | `0 7 * * *` | 毎朝 7:00 JST、今日の便り |
| `evening-digest.md` | `30 22 * * *` | 毎晩 22:30 JST、明日の便り |
| `weekly-review.md` | `0 23 * * 0` | 日曜 23:00、進捗振り返り → WEEKLY_REVIEW.md 追記 + Telegram |
| `achievements-log.md` | `30 23 * * 0` | 日曜 23:30、controlled な達成のみ → ACHIEVEMENTS.md 追記 + Telegram |
| `mycontext-update.md` | `45 23 * * 0` | 日曜 23:45、MyContext.md 再生成 (Telegram 通知なし、裏方) |
| `monthly-strategy-review.md` | `0 9 1 * *` | 月初 9:00、career_milestones 提案 → 承認カード Telegram |
| `inbox-triage.md` | `0 18 * * 6` | 土曜 18:00、滞留 inbox の振り分け促し |
| `code-review.md` | `0 21 * * *` | 毎日 21:00、対象 repo の直近 24h commit 整合性チェック (Phase 1 T4、PoC: oncall-app のみ) |
| `priority-assist.md` | `0 6 * * *` | 毎朝 6:00、当日負荷を集約し `claude-shared/TODAY.md` 書き出し (Phase 11.3、Telegram 通知なし) |
| `progress-nudge.md` | `0 18 * * *` | 毎日 18:00、停滞 project (7 日以上動かず) を 1 件だけ優しく声かけ (Phase 11.3) |
| `inbox-merge-suggest.md` | `0 23 * * *` | 毎日 23:00、inbox の重複候補を検出 → 統合提案を承認カードで送信 (Phase 11.3) |

cron は claude.ai/code/routines に登録するもの。月次プリセットが UI に無いため、monthly-strategy-review は CLI `/schedule update` で `0 9 1 * *` を設定する必要あり。

## v2 (data-model-v2.md) との関係

v2 では `lifecycle_stage` が `state` に置換され、Inbox は「動的 view (`context_id=null AND created_at >= now-7d`)」化される予定。**現行 routine は dual-mode (legacy `lifecycle_stage` 参照) で動作**。migration 023 で旧カラム drop 後、routines を `state` ベースに書き換える必要あり (TODO)。

## 共通スタイル制約

`docs/phase11-secretary-quality.md` §3-A〜F:

- Markdown 記法 (`##` `**` `_` backtick `[]()`) 禁止
- bullet 文字 (`・` `-` `*`) 行頭で禁止
- セクション見出しは `【】` で囲む
- 絵文字禁止 (🌅 📥 🏆 等)
- ラベル付け禁止 (「優先度: 高」「達成 1:」等の plain ラベル)
- 1 行 30 字目安、全体 8〜20 行 (routine 別)
- 統合された文章で書く (羅列禁止)

## 廃止された依存

実データ調査 (2026-05-05) で以下が機能していないと判明:

- `mental_signals_recent_avg`: claude.ai 経由では mental_signals テーブルに書き込まれない (旧 nano classifier だけが書いていた)。**Routine から参照しない**
- `identity_facts`: 0 件、手動マスター扱い (MyContext.md "私について" は手動更新)
- 自発的 dev_note 記録: claude.ai が会話中に `create_item(type='dev_note')` を呼ぶ運用は定着していない。**進捗の主ソースは git commits に転換**

## bootstrap pattern (2026-05-10 採用、実行層 Phase 1 T1)

### 仕組み

claude.ai/code/routines に登録するプロンプトは下記 **3 行のテンプレートだけ**:

```
このルーチンの最新仕様を以下から取得して実行してください:
https://raw.githubusercontent.com/asano1234567890my/ai-secretary-routines/main/routines/<name>.md
最新の v2 仕様で動作してください。
```

(`<name>` は `morning-digest` 等の routine 名)

### 自動同期の流れ

1. ユーザー or AI が `ai-secretary/routines/<name>.md` を編集
2. push to master → GitHub Actions `sync-routines-public.yml` が発火
3. `asano1234567890my/ai-secretary-routines/routines/<name>.md` (public repo) にミラー (一方向)
4. 次回 Routine 実行時、claude.ai が raw URL から最新版を fetch して prompt として実行

### mirror 先が claude-shared でなく ai-secretary-routines な理由

claude-shared は private で raw URL が 404 になるため、bootstrap pattern が機能しない。
LIFE_STRATEGY.md / MyContext.md 等の sensitive ファイルは public 化したくないので
claude-shared は private のまま、routines だけを別 public repo `ai-secretary-routines` に
ミラーする方式を採用 (2026-05-10 pivot)。

### 利点

- routines 編集 → push だけで反映、Web UI 再登録不要
- prompt 履歴が git に残る
- 7 本まとめて同じ仕組み
- claude.ai 会話で「morning-digest のこの行変えて」→ commit → 翌朝反映

### B 案 (PAT を Routine 本文に貼る) を採用しない理由

`docs/phase11-secretary-quality.md` §8 + dev_note `0b094623`: 2026-05-03 の `.claude/settings.local.json` シークレット漏洩事故と同型のリスクのため A 案 (claude-shared mirror) を採用。

### 初回セットアップ (MASA 作業、1 回だけ)

#### 1. GitHub Actions secret 登録 (1 回)

ai-secretary repo に Personal Access Token を Actions secret として登録する。既存 `GITHUB_PAT` (Render env で使用中、`contents:write` scope あり) を流用可能:

```
gh secret set CLAUDE_SHARED_SYNC_PAT -R asano1234567890my/ai-secretary
# (PAT を貼り付け)
```

scope 要件: `ai-secretary-routines` repo に対して `contents:write` (push 権限)。fine-grained PAT を新規発行する場合は Repository access = Only `ai-secretary-routines`、Permissions = Contents: Read and write。

(secret 名 `CLAUDE_SHARED_SYNC_PAT` は historical reasons で残るが実態は `ai-secretary-routines` への push に使う。リネームしたい場合は `gh secret set ROUTINES_SYNC_PAT` で別名登録 + workflow YAML の `${{ secrets.X }}` を書き換え)

#### 2. claude.ai/code/routines に bootstrap 3 行を 7 本登録

各 routine (morning-digest / evening-digest / weekly-review / achievements-log / mycontext-update / monthly-strategy-review / inbox-triage) について claude.ai/code/routines で本文を上記 3 行テンプレートに置き換え。`<name>` は routine 名で置換。

cron は各 routine の冒頭コメントに記載 (例: morning-digest = `0 7 * * *`)。monthly-strategy-review は UI に月次プリセット無いため CLI `/schedule update` で `0 9 1 * *` に設定。

#### 3. 動作確認

`ai-secretary/routines/morning-digest.md` を 1 行編集 → push → GitHub Actions が走り claude-shared にミラーされるのを確認 → 翌朝 Routine が新版を fetch して実行されるか Telegram 出力で確認。

OK なら 7 本展開。

### 既存 daily 化していた Routine の削除

claude.ai/code/routines で旧版 (毎日 career_milestones proposal を生成していたもの) は削除する。本リポの新版 monthly-strategy-review は **monthly-only** (`0 9 1 * *`) で運用。

## 作業中

ユーザー判断:
- メンタル signal 入力経路 (案 A Health Sync vs 案 B 朝夕気分カード) → 別フェーズで実装予定
- v2 完全移行 (migration 023) 後の routines 書き換え
