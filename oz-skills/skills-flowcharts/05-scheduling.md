# 五、本机调度与自动化

这一类目前只有 `scheduler`，但它代表了 `oz-skills` 的一个新边界：只做用户设备上的提醒或轻量任务，不接管云端 agent jobs。


### `scheduler`

用途：调度本机提醒或轻量本地动作，支持一次性、相对时间和重复 schedule，以及通知、Slack 或命令执行等交付方式。

```text
+------------------------------------------------------------+
| 触发：用户要求 remind me、schedule、run later、every weekday |
|       等未来动作。                                           |
|                                                            |
| 明确排除：cloud agents、background agentic jobs、Oz-managed  |
| workflows。                                                  |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. 解析请求                                                  |
| - Intent：reminder 还是 task                                 |
| - Action：提醒消息或要运行的命令/操作                        |
| - Schedule：absolute / relative / recurring                  |
| - Delivery/execution：notification / Slack / command 等      |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：信息是否足够？                                         |
|                                                            |
| 不足：问清楚 delivery、timezone、模糊时间或 task/reminder。   |
| 未说明类型：默认按 reminder 处理并确认。                     |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. 选择 backend                                              |
| macOS：launchd、osascript notification、AppleScript/JXA、shell |
| Linux：cron、systemd timers、notify-send、terminal fallback  |
| Windows：Task Scheduler、Toast notification、PowerShell      |
| External：Slack，必须用户明确指定且已配置。                  |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. Normalize schedule                                        |
| - 使用用户本地时区，除非用户另有说明。                       |
| - 转成 absolute timestamp / relative delay / recurring rule。 |
| - backend 需要 cron 等格式时内部转换。                       |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. 生成稳定名称                                              |
| - 短 kebab-case，例如 review-prs、weekly-backup。             |
| - 冲突时追加数字后缀。                                       |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 5. 创建 scheduled item                                       |
| - 创建 scheduler entry。                                     |
| - 必要时写小 helper script。                                  |
| - 存储后续 list/pause/update/delete 需要的 metadata。         |
| - 不假设 repo、git、secret 或 network。                       |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 6. 向用户确认                                                |
| 必须说明：                                                   |
| - Name                                                       |
| - What will happen                                           |
| - When, in local time                                        |
| - Delivery/execution method                                  |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 后续管理                                                     |
| - list：name/type/schedule/delivery/status                   |
| - pause / unpause / delete / update schedule / update action |
| - 删除前确认；指代不明时问清楚。                             |
+------------------------------------------------------------+
```

解释：原 skill：[scheduler](https://github.com/warpdotdev/oz-skills/blob/main/.agents/skills/scheduler/SKILL.md)。

要点：
- 它的关键边界是“on-device reminders and local actions only”，避免把本地提醒和云端 agent 工作流混为一谈。
- delivery method 不能默认假设；用户没说是通知、Slack 还是后台任务时必须问。
- 容易踩的坑：静默降级通知方式、用系统级 scheduler 而非用户级任务、调度 destructive action 前没确认、或没有留下可管理 metadata。
