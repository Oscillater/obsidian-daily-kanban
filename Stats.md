# Stats

```dataviewjs
const DAY = 86400000;
const WEEKS = 53;
const CELL = 11;
const KANBAN = "KanBan.md";
const ROUTINE = "Routine.md";
const MONTHS = ["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"];
const TASK_RE = /^(\s*)([-*+])\s+\[([ xX])\]\s*(.*)$/;

// Same palette, same index rule as the daily notes: colour = PALETTE[index % length].
// Nothing is bound to a category, so new kanban headings or routine items just take the next colour.
const PALETTE = [
  "var(--color-red, #e06c75)",
  "var(--color-blue, #61afef)",
  "var(--color-green, #6a9955)",
  "var(--color-orange, #d19a66)",
  "var(--color-purple, #c678dd)",
  "var(--color-cyan, #4d9390)",
  "var(--color-pink, #c25d7e)",
  "var(--color-yellow, #a87c1f)"
];

const pad = (n) => (n < 10 ? "0" : "") + n;
const now = new Date();
const isoOf = (d) => d.getFullYear() + "-" + pad(d.getMonth() + 1) + "-" + pad(d.getDate());

const host = dv.container;
host.innerHTML = "";

// ---------- colours ----------

const hexOf = (colour) => {
  const m = colour.match(/#[0-9a-fA-F]{6}/);
  return m === null ? "#808080" : m[0];
};
const rgbaOf = (colour, alpha) => {
  const h = hexOf(colour);
  const r = parseInt(h.slice(1, 3), 16);
  const g = parseInt(h.slice(3, 5), 16);
  const b = parseInt(h.slice(5, 7), 16);
  return "rgba(" + r + ", " + g + ", " + b + ", " + alpha + ")";
};
// The rgba() line is the always-valid fallback; assigning an unsupported
// color-mix() is silently ignored by the browser, so it only takes effect where it works.
const tintOf = (colour, alpha) => {
  const rgba = rgbaOf(colour, alpha);
  const mix = "color-mix(in srgb, " + colour + " " + Math.round(alpha * 100) + "%, transparent)";
  return { rgba: rgba, mix: mix };
};
const EMPTY = "var(--background-modifier-border, #d0d7de)";
const fillsOf = (colour) => [EMPTY, tintOf(colour, 0.28), tintOf(colour, 0.48), tintOf(colour, 0.72), tintOf(colour, 0.95)];
const paint = (el, fill) => {
  el.style.background = fill.rgba;
  el.style.background = fill.mix;
};

// ---------- the grid ----------

const today = new Date(now.getFullYear(), now.getMonth(), now.getDate());
const start = new Date(today.getTime() - (WEEKS * 7 - 1) * DAY);
while (start.getDay() !== 0) start.setDate(start.getDate() - 1);
const weeks = Math.floor((today - start) / (7 * DAY)) + 1;
const days = [];
for (let d = new Date(start); d <= today; d.setDate(d.getDate() + 1)) days.push(new Date(d));

const heat = (title, stat, colour, valueOf, fmt) => {
  const fills = fillsOf(colour);

  const head = document.createElement("div");
  head.style.cssText = "display:flex;align-items:baseline;gap:8px;margin:16px 0 4px";
  const name = document.createElement("span");
  name.textContent = title;
  name.style.cssText = "font-size:0.85em;font-weight:600;color:" + colour;
  head.appendChild(name);
  const extra = document.createElement("span");
  extra.textContent = stat;
  extra.style.cssText = "font-size:0.75em;color:var(--text-muted)";
  head.appendChild(extra);
  host.appendChild(head);

  const wrap = document.createElement("div");
  wrap.style.cssText = "overflow-x:auto;padding-bottom:4px";
  host.appendChild(wrap);

  const inner = document.createElement("div");
  inner.style.cssText = "display:inline-block";
  wrap.appendChild(inner);

  const months = document.createElement("div");
  months.style.cssText =
    "display:grid;grid-template-columns:repeat(" + weeks + "," + CELL + "px);gap:2px;" +
    "margin-left:30px;height:14px;font-size:0.7em;color:var(--text-muted)";
  inner.appendChild(months);

  const body = document.createElement("div");
  body.style.cssText = "display:flex;gap:4px";
  inner.appendChild(body);

  const wk = document.createElement("div");
  wk.style.cssText =
    "display:grid;grid-template-rows:repeat(7," + CELL + "px);gap:2px;width:26px;" +
    "font-size:0.7em;color:var(--text-muted)";
  ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"].forEach((d, i) => {
    const s = document.createElement("div");
    s.textContent = i % 2 === 1 ? d : "";
    s.style.lineHeight = CELL + "px";
    wk.appendChild(s);
  });
  body.appendChild(wk);

  const grid = document.createElement("div");
  grid.style.cssText =
    "display:grid;grid-template-rows:repeat(7," + CELL + "px);grid-auto-flow:column;" +
    "grid-auto-columns:" + CELL + "px;gap:2px";
  body.appendChild(grid);

  let lastMonth = -1;
  for (let i = 0; i < days.length; i++) {
    const d = days[i];
    if (d.getDay() === 0 && d.getMonth() !== lastMonth) {
      lastMonth = d.getMonth();
      const tag = document.createElement("div");
      tag.textContent = MONTHS[lastMonth];
      const col = Math.floor(i / 7);
      tag.style.cssText = "grid-column:" + (col + 1) + " / span " + Math.min(3, weeks - col) + ";white-space:nowrap";
      months.appendChild(tag);
    }
    const v = valueOf(d);
    const box = document.createElement("div");
    box.style.cssText = "width:" + CELL + "px;height:" + CELL + "px;border-radius:2px";
    if (v.level >= 0) paint(box, fills[v.level]);
    box.title = isoOf(d) + " - " + fmt(v);
    grid.appendChild(box);
  }

  const legend = document.createElement("div");
  legend.style.cssText =
    "display:flex;align-items:center;gap:3px;margin-top:6px;font-size:0.7em;color:var(--text-muted)";
  const less = document.createElement("span");
  less.textContent = "Less";
  less.style.marginRight = "3px";
  legend.appendChild(less);
  fills.forEach((f) => {
    const s = document.createElement("div");
    s.style.cssText = "width:" + CELL + "px;height:" + CELL + "px;border-radius:2px";
    if (typeof f === "string") s.style.background = f;
    else paint(s, f);
    legend.appendChild(s);
  });
  const more = document.createElement("span");
  more.textContent = "More";
  more.style.marginLeft = "3px";
  legend.appendChild(more);
  host.appendChild(legend);
};

// ---------- data ----------

// Which routine items each daily note has, and which of them were ticked off.
// Only notes whose mtime changed are re-read.
const store =
  window.__mydayHeat && window.__mydayHeat.v === 2
    ? window.__mydayHeat
    : (window.__mydayHeat = { v: 2, mtimes: {}, routine: {} });
const scanRoutine = (text) => {
  const present = {};
  const checked = {};
  let inSection = false;
  for (const raw of text.split(/\r?\n/)) {
    const t = raw.trim();
    if (/^#{1,6}\s/.test(t)) {
      inSection = t === "## Routine";
      continue;
    }
    if (inSection && /^- \[[ xX]\] /.test(t)) {
      const label = t.replace(/^- \[[ xX]\] /, "").trim();
      present[label] = true;
      if (t.toLowerCase().startsWith("- [x]")) checked[label] = true;
    }
  }
  return { present, checked };
};

for (const f of app.vault.getMarkdownFiles()) {
  if (!f.path.startsWith("daily/") || !/^\d{4}-\d{2}-\d{2}$/.test(f.basename)) continue;
  if (store.mtimes[f.path] === f.stat.mtime) continue;
  store.mtimes[f.path] = f.stat.mtime;
  store.routine[f.basename] = scanRoutine(await app.vault.read(f));
}

const routineFile = app.vault.getAbstractFileByPath(ROUTINE);
const routineItems =
  routineFile === null
    ? []
    : (await app.vault.read(routineFile))
        .split(/\r?\n/)
        .map((l) => l.trim())
        .filter((l) => /^- \[[ xX]\] /.test(l))
        .map((l) => l.replace(/^- \[[ xX]\] /, "").trim());

const kanHeadings = [];
const kanCounts = {};
const kanFile = app.vault.getAbstractFileByPath(KANBAN);
if (kanFile !== null) {
  let current = "";
  for (const line of (await app.vault.read(kanFile)).split(/\r?\n/)) {
    const h = line.match(/^#{1,6}\s+(.*)$/);
    if (h !== null) {
      current = h[1].trim();
      if (current !== "" && kanHeadings.indexOf(current) < 0) kanHeadings.push(current);
      continue;
    }
    const m = line.match(TASK_RE);
    if (m === null || current === "") continue;
    const d = m[4].match(/done::\s*(\d{4}-\d{2}-\d{2})/);
    if (d === null) continue;
    if (kanCounts[current] === undefined) kanCounts[current] = {};
    kanCounts[current][d[1]] = (kanCounts[current][d[1]] || 0) + 1;
  }
}

const sectionTitle = (text) => {
  const el = document.createElement("div");
  el.textContent = text;
  el.style.cssText =
    "font-size:1.05em;font-weight:600;margin:22px 0 10px;padding-bottom:4px;" +
    "border-bottom:1px solid var(--background-modifier-border)";
  host.appendChild(el);
};

// ---------- render ----------

sectionTitle("Routine");

routineItems.forEach((item, i) => {
  const colour = PALETTE[i % PALETTE.length];
  let kept = 0;
  let tracked = 0;
  for (const d of days) {
    const r = store.routine[isoOf(d)];
    if (r !== undefined && r.present && r.present[item]) {
      tracked += 1;
      if (r.checked && r.checked[item]) kept += 1;
    }
  }
  heat(
    item,
    kept + "/" + tracked + " days kept",
    colour,
    (d) => {
      const r = store.routine[isoOf(d)];
      if (r === undefined || !r.present || !r.present[item]) return { level: -1, on: false, seen: false };
      return { level: r.checked && r.checked[item] ? 4 : 0, on: !!(r.checked && r.checked[item]), seen: true };
    },
    (v) => (v.seen ? (v.on ? "kept" : "missed") : "not tracked")
  );
});

sectionTitle("Kanban");

kanHeadings.forEach((heading, i) => {
  const colour = PALETTE[i % PALETTE.length];
  const counts = kanCounts[heading] || {};
  let total = 0;
  for (const key in counts) total += counts[key];
  heat(
    heading,
    total + " completed",
    colour,
    (d) => {
      const n = counts[isoOf(d)] || 0;
      return { level: Math.min(4, n), n: n };
    },
    (v) => v.n + " completed"
  );
});
```
