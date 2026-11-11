---
name: mini-program-video
description: 用 motion-showreel 做「微信小程序 / App 产品介绍片」的实战流程与踩坑清单。当用户要给自己的小程序、App、Web 产品做产品介绍视频、功能演示片、UI 动效片，尤其是要「真实界面截图 + 抖音式翻页 + 海报/结果卡收尾」时使用。已经包含素材截取、防裁切、进度条固定、音乐授权与真峰值、成片验收的完整做法，避免重复踩坑。注意：本 skill 是 motion-showreel 的配套补充，画面与音轨的制作规范仍以 motion-showreel 为准。
---

# 小程序 / App 产品介绍片

这是 `motion-showreel` 的配套 skill。`motion-showreel` 管「怎么把片子做出来」（页面契约、确定性、审片、导出），本 skill 管「做产品介绍片这一类片子时的具体做法和坑」。

两者一起用：先按 motion-showreel 走 0–8 步流程，遇到下面这些环节时按本 skill 的做法处理。

## 什么时候用

- 用户有一个真实产品（小程序 / App / 网页），要做 15–40 秒的介绍片
- 要求界面**真实**（不能用代码画个假的冒充）
- 想要「抖音刷视频」那种一页一停、上滑切换的节奏
- 结尾是结果卡、海报、二维码之类的**产物**，而不是单纯 logo 出场
- 上次做同类片子踩过：素材被裁、进度条跟着滚走、小人被切开、音轨真峰值超限

## 0. 环境（一次）

```bash
sh <motion-showreel>/scripts/setup.sh          # Chromium + playwright
node <motion-showreel>/scripts/check-env.mjs   # 会提示 ffmpeg 缺失
```

ffmpeg 不要装系统级。下便携版到项目里，用的时候临时加 PATH：

```bash
curl -L -o ffmpeg.zip https://www.gyan.dev/ffmpeg/builds/ffmpeg-release-essentials.zip
python -c "import zipfile;zipfile.ZipFile('ffmpeg.zip').extractall('tools')"
export PATH="$PWD/tools/ffmpeg-*/bin:$PATH"
```

**坑**：Git Bash 里 `export PATH="D:/..."` 不生效，必须用 POSIX 形式 `/d/...`。`build.sh` 的子进程会读这个 PATH。

## 1. 素材：从产品代码里截真实界面

产品通常自带一个浏览器预览页（本项目是 `docs/ui-preview.html`，用真实 WXML/WXSS 渲染三个 iframe）。用它截图，比在模拟器里截更稳。

```js
const { chromium } = require("<skill>/scripts/node_modules/playwright");
const b = await chromium.launch({ channel: "msedge" });   // 没装 Chromium 时用系统 Edge
const p = await b.newPage({ viewport: { width: 1500, height: 1300 }, deviceScaleFactor: 3 });
await p.goto("file:///<project>/docs/ui-preview.html");
await p.waitForTimeout(1500);

// 关键：把答题 iframe 调高，否则内容比视口高时会被裁掉
await p.evaluate(() => { document.querySelectorAll("iframe")[1].style.height = "900px"; });

const frame = p.frames()[2];                 // iframe 的 frame 对象
for (let q = 0; q < 12; q++) {
  await p.selectOption("#question", String(q));
  await p.waitForTimeout(450);
  await frame.locator("body").screenshot({ path: `assets/ui/g${String(q+1).padStart(2,"0")}.png` });
}
```

### 三个必须处理的坑

1. **截图带 1px 边框**。预览页的 `iframe{border:1px solid #202020}` 会被截进图里，放到视频里就是一条贴边的黑线。绘制时裁掉：`drawImage(img, 4, 4, w-8, h-8, ...)`，或截图后统一裁 4px（@3x）。
2. **内容比视口高就被裁**。产品页在 375×760 下，有些题目的内容到 781px，截图底部直接把按钮切了。**先把 iframe 调高到 900 再截**，不要靠后期缩放补救——缺失的像素补不回来。
3. **截完记得裁掉尾部空白**。调高 iframe 后短页面会带一大片空白，逐页把「最后一行非背景色像素 + 24px」以下裁掉，并**记录每页高度**（后面按高度适配要用）。

每页的原始高度（CSS px）写进代码常量，比如 `QH = [2030,2096,...]`（@3x 的像素值）。

## 2. 结构模板（30–40 秒）

| 段落 | 时长 | 做法 |
|---|---|---|
| 开场 | 2.5–3.5s | 终端打字（居中，8–11 行，逐行 0.23s / 单行 0.26s）或字标弹入 |
| 首页 | 2–2.5s | 卡片从屏幕外升起，光标走位并点「开始」 |
| 内容页 | 每页 1.1s | 抖音式：停 1.1s → 上滑 0.36s 吸附 |
| 结果 | 2.5–3s | 结果页推近 |
| 人格/特性 | 2.5–3s | 网格弹入，重点项高亮 |
| 结尾 | 3.5–4s | 海报 / CTA 卡 / 结果卡，**不要只出 logo** |

结尾的最后一帧要能当缩略图用，并且只留一个行动提示。

## 3. 抖音式翻页：固定表头 + 内容上滑

用户最容易觉得"高级"的一处，也是最容易做坏的一处。

### 正确做法

- 截图顶部固定高度（本项目 570 设备像素）作为**固定表头**，包含：返回按钮、吉祥物、`情境 N/12`、彩虹进度条、奔跑小人
- 表头**直接取每一页自己的原图顶部**，换页时 0.36s 交叉淡入
- 只有表头以下的内容做整屏上滑

```js
// 表头：每页原图顶部交叉淡入，绝不重绘
function drawHeader(i, k) {
  const draw = (idx, a) => {
    if (a <= 0.002) return;
    ctx.save(); ctx.globalAlpha = a;
    ctx.drawImage(QS[idx], 0, 0, QW, QHDR, r.x, r.y, r.w, hdrH);
    ctx.restore();
  };
  if (k <= 0) draw(i, 1);
  else { draw(i, 1 - k); draw(Math.min(i + 1, 11), k); }
}

// 内容：按每页真实高度适配，底部留白
const srcH = QH[i] - QHDR;
const base = Math.min(w / QW, dh / srcH);   // 宽/高取小，保证不出框
ctx.drawImage(QS[i], 0, QHDR, QW, srcH, x + (w - QW*base)/2, dy0 + off, QW*base, srcH*base);
```

### 为什么不能重绘进度条

试过把小人从截图里抠成精灵图再自己画进度条，结果是：

- 精灵图里自带一截旧进度条，小人移动时被"横切一刀"
- 抠掉那条，小人身体就断成两截（头和腿分开）
- 竖向补洞，补成一块模糊的矩形

**结论：凡是"移动的 UI 元素"，优先从每页原图取，靠淡入淡出过渡，而不是抠图重画。** 抠图只适合完全不动的元素。

### 上滑参数

```js
const PAGE = 1.4583;   // 一页总时长
const SW   = 0.36;     // 上滑时长，剩余 1.0983s 是停留
const k = ease.inOutCubic(seg(local, PAGE - SW, PAGE));
page(idx, -dh * k);         // 旧页向上
page(idx + 1, dh * (1 - k)); // 新页从下方进
```

停留时整页静止会被验收工具标成 `frozen`（死帧）。给内容加 1–1.2% 的落定回缩，或让精灵有轻微起伏。

## 4. 版式：别让内容贴边

- 内容裁剪区高度 = 卡片高度 − 表头高度 − **底部留白（约 38px / 1080 宽）**
- 页面按「按宽适配 / 按高适配取小」缩放；长页面会自动变小 5% 左右，这是保证不切内容的代价
- 文字一律用 `render.mjs --text-check` 验：顶部/底部各 12% 是安全区，字幕、计数、落版文案不能进去
- 卡片内部底色要用截图里的页面底色（本项目 `#FFF4DA`），不能用白色——否则缩放后四周会露白边

## 5. 音乐：三种来源，各有各的坑

### 自合成（默认）

motion-showreel 的 `score.mjs` 能出声音，但容易"太电子"。用户嫌电子味时不要硬调合成器参数，换现成曲子更快。

### 现成曲子（推荐给产品片）

可商用免署名：Pixabay（下载要他们站内 token，脚本抓不到，得手工下）。
可商用需署名：Incompetech（Kevin MacLeod），**直链可 curl**，但 CC BY 4.0 必须署名：

```
Music: "<Track>" — Kevin MacLeod (incompetech.com)
Licensed under Creative Commons: By Attribution 4.0
```

`audio.mjs beats <track>` 量 BPM 与第一个强拍，把切点倒推，让结尾那一段落在强拍上：

```js
offset = firstDownbeat + k * bar - finaleTime   // 取第一个 ≥ 0 的 k
```

### 音频三个必踩的坑

1. **24-bit WAV 浏览器放不出**。合成器默认输出 24-bit PCM，`<audio>` 时长会是 NaN、点了没声。交付前转 MP3：
   `ffmpeg -i music.wav -codec:a libmp3lame -b:a 192k music.mp3`
2. **AAC 编码会让真峰值超标**。母带已经压到 −1.5 dBTP，`-c:a aac` 之后可能变成 +0.1 dBTP，`inspect.mjs qa` 直接 FAIL。build.sh 不管这一步，导出后自己重封一次：

```bash
ffmpeg -y -i out/final.mp4 -i audio/final.wav -map 0:v -map 1:a -c:v copy \
  -af "alimiter=limit=0.82:level=disabled" -c:a aac -b:a 192k -shortest out/final-c.mp4
```

3. **换音乐要重算时长**。改片子时长就重新裁曲、重新对齐，不要沿用上一版音频。

## 6. 验收：必须全 PASS 才算交付

```bash
node <skill>/scripts/inspect.mjs qa out/final.mp4 --out qa/x \
  --expect-size 1080x1920 --expect-fps 60 --expect-duration 34 --audio required
```

九项：解码、尺寸、帧率、时长、帧数、音轨、黑帧、响度、真峰值。有 `NOT CHECKED` 也不算过。

另外三项自查：

- `render.mjs --determinism`：帧只由 t 决定
- `render.mjs --text-check`：没有文字越界/重叠/太小
- `build.sh` 输出里的 `frozen ≥ 0.8s`：每段静止要么是刻意的（写进 `HOLDS`），要么加动效

## 7. 交付清单

- `out/final.mp4`、`out/poster.png`
- `brief.md`（结构化提示词 + 确认记录）
- `qa/review_log.md`（技术检查、修过的问题、未解决项）
- 音乐署名（CC BY 必带）、字体与素材授权
- 重渲命令（含 ffmpeg PATH 与最后的重封步骤）

## 8. 与用户沟通的节奏

产品介绍片的返工大多来自"信息量"和"节奏"，不是技术问题：

- 第一版给 20–30 秒，用户常常觉得"太短太空"。先把**信息点数量**列出来（几屏、几个功能、几个人设），再谈时长
- 开场/结尾这种审美选择，**做成网页让用户点选**，每个选项都是真实在动的 9:16 小样，比文字描述有效十倍
- 用户说"不高级"时，先看是不是"只有一个元素在动"，而不是加更多特效
- 每一轮改完都要重新导出并复述改了什么、有没有连带影响

## 参考

- `references/snippets.md`：可直接复制的代码（抖音翻页、按页适配、表头淡入、截图脚本、重封命令）
- `references/pitfalls.md`：按现象查原因的速查表
- 制作规范、页面契约、审片标准仍看 `motion-showreel` 的 SKILL.md 与 `references/`
