# কোড-টেমপ্লেট ও ইমপ্লিমেন্টেশন-নোট

**কখন পড়বে:** প্রকল্প গড়ার সময়। টাইমিং/SRT/ক্যাপশন হেল্পার এখানে পরীক্ষিত; বাকি টেমপ্লেট প্রথম রেন্ডারে দেখে নিও।

## ক. টাইমিং ও ক্যাপশন (হেল্পার + Caption কম্পোনেন্ট)

**পরীক্ষিত হেল্পার (`helpers.mjs`):**
```js
const seg = new Intl.Segmenter("bn", { granularity: "grapheme" });
export const glen = (s) => [...seg.segment(s)].length;

export function chunkWords(words, { maxWords = 3, maxChars = 24, gap = 0.45 } = {}) {
  const chunks = []; let cur = [];
  const flush = () => { if (cur.length) { chunks.push(cur); cur = []; } };
  for (const w of words) {
    const chars = glen([...cur, w].map(x => x.w).join(" "));
    if (cur.length && (cur.length >= maxWords || chars > maxChars)) flush();
    if (cur.length && w.s - cur[cur.length - 1].e > gap) flush();
    cur.push(w);
    if (/[।,?!;:—]$/.test(w.w)) flush();
  }
  flush();
  return chunks.map(c => ({ words: c, start: c[0].s, end: c[c.length - 1].e }));
}

// t সময়ে কোন চাঙ্ক ও কোন শব্দ সক্রিয়
export function captionState(chunks, t, hold = 0.35) {
  for (let i = 0; i < chunks.length; i++) {
    const c = chunks[i], next = chunks[i + 1];
    const until = next ? Math.min(next.start, c.end + hold) : c.end + hold;
    if (t >= c.start - 0.02 && t < until) {
      let active = -1;
      c.words.forEach((w, k) => { if (t >= w.s) active = k; });
      return { chunk: c, active };
    }
  }
  return null;
}

export const countAt = (t, startSec, durSec, from, to) => {
  const p = Math.min(1, Math.max(0, (t - startSec) / durSec));
  return Math.round(from + (to - from) * (1 - Math.pow(1 - p, 3)));
};
export const toBn = (n) => String(n).replace(/\d/g, d => "০১২৩৪৫৬৭৮৯"[d]);

// ---- SRT (পরীক্ষিত) ----
const tc = (h, m, s, ms) => (+h) * 3600 + (+m) * 60 + (+s) + (+ms.padEnd(3, "0")) / 1000;
export function parseSrt(text) {
  const t = text.replace(/^/, "").replace(/\r/g, "").trim();
  const cues = [];
  for (const block of t.split(/\n{2,}/)) {
    const lines = block.split("\n").map(l => l.trim()).filter(Boolean);
    const i = lines.findIndex(l => /-->/.test(l));
    if (i < 0) continue;
    const m = lines[i].match(/(\d+):(\d+):(\d+)[,.](\d+)\s*-->\s*(\d+):(\d+):(\d+)[,.](\d+)/);
    if (!m) continue;
    const txt = lines.slice(i + 1).join(" ").replace(/<[^>]+>/g, "").replace(/\{[^}]*\}/g, "").replace(/\s+/g, " ").trim();
    if (!txt) continue;
    cues.push({ s: tc(m[1], m[2], m[3], m[4]), e: tc(m[5], m[6], m[7], m[8]), text: txt });
  }
  cues.sort((a, b) => a.s - b.s);
  for (let k = 0; k < cues.length - 1; k++) if (cues[k].e > cues[k + 1].s) cues[k].e = cues[k + 1].s; // ওভারল্যাপ কাটো
  return cues.filter(c => c.e > c.s);
}

// ASR ছাড়া: প্রতিটি cue-র ভেতরে শব্দ-সময় গ্রাফিম-অনুপাতে ভাগ (cue-র বাইরে যায় না)
export function distributeWords(cues) {
  const words = [];
  for (const c of cues) {
    const ws = c.text.split(" ").filter(Boolean);
    const total = ws.reduce((a, w) => a + glen(w), 0) || 1;
    let t = c.s;
    for (const w of ws) {
      const d = (c.e - c.s) * glen(w) / total;
      words.push({ w, s: +t.toFixed(3), e: +(t + d).toFixed(3) });
      t += d;
    }
  }
  return words;
}
```

**Caption কম্পোনেন্ট (টেমপ্লেট; প্রথম ফ্রেমে দেখে নিও):**
```tsx
export const Caption: React.FC<{ chunks: Chunk[]; theme: Theme; fontFamily: string; layout: Layout }> = ({ chunks, theme, fontFamily, layout }) => {
  const frame = useCurrentFrame(); const { fps } = useVideoConfig(); const t = frame / fps;
  const st = captionState(chunks, t); if (!st) return null;
  const enter = interpolate(t - st.chunk.start, [0, 0.12], [0, 1], { extrapolateRight: "clamp" });
  return (
    <div style={{ position: "absolute", left: 0, right: 0, top: layout.caption.y, transform: "translateY(-50%)", display: "flex", justifyContent: "center", opacity: enter }}>
      <div style={{ display: "flex", gap: 14, padding: "16px 28px", borderRadius: px(layout, 30), background: theme.surfaceHi, border: `1.5px solid ${theme.border}`, fontFamily, fontWeight: 700, fontSize: layout.caption.font, lineHeight: 1.15 }}>
        {st.chunk.words.map((w, k) => {
          const on = k === st.active, done = k < st.active;
          return <span key={k} style={{ padding: "0 14px", borderRadius: 14, background: on ? theme.accent : "transparent", color: on ? theme.onAccent : done ? theme.text : theme.muted, opacity: on || done ? 1 : 0.5 }}>{w.w}</span>;
        })}
      </div>
    </div>
  );
};
```

## খ. প্রকল্প-গঠন, অনুপাত-প্রিসেট, ফন্ট, পথ, স্প্রিং

**প্রকল্প-গঠন:**
```
src/
  theme.ts        // ভূমিকা-ভিত্তিক রং (ব্যবহারকারীর রং এখানে বসে)
  layout.ts       // অনুপাত-প্রিসেট (৯:১৬ / ১৬:৯ / অন্য): ক্যানভাস, জোন, k, ক্যাপশন-মাপ; ইনপুটের অনুপাত থেকে বাছা
  motion.ts       // SPRING, SMOOTH, INOUT, stagger, cue()
  timing.json     // শব্দ-টাইমিং
  corrections.json // SRT-সংশোধন-লগ ও সন্দেহ-তালিকা
  scenes.ts       // সিন-তালিকা: { recipe, from, to, cues:{ keyword -> শব্দ-ইনডেক্স } } + দৃশ্য-বোর্ড
  icons/          // ছোট সহায়ক আইকন (আউটলাইন)
  art/            // ইলাস্ট্রেশন-কিট: আঁকা মানুষ/ভবন/যন্ত্র/প্রকৃতি, ভূমি, বাধা-বস্তু, ভূত-ভার্সন (illustration-kit.md)
  components/     // Caption, TabBar, FlowPath, Actor, Dots, MetricChip, TimerChip, Card, Stamp, Ruler, Beam, ...
  scenes/         // R১–R১৭
  Root.tsx
```

**অনুপাত-প্রিসেট (`layout.ts`):** সব জোন ও মাপ এক জায়গা থেকে আসবে। কম্পোনেন্টে ১০৮০, ১৯২০, ১৩৯৮-ধরনের মান সরাসরি লিখবে না।
```ts
export const RATIOS = {
  "9:16": { w: 1080, h: 1920, margin: 70,  tab: { y: 150, h: 62 }, label: { y: 262 }, stage: { top: 280, bottom: 1300 }, caption: { y: 1398, font: 62, pillH: 115 }, reserveFrom: 1500, k: 1,   glowR: 700 },
  "16:9": { w: 1920, h: 1080, margin: 100, tab: { y: 44,  h: 56 }, label: { y: 128 }, stage: { top: 160, bottom: 850 },  caption: { y: 940,  font: 52, pillH: 100 }, reserveFrom: 1000, k: 0.7, glowR: 850 },
} as const;
export type RatioKey = keyof typeof RATIOS;       // অন্য অনুপাত: SKILL.md-র অনুপাত-সূত্রে একই আকারের অবজেক্ট বানাও
export type Layout = (typeof RATIOS)[RatioKey];
export const px = (L: { k: number }, n: number) => Math.round(n * L.k);   // ৯:১৬-এর মাপকে স্কেল করতে
```
`Root.tsx`-এ `<Composition width={L.w} height={L.h} fps={30} … />`; ভিডিওর একটা `ratio` const (ইনপুট থেকে) দিয়ে `L = RATIOS[ratio]` নাও এবং সব কম্পোনেন্টে `layout={L}` পাস করো (বা React context)। রেন্ডারের কমান্ডে রেজল্যুশন Composition থেকেই আসে, আলাদা করে বদলাতে হয় না।

**ফন্ট:**
```ts
import { loadFont } from "@remotion/google-fonts/HindSiliguri";
export const { fontFamily: bn } = loadFont("normal", { weights: ["600", "700"], subsets: ["bengali", "latin"] });
```
নেটওয়ার্ক বন্ধ থাকলে ব্যবহারকারীর দেওয়া `.ttf/.woff2` ফাইল `staticFile` দিয়ে `@font-face`-এ লোড করো, অথবা npm-এর `@fontsource/hind-siliguri` প্যাকেজ।

**পথ ও ডটের প্রবাহ (`@remotion/paths`, নামগুলো যাচাই করা):**
```ts
import { getLength, getPointAtLength, getTangentAtLength, evolvePath } from "@remotion/paths";
const PATH = "M 520 1260 C 380 1060 700 900 540 700 S 420 420 560 300";   // উদাহরণ S-বাঁক (৯:১৬-এর); দৃশ্য অনুযায়ী বদলাও। ১৬:৯-এ পথ বাঁ→ডান, স্থানাঙ্ক L.w ও L.stage থেকে হিসেব করো
const L = getLength(PATH);
export function dotPos(i: number, n: number, t: number, speed = 200, lanes = 4, gap = 22) {
  const s = (((i / n) * L + t * speed) % L + L) % L;
  const p = getPointAtLength(PATH, s), tg = getTangentAtLength(PATH, s);
  const lane = ((i * 37) % lanes) - (lanes - 1) / 2;            // নির্ণায়ক লেন
  return { x: p.x - tg.y * lane * gap, y: p.y + tg.x * lane * gap };
}
const draw = (p: number) => evolvePath(p, PATH);               // { strokeDasharray, strokeDashoffset }
```
ভিড়/ধীর হওয়া: ওই অংশের দৈর্ঘ্য-পরিসরে `speed` কমাও ও `lane`-ফাঁক কমাও, ডট ঘন হবে। একই ফাংশন যেকোনো পথে কাজ করে: সোজা লাইন, বক্ররেখা, সিঁড়ি, তার, চক্র, সময়রেখা। **সংকেত-রেখা (beam) বাঁকানো:** দুই অংশের পথ (`M উৎস L ঠেকার-বিন্দু L নায়ক-এলাকা`) এক `evolvePath`-এ আঁকো; ঠেকার বিন্দুর `progress` ধরে রিং-পালস ট্রিগার করো।

**স্প্রিং ও প্রবেশ:**
```ts
const p = spring({ frame: frame - cueFrame, fps, config: SPRING.pop });
const y = interpolate(p, [0, 1], [40, 0]);
const blur = interpolate(p, [0, 1], [8, 0]);
```

**অডিও:** `<Audio src={staticFile("voice.mp3")} />` বা শেষে `ffmpeg` দিয়ে মিউক্স। রেন্ডার: `npx remotion render ... --codec=h264 --crf=18`, `yuv420p`, AAC।

**পারফরম্যান্স:** বড় এলাকায় blur/filter কম; কাট-স্ট্যাগার গুচ্ছ `memo`; ফ্রেম ধীর হলে `--concurrency` পরিমিত বাড়াও। বড় আঁকা দৃশ্য (জানালা-গ্রিড ইত্যাদি) SVG `<pattern>` বা একবার-আঁকা `memo` কম্পোনেন্টে রাখো।
