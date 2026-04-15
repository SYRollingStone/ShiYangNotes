# Astrocade 存档 API

存档功能不使用 `localStorage`，走 Astrocade 平台云端存档。

## API

```js
// 保存存档（state 必须是可 JSON 序列化的对象）
await lib.saveUserGameState(state)
// → Promise<UserGameStateResponse>

// 读取存档（response.state 为 null 表示没有存档）
await lib.getUserGameState()
// → Promise<UserGameStateResponse>

// 删除存档（适合"新游戏"重置）
await lib.deleteUserGameState()
// → Promise<{ success: boolean }>
```

## 标准存档模式

### 存档（关卡完成时）

```js
try {
    let totalScore = thisLevelScore;
    let highestLevel = currentLevel + 1; // 下一关的索引

    // 先读已有存档，再叠加
    try {
        const saved = await lib.getUserGameState();
        if (saved && saved.state) {
            totalScore = (saved.state.totalScore ?? 0) + thisLevelScore;
            highestLevel = Math.max(highestLevel, saved.state.highestLevel ?? 0);
        }
    } catch(e) {}

    await lib.saveUserGameState({ totalScore, highestLevel });
} catch(e) {}
```

### 读档（游戏初始化时）

```js
try {
    const saved = await lib.getUserGameState();
    if (saved && saved.state && saved.state.highestLevel > 0) {
        const hl = saved.state.highestLevel;
        const lvls = window.gameConfig?.levels ?? [];
        if (hl < lvls.length) {
            window._savedLevel = hl;
            // 更新开始按钮文字，让玩家继续上次进度
            const pb = document.getElementById('play-btn');
            if (pb) pb.textContent = 'Level ' + (hl + 1);
        }
    }
} catch(e) {}
```

### 重置（新游戏）

```js
try {
    await lib.deleteUserGameState();
    window._savedLevel = 0;
    document.getElementById('play-btn').textContent = 'Play';
} catch(e) {}
```

## 存档字段约定

| 字段 | 类型 | 说明 |
|------|------|------|
| `totalScore` | number | 所有关卡积分累计 |
| `highestLevel` | number | 已解锁的下一关索引（0 = 未开始，1 = 第2关已解锁） |

如需扩展（如各关单独分数、收集物状态等），直接在 state 对象中增加字段即可。

## 注意事项

- 所有调用均为 `async`，务必 `await`
- 用空 `catch(e){}` 静默处理网络失败，不要让存档失败崩溃游戏
- `getUserGameState()` 返回的 `response.state` 是 `null` 而不是 `undefined`，判断时用 `saved.state` 而不是 `saved`
- `highestLevel` 存的是**下一关索引**（完成第0关后存1），读取时注意换算为显示用的关卡号（+1）
