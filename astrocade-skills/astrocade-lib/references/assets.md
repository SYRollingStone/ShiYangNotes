# Astrocade 资产加载 API

## API

```js
// 获取资产信息（同步）
lib.getAsset(id)
// → { url: string, ...其他元数据 } | null

// 获取动画播放器（用于 spritesheet 动画）
lib.getAnimationPlayer(assetId)
// → AnimationPlayer

// 预加载动画（异步）
await lib.preloadAnimation(assetId)
```

资产 ID 来自 `game_config.json` 的 asset_map，通过 `window.gameConfig` 访问。

---

## 加载图片

```js
function loadImage(id) {
    return new Promise((resolve) => {
        const assetInfo = lib.getAsset(id);
        // 资产不存在时 resolve(null)，不要 reject
        if (!assetInfo || !assetInfo.url) { resolve(null); return; }
        const img = new Image();
        img.onload  = () => resolve({ img, info: assetInfo });
        img.onerror = () => resolve(null);
        img.src = assetInfo.url;
    });
}

// 使用
const result = await loadImage('car_sprite');
if (result) {
    ctx.drawImage(result.img, x, y, w, h);
}
```

## 加载音频（Web Audio API）

```js
function loadAudio(id) {
    return new Promise((resolve) => {
        const assetInfo = lib.getAsset(id);
        if (!assetInfo || !assetInfo.url) { resolve(null); return; }
        fetch(assetInfo.url)
            .then(r => r.arrayBuffer())
            .then(buf => audioContext.decodeAudioData(buf))
            .then(decoded => resolve(decoded))
            .catch(() => resolve(null));
    });
}

// 使用：存入 audioBuffers 字典
const audioBuffers = {};
audioBuffers['bgm'] = await loadAudio('background_music');
```

## 批量并行加载（推荐做法）

```js
const [carResult, trackResult, bgmBuffer, sfxBuffer] = await Promise.all([
    loadImage('car_sprite'),
    loadImage('track_bg'),
    loadAudio('background_music'),
    loadAudio('drift_sfx'),
]);
```

加载进度显示示例：
```js
let loaded = 0;
const total = assetIds.length;
const results = await Promise.all(
    assetIds.map(id => loadImage(id).then(r => {
        loaded++;
        updateLoadingBar(loaded / total);
        return r;
    }))
);
```

## AnimationPlayer（spritesheet 动画）

```js
// 预加载（异步）
await lib.preloadAnimation('explosion');

// 获取播放器（同步，preload 完成后才有效）
const player = lib.getAnimationPlayer('explosion');

// 在游戏循环中更新和绘制
function gameLoop(timestamp) {
    player.update(timestamp);                    // 推进帧
    player.draw(ctx, x, y, width, height);      // 绘制当前帧
}

// 其他方法
player.reset();              // 重置到第一帧
player.getCurrentFrame();   // 返回当前帧索引
```

## 注意事项

- `lib.getAsset(id)` 是**同步**的，直接返回对象；图片/音频加载本身是异步的
- 资产不存在时 `getAsset` 返回 `null`，务必做判空再访问 `.url`
- 音频需要先创建 `AudioContext`，在用户交互后 resume（浏览器自动播放策略）
- 所有加载函数遇到失败应 `resolve(null)` 而不是 `reject`，避免 `Promise.all` 中断整体加载
- `preloadAnimation` 必须 `await` 完成后再调 `getAnimationPlayer`，否则 player 可能返回空对象

## 播放音效（AudioBuffer → 声音节点）

```js
function playSound(buffer, loop = false, volume = 1.0) {
    if (!buffer || !audioContext) return null;
    const source = audioContext.createBufferSource();
    const gainNode = audioContext.createGain();
    source.buffer = buffer;
    source.loop = loop;
    gainNode.gain.value = volume;
    source.connect(gainNode);
    gainNode.connect(audioContext.destination);
    source.start();
    return { source, gainNode };
}

// 停止
const bgm = playSound(audioBuffers['bgm'], true, 0.5);
// ...
bgm.source.stop();
```
