# Figma Motion Plugin API — 表面、常见错误、校验

配合 `figma-use-motion` skill 使用。那份文档是官方 API 说明，这份是实际项目里踩过的问题。

## API 表面

| 用途 | 调用 |
|---|---|
| 读全部轨道 | `node.manualKeyframeTracks` |
| 写单条轨道 | `node.applyManualKeyframeTrack(field, track)` |
| 删单条轨道 | `node.removeManualKeyframeTrack(field)` |
| 读时间轴 | `node.timelines` → `[{ id, duration }]`，duration 单位秒 |
| 写时长 | `node.setTimelineDuration(id, seconds)` |

`field` 有两种形态：

```js
{ type: 'PROPERTY', name: 'TRANSLATION_X' }                                   // 几何 / 不透明度 / 圆角
{ type: 'INDEXED_ITEM', collection: 'effects', index: 0, field: 'RADIUS' }    // 投影 / 模糊
{ type: 'INDEXED_ITEM', collection: 'fills',   index: 0, field: 'COLOR' }     // 填充色
```

关键帧结构：

```js
{ timelinePosition: 1.24,                    // 秒
  value: { type: 'FLOAT', value: 120 },      // 或 VECTOR / COLOR
  easing: { type: 'CUSTOM_CUBIC_BEZIER',
            easingFunctionCubicBezier: { x1: .23, y1: 1, x2: .32, y2: 1 } } }
```

### PROPERTY 可动字段

变换类，与节点自身变换**复合**，中性值 0（缩放为 1）：
`TRANSLATION_X` `TRANSLATION_Y` `TRANSLATION_XY` `ROTATION` `SCALE_X` `SCALE_Y` `SCALE_XY`

绝对类，在动画窗口内**替换**节点的值：
`OPACITY` `CORNER_RADIUS` `RECTANGLE_*_CORNER_RADIUS` `STROKE_WEIGHT` `BORDER_*_WEIGHT` `STACK_SPACING` `STACK_PADDING_*` `GRID_ROW_GAP` `GRID_COLUMN_GAP` `PATH_TRIM_START` `PATH_TRIM_END` `WIDTH` `HEIGHT`

`WIDTH` / `HEIGHT` 对 GROUP 和矢量图形无效。

### INDEXED_ITEM 可动字段

`effects` 集合：`RADIUS` `SPREAD` `OFFSET_X` `OFFSET_Y` `COLOR` `EFFECT_OPACITY`，以及玻璃 / 噪声类的 `REFRACTION_*` `SPECULAR_*` `CHROMATIC_ABERRATION` `SPLAY` `START_RADIUS` `NOISE_SIZE_*` `DENSITY` `SECONDARY_COLOR`。

`COLOR` / `SECONDARY_COLOR` 要 `{ type:'COLOR', value:{r,g,b,a} }`，其余全是 `{ type:'FLOAT', value }`。写之前确认 `node.effects[index]` 存在。

### 缓动

`EASE_IN` `EASE_OUT` `EASE_IN_AND_OUT` `LINEAR` `EASE_IN_BACK` `EASE_OUT_BACK` `EASE_IN_AND_OUT_BACK` `CUSTOM_CUBIC_BEZIER` `GENTLE` `QUICK` `BOUNCY` `SLOW` `CUSTOM_SPRING` `HOLD`

弹簧用 `{ type:'CUSTOM_SPRING', easingFunctionSpring:{ bounce } }`，bounce 取 0–1。物理参数转换用 `figma.motion.physicalSpringToNormalized({mass, stiffness, damping})`。

## 12 条常见错误

**1. `field` 不能传字符串。** `applyManualKeyframeTrack('TRANSLATION_X', track)` 会报错，第一个参数必须是描述符对象。

**2. effect 报错不代表 effect 不能动。** 用 `{type:'PROPERTY', name:'RADIUS'}` 去打投影会返回一串几何属性的枚举，容易误判成「投影不可打帧」。投影走 `INDEXED_ITEM` 那条路。这个误判会让「缩放时投影跟着放大」的问题被错误地判定为无解。

**3. 缓动写在到达的关键帧上。** 第 k 个关键帧的 `easing` 描述的是从 k−1 到 k 的过程，第一个关键帧的 easing 无效。

**4. 首尾关键帧自动保持。** 第一个关键帧的值向前保持到 t=0，最后一个向后保持到结尾。不需要补 t=0 的冗余关键帧。

**5. 枚举名不能简写。** `EASE_IN_AND_OUT` 有效，`EASE_IN_OUT` 被拒。内部名 `OUT_CUBIC` `INOUT_CUBIC` 一律被拒。

**6. 时间单位是秒。** 传毫秒会得到一条几千秒的时间轴。

**7. 顶层 Frame 不能打关键帧。** 画布直接子级的 Frame 是时间轴容器，动画写在它的后代上。

**8. 位置和图层是两套顺序。** 改完子节点的 `x` 必须同步确认 `children` 顺序，`children[0]` 在最底层、末位在最上层。重排卡片 x 之后不重排图层，会出现本该在下层的元素盖到上层、露出后面内容的现象。

**9. 缩放会连圆角和投影一起缩放。** 视觉上要保持恒定就得写补偿轨道，见 [motion-math.md](motion-math.md)。

**10. 稀疏关键帧之间是线性插值。** 导数变化快的区间用等间隔采样会产生「匀速一段然后急停」的假象。实测案例：某段偏移量的速率在段末从 380/s 掉到 21/s，50ms 等间隔采样把它压成了恒定 232/s，把采样点从 9 个加到 14 个（末端加密）才恢复。

**11. 旋转节点的渐变方向要用 `relativeTransform` 反推。** 从 `x/y/width/height` 推方向在节点有 `rotation` 时是错的。实测案例：一个旋转了 −15.34° 的扫光层，按轴对齐假设改渐变，把原本平行于遮罩边缘的光带改成了 30.94° 斜角。

**12. 循环模式 Plugin API 不暴露。** 交付时提醒用户在 Figma 的 Motion 面板手动设。

## 常用脚本

### 写一条轨道

```js
node.applyManualKeyframeTrack(
  { type: 'PROPERTY', name: 'TRANSLATION_X' },
  { keyframes: [
      { timelinePosition: 5.04, value: { type: 'FLOAT', value: 2765.8 } },
      { timelinePosition: 5.44, value: { type: 'FLOAT', value: 977.86 },
        easing: { type: 'CUSTOM_CUBIC_BEZIER',
                  easingFunctionCubicBezier: { x1: .55, y1: 0, x2: .75, y2: .9672 } } },
  ] });
```

### 复制画板（每轮改动的第一步）

```js
const src = await figma.getNodeByIdAsync(SRC_ID);
const next = src.clone();
next.name = 'xxx v{n+1}';
next.x = src.x + 485; next.y = src.y;
src.parent.appendChild(next);
// clone 会完整保留 manualKeyframeTracks 和 timelines[0].duration
```

### 按索引路径迁移轨道到结构相同的另一棵树

替换素材（换语言、换主题、换样式）时用这个，不重建关键帧。**迁移前必须逐个比对节点名，对不上就抛错中止**，否则会静默写到错误的位置。

```js
const resolve = (root, path) => path.reduce((n, i) => n.children[i], root);
const specs = [];
(function walk(n, path) {
  const tr = n.manualKeyframeTracks;
  if (tr) for (const [f, t] of Object.entries(tr)) specs.push({ path, nm: n.name, field: f, track: t });
  if ('children' in n) n.children.forEach((c, i) => walk(c, path.concat(i)));
})(oldRoot, []);

for (const s of specs) {
  const t = resolve(newRoot, s.path);
  if (!t || t.name !== s.nm) throw new Error('path mismatch @' + s.path.join('.'));
}
oldRoot.remove();
parent.appendChild(newRoot);
for (const s of specs)
  resolve(newRoot, s.path).applyManualKeyframeTrack({ type: 'PROPERTY', name: s.field }, s.track);
```

### 整棵树时间重映射（改节奏）

```js
const SEG = [[0, 5.68, 0, 5.68], [5.68, 7.58, 5.58, 7.08]];   // [旧起, 旧止, 新起, 新止]
const map = t => {
  for (const [a, b, c, d] of SEG)
    if (t <= b + 1e-6) return Math.round((c + (t - a) / (b - a) * (d - c)) * 1e4) / 1e4;
  return Math.round((t - 0.50) * 1e4) / 1e4;                   // 尾段整体平移
};
(function walk(x) {
  const tr = x.manualKeyframeTracks;
  if (tr) for (const [f, t] of Object.entries(tr))
    x.applyManualKeyframeTrack({ type: 'PROPERTY', name: f },
      { ...t, keyframes: t.keyframes.map(k => ({ ...k, timelinePosition: map(k.timelinePosition) })) });
  if ('children' in x) x.children.forEach(walk);
})(root);
```

### 覆盖前扫外来改动

同一次 `use_figma` 调用写入的关键帧共用一个 node id 前缀段。前缀与本次不同的即来自其他来源（Figma 界面手改或另一次调用）。覆盖前按前缀分桶，把外来项列给用户确认。

```js
const buckets = {};
for (const [f, t] of Object.entries(node.manualKeyframeTracks))
  for (const k of t.keyframes) {
    const p = (k.id || '').split(':')[0];
    (buckets[p] = buckets[p] || []).push(`${f}@${k.timelinePosition}`);
  }
return buckets;   // 多个前缀 = 有外来改动
```

## 校验

不每轮渲染视频。默认按这四步验：

**1. 读回关键帧值。** 写完把轨道读回来，逐条核时间点、数值、缓动。这是唯一能确认「写进去的和想写的一致」的方法。

**2. 数值采样。** 在脚本里按 10–20ms 步长复算目标量，检查约束是否成立。常查的三项：

- **单调性**：逐帧算屏幕位置，累计回摆量应为 0。
- **速度连续性**：段与段交界处的速度台阶应在几个百分点以内。
- **几何约束**：行间距、覆盖范围、视觉圆角偏差。

采样代码要自己实现缓动求值（cubic bezier 按 x 解 t 再取 y），不能依赖 Figma 渲染。

**3. 结构核对。** 动画节点数 / 轨道数 / 关键帧总数 / 时间轴时长，改版前后对比。数字对不上说明有轨道被漏写或误删。

**4. 视觉抽查。** `get_screenshot` 只出静止状态，用来确认素材换对了、图层顺序对了。真要看运动才用 `export_video`，一次渲染尽量多采几帧（`ffmpeg -ss <t> -i anim.mp4 -frames:v 1 out.png`，抽帧免费）。

**校验脚本不能重复被验对象的假设。** 用同一套错误前提写的验证会自我确认。校验要从独立的量出发，例如直接读 `relativeTransform` 算屏幕坐标，而不是复用生成时的推导。
