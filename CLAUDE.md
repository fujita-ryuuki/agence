# CLAUDE.md — Agence Agent Builder PoC

このリポジトリは「専門家が自分の知識を Agent に詰める」Web アプリ（PoC）です。購入者向け機能は作りません。仕様は `docs/` にあり、ここが唯一の正です。仕様に無いことは推測で実装せず、`docs/11_決定管理表.md` に「要確認」として追記してから進めてください。

## まず読む順
1. `docs/00_README.md`（全体像と用語）
2. `docs/01_要求定義書.md`（何のために作るか・成功の定義）
3. `docs/02_要件定義書_機能要件.md`（FR-ID ごとの受け入れ基準）
4. `docs/03_画面設計仕様.md`（画面 ID・ルーティング）
5. `docs/04_データモデル.md`（テーブル・RLS）
6. `docs/05_アーキテクチャ_技術選定.md`（構成・パイプライン）
7. `docs/06_API仕様.md`（Server Actions・ジョブ契約）
8. `docs/07_Organize仕様_Vault構造.md`（5 層ノート・Vault・プロンプト）
9. `docs/08_評価計測設計.md`、`docs/09_非機能要件_セキュリティ.md`
10. `docs/10_実装計画.md`（マイルストーン順に実装）
11. `docs/11_決定管理表.md`（【仮置き】値の一覧。config.ts と 1:1）

## 技術スタック（【仮置き】、docs/05 参照）
Next.js 15 App Router + TypeScript + Tailwind + shadcn/ui ／ Supabase（Postgres + pgvector + Auth + Storage + RLS）／ pg-boss Worker（Node）／ Claude API（sonnet: 整理・回答、haiku: 分類・匿名化）／ OpenAI embeddings ／ Deepgram（文字起こし）／ Slack・Google OAuth。
pnpm workspaces：`apps/web`, `apps/worker`, `packages/core`, `packages/db`。

## 実装ルール
- 1 マイルストーン = 1 PR 群。`docs/10_実装計画.md` の DoD を満たす Playwright テストを必ず追加する。
- Server Action は zod で入力検証し、先頭で `getSessionRole()` を呼ぶ。ロール不一致は 403。
- 全テーブルに `expert_id` と RLS。マイグレーションに `enable row level security` を必ず含める。
- LLM 呼び出しは `packages/core/llm` 経由のみ。プロンプトは `packages/core/prompts/*.vN.md`。変更は新バージョン追加で行う。
- 全ジョブは冪等。`jobs` と `cost_logs` に開始/終了/コストを必ず記録する。
- 評価用相談（`consultations.is_eval = true`）を Organize の入力に含めない（テストで担保）。
- 専門家 UI に Markdown 記法・wikilink・ファイルパスを表示しない（`notes.structured` を描画する）。
- 【仮置き】値は `packages/core/config.ts` にのみ書き、`settings` テーブルで上書き可能にする。ハードコード禁止。
- ログに PII・トークン・本文全文を出さない。
- UI は Claude Design の出力（`design/` 配下、DESIGN_BRIEF.md）を正とし、shadcn/ui コンポーネントで実装する。

## よく使うコマンド（想定）
pnpm i ／ pnpm dev（web + worker）／ pnpm db:migrate ／ pnpm db:types ／ pnpm test ／ pnpm e2e

## 用語
expert=専門家、reviewer=ブラインド評価者、operator=運用者、Batch=投入と評価の区切り、Organize=5 層構造化、Vault=Obsidian 形式の知識スナップショット、structured/raw_rag=回答生成の 2 系統。
