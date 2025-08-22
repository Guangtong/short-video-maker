# 素材检索与网络获取审计报告

本报告分析了该项目自动联网搜索素材（视频、音乐等）的能力及其实现方式。

## 提供方与端点表

| 提供方 | 端点/来源 | 使用的库/模块 | 鉴权 | 主要用途 |
| :--- | :--- | :--- | :--- | :--- |
| **Pexels** | `https://api.pexels.com/videos/search` | `src/short-creator/libraries/Pexels.ts` | API Key (在 `Authorization` header 中) | B-roll 视频素材 |
| **YouTube Audio Library** | 本地目录: `static/music/` | `src/short-creator/music.ts` | 无 (本地文件) | 背景音乐 |
| **Kokoro** | 本地模型: `onnx-community/Kokoro-82M-v1.0-ONNX` | `src/short-creator/libraries/Kokoro.ts` | 无 (本地模型) | 文本转语音 (TTS) |
| **Whisper** | 本地模型 (例如 `medium.en`) | `src/short-creator/libraries/Whisper.ts`| 无 (本地模型) | 音频转字幕 |

---

## 查询构造流水线

此流程描述了如何为视频的每个场景生成和获取素材。

1.  **输入**: 接收一个场景数组 (`SceneInput[]`) 作为输入。每个场景对象包含：
    *   `text`: 用于生成画外音的文本。
    *   `searchTerms`: 一个字符串数组，用于搜索视频素材。

2.  **音频生成 (TTS)**:
    *   `Kokoro` 库接收 `text` 并生成 `.wav` 格式的音频流。
    *   音频的时长被计算出来，这个时长将决定该场景视频片段的最小长度。

3.  **视频搜索 (Pexels)**:
    *   `Pexels.ts` 库接收 `searchTerms` 数组。
    *   它会随机打乱 `searchTerms` 的顺序以实现多样性。
    *   如果 `searchTerms` 未找到结果，它会使用一组备用的 "joker" 关键词 (`nature`, `globe`, `space`, `ocean`) 进行后备搜索。
    *   向 Pexels API 发起请求，查询参数包括：
        *   `orientation`: 方向 (默认为 `portrait`)。
        *   `size`: `medium`。
        *   `per_page`: `80`。
        *   `query`: 编码后的搜索词。

4.  **字幕生成**:
    *   `Whisper` 库处理之前生成的 `.wav` 文件，为其创建字幕数据。

5.  **背景音乐选择**:
    *   根据视频总时长和可选的 `MusicMoodEnum` 标签，从 `static/music/` 目录下的本地音乐库中随机选择一个音轨。

---

## 下载/缓存/过滤逻辑摘要

**Pexels (视频)**

*   **下载**:
    *   `Pexels.ts` 首先获取视频元数据，包含一个 `.mp4` 的下载链接 (`video.url`)。
    *   `ShortCreator.ts` 使用 Node.js 的 `https` 模块通过该链接下载视频文件。
    *   下载是流式的，直接写入临时文件系统。
*   **限速与重试**:
    *   **限速**: 没有明确的速率限制。但系统采用队列机制 (`ShortCreator.queue`)，一次只处理一个视频的创建请求，这间接避免了并发下载。
    *   **重试**: `Pexels.ts` 中实现了针对 `TimeoutError` 的重试逻辑，默认重试 3 次。
*   **缓存与去重**:
    *   **缓存**: 视频文件被下载到由 `config.tempDirPath` 定义的临时目录中。这些文件在视频渲染完成后会被删除，没有持久化缓存。
    *   **去重**: 在同一次视频创作过程中，已使用的 Pexels 视频 ID 会被记录在 `excludeVideoIds` 数组中，以避免在不同场景中使用重复的视频片段。
*   **许可证过滤**:
    *   代码中没有明确的许可证过滤逻辑。它依赖于 Pexels API 返回的内容，Pexels 的许可证通常允许免费使用。
*   **安全/过滤**:
    *   代码中没有明确的 NSFW (不适宜工作场所) 内容过滤。完全依赖 Pexels 平台自身的安全策略。

**本地素材 (音乐)**

*   **下载/缓存**: 无需下载。所有音乐文件都预先存放在 `static/music/` 目录中，是项目的一部分。
*   **文件组织**:
    *   音乐文件: 存储在 `static/music/`。
    *   元数据 (文件名、情绪、起止时间): 硬编码在 `src/short-creator/music.ts` 的 `musicList` 数组中。

---

## 素材 → Remotion 轨道映射

| 素材来源 | Remotion 时间线集成方式 | 选择与裁剪 | 时长适配 |
| :--- | :--- | :--- | :--- |
| **Pexels 视频** | 作为 `<Video>` 组件的 `src` | **选择**: 为每个场景随机选择一个符合搜索词和时长要求的视频。 <br> **裁剪**: Remotion 可能会根据场景音频时长裁剪视频，但代码层面主要是选择一个足够长的视频。| 视频片段的播放时长由对应场景的 TTS 音频时长 (`audioLength`) 决定。 |
| **Kokoro TTS 音频** | 作为 `<Audio>` 组件的 `src` | **选择**: 根据场景文本生成。 <br> **裁剪**: 无，使用完整生成的音频。 | 音频本身的长度定义了场景的持续时间。 |
| **YouTube 背景音乐** | 作为顶层 `<Audio>` 组件的 `src`，在所有场景背后播放 | **选择**: 根据 `MusicMoodEnum` 标签从本地库中随机选择一个文件。 <br> **裁剪**: 音乐的 `start` 和 `end` 时间在 `music.ts` 中预定义，但 Remotion 组件似乎未使用它们，而是从头播放。 | 音乐从视频开始处播放，直到视频结束或音乐本身结束。 |
| **Whisper 字幕** | 作为自定义字幕组件的数据源 | **选择**: 从 TTS 音频生成。 <br> **裁剪**: 无，使用完整的字幕数据。 | 字幕的时间戳与 TTS 音频精确同步。 |

---

## 未决问题与待查看文件清单

- **[已解决]** Pexels API 的密钥管理 (`.env` 文件, `src/config.ts`)。
- **[已解决]** 查询关键词的生成逻辑 (`src/short-creator/ShortCreator.ts`)。
- **[已解决]** 视频下载和临时文件的处理 (`src/short-creator/ShortCreator.ts`)。
- **[已解决]** 音乐的来源和选择逻辑 (`static/music/README.md`, `src/short-creator/music.ts`)。
- **[澄清]** `Kokoro` 和 `Whisper` 是作为外部 API 调用还是本地依赖项运行？
  - **结论**: 从 `src/config.ts` 和 `src/scripts/install.ts` 的逻辑来看，它们是作为本地模型/库运行的，而不是外部网络服务。

---

## 参考链接

- **Pexels API 文档**: [https://www.pexels.com/api/documentation/](https://www.pexels.com/api/documentation/)
- **YouTube 音频库**: [https://www.youtube.com/audiolibrary/](https://www.youtube.com/audiolibrary/)
- **项目使用的 Kokoro 模型**: [https://huggingface.co/onnx-community/Kokoro-82M-v1.0-ONNX](https://huggingface.co/onnx-community/Kokoro-82M-v1.0-ONNX)
