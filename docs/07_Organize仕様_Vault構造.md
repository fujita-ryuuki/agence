# 07 Organize 仕様 — 5層ノート構造・Obsidian Vault 形式・プロンプト設計

版：v1.0-draft｜2026-09-07｜【仮置き：論点#3】5層を標準とし、領域ごとに層の名称・追加を許容する。

## 1. 5層ノート構造

| 層 | layer | 内容 | structured スキーマ（UI 表示用 JSON） |
|---|---|---|---|
| 原則 | principle | 専門家が一貫して依拠する考え方・価値観・前提 | {statement, rationale, applies_when, exceptions[], related_criteria[]} |
| 判断基準・数値ライン | criterion | 「条件 X なら判断 Y」。閾値・数値ライン | {condition, judgement, thresholds[{metric, operator, value, unit}], caveats[], evidence_refs[]} |
| 診断フレーム | diagnostic | 相談を分類し原因を特定する手順 | {trigger, steps[{ask, if_yes, if_no}], outcomes[], linked_criteria[]} |
| 事例 | case | 疑似相談＋本人回答、実資料由来のケース | {situation, question, expert_answer, key_judgements[], applied_principles[], applied_criteria[]} |
| 出典 | source_ref | 元ファイル・疑似相談への参照（自動生成、専門家 UI では各ノートの出典パネルとして表示） | {source_type, source_name, locator, excerpt} |

領域可変ルール：`domains.layer_overrides`（jsonb）で層の表示名変更・追加層（例：法務なら「条文・判例」）を許容。追加層は `layer = 'custom:<slug>'` として保存し、回答生成では criterion 相当として扱う。

## 2. Obsidian Vault 形式

```
vault/
  00_index.md                     # MOC（Map of Content）：層別リンク一覧
  principles/<slug>.md
  criteria/<slug>.md
  diagnostics/<slug>.md
  cases/<slug>.md
  sources/<slug>.md
  _meta/manifest.json             # note_id↔path, versions, prompt_version, model, snapshot_id
```

frontmatter（各ノート共通）
```
---
id: <note uuid>
layer: criterion
title: 粗利率が20%を下回る場合の値上げ判断
confidence: 0.82
tags: [pricing, smb]
sources: [chunk:<uuid>, answer:<uuid>]
version: 3
batch: 2
review_status: approved
---
```
本文は Markdown。関連ノートは `[[principles/<slug>]]` の wikilink。専門家 UI では body_md ではなく structured を表示する（記法非露出）。

## 3. Organize プロンプト設計（packages/core/prompts/）

| ファイル | 用途 | モデル |
|---|---|---|
| extract_notes.v1.md | chunk 群→候補ノート（層・title・body・structured・confidence・出典抽粋）を JSON で出力 | sonnet |
| merge_notes.v1.md | 候補ノートと既存類似ノート→ update / new / duplicate の判定と統合本文 | sonnet |
| classify_extraction.v1.md | 会話系 chunk が「専門家の判断を含むか」を yes/no＋理由 | haiku |
| detect_conflict.v1.md | 類似 criterion 2件→ 矛盾か（同条件で異なる判断）・理由 | sonnet |
| generate_consultation.v1.md | 領域・難易度・persona→ 疑似相談 | sonnet |
| answer_structured.v1.md | 質問＋取得ノート→ 回答（診断フレーム順に思考、根拠 note_id を列挙） | sonnet |
| answer_raw_rag.v1.md | 質問＋chunk→ 回答 | sonnet |
| anonymize.v1.md | PII 抽出→ alias 表 | haiku |

extract_notes の必須ルール（プロンプトに明記）
- 一般論を書かない。専門家の資料・回答に「書かれていること」だけをノート化し、推測は confidence を下げる。
- 各ノートに出典抽粋（原文 40〜200 字）を必ず付ける。付けられない場合は出力しない。
- 数値・閾値は原文の単位のまま保持する。
- 1 chunk 群あたり最大 15 ノート。似た内容は 1 ノートに統合する。
- 出力は JSON Schema（packages/core/schemas/note.schema.json）に準拠。

## 4. 形式別の抽出方針（B-4 初期設定）

| 形式 | 優先抽出 | 除外 | 備考 |
|---|---|---|---|
| PDF / PPT / テキスト | 全文（見出し構造を保持） | 目次・ページ番号・定型フッター | スライドはノート欄も対象 |
| Excel | 見出し行＋数値ライン（閾値・比率・単価）を criterion 候補として重点抽出 | 空行・計算過程のみのシート | 表は Markdown テーブルで chunk 化 |
| 動画 | 専門家の発言（話者ラベルが専門家）を優先。質疑応答は case 候補 | 前置き・雑談・技術トラブル | 話者名は専門家が S03a で確定 |
| Slack | 専門家が「判断・助言・指摘」をしている発言とそのスレッド文脈 | 挨拶・日程調整・事務連絡・bot 投稿・リアクションのみ | スレッド単位で chunk |
| E-mail | 専門家の返信本文（助言・回答部分） | 署名・引用返信・自動通知・ニュースレター | スレッド単位で chunk |

分類器（classify_extraction）の出力を `chunks.extraction_decision` に保存。専門家は S05 の出典パネルまたは S03 の「抽出結果を見る」から反転できる。

## 5. 矛盾・重複・出典不明の判定基準【仮置き】
- 重複：埋め込み cosine 類似度 > 0.92 かつ merge_notes が duplicate 判定
- 矛盾：同 layer=criterion で condition の類似度 > 0.85 かつ judgement/thresholds が異なると detect_conflict が判定
- 出典不明：note_sources が 0 件、または全て orphaned

## 6. 確信度の付与
- extract_notes の自己申告 confidence（0〜1）を基礎に、出典抽粋の一致率（原文との文字列一致）で補正。出典が expert_answer 由来なら +0.1。
- 0.6 未満は「要確認」。専門家の approve で 1.0 に更新。

## 7. 構造化 vs 生RAG（H3）の対照条件
- 同一 chunks、同一モデル、同一評価セット、同一 top-k（8）、同一温度（0.2）。差分は「取得対象（notes vs chunks）」と「回答プロンプト」のみ。
