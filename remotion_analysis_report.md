### Remotion 集成与渲染流程分析报告

首席 Remotion 工程师 Jules

#### **A 节：Compositions JSON**

项目中定义了三个核心 Composition。其中两个是动态的，一个是用于测试的静态 Composition。

```json
[
  {
    "id": "ShortVideo",
    "componentName": "PortraitVideo",
    "width": 1080,
    "height": 1920,
    "fps": 25,
    "durationInFrames": "动态计算",
    "durationCalculation": "Math.floor((props.config.durationMs / 1000) * 25)",
    "description": "用于渲染竖屏格式的短视频（例如 TikTok, Reels, Shorts）。"
  },
  {
    "id": "LandscapeVideo",
    "componentName": "LandscapeVideo",
    "width": 1920,
    "height": 1080,
    "fps": 25,
    "durationInFrames": "动态计算",
    "durationCalculation": "Math.floor((props.config.durationMs / 1000) * 25)",
    "description": "用于渲染横屏格式的视频。"
  },
  {
    "id": "TestVideo",
    "componentName": "TestVideo",
    "width": 100,
    "height": 100,
    "fps": 23,
    "durationInFrames": 14,
    "durationCalculation": "固定值",
    "description": "一个用于内部测试的简单、静态的 Composition。"
  }
]
```

- **注册入口**: Root component (`RemotionRoot`) 在 `src/components/root/index.ts` 中通过 `registerRoot(RemotionRoot)` 进行注册。
- **定义位置**: 所有 `Composition` 组件都在 `src/components/root/Root.tsx` 文件中定义。

---

#### **B 节：时间线地图**

视频的时间线是数据驱动的，其结构和时序完全由传入的 `props` 决定。`ShortVideo` 和 `LandscapeVideo` 共享相同的结构逻辑。

1.  **全局音轨 (Music)**:
    *   一个 `<Audio>` 组件贯穿整个视频，播放由 `props.music.url` 定义的背景音乐。
    *   音量由 `props.config.musicVolume` 控制。

2.  **场景序列 (Scenes)**:
    *   视频的主体由 `props.scenes` 数组驱动，每个数组成员构成一个独立的场景。
    *   这些场景很可能是通过 `map` 渲染为一系列的 `<Sequence>` 组件。
    *   **轨道分层**: 在每个场景内部，轨道分层如下：
        *   **视频层 (Video Layer)**: `<Video>` 组件播放 `scene.video` 提供的背景视频素材，可能使用 `<AbsoluteFill>` 填充整个画布。
        *   **语音层 (TTS Audio Layer)**: `<Audio>` 组件播放 `scene.audio.url` 提供的文本转语音（TTS）音频。此音轨的时长与场景同步。

3.  **字幕轨道 (Captions)**:
    *   字幕数据位于 `scene.captions` 数组中，包含了每个单词的文本和毫秒级时间戳。
    *   `src/components/utils.ts` 中的 `createCaptionPages` 函数是关键。它将单词聚合成“行”和“页”，而不是逐字显示。
    *   时间线上的字幕动画，是基于这些“页”的 `startMs` 和 `endMs` 来控制其出现和消失，实现了更自然的阅读体验。样式（如背景色 `captionBackgroundColor` 和位置 `captionPosition`）也由 props 控制。

---

#### **C 节：渲染入口与命令**

该项目**不使用**标准的 Remotion CLI (`npx remotion render`) 进行渲染。渲染过程是完全程序化的。

-   **渲染引擎**: 核心渲染引擎是 `@remotion/renderer` Node.js 库。
-   **渲染入口**: 渲染的统一入口点是 `src/short-creator/libraries/Remotion.ts` 文件中的 `Remotion` 类的 `render` 方法。
-   **CI/CD 友好**: 这种将打包 (`bundle`) 和渲染 (`renderMedia`) 封装在类中的结构，使其非常容易集成到自动化流程中，例如在 Docker 容器、CI/CD 流水线或 Serverless 函数中调用。

##### **最小可运行命令（概念性脚本）**

要在本地触发一次渲染，您需要编写一个类似于下面的 TypeScript 脚本，而不是运行命令行工具。

```typescript
// 文件: run-render.ts
import { Remotion } from "./src/short-creator/libraries/Remotion";
import { Config } from "./src/config"; // 假设的配置加载器
import { shortVideoSchema } from "./src/components/utils";
import { OrientationEnum } from "./src/types/shorts";
import { z } from "zod";
import path from "path";

// 1. 提供符合 Zod  schema 的示例数据
const sampleData: z.infer<typeof shortVideoSchema> = {
  music: {
    url: "file:///path/to/your/music.mp3",
    file: "music.mp3",
    start: 0,
    end: 15000,
  },
  scenes: [
    {
      captions: [
        { text: "This", startMs: 100, endMs: 500 },
        { text: "is", startMs: 500, endMs: 700 },
        { text: "a test.", startMs: 700, endMs: 1500 },
      ],
      video: "file:///path/to/your/background.mp4",
      audio: {
        url: "file:///path/to/your/tts.mp3",
        duration: 2.0,
      },
    },
  ],
  config: {
    durationMs: 3000,
    paddingBack: 1000,
    captionBackgroundColor: "rgba(0,0,0,0.5)",
    captionPosition: "bottom",
  },
};

// 2. 异步渲染函数
async function renderMyVideo() {
  console.log("启动渲染...");

  // 假设的配置对象
  const config: any = {
    packageDirPath: process.cwd(),
    videosDirPath: path.join(process.cwd(), "videos"),
    devMode: true,
    concurrency: 1,
    videoCacheSizeInBytes: 1024 * 1024 * 1024,
  };

  try {
    // 3. 初始化 Remotion 类 (这将首先打包项目)
    const remotion = await Remotion.init(config);

    // 4. 调用 render 方法
    await remotion.render(
      sampleData,
      "my-first-video", // 输出文件名将是 my-first-video.mp4
      OrientationEnum.Portrait
    );

    console.log("渲染成功！");
    console.log(`视频已保存至: ${path.join(config.videosDirPath, "my-first-video.mp4")}`);

  } catch (error) {
    console.error("渲染过程中发生错误:", error);
  }
}

// 5. 运行
renderMyVideo();
```

---

#### **D 节：数据/props 来源与校验**

-   **数据来源**: 所有动态数据都通过 `Remotion.render()` 方法的 `data` 参数传入。
-   **数据流**: 这个 `data` 对象被直接传递给 `@remotion/renderer` 中 `selectComposition` 和 `renderMedia` 函数的 `inputProps` 字段，最终成为 Remotion 组件树的顶层 props。
-   **数据校验**: 项目使用 `Zod` 进行严格的数据结构校验。`data` 参数必须符合在 `src/components/utils.ts` 中定义的 `shortVideoSchema`。这确保了传入的数据在启动渲染前是有效和完整的。

---

#### **E 节：未决问题**

本次分析已覆盖了项目的集成方式、数据流和渲染触发流程。目前没有重大的未决问题。

-   **下一步探索**: 如果需要深入了解具体的动画实现（例如字幕的淡入淡出效果、视频的过渡），下一步需要阅读的文件将是 React 组件本身：
    -   `src/components/videos/PortraitVideo.tsx`
    -   `src/components/videos/LandscapeVideo.tsx`

---

#### **参考链接**

-   **@remotion/renderer API**: [https://www.remotion.dev/docs/renderer](https://www.remotion.dev/docs/renderer)
    -   这是本项目使用的核心库，用于在 Node.js 环境中进行程序化渲染。
-   **calculateMetadata()**: [https://www.remotion.dev/docs/calculate-metadata](https://www.remotion.dev/docs/calculate-metadata)
    -   该函数用于根据传入的 props 动态计算 Composition 的元数据，例如 `durationInFrames`。
