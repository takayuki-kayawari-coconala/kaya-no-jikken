# video_meeting.rake 移行に伴う技術的懸念調査レポート

## Context

`jobmatching-api-rails`（Rails 5.2.3、EOL前提で廃止予定）の `lib/tasks/video_meeting.rake` に定義された rake タスクを、Discussion #1216 の方針に従って以下の2方向に移行する。本レポートは「各タスクの技術的移行懸念」をタスク単位で洗い出すもの。スケジューラ（k8s workflow）の議論や移行体制面の所見は対象外。

| # | タスク | 方針 |
|---|---|---|
| 1 | `video_meeting:disconnect_expired` | 🙌 NestJS (jobmatching-api) へ |
| 2 | `video_meeting:resend_reminder_on_previous_day` (手動) | 👹 coconala-api-rails へ |
| 3 | `video_meeting:reminder_on_previous_day` | 👹 coconala-api-rails へ |
| 4 | `video_meeting:transfer_recording` | 🙌 NestJS へ |
| 5 | `video_meeting:kickoff:disconnect_expired` | 🙌 NestJS へ |
| 6 | `video_meeting:kickoff:reminder_on_previous_day` | 👹 coconala-api-rails へ |
| 7 | `video_meeting:kickoff:resend_reminder_on_previous_day` (手動) | 👹 coconala-api-rails へ |

resend 系（手動実行）は通常版と同じ方針（ユーザー確認済み）。

---

## 🙌 NestJS（jobmatching-api）への移行組

### 共通の前提（既存資産・パターン）

- **CLI コマンド基盤**: `nest-commander` 導入済み。`src/cli.ts` がエントリポイント、`src/commands/interview-completion-notification.command.ts:1-57` に既存実装サンプルあり（三者面談**完了通知**バッチ。本件の `disconnect_expired` 系の通知部分にあたる）
- **バッチサービス雛形**: `src/outsource-online-meeting/service/interview-completion-notification.service.ts:1-240` がリファレンス。`execute()` が `{ processedCount, skippedCount }` を返す。冪等性チェック、`MAX_MEETINGS_PER_BATCH = 100` 制限あり
- **通知の委譲設計**: `src/transaction-activity/service/abstract-notifier.service.ts:43-58` の通り、NestJS は通知の実体を Rails の `/api/v1/job_matching/notification/{endpoint}` に HTTP POST するだけ。**メーラー（Sendgrid）と TransactionActivity 作成は Rails 側に残る**
- **Prisma スキーマ**: `prisma/schema/outsource/` に `OutsourceOnlineMeeting`, `OutsourceKickoffMeeting`, `ZoomMeeting`, `ZoomMeetingRegistrant` 定義済み
- **S3 クライアント**: `src/package/aws/s3/s3-client.service.ts` 整備済み
- **未整備**: Zoom API クライアント、ActiveJob 相当の非同期実行基盤、`@nestjs/schedule`（k8s workflow 前提のため不要）

### #1 `video_meeting:disconnect_expired`（三者面談 終了処理）

対象サービス: `app/services/outsources/online_meeting/disconnect_expired_meeting_service.rb:1-38`

#### 既存処理の構造

```
expired_meetings.find_each(batch_size: 100) do |online_meeting|
  ZoomMeetingEndUpJob.perform_later(zoom_meeting)        # 非同期で Zoom API end
  Outsources::OnlineMeeting::EndUpService.execute        # ステータス遷移 + 通知
rescue StandardError
  SlackService.post_message(...)
end
```

#### 技術的移行懸念

1. **Zoom API クライアント未整備（最重）**
   - Rails 側 `lib/zoom/meeting_api_client.rb:204-226` は OAuth2 Server-to-Server + Redis（`Rails.cache`）でアクセストークンを45分キャッシュ
   - NestJS にこのクライアント実装がない。新規構築には：トークン取得・キャッシュ（Redis 接続も含む）、Faraday 相当のリトライ（10秒間隔×3回）、`ResponseError` / `RecordingUncompleteError` 相当のエラー型設計が必要
   - 認証情報の管理場所が Rails: `Settings.zoom_meeting.*` → NestJS: `@nestjs/config` への移植が必要

2. **`ZoomMeetingEndUpJob`（ActiveJob 非同期）の代替**
   - 現状は Zoom API 呼び出しを非同期化してバッチ実行時間を短縮している
   - NestJS は ActiveJob 相当の汎用キュー（Bull など）未導入。同期実行に倒すか、Bull/RabbitMQ を新規導入する判断が必要
   - 同期実行に倒す場合、バッチ全体の実行時間が Zoom API レイテンシ × 件数に比例する点を許容できるか要確認

3. **`Outsources::OnlineMeeting::EndUpService` の移植範囲**
   - このサービスは「ステータス遷移」+「アプリ内タスク（TransactionActivity）作成」+「メーラー送信」を行う複合サービス
   - NestJS の通知委譲設計（`AbstractNotifierService`）に従うなら、通知部分は **Rails 側に `/api/v1/job_matching/notification/online_meeting_finished` 相当のエンドポイントを追加し、NestJS から POST する** 形になる
   - ただし Rails 側がいずれ廃止される前提なので、廃止タイミングと通知委譲設計の寿命に整合性があるかを設計時に確認すべき
   - ステータス遷移ロジック（`ApplicationStatusTransitionService`）は NestJS 側で Prisma トランザクションとして実装し直す必要

4. **`OutsourceOnlineMeeting.expired` スコープの Prisma 化**
   - Rails スコープは JOIN（`zoom_meetings`）+ 時刻条件（`MEETING_ENDING_BUFFER = 60.minutes`）の複合
   - Prisma クエリへ書き換える際、`include` + `where` の組み合わせで再現は可能だが、対象抽出ロジックの単体テスト整備が必須

5. **Slack 通知**
   - Rails: `SlackService` + Faraday + `Settings.slack.zoom_meeting_notify.*`
   - NestJS: 既に `@slack/web-api` + `src/package/slack/slack-client.service.ts` 整備済み。チャネル設定の移植のみで対応可能

6. **既存 `interview-completion-notification` との関係整理**
   - NestJS 側に類似機能 `InterviewCompletionNotificationService`（面談開始時刻 + N分後の通知）が既に存在する
   - これは Rails 側の `disconnect_expired` とは別の概念だが、対象抽出ロジック（面談終了時間帯の判定）が部分的に重なる。**冪等性チェック・除外条件が二重実行で衝突しないか**、設計時に必ず突き合わせが必要

### #5 `video_meeting:kickoff:disconnect_expired`（打ち合わせ面談 終了処理）

対象サービス: `app/services/outsources/kickoff_meeting/disconnect_expired_meeting_service.rb:1-32`

#### #1 との差分

- ステータス遷移先が異なる（`:interviewed` → `:kickoff_finished`）
- `EndUpService` を呼ばず、`ApplicationStatusTransitionService` のみ呼ぶ（通知ロジックは別経路）

#### 技術的移行懸念

- **#1 と同じ Zoom API・ActiveJob・スコープ書き換えの懸念がすべて当てはまる**
- 追加で、`OutsourceKickoffMeeting.expired` スコープが `outsource_applications: { status: :kickoff_scheduled }` を含む多段 JOIN になっている。Prisma 化時にステータス enum 値が NestJS 側と一致しているか要確認
- 通知が直接呼ばれない分、#1 より移植は軽い

### #4 `video_meeting:transfer_recording`（Zoom 録画の S3 転送）

対象サービス: `app/services/zoom/transfer_meeting_recording_service.rb:1-78`

#### 技術的移行懸念（**最も重い**）

1. **Zoom API クライアント未整備**（#1 と共通）

2. **多層リトライ機構の再現**
   - Zoom 録画取得: `Retryable.retryable(sleep: 300, tries: 3)` で `RecordingUncompleteError` を待つ設計（録画が Zoom 側で生成中の場合の待機）
   - S3 アップロード: `Retryable.retryable(on: [Aws::S3::MultipartUploadError], sleep: 0.5, tries: 3)`
   - Faraday 内部のコネクションリトライ（10秒×3）
   - これら3層のリトライセマンティクスを Node.js（`p-retry` / `async-retry` 等）で同等に再現する必要

3. **AWS S3 ストリーミングアップロード**
   - Rails: `S3IamClient#upload_stream` でブロックを介したストリーミングアップロード（`@zoom_client.download_recording` のチャンクを `stream << chunk` で逐次書き込み）
   - NestJS: `@aws-sdk/client-s3` の `Upload`（`@aws-sdk/lib-storage`）で同等のマルチパートストリーミングを実装する必要。バイナリ・大容量ファイル（録画）でのメモリ効率検証が必須

4. **`ZoomMeeting.ended_up` スコープ**
   - `where(ended_at: Date.yesterday.all_day)` → タイムゾーン依存。`DateService` 経由で JST の前日範囲を生成する必要あり

5. **`ZoomMeetingRecordingDeleteJob`（ActiveJob 非同期で Zoom 側録画削除）**
   - 転送成功後に Zoom 側録画を削除するジョブ。NestJS 同期化 or 別バッチ化を選択する必要
   - 削除失敗時のリトライ・冪等性設計（S3 アップロード済みなのに Zoom 削除失敗で次回も再転送される、等）の設計検討必須

6. **AWS 認証情報**
   - 本番は IAM Role（EC2 インスタンスプロフィール）想定。**NestJS が動く k8s クラスタの IAM Role / IRSA 構成と整合するか**、S3 バケット `Settings.s3_bucket.video_files` への書き込み権限を持つか要確認

7. **coconala-api-rails にもほぼ同一の実装が存在**
   - 「独立性が高く NestJS に寄せやすい」と判断されているが、結果的に **NestJS / coconala-api-rails / jobmatching-api-rails の3箇所** に類似コードが存在する状態を経由する点に注意。古い実装の停止漏れ防止のためにオペレーション設計（旧バッチの実行を確実に止める手順）が必要

---

## 👹 coconala-api-rails への移行組

### 共通の前提（既存資産・課題）

- **同じ Rails 5.2.3 / Ruby 2.5.5**。コード規約は同等で平移行はしやすい
- **通知メカニズムが異なる**: jobmatching-api-rails は「`Mailer.deliver_later` + `Outsources::TransactionActivities::*Service`」の2系統、coconala-api-rails は「`TransactionActivity.create_*_notice` 一本」。**そのままコピペでは動作しない**
- **`OutsourceOnlineMeeting` 系モデルは coconala 側にも存在**（既に類似 reminder サービスあり: `coconala-api-rails/app/services/outsources/online_meeting/reminder_on_previous_day_service.rb`）
- **`OutsourceKickoffMeeting` 系モデルが coconala 側に存在しない（要確認）**: テーブル管理が物理的に分離されているのか、DB が共有でモデルだけ未定義なのかで対応コストが大きく変わる → 事前検証必須

### #3 `video_meeting:reminder_on_previous_day`（三者面談 前日通知）

対象サービス: `app/services/outsources/online_meeting/reminder_on_previous_day_service.rb:1-102`

#### 技術的移行懸念

1. **通知メカニズムの差異吸収**
   - jobmatching 側: `Outsources::ClientMailer.meeting_schedule.deliver_later` + `Outsources::TransactionActivities::OnlineMeetingRemindForClientService`
   - coconala 側: 既存 reminder サービスは `TransactionActivity.create_outsource_application_notice(params)` 一本
   - 同等の通知挙動を coconala のパターンに合わせて書き換える際、**現在送られているメールやアプリ内タスクの「件数・宛先・本文」が変わらないこと**を保証する設計検証が必須

2. **両リポジトリの reminder 実装の差分把握**
   - coconala-api-rails には既に `Outsources::OnlineMeeting::ReminderOnPreviousDayService` が存在する
   - **既存 coconala 実装をそのまま使えば足りるのか、jobmatching 側のロジック（応募者/募集者の区別、`scheduled` ステータスフィルタ、`ZoomMeetingRegistrant.has_online_meetings_tomorrow` スコープなど）に独自要素があるのか**、機能差分の洗い出しが移行前に必要

3. **DB 共有・参照可否の確認**
   - `OutsourceOnlineMeeting`, `OutsourceApplication`, `ZoomMeeting`, `ZoomMeetingRegistrant` が両リポジトリの ActiveRecord から同じテーブルを参照しているか。テーブル管理リポジトリと マイグレーション運用がどちらにあるか要確認

4. **メーラーテンプレート**
   - `Outsources::ClientMailer.meeting_schedule` / `SupplierMailer.meeting_schedule` のテンプレートが coconala 側に存在するか確認。なければ移植が必要

### #2 `video_meeting:resend_reminder_on_previous_day`（手動再送）

対象サービス: `app/services/outsources/online_meeting/resend_reminder_on_previous_day_service.rb:1-135`

#### 技術的移行懸念

- **#3 と同じ通知メカニズム差異の問題がすべて当てはまる**
- 加えて、rake タスク引数のパース（`[1,2,3]` 形式）と `target_outsource_application_ids` パラメータの受け渡しを coconala 側 rake で同じ UX で再現する必要
- 手動実行頻度が低いため、移行コストの優先度は #3 完了後でよいと考えられる

### #6 `video_meeting:kickoff:reminder_on_previous_day`（打ち合わせ 前日通知）

対象サービス: `app/services/outsources/kickoff_meeting/reminder_on_previous_day_service.rb:1-108`

#### 技術的移行懸念（**この方針の最大の論点**）

1. **`OutsourceKickoffMeeting` モデル・テーブルが coconala-api-rails 側に存在しない可能性**
   - 別並列調査では `coconala-api-rails/app/models/outsource_kickoff_meeting.rb` が見当たらない
   - パターン A: 同一 DB を共有しているがモデル未定義 → coconala-api-rails に **モデル・スコープ・関連定義を追加するだけ**で済む（中程度コスト）
   - パターン B: テーブルそのものが jobmatching-api-rails 側固有 DB → **テーブル移管・データ移行を含む大規模対応** が必要（移行先選定の再考が必要なレベルの大きな懸念）
   - **どちらかをまず確定させるのが移行可否判断の前提**

2. **`KickoffMeeting` 系の TransactionActivities が coconala 側に存在しない**
   - `Outsources::TransactionActivities::KickoffMeetingRemindForClientService` / `ForSupplierService` 相当が coconala 側にない可能性が高い
   - coconala の通知パターン（`TransactionActivity.create_*_notice` 一本）に合わせて新規実装する必要

3. **メーラーの新規追加**
   - `Outsources::ClientMailer#kickoff_meeting_schedule` / `SupplierMailer#kickoff_meeting_schedule` を coconala 側に新規追加する必要（テンプレート・i18n 含む）

4. **ステータス enum**
   - `OutsourceApplication.statuses[:kickoff_scheduled]` が coconala 側 enum に存在するか確認。存在しない場合は enum 拡張のマイグレーションが必要

### #7 `video_meeting:kickoff:resend_reminder_on_previous_day`（手動再送）

対象サービス: `app/services/outsources/kickoff_meeting/resend_reminder_on_previous_day_service.rb:1-134`

#### 技術的移行懸念

- **#6 の問題（モデル・テーブル・通知・メーラー新規追加）がすべて当てはまる**
- #6 が解決して初めて成立するため、#6 の方針確定が前提

---

## 横断的に押さえておくべき技術的不確実性

| #   | 確認項目                                                                                                                     | なぜ重要か                                             |
| --- | ------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------- |
| 1   | `OutsourceKickoffMeeting` 系テーブルの管理リポジトリ                                                                                  | #6 / #7 の coconala 移行可否を左右する最大の不確実性               |
| 2   | Rails と NestJS が同じ DB（同じテーブル）を共有しているか                                                                                    | Prisma スキーマと ActiveRecord モデルの整合性、書き込み排他制御の必要性に直結 |
| 3   | NestJS 動作環境（k8s）の IAM Role が S3 `video_files` バケットに書けるか                                                                  | #4 の前提                                            |
| 4   | NestJS 環境で Redis が利用可能か                                                                                                  | Zoom OAuth トークンキャッシュの前提                           |
| 5   | Rails 側の通知委譲 API（`/api/v1/job_matching/notification/*`）に `online_meeting_finished` / `kickoff_meeting_finished` 相当が既にあるか | #1 / #5 の通知設計を委譲方式に倒すかどうかの判断材料                    |
| 6   | 既存 `interview-completion-notification` バッチと #1 の対象抽出条件の重複範囲                                                              | 二重通知・二重ステータス遷移を防ぐ前提                               |

---

## 検証（移行着手前に必ず行うべき確認）

実装着手前に以下を **実環境ないしリポジトリの最新状態で** 検証してから個別タスクの実装計画に進む。

1. `find /Users/takayuki.kayawari/ghq/github.com/welself/coconala-api-rails -name "outsource_kickoff_meeting*"` で coconala 側にモデル/マイグレーション/テストが存在するか確認
2. 両 Rails リポジトリの `db/schema.rb`（または同等）で `outsource_kickoff_meetings` / `outsource_online_meetings` / `zoom_meetings` テーブル定義の有無を突き合わせる
3. NestJS の `src/transaction-activity/` 配下に `online-meeting-finished-notifier.service.ts` 等が **既に存在する** ことを確認した（`#1 #5` の通知委譲の前例として活用可能）
4. Rails の `config/routes.rb` で `/api/v1/job_matching/notification/*` のルーティングを確認し、`disconnect_expired` の通知に流用できる既存 endpoint があるかを洗い出す
5. `jobmatching-api/CLAUDE.md` の `.claude/rules/service-patterns.md` を再読し、新規バッチサービスの実装規約を遵守できる構造に落とせるか確認
6. Argo Workflows の既存 CronWorkflow（`interview-completion-notification` 用）を参考に、対象3バッチ（`disconnect_expired` × 2 + `transfer_recording`）の `cron` 設定とリトライ設定を起こす

## 参照ファイル

- `lib/tasks/video_meeting.rake:1-46` （本件タスク定義）
- `app/services/outsources/online_meeting/disconnect_expired_meeting_service.rb:1-38`
- `app/services/outsources/online_meeting/reminder_on_previous_day_service.rb:1-102`
- `app/services/outsources/online_meeting/resend_reminder_on_previous_day_service.rb:1-135`
- `app/services/outsources/kickoff_meeting/disconnect_expired_meeting_service.rb:1-32`
- `app/services/outsources/kickoff_meeting/reminder_on_previous_day_service.rb:1-108`
- `app/services/outsources/kickoff_meeting/resend_reminder_on_previous_day_service.rb:1-134`
- `app/services/zoom/transfer_meeting_recording_service.rb:1-78`
- `lib/zoom/meeting_api_client.rb:1-245`
- `app/models/s3_iam_client.rb:1-85`
- `jobmatching-api/src/commands/interview-completion-notification.command.ts:1-57`
- `jobmatching-api/src/outsource-online-meeting/service/interview-completion-notification.service.ts:1-240`
- `jobmatching-api/src/transaction-activity/service/abstract-notifier.service.ts:1-86`
- `jobmatching-api/prisma/schema/outsource/` （Prisma スキーマ群）
- `coconala-api-rails/app/services/outsources/online_meeting/reminder_on_previous_day_service.rb:1-107`（既存実装）
