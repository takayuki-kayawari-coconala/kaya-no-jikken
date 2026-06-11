👹：移行しない
🙌：移行する・したい
## rakeタスク定義ファイル一覧

1. 👹`send_from_pool_outsource_notification_activity.yaml`
    -　 rakeタスク: `jobmatching_notification_activity:send_from_pool`
    - 定義ファイル: `lib/tasks/jobmatching_notification_activity.rake:28`
    - 方針：NotificationActivityサービスにかなり依存している。移行しない
2. 👹`add_to_pool_outsource_notification_activity.yaml`
    - rakeタスク: `jobmatching_notification_activity:add_to_pool`
    - 定義ファイル: `lib/tasks/jobmatching_notification_activity.rake:21`
    - 方針：NotificationActivityサービスに依存はしていないが、users, profile db, active-record gemなどのrails固有に依存している。移行できなくはないが、notify系はrailsに寄せたほうが集約できて管理がしやすいのでは？と思う
3. 👹`expire_pool_outsource_notification_activity.yaml`
    - rake@タスク: `jobmatching_notification_activity:expire_pool`
    - 定義ファイル: `lib/tasks/jobmatching_notification_activity.rake:35
    - 方針：処理が簡単なので移行できなくはないが、移行コストに比べてメリットが少ない。コスパが悪い
4. 🙌`tech_job_feed_validate_and_register.yaml`
    - rakeタスク: `tech_job_feed:validate_and_register`
    - 定義ファイル: `lib/tasks/tech_job_feed.rake:3`
    - 方針：S3、Slack、テック案件フィードの取得だけでcoconala-api-railsなどの依存はない。jm-apiになっても同じ構成を維持できる
5. 👹`notify_outsource.yaml`
    - rakeタスク: `jobmatching_notification_activity:notify_outsource`
    - 定義ファイル: `lib/tasks/jobmatching_notification_activity.rake:5`
    - 方針：これもnotification_activitiesに強依存
6. 👹`delete_outsource_notification_activity.yaml`
    - rakeタスク: `jobmatching_notification_activity:delete_outsource_notification_activity`
    - 定義ファイル: `lib/tasks/jobmatching_notification_activity.rake:12`
    - 移行しても良いが、該当のレコードを取得して、キャッシュを削除するだけ。移行コストの割に移行後のメリットが少ない
7. 👹`oneshot_migrate_outsource_published_at.yaml`
    - rakeタスク: `oneshot:migrate_outsource_published_at`
    - 定義ファイル: `lib/tasks/oneshot_migrate_outsource_published_at.rake:64`
    - 方針：そもそもいらないと思うので削除してよさそう
8. 🙌`video_meeting_disconnect_expired.yaml`
    - rakeタスク: `video_meeting:disconnect_expired`
    - 定義ファイル: `lib/tasks/video_meeting.rake:5`
9. 🙌`video_meeting_kickoff_disconnect_expired.yaml`
    - rakeタスク: `video_meeting:kickoff:disconnect_expired`
    - 定義ファイル: `lib/tasks/video_meeting.rake:29`
10. 🙌`video_meeting_transfer_recording.yaml`
    - rakeタスク: `video_meeting:transfer_recording`
    - 定義ファイル: `lib/tasks/video_meeting.rake:10`
11. 👹`video_meeting_reminder_on_previous_day.yaml`
    - rakeタスク: `video_meeting:reminder_on_previous_day`
    - 定義ファイル: `lib/tasks/video_meeting.rake:15`
12. 👹`video_meeting_kickoff_reminder_on_previous_day.yaml`
    - rakeタスク: `video_meeting:kickoff:reminder_on_previous_day`
    - 定義ファイル: `lib/tasks/video_meeting.rake:34`


定義ファイルは以下の4ファイルに集約されています:
- `lib/tasks/jobmatching_notification_activity.rake` (5タスク)
- `lib/tasks/video_meeting.rake` (4タスク: `video_meeting` namespace 2件 + `video_meeting:kickoff` namespace 2件)
- `lib/tasks/tech_job_feed.rake` (1タスク)
- `lib/tasks/oneshot_migrate_outsource_published_at.rake` (1タスク)


jobmatching-api-railsに依存をしているなら、coconala-api-railsにまとめたほうがいいと思っている

