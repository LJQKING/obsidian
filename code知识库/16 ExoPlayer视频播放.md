# 一、ExoPlayer 是什么（核心定位）

ExoPlayer 是 Google 提供的 **Android 媒体播放引擎**，用于替代传统 `MediaPlayer`。

它的核心特点：

- 解耦式播放器（组件化）
    
- 支持 DASH / HLS / SmoothStreaming / MP4
    
- 支持缓存、DRM、倍速、音轨切换
    
- 可扩展 TrackSelector / LoadControl / Renderer
    

---

# 二、ExoPlayer 架构原理（重点）

ExoPlayer 本质是一个**流水线架构（Pipeline）**：

```
MediaSource -> LoadControl -> Renderer -> Decoder -> Surface
```

拆解如下：

## 1. MediaSource（数据源层）

负责“从哪里读数据”：

- ProgressiveMediaSource（mp4）
    
- HlsMediaSource（m3u8）
    
- DashMediaSource
    

👉 作用：**负责分片、加载策略**

---

## 2. LoadControl（缓冲控制）

控制：

- 何时开始播放
    
- 缓冲多少开始播放
    
- 是否继续加载
    

核心逻辑：

```text
bufferedDuration < minBufferMs → 继续加载
bufferedDuration > maxBufferMs → 暂停加载
```

---

## 3. Renderer（渲染层）

ExoPlayer 内部有多个 Renderer：

- VideoRenderer（视频）
    
- AudioRenderer（音频）
    
- TextRenderer（字幕）
    

---

## 4. Decoder（解码）

通常走：

- MediaCodec（硬解）
    
- ffmpeg（扩展）
    

---

## 5. Timeline / Period（关键点）

ExoPlayer 支持流媒体时间轴：

- Live
    
- VOD
    
- 多 Period 切片
    

👉 这是“拖动 seek 不会卡死”的核心基础之一

---

# 三、ExoPlayer 播放流程

```
prepare()
   ↓
Buffer MediaSource
   ↓
LoadControl 判断 buffer
   ↓
Decoder 解码
   ↓
Renderer 输出 Surface
```

---

# 四、标准使用方式（Media3 ExoPlayer）

## 1. 依赖

```gradle
implementation "androidx.media3:media3-exoplayer:1.3.1"
implementation "androidx.media3:media3-ui:1.3.1"
```

---

## 2. 初始化 Player

```kotlin
val player = ExoPlayer.Builder(context).build()

playerView.player = player
```

---

## 3. 设置数据源

```kotlin
val mediaItem = MediaItem.fromUri(videoUrl)

player.setMediaItem(mediaItem)
player.prepare()
player.play()
```

---

## 4. 生命周期管理

```kotlin
override fun onStop() {
    player.pause()
}

override fun onDestroy() {
    player.release()
}
```

---

# 五、拖动进度条不卡顿的核心原理（重点）

你问的核心问题就在这里 👇

---

## ❗ 为什么 MediaPlayer 拖动会卡？

因为：

- seek 是“阻塞式”
    
- 需要重新 prepare
    
- 解码器可能重建
    

---

## ✅ ExoPlayer 为什么不卡？

ExoPlayer 做了三件关键优化：

---

## 1. 非阻塞 Seek（Asynchronous Seek）

```text
seekTo()
→ 只更新 Timeline position
→ 不立即重建 decoder
→ 后台继续 buffer
```

👉 UI 不会卡

---

## 2. 精确分段定位（Segment Seek）

对于 HLS/DASH：

- 按 segment（分片）seek
    
- 不是逐帧 seek
    

👉 速度非常快

---

## 3. Buffer 不中断策略

Seek 时：

- decoder 不一定 reset
    
- cache 保留
    
- 继续加载 nearest keyframe
    

---

# 六、进度条拖动不卡顿（工程正确写法）

---

## ❌ 错误写法（会卡）

```kotlin
seekBar.setOnSeekBarChangeListener(object : SeekBar.OnSeekBarChangeListener {
    override fun onProgressChanged(seekBar: SeekBar?, progress: Int, fromUser: Boolean) {
        player.seekTo(progress.toLong()) // ❌ 高频调用
    }
})
```

👉 问题：

- 每一帧都 seek
    
- decoder疯狂重置
    

---

## ✅ 正确写法（关键）

### 方案1：松手才 seek（推荐）

```kotlin
seekBar.setOnSeekBarChangeListener(object : SeekBar.OnSeekBarChangeListener {

    override fun onStartTrackingTouch(seekBar: SeekBar) {
        player.pause()
    }

    override fun onProgressChanged(seekBar: SeekBar, progress: Int, fromUser: Boolean) {}

    override fun onStopTrackingTouch(seekBar: SeekBar) {
        player.seekTo(seekBar.progress.toLong())
        player.play()
    }
})
```

---

### 方案2：拖动中“节流 seek”（进阶）

```kotlin
private var lastSeekTime = 0L

seekBar.setOnSeekBarChangeListener(object : SeekBar.OnSeekBarChangeListener {

    override fun onProgressChanged(seekBar: SeekBar, progress: Int, fromUser: Boolean) {
        if (!fromUser) return

        val now = System.currentTimeMillis()
        if (now - lastSeekTime > 80) { // 80ms节流
            player.seekTo(progress.toLong())
            lastSeekTime = now
        }
    }

    override fun onStartTrackingTouch(seekBar: SeekBar) {}

    override fun onStopTrackingTouch(seekBar: SeekBar) {
        player.seekTo(seekBar.progress.toLong())
    }
})
```

---

## ✅ 方案3：UI拖动“假进度条”（最丝滑）

核心思想：

👉 拖动时只更新 UI，不 seek

```kotlin
override fun onProgressChanged(seekBar: SeekBar?, progress: Int, fromUser: Boolean) {
    if (fromUser) {
        timeText.text = format(progress) // UI模拟
    }
}
```

松手才：

```kotlin
player.seekTo(progress.toLong())
```

---

# 七、进阶优化（让 seek 更丝滑）

---

## 1. 调整 Buffer Strategy

```kotlin
val loadControl = DefaultLoadControl.Builder()
    .setBufferDurationsMs(
        5_000,  // minBuffer
        50_000, // maxBuffer
        1_500,
        2_000
    )
    .build()
```

---

## 2. 预加载关键帧

ExoPlayer 自动做，但 HLS 依赖：

- GOP（关键帧间隔）
    
- segment 切分
    

👉 视频编码影响 seek 流畅度

---

## 3. 使用 Cache（强烈推荐）

```kotlin
val cache = SimpleCache(...)
val dataSourceFactory = CacheDataSource.Factory()
```

👉 二次 seek 几乎无感

---

## 4. 使用 UI 层“缓冲提示”

```kotlin
player.addListener(object : Player.Listener {
    override fun onPlaybackStateChanged(state: Int) {
        // BUFFERING / READY
    }
})
```

---

# 八、总结（核心记忆点）

ExoPlayer 拖动不卡顿的本质：

> ❗“Seek 不重建播放链，只移动时间轴 + 后台重新对齐 keyframe”

---

## 面试一句话总结：

ExoPlayer 通过 **非阻塞 seek + 分片媒体源 + timeline 驱动 + buffer 续播机制**，实现了远优于 MediaPlayer 的拖动体验。

---

如果你需要，我可以再帮你补一版：

👉 **ExoPlayer 源码级 seekTo() 流程分析（从 PlayerImpl → MediaPeriod → Renderer 全链路）**  
👉 或者 **抖音级播放器架构（缓存 + preload + ABR）**