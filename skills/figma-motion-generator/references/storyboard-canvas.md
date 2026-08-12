# 动画时间板 — 几何契约与生成

Phase A 在用户指定的 Figma 页面上生成这块板。结构对应设计师手画分镜板的常见形态，横轴是严格的时间轴。

## 结构

三行加一块参数面板：

| 行 | 内容 | 谁填 |
|---|---|---|
| 关键帧 | N 个舞台尺寸的空格子，代表 N 个静止状态 | 用户画 |
| 动画帧 | N−1 个空格子，代表相邻两个关键帧之间的过渡 | 用户画或只写说明 |
| 时长 | N−1 根横条，每根写该段的毫秒数 | 用户改数字 |

每个格子下面有一行说明栏。关键帧的说明写「画面上是什么」，动画帧的说明写「怎么动」。

## 坐标

以下都相对 `动画时间板` Frame 原点。`SW` / `SH` 为舞台宽高，默认 365×210。

```
SCALE   = max(600, (SW + 55) / min(每段秒数))      px / 秒，保证最短的一段也放得下一个格子
X0      = 240 + SW/2                              t=0 对应的横坐标
t_i     = 前 i 段时长累加（秒）

关键帧 i:   frame  (X0 + t_i*SCALE - SW/2, 74)              尺寸 SW×SH   名 KF-{i+1}
            说明   (同上 x, 74 + SH + 20)                    宽 SW+7      名 KF-{i+1}-说明
分隔线:     (240, 74 + SH + 159) → 横贯整块板
动画帧 j:   bar_x  = X0 + t_j*SCALE
            bar_w  = (t_{j+1} - t_j) * SCALE
            frame  (bar_x + (bar_w - SW)/2, DIV_Y + 107)     尺寸 SW×SH   名 AF-{j+1}
            说明   (同上 x, AF_Y + SH + 20)                   宽 SW+7      名 AF-{j+1}-说明
时长条 j:   (bar_x, AF说明_Y + 137)                           尺寸 bar_w×93  名 DUR-{j+1}
            条内居中 TEXT「{毫秒} ms」                                      名 DUR-{j+1}-值

行标签:     「关键帧」「动画帧」「时长」在 x=40，各自纵向居中于所在行
板宽:       X0 + T_total*SCALE + SW/2 + 240
板高:       时长条 y + 93 + 119
底色:       #FCFCFC
```

参数面板是一个单独的 Frame，放在板的正上方 260px 处，宽 700 高 220，名「参数」。

## 时长的真源是条里的数字

改毫秒数比拖条宽度准。**条内的 `{n} ms` 文本是权威值，条宽只是可视化。** Phase B 解析这段文本，不量条宽。改完数字重跑布局函数即可重新排版。

## 字体

先取页面上已有 TEXT 的字体，取不到再退回 Inter Regular。中文说明栏用 Inter 会缺字。

```js
async function pickFont(page) {
  const t = page.findOne(n => n.type === 'TEXT');
  if (t) {
    const segs = t.getStyledTextSegments(['fontName']);
    if (segs.length) { await figma.loadFontAsync(segs[0].fontName); return segs[0].fontName; }
  }
  const f = { family: 'Inter', style: 'Regular' };
  await figma.loadFontAsync(f);
  return f;
}
```

创建任何 TEXT 之前必须 `await figma.loadFontAsync(...)`，漏了会抛 `Cannot write to node with unloaded font`。

## 生成脚本

```js
// segments: [{ name: '浮起', ms: 480 }, ...]  N-1 段 → N 个关键帧
const SW = 365, SH = 210, GUTTER = 240;
const secs = segments.map(s => s.ms / 1000);
const SCALE = Math.max(600, (SW + 55) / Math.min(...secs));
const X0 = GUTTER + SW / 2;
const T = secs.reduce((a, b) => a + b, 0);

const KF_Y = 74, CAP1_Y = KF_Y + SH + 20;
const DIV_Y = KF_Y + SH + 159, AF_Y = DIV_Y + 107;
const CAP2_Y = AF_Y + SH + 20, BAR_Y = CAP2_Y + 137, BAR_H = 93;
const W = X0 + T * SCALE + SW / 2 + GUTTER, H = BAR_Y + BAR_H + 119;

const font = await pickFont(figma.currentPage);
const board = figma.createFrame();
board.name = '动画时间板';
board.resize(W, H);
board.fills = [{ type: 'SOLID', color: { r: .988, g: .988, b: .988 } }];
board.clipsContent = false;

const mkText = (s, x, y, w, size) => {
  const t = figma.createText();
  t.fontName = font; t.characters = s; t.fontSize = size;
  board.appendChild(t); t.x = x; t.y = y;
  if (w) { t.textAutoResize = 'HEIGHT'; t.resize(w, t.height); }
  return t;
};
const mkSlot = (name, x, y) => {
  const f = figma.createFrame();
  f.name = name; f.resize(SW, SH); f.cornerRadius = 6;
  f.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];
  f.strokes = [{ type: 'SOLID', color: { r: .85, g: .85, b: .87 } }];
  f.strokeWeight = 1; f.dashPattern = [6, 4];
  board.appendChild(f); f.x = x; f.y = y;
  return f;
};

let t = 0;
for (let i = 0; i <= segments.length; i++) {
  const x = X0 + t * SCALE - SW / 2;
  mkSlot(`KF-${i + 1}`, x, KF_Y);
  const label = i === 0 ? '首帧：' : (i === segments.length ? '末帧（回到首帧）：' : '');
  mkText(label, x, CAP1_Y, SW + 7, 16).name = `KF-${i + 1}-说明`;
  if (i < segments.length) t += secs[i];
}

t = 0;
for (let j = 0; j < segments.length; j++) {
  const barX = X0 + t * SCALE, barW = secs[j] * SCALE;
  mkSlot(`AF-${j + 1}`, barX + (barW - SW) / 2, AF_Y);
  mkText(`动画：${segments[j].name}`, barX + (barW - SW) / 2, CAP2_Y, SW + 7, 16)
    .name = `AF-${j + 1}-说明`;
  const bar = figma.createFrame();
  bar.name = `DUR-${j + 1}`; bar.resize(barW, BAR_H); bar.cornerRadius = 8;
  bar.fills = [{ type: 'SOLID', color: { r: .92, g: .94, b: 1 } }];
  board.appendChild(bar); bar.x = barX; bar.y = BAR_Y;
  const v = mkText(`${segments[j].ms} ms`, 0, 0, 0, 18);
  bar.appendChild(v); v.x = (barW - v.width) / 2; v.y = (BAR_H - v.height) / 2;
  v.name = `DUR-${j + 1}-值`;
  t += secs[j];
}

for (const [s, y, h] of [['关键帧', KF_Y, SH], ['动画帧', AF_Y, SH], ['时长', BAR_Y, BAR_H]])
  mkText(s, 40, y + (h - 30) / 2, 0, 22);

const line = figma.createLine();
board.appendChild(line); line.resize(W - GUTTER - 40, 0);
line.x = GUTTER; line.y = DIV_Y;
line.strokes = [{ type: 'SOLID', color: { r: .87, g: .87, b: .89 } }];

return { createdNodeIds: [board.id], width: W, height: H, scale: SCALE, total: T };
```

参数面板另建一个 Frame，内容套 [templates/motion-spec.md](../templates/motion-spec.md) 的参数表。

## Phase B 怎么读回

```js
const board = /* 找到名为 动画时间板 的 Frame */;
const q = n => board.children.filter(c => c.name.startsWith(n));
const kf  = q('KF-').filter(c => c.type === 'FRAME')
             .sort((a, b) => +a.name.split('-')[1] - +b.name.split('-')[1]);
const cap = Object.fromEntries(board.children.filter(c => c.name.endsWith('-说明'))
             .map(c => [c.name.replace('-说明', ''), c.characters]));
const dur = q('DUR-').filter(c => c.type === 'FRAME')
             .map(c => ({ name: c.name, ms: parseFloat(c.children[0].characters) }));
```

关键帧格子里的图层结构就是实现画板的起点。`KF-1` 的内容即 t=0 的静止状态，直接 `clone()` 进实现画板作为基底。
