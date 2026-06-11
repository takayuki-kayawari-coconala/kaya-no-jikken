# 応募促進通知バッチ仕様

`lib/tasks/jobmatching_notification_activity.rake` で定義されている、継続募集の応募促進通知（あなた宛通知）に関するバッチ処理の設計と仕様をまとめる。

関連 Linear: COC2-38 / 関連設計仕様書: `coconala-jobmatching/planning/応募促進/通知プール機能/設計仕様書.md`

---

## 1. 概要

### 1.1 目的
継続募集（`Outsource`）の中からマッチ条件を満たすユーザーに対して、応募促進のために「あなた宛通知」（`NotificationActivity`）を日次で送信する。

### 1.2 解決した課題
旧 `NotifyOutsourceService` は **1日1件制限をメモリ内カウントで実装**していたため、同日に複数の募集がマッチしたユーザーには **2件目以降の通知が破棄され、永久に送信されない**問題があった（翌日のバッチは前日公開分しか見ないため）。

これを解消するために `outsource_notification_activity_pools` テーブルを新設し、3つのタスク（`add_to_pool` / `send_from_pool` / `expire_pool`）で「プール方式」を実現した。

---

## 2. タスク一覧と実行スケジュール

`lib/tasks/jobmatching_notification_activity.rake` で定義されている 5 つのタスク：

| # | タスク名 | サービスクラス | 想定実行時刻 | 区分 |
|---|---|---|---|---|
| 1 | `notify_outsource` | `NotifyOutsourceService` | 毎日 16:00 | **旧実装**（移行後に廃止予定） |
| 2 | `delete_outsource_notification_activity` | `DeleteOutsourceNotificationActivityService` | 日次 | 履歴クリーンアップ |
| 3 | `add_to_pool` | `AddToPoolService` | 毎日 16:00 | **新実装** |
| 4 | `send_from_pool` | `SendFromPoolService` | 毎日 16:05 | **新実装** |
| 5 | `expire_pool` | `ExpirePoolService` | 毎日 16:10 | **新実装** |

> ⚠️ 現状の rake には旧 `notify_outsource` と新規3タスクが並列に定義されている。Rundeck 側で無効化されているかは別途確認が必要。

---

## 3. 関連DBテーブル

### 3.1 主要テーブル

| テーブル | DB | 役割 | データ寿命 |
|---|---|---|---|
| `outsource_notification_activity_pools` | main | 送信待ちキュー（新規） | `expired_at` まで（7日）または送信後翌日まで |
| `outsource_notification_activity_histories` | main | 送信履歴（重複防止＋削除判定キー） | 14日間 |
| `notification_activities` | main | ユーザーに表示される通知本体 | `expiration_datetime`（7日後）まで表示 |

### 3.2 参照テーブル

| テーブル | DB | 用途 |
|---|---|---|
| `outsources` | main | 募集本体（status, published_at 等） |
| `job_postings` | main | `is_public` フラグ |
| `users` | main | 退会判定 |
| `last_logins` | main | 60日以内ログイン判定 |
| `users_black_lists` | main | ブラックリスト除外 |
| `jobmatching_notification_activity_ng_users` | main | 通知NGユーザー除外 |
| `profile_data.profiles` | profile | 経験職種マッチング起点 |
| `profile_data.user_experience_jobs` | profile | 経験職種2年以上の判定 |
| `profile_data.user_jobs` / `profile_data.jobs` | profile | 業務領域マッチ |
| `profile_data.experience_job_category_job_mappings` | profile | 業務領域↔経験職種のマッピング |
| `profile_data.experience_job_categories` | profile | 通知本文のカテゴリ名取得 |

---

## 4. 全体フロー

```
                     ┌─────────────────────────────────────┐
                     │ 旧実装（廃止予定）                  │
                     │                                     │
   募集公開 ────────▶│ NotifyOutsourceService              │
                     │  ─ マッチング+送信を一気通貫        │
                     │  ─ メモリ内カウントで1件制限        │
                     │  ─ 2件目以降は破棄                  │
                     └─────────────────────────────────────┘

                     ┌─────────────────────────────────────────────────┐
                     │ 新実装（3タスク構成）                            │
                     │                                                 │
   募集公開 ────────▶│ ① AddToPoolService（16:00）                   │
                     │   前日公開募集 × マッチユーザー → プールに積む  │
                     │            │                                    │
                     │            ▼                                    │
                     │   outsource_notification_activity_pools         │
                     │   (UNIQUE: outsource_id × user_id, 7日有効)    │
                     │            │                                    │
                     │            ▼                                    │
                     │ ② SendFromPoolService（16:05）                │
                     │   ─ 公開中の募集のみ再フィルタ                 │
                     │   ─ ユーザー単位グループ化                     │
                     │   ─ 最新公開順に1件選択                        │
                     │   ─ 履歴未送信のもののみ送信                   │
                     │            │                                    │
                     │            ▼ INSERT                             │
                     │   notification_activities（通知本体）           │
                     │   outsource_notification_activity_histories     │
                     │            │                                    │
                     │            ▼                                    │
                     │ ③ ExpirePoolService（16:10）                  │
                     │   ─ 期限切れプール削除                         │
                     │   ─ 前日以前送信済みのプール削除               │
                     │     （BQ転送 AM2:00 完了後を狙う）             │
                     └─────────────────────────────────────────────────┘

                     ┌─────────────────────────────────────┐
                     │ 別系統（履歴クリーナー、日次）       │
                     │                                     │
                     │ DeleteOutsourceNotificationActivity │
                     │ Service                             │
                     │   ─ 14日経過した履歴を物理削除      │
                     │   ─ Redis キャッシュ全削除          │
                     └─────────────────────────────────────┘
```

---

## 5. 各タスクの詳細仕様

### 5.1 `notify_outsource`（旧実装）

**サービス**: `Outsources::NotificationActivity::NotifyOutsourceService`

#### 処理内容
1. 前日公開（`published_at: 1.day.ago.all_day`）かつ `status: published` かつ `job_postings.is_public: true` の募集を取得
2. 募集ごとに対象ユーザーを取得（退会していない / 60日以内ログイン / ブラックリスト外 / 経験職種マッチ）
3. 募集者本人・通知NGユーザーを除外
4. **メモリ内カウント** `@sent_notification_counts` を参照し、1日1件超なら破棄
5. `notification_activities` と `outsource_notification_activity_histories` をユーザー1000人単位で `import!`

#### 問題点（仕様書 §3.3）
| 観点 | 旧実装の挙動 |
|---|---|
| 1日1件制限 | メモリ内カウント（バッチ実行中のみ有効） |
| 2件目以降の通知 | **破棄され、永久に送信されない** |
| 翌日のリカバリ | できない（前日公開分しか対象にしない） |

→ 新実装（プール方式）に置き換える対象。

---

### 5.2 `add_to_pool`（新実装①：プール追加）

**サービス**: `Outsources::NotificationActivity::AddToPoolService`

#### 処理内容
1. 前日公開の募集を取得（除外職種 ID 212〜230 を除く）
2. 募集を `experience_job_category_id` でグループ化（N+1 回避のため、職種単位でユーザー取得）
3. 各職種について対象ユーザーを取得：
   - `User.not_withdrawal`
   - 60日以内に `last_logins.login_time` あり
   - `users_black_lists` に該当しない
   - profile DB で「経験職種2年以上 OR 業務領域マッチ」のいずれか
4. 除外条件適用：
   - 募集者本人（`user.id == outsource.user_id`）
   - `JobmatchingNotificationActivityNgUsers` に登録されたユーザー
5. 1000人単位で `OutsourceNotificationActivityPool.import!(on_duplicate_key_ignore: true)`
   - UNIQUE 制約 `(outsource_id, user_id)` で重複を無視
   - `expired_at = 7日後 end_of_day`

#### 書き込み先
| テーブル | 操作 |
|---|---|
| `outsource_notification_activity_pools` | INSERT（重複は無視） |

#### 主な定数
| 定数 | 値 | 意味 |
|---|---|---|
| `EXCLUDE_EXPERIENCE_JOB_CATEGORY_ID` | 212..230 | 通知対象外職種 |
| `EXPERIENCE_YEARS_COUNT` | 2 | 経験年数しきい値 |
| `LAST_LOGIN_ACTIVE_DATE` | 60 | 最終ログイン許容日数 |
| `POOL_EXPIRATION_DAYS` | 7 | プール有効期限 |

---

### 5.3 `send_from_pool`（新実装②：プールから送信）

**サービス**: `Outsources::NotificationActivity::SendFromPoolService`

#### 処理内容
1. `OutsourceNotificationActivityPool.sendable`（`expired_at > 現在`）から候補 ID 取得
2. **送信時点で有効な募集のみ再フィルタ**：
   - `Outsource.published`（status='PUBLISHED'）
   - `job_postings.is_public: true`
3. プールレコードを `outsource_published_at desc` でソート→ user_id でグループ化
4. 各ユーザーについて履歴 (`outsource_notification_activity_histories`) と突合し、**未送信で最新の募集 1 件**を選択
5. トランザクション内で：
   - `NotificationActivity` を `save!`（タイトル30文字超は `…` 切り詰め、空ならカテゴリ名のみのフォーマット）
   - `OutsourceNotificationActivityHistory.find_or_create_by!`
6. 末尾で `ActivityCacheService.all_delete`（Redis `flushdb`）

#### 書き込み先
| テーブル | 操作 |
|---|---|
| `notification_activities` | INSERT |
| `outsource_notification_activity_histories` | INSERT（既存ならスキップ） |
| `outsource_notification_activity_pools` | **削除しない**（後述） |

#### 重要な設計ポイント
- **1日1件制限の実現方法**: ユーザー単位グループ化 + 履歴突合のみで、メモリ内カウント不要
- **送信時の再フィルタ**: プール追加後に募集が削除・非公開化された場合、ここで除外される（プールには残るが通知は送られない）
- **プールを即削除しない理由**: BigQuery への append モード転送（AM 2:00）の制約のため、送信当日はプールに残し、翌日 `expire_pool` で削除する

#### 主な定数
| 定数 | 値 | 意味 |
|---|---|---|
| `MAX_NOTIFICATIONS_PER_USER_PER_DAY` | 1 | 1日あたりの送信上限（コメント上の文書化のみ） |
| `NOTIFICATION_ACTIVITY_EXPIRATION_DAYS` | 7 | 通知の有効期限 |

---

### 5.4 `expire_pool`（新実装③：プールのクリーンアップ）

**サービス**: `Outsources::NotificationActivity::ExpirePoolService`

#### 処理内容（`in_batches` + 1秒 sleep で負荷軽減）

##### (1) 期限切れ削除
- `OutsourceNotificationActivityPool.where('expired_at < ?', Time.current).in_batches { |pools| pools.delete_all }`

##### (2) 送信済み削除
- プールを 5000 件単位で走査
- 履歴テーブルから「`created_at < 当日0:00` かつ outsource_id がプールにあるもの」の `(outsource_id, user_id)` ペアを取得
- 一致するプールレコードを `delete_all`

#### 書き込み先
| テーブル | 操作 |
|---|---|
| `outsource_notification_activity_pools` | DELETE |

#### 重要な設計ポイント
- **「前日以前」に限定する理由**: BigQuery の append 転送（AM 2:00）が走るまで当日送信分は残す
- **プールテーブル起点でバッチ処理**: 履歴テーブル（100万件以上）よりプールテーブル（数万件）の方が小さいため

---

### 5.5 `delete_outsource_notification_activity`（履歴クリーナー）

**サービス**: `Outsources::NotificationActivity::DeleteOutsourceNotificationActivityService`

#### 処理内容
1. `OutsourceNotificationActivityHistory.where('created_at < 14日前')` を `in_batches` で `delete_all`（1秒 sleep）
2. 末尾で `ActivityCacheService.all_delete`

#### 書き込み先
| テーブル | 操作 |
|---|---|
| `outsource_notification_activity_histories` | DELETE（14日経過分） |

#### 効果
- 14日後に履歴がクリアされることで、**同じ募集をユーザーに再度通知できる状態に戻す**
- 注: プール方式では 7 日の有効期限内に送信されているはずなので、14日後の履歴削除はほぼ「再通知可能化」のリセット用途

#### 主な定数
| 定数 | 値 | 意味 |
|---|---|---|
| `OUTSOURCE_NOTIFICATION_ACTIVITY_EXPIRATION_DAYS` | 14 | 履歴の保持日数 |

---

## 6. タスク間の関係

### 6.1 テーブルアクセス対応表

| | pools | histories | notification_activities | outsources | profile DB |
|---|---|---|---|---|---|
| `notify_outsource`（旧） | ─ | R + W | W | R | R |
| `add_to_pool` | W (import) | ─ | ─ | R | R |
| `send_from_pool` | R (sendable) | R + W | W | R | R |
| `expire_pool` | R + DELETE | R | ─ | ─ | ─ |
| `delete_outsource_notification_activity` | ─ | DELETE | ─ | ─ | ─ |

### 6.2 履歴テーブルの二重用途

`outsource_notification_activity_histories` は3つの役割を持つ：

| 用途 | 利用するサービス |
|---|---|
| 重複送信防止フィルタ | `SendFromPoolService`（同じ募集×ユーザーへの再送を防ぐ） |
| プール削除判定キー | `ExpirePoolService`（送信済みプールを消すための突合キー） |
| 14日後リセット | `DeleteOutsourceNotificationActivityService`（再通知可能化） |

### 6.3 旧→新の置き換え関係

| 旧 `NotifyOutsourceService` の責務 | 新実装での担当 |
|---|---|
| 前日公開募集の取得 | `AddToPoolService`（同一クエリをコピー） |
| 対象ユーザーの抽出 | `AddToPoolService`（同一クエリをコピー） |
| 募集者本人・NGユーザー除外 | `AddToPoolService` |
| 1日1件制限（メモリ内カウント） | `SendFromPoolService`（DBグループ化＋履歴突合に置き換え） |
| 通知本体作成（`NotificationActivity`） | `SendFromPoolService` |
| 履歴作成（`OutsourceNotificationActivityHistory`） | `SendFromPoolService` |
| 「破棄された2件目以降」の救済 | **新規追加**: プール永続化により翌日以降に送信可能 |

---

## 7. 共通仕様

### 7.1 マッチング条件（`notify_outsource` / `add_to_pool` で共通）

#### 対象募集
```ruby
Outsource.includes(:job_posting)
         .published
         .where.not(experience_job_category_id: 212..230)
         .where(published_at: 1.day.ago.all_day)
         .where(job_postings: { is_public: true })
         .order(id: :asc)
```

#### 対象ユーザー（main DB）
- `User.not_withdrawal`
- `last_logins.login_time >= 60.day.ago.beginning_of_day`
- `users_black_lists: { id: nil }`

#### 対象ユーザー（profile DB、OR 条件）
1. 経験職種として募集職種を **2年以上**登録している
2. 募集職種にマッピングされた業務領域を登録している

#### 除外
- 募集者本人
- `JobmatchingNotificationActivityNgUsers` に該当するユーザー

### 7.2 通知本文フォーマット（`SendFromPoolService` / `NotifyOutsourceService` 共通）

```
タイトルあり: 「経歴にあった{カテゴリ名}案件をご紹介！「{title (30文字超は…で切り詰め)}」」
タイトル空:  「経歴にあった{カテゴリ名}案件をご紹介！」（警告ログ出力）
```

### 7.3 通知レコード仕様（`notification_activities`）

| カラム | 値 |
|---|---|
| `from_user_id` | `Settings.system_user_id` |
| `notification_type` | `:new_outsource`（=9） |
| `url` | `/job_matching/outsources/{outsource_ulid}?ref=activity` |
| `opened` | `:unread` |
| `expiration_datetime` | 7日後 end_of_day |
| `icon_type` | `:system_icon` |
| `unit_type` | `nil`（おまとめしない） |

---

## 8. 設計上の重要ポイント

### 8.1 BigQuery 転送との時間調整
- BQ 転送（appendモード）は **AM 2:00** に走る
- `SendFromPoolService` が送信直後にプールを消すと BQ に転送されない
- 解決策: `ExpirePoolService` が**翌日**（`created_at < 当日0:00` の履歴）に削除する設計
- ※ 仕様書 §4.2 Note に記載。シーケンス図には「送信時に即削除」と古い記述が残っているが、実装は仕様書 Note 方針に従っている

### 8.2 送信時の再フィルタ
- プール追加時に有効だった募集が、送信時点で削除・非公開化されている可能性がある
- `SendFromPoolService` は `Outsource.published` ∧ `job_postings.is_public: true` で再フィルタするため、**プールに残っていても送られない**
- 例: Day1 公開 → Day2 削除 → Day3 send_from_pool → 通知されない
- プールには `expired_at` まで残るが、最終的に `ExpirePoolService` が物理削除

### 8.3 マッチング結果の冪等性
- `outsource_notification_activity_pools` の UNIQUE 制約 `(outsource_id, user_id)`
- `AddToPoolService` は `on_duplicate_key_ignore: true` で再実行に強い
- `SendFromPoolService` は履歴を `find_or_create_by!` で作成するため、リトライ時の二重送信を防げる

### 8.4 1日1件制限の進化
| 実装 | 仕組み | 弱点 |
|---|---|---|
| 旧（メモリカウント） | `@sent_notification_counts[user_id]` | 2件目以降が永久ロスト |
| 新（DBグループ化＋履歴） | `pools.group_by(&:user_id).find { 未送信 }` | プール期限切れ後は機会喪失だが、7日間のリトライ猶予がある |

---

## 9. 外部インフラ依存

| インフラ | 利用箇所 | 用途 |
|---|---|---|
| MySQL (main DB) | 全タスク | メインデータ |
| MySQL (profile DB) | `notify_outsource` / `add_to_pool` | 経験職種マッチング |
| Redis | `send_from_pool` / `delete_outsource_notification_activity`（`ActivityCacheService.all_delete`） | キャッシュフラッシュ |
| BigQuery | `expire_pool` の前提 | プール送信実績の append 転送（AM 2:00、別経路） |
| Rundeck | 全タスク | スケジューラ |

---
![[スクリーンショット 2026-06-08 15.37.47.png]]


## 10. 参考資料

- 設計仕様書: `coconala-jobmatching/planning/応募促進/通知プール機能/設計仕様書.md`
- 処理シーケンス図: `coconala-jobmatching/planning/応募促進/通知プール機能/処理シーケンス図.md`
- Linear: COC2-38
- 関連PR
  - jobmatching-api-rails: https://github.com/welself/jobmatching-api-rails/pull/560
  - schema-ridgepole: https://github.com/welself/schema-ridgepole/pull/1377

---

## 11. ファイル対応表

| 対象 | パス |
|---|---|
| rake タスク定義 | `lib/tasks/jobmatching_notification_activity.rake` |
| 旧サービス | `app/services/outsources/notification_activity/notify_outsource_service.rb` |
| プール追加 | `app/services/outsources/notification_activity/add_to_pool_service.rb` |
| プール送信 | `app/services/outsources/notification_activity/send_from_pool_service.rb` |
| プール期限切れ | `app/services/outsources/notification_activity/expire_pool_service.rb` |
| 履歴削除 | `app/services/outsources/notification_activity/delete_outsource_notification_activity_service.rb` |
| プールモデル | `app/models/outsource_notification_activity_pool.rb` |
| 履歴モデル | `app/models/outsource_notification_activity_history.rb` |
| 通知本体モデル | `app/models/notification_activity.rb` |
| キャッシュサービス | `app/services/activity_cache_service.rb` |
