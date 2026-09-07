# 00 README — Agence Agent Builder PoC 要件定義一式

版：v1.0-draft｜2026-09-07｜元資料：Agence_PoC_要求定義キックオフ_v0.2（2026-09-05）｜作成：藤田（AI生成・レビュー前ドラフト）

## 何を作るか（30秒版）
専門家が Web UI で「資料を入れる／疑似相談に答える／AI の整理結果を確かめる／Agent の回答を採点する」を繰り返すと、専門家本人が納得する回答を出す Agent が育つ——それを 3ヶ月・専門家3〜5名で検証する PoC アプリ。購入者向け機能（マーケットプレイス・課金・MCP 配信・Native アプリ）は作らない。

## ドキュメント構成と読者

| # | ファイル | 内容 | 主な読者 |
|---|---|---|---|
| 00 | 00_README.md | 本書。全体像・用語・使い方 | 全員 |
| 01 | 01_要求定義書.md | 目的・仮説・成功の定義・スコープ・機能要求一覧・検証プロトコル | 社長・PdM |
| 02 | 02_要件定義書_機能要件.md | FR-ID ごとの受け入れ基準（実装単位） | Claude Code・開発 |
| 03 | 03_画面設計仕様.md | 画面 ID・ルーティング・画面ごとの要素と状態 | Claude Design・Claude Code |
| 04 | 04_データモデル.md | テーブル・RLS・インデックス | Claude Code |
| 05 | 05_アーキテクチャ_技術選定.md | 構成図・技術選定と代替案・パイプライン・ディレクトリ | Claude Code・開発 |
| 06 | 06_API仕様.md | Server Actions・Route Handlers・ジョブ契約 | Claude Code |
| 07 | 07_Organize仕様_Vault構造.md | 5層ノート・Obsidian Vault 形式・プロンプト設計・形式別抽出方針 | Claude Code・PdM |
| 08 | 08_評価計測設計.md | 指標の SQL 定義・仮説との対応・評価セット運用 | PdM・Claude Code |
| 09 | 09_非機能要件_セキュリティ.md | 分離・非露出・OAuth・PII・再現性・チェックリスト | 開発 |
| 10 | 10_実装計画.md | マイルストーン M0〜M8・DoD・リスク | Claude Code・PdM |
| 11 | 11_決定管理表.md | 【仮置き】値の一覧と論点DBリンク、差し替え手順 | 全員 |
| — | CLAUDE.md | Claude Code の起点（読む順・実装ルール） | Claude Code |
| — | DESIGN_BRIEF.md | Claude Design の起点（原則・トーン・生成する画面） | Claude Design |

## 使い方
- Claude Design：`DESIGN_BRIEF.md` と `docs/03_画面設計仕様.md` を読み込ませ、優先順の画面から生成。出力は `design/` に置く。
- Claude Code：リポジトリ直下に `CLAUDE.md` と `docs/` を置いた状態で起動し、「docs/10_実装計画.md の M0 から着手」と指示する。【仮置き】値は `docs/11_決定管理表.md` → `packages/core/config.ts` で管理。
- 人：01 → 11 の順で読み、論点DB（Maxeff 論点管理_実行前意思決定DB v2 の「【Agence PoC】#1〜#9」）で判断する。

## 用語

| 用語 | 意味 |
|---|---|
| expert / reviewer / operator | 専門家／ブラインド評価者／運用者 |
| Batch | 投入と評価の区切り。Batch 0=既存データのみ、Batch 1〜3=疑似相談回答を追加 |
| 疑似相談（consultation） | LLM 生成または持ち込みの相談文。学習用と評価用（固定 25 件）に分かれる |
| Organize | 投入データと本人回答を 5層ノート（原則／判断基準・数値ライン／診断フレーム／事例／出典）に構造化する処理 |
| Vault | Obsidian 形式の知識スナップショット（サーバー内部のみ。専門家には見せない） |
| structured / raw_rag | 回答生成の 2 系統。5層ノート検索 vs 生チャンク検索（H3 比較用） |
| 納得度スコア／合格率／減点タグ／ブラインド比較 | 01 §4 の指標 |

## 決定状況の凡例
【確定】社長判断済み／【仮置き】PdM 推奨案で仮採用（11 で管理）／【対象外】PoC では作らない
