# Astrocade Leaderboard API

## API

```js
// 提交分数（同时返回排行榜数据）
await lib.addPlayerScoreToLeaderboard(score, numEntries?)
// → Promise<LeaderboardResponse>

// 仅查询，不提交
await lib.getTopNEntriesFromLeaderboard(numEntries?)
// → Promise<LeaderboardResponse>
```

**LeaderboardResponse 结构：**
```js
{
  entries: [
    { username: string, profilePicture: string, score: number }
  ],
  userRank: number | null   // null = 玩家未上榜
}
```

- `numEntries` 默认值由平台决定，通常取 10
- `profilePicture` 是头像图片 URL，可直接用于 `<img src>`

## 标准用法：关卡结束后提交并展示

```js
// 提交分数
try {
    const lb = await lib.addPlayerScoreToLeaderboard(score.total);
    renderLeaderboard(lb);
} catch(e) {}

function renderLeaderboard(lb) {
    // 显示玩家排名徽章
    const rankContainer = document.getElementById('player-rank-container');
    if (lb.userRank != null) {
        rankContainer.innerHTML = `
            <div class="player-rank-badge">Your Rank: #${lb.userRank}</div>`;
    }

    // 渲染 Top N 列表
    const container = document.getElementById('leaderboard-container');
    const entriesEl = document.getElementById('leaderboard-entries');
    if (!lb.entries?.length) return;

    entriesEl.innerHTML = lb.entries.map((e, i) => {
        const rank = i + 1;
        const rankClass = rank <= 3 ? `rank-${rank}` : '';
        const isPlayer = lb.userRank === rank;
        return `
        <div class="leaderboard-entry${isPlayer ? ' player-entry' : ''}">
            <span class="leaderboard-rank ${rankClass}">${rank}</span>
            <img class="leaderboard-avatar" src="${e.profilePicture}" onerror="this.style.display='none'">
            <span class="leaderboard-name">${e.username}</span>
            <span class="leaderboard-score">${e.score.toLocaleString()}</span>
        </div>`;
    }).join('');

    container.classList.remove('hidden');
}
```

## 必要的 HTML 结构

```html
<div id="player-rank-container"></div>
<div class="leaderboard-container hidden" id="leaderboard-container">
    <div class="leaderboard-title">Top Racers</div>
    <div id="leaderboard-entries"></div>
</div>
```

## 常用 CSS 类

```css
.leaderboard-entry          /* 每行容器 */
.leaderboard-entry.player-entry  /* 当前玩家高亮行（绿色边框） */
.leaderboard-rank           /* 排名数字 */
.leaderboard-rank.rank-1    /* 金色 */
.leaderboard-rank.rank-2    /* 银色 */
.leaderboard-rank.rank-3    /* 铜色 */
.leaderboard-avatar         /* 64×64 圆形头像 */
.leaderboard-name           /* 玩家名字 */
.leaderboard-score          /* 分数 */
.player-rank-badge          /* 玩家排名徽章（绿色胶囊） */
```

## 注意事项

- 用空 `catch(e){}` 静默处理，排行榜失败不应崩溃游戏
- `addPlayerScoreToLeaderboard` 会同时提交 + 返回榜单，不需要再单独调 `getTopNEntries`
- `userRank` 为 `null` 时不显示排名徽章，不要显示 "Rank: null"
- `onerror="this.style.display='none'"` 防止头像加载失败时显示破图
