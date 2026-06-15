# 删改记录

公开仓库创建后，从 README、CHANGELOG、Release 正文中移除的特性描述及原因。

| # | 移除项 | 位置 | 原因 | 日期 |
|---|--------|------|------|------|
| 1 | MIGT (Migration Guidance) | README(EN+CN), CHANGELOG, Release | 竞品敏感，不宜过早公开 | 2026-06-15 |
| 2 | Metis Scheduler Feedback | README(EN+CN), CHANGELOG, Release | 竞品敏感，不宜过早公开 | 2026-06-15 |
| 3 | APEX 透明重定向 | README(EN+CN), CHANGELOG, Release | 内核截获技术细节不宜公开 | 2026-06-16 |
| 4 | 内核态 ELF 注入器 | README(EN+CN), CHANGELOG, Release | 敏感技术项 | 2026-06-16 |
| 5 | PHBT 描述中 ART 字眼 | README(EN) | 淡化 PHBT 与 ART 关联 | 2026-06-16 |

## 恢复条件

- 1/2：内部源码审查通过，确认不影响竞品态势后逐项评估
- 3/4：安全审查 + 法律评估通过
- 5：整体 ART 策略确定后考虑

## 版本

首次公开发布：v2026.06.14
