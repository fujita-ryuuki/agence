# 06 API 仕様（BFF / Server Actions）

版：v1.0-draft｜2026-09-07｜方針：画面操作は Next.js Server Actions、外部連携コールバックと Worker 用は Route Handlers（`/api/*`）。全て Supabase セッション必須。認可は RLS＋ロールチェック。

## 1. Server Actions（apps/web/lib/actions/*）

| 関数 | 入力 | 出力 | ロール | FR |
|---|---|---|---|---|
| saveDomain | {domainIds[], domainNote} | ok | expert | A-1 |
| previewConsultations | {} | [{question, persona, difficulty}]×3 | expert | A-1 |
| createUploadSource | {type, fileName, size, anonymize} | {sourceId, signedUploadUrl} | expert | A-2 |
| finalizeUploadSource | {sourceId} | job enqueued | expert | A-2 |
| createVideoUrlSource | {url, anonymize} | {sourceId} | expert | A-2 |
| listSources | {status?, type?, batchId?} | Source[] | expert/operator | A-2 |
| retrySource / deleteSource | {sourceId} | ok | expert | A-2 |
| getTranscript / updateTranscript / confirmTranscript | {transcriptId, segments?} | Transcript | expert | A-2-TR |
| listSlackChannels | {connectionId} | Channel[] | expert | A-2 |
| importSlack | {connectionId, channelIds[], from, to, includeDm, anonymize} | {sourceIds[]} | expert | A-2 |
| listGmailLabels | {connectionId} | Label[] | expert | A-2 |
| importGmail | {connectionId, labelIds[], from, to, participants[], anonymize} | {sourceIds[]} | expert | A-2 |
| listConsultations | {batchId?, status?} | Consultation[] | expert | A-3 |
| importConsultation | {question, situation?, anonymize} | Consultation | expert | A-3 |
| saveExpertAnswer | {consultationId, mode, content, status} | ExpertAnswer | expert | A-3 |
| uploadAnswerAudio | {consultationId} | {signedUploadUrl, transcriptId} | expert | A-3 |
| flagConsultation | {consultationId, flag} | ok | expert | A-3 |
| requestBatchClose | {batchId} | ok（operator へ通知） | expert | A-4 |
| listNotes | {layer, needsReviewOnly, minConfidence, q, batchId} | NoteCard[] | expert | B-2 |
| getNote | {noteId} | NoteDetail（structured, links, sources） | expert/operator | B-2, C-4 |
| reviewNote | {noteId, action, fixText?, reason?} | ok | expert | B-2 |
| listConflicts / resolveConflict | {conflictId, resolution, note?} | ok | expert | B-3 |
| toggleChunkExtraction | {chunkId, decision} | ok | expert | B-4 |
| listEvalItems | {batchId} | EvalItem[]（progress 込み） | expert | C-2 |
| getEvalItem | {consultationId} | {consultation, answersByBatch[], citations} | expert | C-1, C-4 |
| saveEvaluation | {agentAnswerId, score, tags[], comment?, myAnswer?, culpritNoteIds[]} | ok | expert | C-2 |
| getProgress | {} | KPI + series | expert/operator | C-3 |
| listBlindPairs / getBlindPair / saveBlindJudgement | {pairId, verdict, reason} | ok | reviewer | C-5 |
| logWork | {type, refId, seconds, source} | ok | expert | H5 |
| updateSettings / disconnectProvider / requestDataDeletion | … | ok | expert | 設定 |

operator（apps/web/lib/actions/admin/*）
inviteUser, suspendUser, listExperts, getExpertOverview, createBatch, closeBatch, regenerateAnswers({batchId, consultationId?, pipeline?}), listJobs, retryJob, generateConsultationSet({expertId, kind: eval|learning, batchId?, config}), lockEvalSet, editConsultation, getVaultTree, getVaultFile, listCosts, updateGlobalSettings, createBlindPairs({expertId, batchId, reviewerId})

## 2. Route Handlers

| パス | メソッド | 用途 |
|---|---|---|
| /api/oauth/slack/start, /callback | GET | Slack OAuth。state に expert_id 署名付き |
| /api/oauth/google/start, /callback | GET | Gmail OAuth |
| /api/storage/complete | POST | アップロード完了 Webhook（Storage イベント） |
| /api/internal/jobs/enqueue | POST | Worker 以外からの enqueue（service role 限定） |
| /api/health | GET | 監視 |

## 3. Worker ジョブ契約（apps/worker/jobs/*）

| job.type | payload | 完了条件 | 失敗時 |
|---|---|---|---|
| extract | {sourceId} | chunks 作成、sources.ready | sources.failed + error_message |
| transcribe | {sourceId | transcriptId, partIndex?} | transcripts.segments 完成 | 分割単位で再試行（attempts ≤3） |
| import_slack / import_email | {sourceId} | chunks 作成 | 上限超過は `limit_exceeded` エラー |
| anonymize | {sourceId} | chunks.content 置換、pii_maps | — |
| classify_extraction | {sourceId} | chunks.extraction_decision | pending のまま UI で手動 |
| embed | {chunkIds[] | noteIds[]} | embedding 設定 | — |
| generate_consultations | {expertId, kind, batchId?, config} | consultations 作成 | — |
| organize | {expertId, batchId} | notes/links/conflicts 更新 | batches.failed |
| snapshot_vault | {expertId, batchId} | vault_snapshots | — |
| generate_answers | {expertId, batchId, pipeline, consultationIds?} | agent_answers | 個別失敗は続行し集計 |
| close_batch（オーケストレータ） | {batchId} | organize→snapshot→answers×2 完了で closed | pipeline_run に段階を記録 |

全ジョブ共通：開始・終了時刻、input_size/output_size、cost_logs 追記、Sentry へ例外送信。

## 4. 型（packages/core/types.ts の要点）
NoteLayer = 'principle'|'criterion'|'diagnostic'|'case'|'source_ref'
DeductionTag = 'fact'|'principle'|'context'|'tone'|'shallow'|'other'
Pipeline = 'structured'|'raw_rag'
SourceType = 'pdf'|'pptx'|'text'|'excel'|'video'|'slack'|'email'
