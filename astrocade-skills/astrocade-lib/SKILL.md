---
name: astrocade-lib
description: Astrocade 平台游戏的 lib API 使用指南，涵盖存档/读档、排行榜、资产与音频加载。当用户在 Astrocade 项目中问到存档、读档、关卡进度、积分保存、新游戏重置、排行榜、提交分数、展示榜单、加载图片/音频/动画资产，或提到 lib.saveUserGameState / lib.getUserGameState / lib.deleteUserGameState / lib.addPlayerScoreToLeaderboard / lib.getTopNEntriesFromLeaderboard / lib.getAsset / lib.getAnimationPlayer / lib.preloadAnimation 时触发。
---

# Astrocade lib API 索引

`lib` 对象在 `game_code.html` 中全局可用。根据话题读取对应的参考文档：

| 主题 | 文件 | 场景 |
|------|------|------|
| 存档 / 读档 | `references/save.md` | 关卡进度、总积分云端保存、新游戏重置 |
| 排行榜 | `references/leaderboard.md` | 提交分数、展示 Top N 榜单 |
| 资产 & 音频加载 | `references/assets.md` | 图片、音频、spritesheet 动画 |
