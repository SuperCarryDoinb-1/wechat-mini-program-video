# 可复用代码

坐标假设：成片 1080×1920，卡片宽 `CARD_W`（本项目 652），截图 @3x 宽 1125。

## 1. 截图（可加高 iframe）

```js
const { chromium } = require("<motion-showreel>/scripts/node_modules/playwright");

(async () => {
  const b = await chromium.launch({ channel: "msedge" });     // 或省略用自带 Chromium
  const p = await b.newPage({ viewport: { width: 1500, height: 1500 }, deviceScaleFactor: 3 });
  await p.goto("file:///<project>/docs/ui-preview.html");
  await p.waitForTimeout(1500);

  // 关键：加高 iframe，避免内容比视口高时被裁
  await p.evaluate(() => { document.querySelectorAll("iframe")[1].style.height = "900px"; });
  await p.waitForTimeout(600);

  const frame = p.frames()[2];
  const heights = [];
  for (let q = 0; q < 12; q++) {
    await p.selectOption("#question", String(q));
    await p.waitForTimeout(450);
    heights.push(await frame.evaluate(() => document.documentElement.scrollHeight));
    await frame.locator("body").screenshot({
      path: `assets/ui/g${String(q + 1).padStart(2, "0")}.png`,
    });
  }
  console.log(JSON.stringify(heights));
  await b.close();
})();
```

## 2. 裁掉截图尾部空白（并把高度写进常量）

```python
from PIL import Image
hs = []
for i in range(1, 13):
    p = f"aiti-intro/assets/ui/g{i:02d}.png"
    im = Image.open(p).convert("RGB"); w, h = im.size; px = im.load()
    last = 0
    for y in range(h - 1, -1, -1):
        if any(not (px[x, y][0] > 246 and px[x, y][1] > 238 and px[x, y][2] > 205)
               for x in range(0, w, 4)):
            last = y; break
    bottom = min(h, last + 24)
    Image.open(p).crop((0, 0, w, bottom)).save(p)
    hs.append(bottom)
print(hs)   # 直接抄进 QH
```

## 3. 抖音式翻页 + 按页适配

```js
const QW = 1125, QHDR = 570;                 // 截图宽度、表头高度（@3x 像素）
const QH = [2030, 2096, 2276, 2321, 2174, 2096, 2231, 2267, 2267, 2267, 2312, 2186];
const PAGE = 1.4583, SW = 0.36;

if (t < quizB) {
  const lt = t - quizA;
  const idx = Math.min(QS.length - 1, Math.floor(lt / PAGE));
  const local = lt - idx * PAGE;
  const k = idx >= QS.length - 1 ? 0 : ease.inOutCubic(seg(local, PAGE - SW, PAGE));

  const kk = w / 1123;
  const hdrH = HDR_SRC * kk;
  const cTop = y + hdrH;
  const dh = h - hdrH - 66 * kk;            // 底部留白 ≈ 38px

  const page = (i, off) => {
    const srcH = QH[i] - QHDR;
    const s = Math.min(w / QW, dh / srcH);  // 宽/高取小 → 不出框
    ctx.drawImage(QS[i], 0, QHDR, QW, srcH,
      x + (w - QW * s) / 2, cTop + 2 * kk + off, QW * s, srcH * s);
  };

  ctx.save();
  ctx.beginPath(); ctx.rect(x, cTop, w, dh); ctx.clip();
  if (k <= 0) page(idx, 0);
  else { page(idx, -dh * k); page(idx + 1, dh * (1 - k)); }
  ctx.restore();

  drawQuizHeader(idx, k);                    // 表头最后画，压在内容上
  return;
}
```

## 4. 固定表头（每页原图交叉淡入）

```js
function drawQuizHeader(idx, k) {
  const hdrH = QHDR * (r.w / QW);
  const draw = (i, a) => {
    if (a <= 0.002) return;
    ctx.save(); ctx.globalAlpha = a;
    ctx.drawImage(QS[i], 0, 0, QW, QHDR, r.x, r.y, r.w, hdrH);
    ctx.restore();
  };
  if (k <= 0) draw(idx, 1);
  else { draw(idx, 1 - k); draw(Math.min(idx + 1, QS.length - 1), k); }
}
```

> 不要重绘进度条和奔跑小人。抠图会带出旧进度条、把小人切成两截。

## 5. 终端打字开场（居中）

```js
const CODE = [
  "$ aiti init --mode cloud",
  "  loading questions ......... 12/12",
  // …8–11 行
];
function drawTerminal(t) {
  const FS = 30, LH = 44, STEP = 0.235, TYPE = 0.26;
  ctx.font = `500 ${FS}px Consolas, monospace`;
  const blockW = Math.max(...CODE.map(l => ctx.measureText(l).width));
  const x0 = CX - blockW / 2;
  const y0 = CY - (CODE.length * LH) / 2 + 10;
  CODE.forEach((line, i) => {
    const k = seg(t, 0.15 + i * STEP, 0.15 + i * STEP + TYPE);
    if (k <= 0) return;
    ctx.fillText(line.slice(0, Math.ceil(line.length * k)), x0, y0 + i * LH);
  });
}
```

打字太快是常见返工点：单行 0.26s、行间隔 0.235s，11 行约 2.9 秒。

## 6. 海报结尾

```js
const pw = 560, ph = pw * 1.5, px = CX - pw / 2, py = 470;
// 外框 + 立绘 + 色条 + 代码名 + 按钮
ctx.drawImage(PORTRAIT["TEM"], px + 22, py + 22, pw - 44, 460);
ctx.fillText("TEM · AI 驯兽师", px + 26, py + 574);
// 保存按钮在 1.2s 后弹出；落版字标在 2.5s 后小签
```

结尾不要停在 logo 上，停在"用户拿到的东西"上（结果卡 / 海报 / 搜索结果）。

## 7. 音乐：量拍点 + 裁切

```bash
node <skill>/scripts/audio.mjs beats "<track>.mp3"    # 输出 BPM、第一个强拍
```

```bash
# 让「结尾段起点」落在强拍上
ffmpeg -y -ss <offset> -t <DUR> -i track.mp3 \
  -af "afade=t=in:st=0:d=0.4,afade=t=out:st=<DUR-0.9>:d=0.9" \
  -codec:a libmp3lame -b:a 192k audio/ext/cut.mp3
```

## 8. 导出后重封（修真峰值）

```bash
ffmpeg -y -i out/final.mp4 -i audio/final.wav -map 0:v -map 1:a -c:v copy \
  -af "alimiter=limit=0.82:level=disabled" -c:a aac -b:a 192k -shortest out/final-c.mp4
```

## 9. 常用命令

```bash
export PATH="/d/<project>/tools/ffmpeg-9.0.2-essentials_build/bin:$PATH"

node <skill>/scripts/render.mjs <proj> --still 3,12,29 --sheet --size 1080x1920 --fps 60
node <skill>/scripts/render.mjs <proj> --text-check --size 1080x1920
node <skill>/scripts/render.mjs <proj> --determinism
AUDIO=file AUDIO_FILE=audio/final.wav sh <proj>/build.sh
node <skill>/scripts/inspect.mjs qa <proj>/out/final.mp4 --out <proj>/qa/x \
  --expect-size 1080x1920 --expect-fps 60 --expect-duration 34 --audio required
```
