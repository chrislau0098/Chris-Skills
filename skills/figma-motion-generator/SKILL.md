---
name: figma-motion-generator
description: 在 Figma 画布上用原生 Motion 功能制作时间轴动画。分两阶段：先在指定 Figma 页面生成「动画时间板」故事板框架（关键帧 / 动画帧 / 时长三行 + 参数面板），用户把分镜填好后读回，产出 motion-spec 并写入 manualKeyframeTracks 生成动画，关键参数全部可配置。Use when 用户要在 Figma 里做动画、把分镜或故事板做成动效、生成动画故事板框架、写 Motion 关键帧、调缓动或动画节奏，或提到 Figma Motion、manualKeyframeTracks、动画时间板、分镜、keyframe、storyboard。不适用于 Web 端动画实现、After Effects 或 Lottie。
---

# Figma Motion 动画生成

在 Figma 文件里用原生 Motion 做时间轴动画。适用于产品 onboarding、功能演示、循环动效、素材过场。Web 端动画实现用 `motion` skill，静态设计用 `figma-use`。

## 第 0 步 · 检查 Figma MCP

动手前先过这四项，任何一项不通就停下来告诉用户怎么修，不要硬跑。

1. **工具在不在。** 需要 `use_figma` / `get_metadata` / `get_screenshot`。以 deferred 形式出现时用一次 ToolSearch 批量加载：`select:use_figma,get_metadata,get_screenshot`。完全找不到说明 Figma MCP 没装或没连上——让用户在**交互式** Claude Code 里跑 `/mcp` 查看连接状态，或用 `claude mcp` 添加 server。非交互会话跑不了 OAuth 流程，不要试。
2. **授权通不通。** 调 `whoami`。返回账号 handle、邮箱和所属 plan 即为已授权；报鉴权错误说明未授权或 token 失效，同样让用户去 `/mcp` 重新授权。**返回值含账号邮箱，属于个人信息，不要写进产出文档，也不要复述给第三方。**
3. **席位够不够。** `whoami` 返回的 `plans[].seat` 为 `View` 时只能读不能写。目标文件所属 plan 必须是 `Full` 或 Editor 席位，否则 `use_figma` 的写操作会失败。席位不对要让用户换账号或申请编辑权限。
4. **Motion 开关开没开。** Motion API 挂在 `metronome` 用户特性开关下。读一次目标节点的 `manualKeyframeTracks` 探测，报 `"manualKeyframeTracks" is not a supported API` 说明当前账号没开，告知用户并停止，不要重试。

## 需要一起加载的 skill

- **`figma-use` + `figma-use-motion`** 必须同时加载，调用 `use_figma` 时 `skillNames` 传 `"figma-use,figma-use-motion"`。两者随 Figma MCP 提供。
- **Phase B 开始前加载 `emil-design-eng` 和 `motion`**。缓动、时长、stagger 的判断依据来自这两个 skill；[references/motion-craft.md](references/motion-craft.md) 是它们的 Figma 适配版，冲突时以原 skill 为准。

## Quick start

```js
// field 是描述符对象不是字符串；时间单位是秒；缓动写在到达的那个关键帧上。
const node = await figma.getNodeByIdAsync('123:456');
const cb = e => ({ type:'CUSTOM_CUBIC_BEZIER', easingFunctionCubicBezier:e });
node.applyManualKeyframeTrack({ type:'PROPERTY', name:'TRANSLATION_X' }, { keyframes:[
  { timelinePosition:0,   value:{ type:'FLOAT', value:0 } },
  { timelinePosition:0.4, value:{ type:'FLOAT', value:120 }, easing:cb({x1:.23,y1:1,x2:.32,y2:1}) },
]});
return { mutatedNodeIds:[node.id] };
```

## 工作流

### Phase A — 生成故事板框架

输入：Figma 文件 URL（含目标页面）、舞台尺寸、段落列表（每段一句话 + 预估毫秒数）。没给段落列表就用 6 段占位。

产出：目标页面上一个「动画时间板」Frame，三行结构加参数面板。用户在关键帧格子里画静止状态、在说明栏写动作、在时长条里改毫秒数。几何契约与生成脚本见 [references/storyboard-canvas.md](references/storyboard-canvas.md)。

**Phase A 结束就停。** 等用户说填好了再进 Phase B，不要自行猜测分镜内容。

### Phase B — 生成 Motion

1. **读回故事板** — 解析关键帧格子的图层结构、说明栏文字、时长条里的毫秒数。用户另给了参考视频就先用 `ffmpeg` 抽帧量出周期、过渡时长、运动方向、以及**元素是同时动还是接力**，方法见 [references/motion-craft.md](references/motion-craft.md)。故事板决定姿态，视频决定节奏。
2. **写 motion-spec** — 套用 [templates/motion-spec.md](templates/motion-spec.md) 落一份本地文档，作为数值的唯一真源。每段写清 t_in / t_out、参与节点、属性轨道、缓动、约束。**先让用户确认 spec，再写关键帧。**
3. **建实现画板** — 新建一个画板承载动画，命名带版本号。舞台本身是顶层 Frame，动画写在它的子节点上。
4. **写轨道** — 按 spec 逐段写 `manualKeyframeTracks`，用 `setTimelineDuration` 设总时长。API 细节与 13 条常见错误见 [references/figma-motion-api.md](references/figma-motion-api.md)。
5. **校验** — 读回关键帧值加数值采样，不渲染视频。方法见同一份文档的「校验」节。
6. **交付** — 报告画板 ID、时间轴、校验结果、未解决项。循环模式需要用户在 Figma 的 Motion 面板手动设置，Plugin API 不暴露。

### 迭代

**每一轮改动都新建画板**，从当前版本 `clone()` 出下一版再改，旧版原地保留。用户会在 Figma 里手改参数试效果，覆盖任何轨道前先按关键帧 id 前缀分桶，把非脚本写入的项列出来确认。

## 可配置参数

写在 motion-spec 的参数表里，改 spec 再重跑，不要散在脚本里。

| 参数 | 默认 | 作用 |
|---|---|---|
| `stage` | 365×210 | 舞台尺寸，决定故事板格子大小 |
| `duration` | 由段落累加 | 时间轴总长（秒） |
| `tail` | 0.5–0.9s | 循环前的空白帧时长 |
| `ease.enter` | `CB(0.23, 1, 0.32, 1)` | 入场、出现 |
| `ease.exit` | `CB(0.5, 0, 0.75, 1)` | 退场、消失 |
| `ease.move` | `CB(0.55, 0, 0.75, 1)` | 屏内位移与形变 |
| `ease.linear` | `CB(0, 0, 1, 1)` | 匀速段与采样轨道 |
| `stagger` | 40ms | 同组元素依次入场的间隔 |
| `segments[]` | — | 每段的 t_in / t_out / 缓动覆盖 |

## 铁律

1. 每轮改动新建画板，不在旧版上改。
2. 数值真源是 motion-spec 文档，画布是视觉真源。
3. 不每轮渲染视频。读回关键帧值加采样验证，用户自己在 Figma 看。
4. 改节点 `x` 之后必须确认 `children` 顺序，位置和图层是两套独立的顺序。
5. 缓动写在**到达**的那个关键帧上，第一个关键帧的缓动无效。
6. 时间单位是秒。
7. 缩放会同时缩放圆角、描边和投影，需要写 `1/s(t)` 补偿轨道。
8. 覆盖已有轨道前先扫外来改动。
9. 稀疏关键帧之间是线性插值，导数变化快的区间要加密采样点。
10. 一次只解决一个问题，不在一版里叠多个大改动。
11. 多元素换位默认让它们**同时动**。别为了消除背景暴露改成接力，那会读成排队。
12. 故事板的中间姿态是路径航点，不是关键帧。路径与速度分两层做，见 motion-math 第 6 节。
13. 没验证的写「未验证」，原因不明的写「原因未查明」。

## 参考文档

| 文档 | 何时读 |
|---|---|
| [references/storyboard-canvas.md](references/storyboard-canvas.md) | Phase A：故事板几何契约与生成脚本 |
| [references/figma-motion-api.md](references/figma-motion-api.md) | Phase B：Plugin API 表面、13 条常见错误、校验方法 |
| [references/motion-craft.md](references/motion-craft.md) | 选缓动、定时长、排 stagger、判断该不该动 |
| [references/motion-math.md](references/motion-math.md) | 补偿轨道、均匀缩放平铺、速度衔接、可行性判定、单调性检查 |
| [templates/motion-spec.md](templates/motion-spec.md) | 写 motion-spec 时套用 |
