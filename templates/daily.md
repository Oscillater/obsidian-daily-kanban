## Routine

%%routine-begin%%
```dataviewjs
const SRC = "Routine.md";
const path = dv.currentFilePath;
// Built by concatenation so that indexOf() never matches the marker inside this script.
const BEGIN = "%%routine" + "-begin%%";
const END = "%%routine" + "-end%%";

if (typeof path === "string" && path.startsWith("daily/")) {
  const srcFile = app.vault.getAbstractFileByPath(SRC);
  const src = srcFile === null ? null : await app.vault.read(srcFile);
  const items =
    src === null
      ? []
      : src
          .split(/\r?\n/)
          .map((l) => l.trim())
          .filter((l) => /^- \[[ xX]\] /.test(l))
          .map((l) => "- [ ] " + l.replace(/^- \[[ xX]\] /, ""));
  const body = items.length > 0 ? items : ["- [ ] (empty list: " + SRC + ")"];

  const selfFile = app.vault.getAbstractFileByPath(path);
  const replace = (data) => {
    const start = data.indexOf(BEGIN);
    const stop = data.indexOf(END, start);
    if (start < 0 || stop <= start) return data;
    const eol = data.includes("\r\n") ? "\r\n" : "\n";
    return data.slice(0, start) + body.join(eol) + data.slice(stop + END.length);
  };

  if (typeof app.vault.process === "function") await app.vault.process(selfFile, replace);
  else {
    const data = await app.vault.read(selfFile);
    const out = replace(data);
    if (out !== data) await app.vault.modify(selfFile, out);
  }
  dv.container.innerHTML = "";
}
```
%%routine-end%%

## Task

%%task-begin%%
%%task-end%%

```dataviewjs
const KANBAN = "KanBan.md";
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
const TODAY = now.getFullYear() + "-" + pad(now.getMonth() + 1) + "-" + pad(now.getDate());

const filePath = typeof dv.currentFilePath === "string" ? dv.currentFilePath : "";
const TASK_RE = /^(\s*)([-*+])\s+\[([ xX])\]\s*(.*)$/;
const TB = "%%task" + "-begin%%";
const TE = "%%task" + "-end%%";
const escapeHtml = (s) => s.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;");
const plain = (text) =>
  text
    .replace(/\s*(planned|done)::\s*\d{4}-\d{2}-\d{2}/g, "")
    .replace(/<span[^>]*>[^<]*<\/span>/g, "")
    .replace(/\s+/g, " ")
    .trim();
const taskText = (line) => {
  const m = line.match(TASK_RE);
  return m === null ? null : plain(m[4]);
};

const host = dv.container;
const snap = window.__mydayTasks || (window.__mydayTasks = { state: {} });

const inNote = async (fn) => {
  const selfFile = app.vault.getAbstractFileByPath(filePath);
  if (selfFile === null) return;
  const run = (data) => {
    const eol = data.includes("\r\n") ? "\r\n" : "\n";
    return fn(data, eol);
  };
  if (typeof app.vault.process === "function") await app.vault.process(selfFile, run);
  else {
    const data = await app.vault.read(selfFile);
    const out = run(data);
    if (out !== data) await app.vault.modify(selfFile, out);
  }
};

const sectionList = (data) => {
  const s = data.indexOf(TB);
  const e = data.indexOf(TE, s);
  if (s < 0 || e <= s) return null;
  return data
    .slice(s + TB.length, e)
    .split(/\r?\n/)
    .map((l) => l.trim())
    .filter((l) => l !== "");
};

const rebuildSection = (list, s, e, eol) => {
  const inner = list.length > 0 ? eol + list.join(eol) + eol : eol;
  return s === null ? null : inner;
};

const insertTask = (data, eol, line, text) => {
  const s = data.indexOf(TB);
  const e = data.indexOf(TE, s);
  if (s < 0 || e <= s) return data;
  const list = sectionList(data);
  if (list.some((l) => taskText(l) === text)) return data;
  list.push(line);
  return data.slice(0, s + TB.length) + rebuildSection(list, s, e, eol) + data.slice(e);
};

const deleteTask = (data, eol, text) => {
  const s = data.indexOf(TB);
  const e = data.indexOf(TE, s);
  if (s < 0 || e <= s) return data;
  const list = sectionList(data).filter((l) => taskText(l) !== text);
  return data.slice(0, s + TB.length) + rebuildSection(list, s, e, eol) + data.slice(e);
};

// Writing the completion time back onto the kanban line.
const stamp = async (text, done) => {
  const kanFile = app.vault.getAbstractFileByPath(KANBAN);
  if (kanFile === null) return;
  const d = new Date();
  const stampDate = d.getFullYear() + "-" + pad(d.getMonth() + 1) + "-" + pad(d.getDate());
  let found = false;
  const run = (data) => {
    const eol = data.includes("\r\n") ? "\r\n" : "\n";
    const ls = data.split(/\r?\n/);
    let changed = false;
    for (let i = 0; i < ls.length; i++) {
      const m = ls[i].match(TASK_RE);
      if (m === null || plain(m[4]) !== text) continue;
      found = true;
      let next = ls[i].replace(/\[([ xX])\]/, "[" + (done ? "x" : " ") + "]");
      if (done) {
        if (!/done::\s*\d{4}-\d{2}-\d{2}/.test(next)) next = next.replace(/\s+$/, "") + " done:: " + stampDate;
      } else {
        next = next.replace(/\s*done::\s*\d{4}-\d{2}-\d{2}/, "");
      }
      if (next !== ls[i]) {
        ls[i] = next;
        changed = true;
      }
    }
    return changed ? ls.join(eol) : data;
  };
  if (typeof app.vault.process === "function") await app.vault.process(kanFile, run);
  else {
    const data = await app.vault.read(kanFile);
    const out = run(data);
    if (out !== data) await app.vault.modify(kanFile, out);
  }
  if (!found) new Notice("KanBan: no line matches - " + text);
};

// Watch this note for checkbox flips; only state *changes* reach the kanban.
const onModify = async () => {
  const selfFile = app.vault.getAbstractFileByPath(filePath);
  if (selfFile === null) return;
  const list = sectionList(await app.vault.read(selfFile));
  if (list === null) return;
  const states = {};
  for (const line of list) {
    const m = line.match(TASK_RE);
    if (m !== null) states[plain(m[4])] = m[3] !== " ";
  }
  const prev = snap.state[filePath];
  snap.state[filePath] = states;
  if (prev === undefined) return;
  for (const text in states) {
    if (prev[text] === undefined || prev[text] === states[text]) continue;
    await stamp(text, states[text]);
  }
};

if (dv.component.__mydayTaskSync !== true) {
  dv.component.__mydayTaskSync = true;
  dv.component.registerEvent(
    app.vault.on("modify", (f) => {
      if (f.path === filePath) onModify();
    })
  );
}

const muted = (text) => {
  const span = document.createElement("span");
  span.textContent = text;
  span.style.color = "var(--text-muted)";
  return span;
};

const build = async () => {
  host.innerHTML = "";

  const selfFile = app.vault.getAbstractFileByPath(filePath);
  const data = selfFile === null ? "" : await app.vault.read(selfFile);
  const list = sectionList(data);

  if (list === null) {
    host.appendChild(muted("Task section markers not found in this note."));
    return;
  }

  const mine = [];
  for (const line of list) {
    const m = line.match(TASK_RE);
    if (m !== null) mine.push({ text: plain(m[4]), checked: m[3] !== " " });
  }
  const states = {};
  mine.forEach((t) => {
    states[t.text] = t.checked;
  });
  if (snap.state[filePath] === undefined) snap.state[filePath] = states;

  const kanFile = app.vault.getAbstractFileByPath(KANBAN);
  if (kanFile === null) {
    host.appendChild(muted("Kanban not found: " + KANBAN));
    return;
  }

  const lines = (await app.vault.read(kanFile)).split(/\r?\n/);
  const headings = [];
  const headingAt = [];
  let current = "";
  for (const line of lines) {
    const h = line.match(/^#{1,6}\s+(.*)$/);
    if (h !== null) {
      current = h[1].trim();
      if (current !== "" && headings.indexOf(current) < 0) headings.push(current);
    }
    headingAt.push(current);
  }
  const colourOf = {};
  headings.forEach((h, i) => {
    colourOf[h] = PALETTE[i % PALETTE.length];
  });
  const colour = (h) => (h !== "" && colourOf[h] !== undefined ? colourOf[h] : "var(--text-faint, #808080)");

  const textToHeading = {};
  lines.forEach((line, i) => {
    const m = line.match(TASK_RE);
    if (m !== null) textToHeading[plain(m[4])] = headingAt[i] === undefined ? "" : headingAt[i];
  });
  const tagFor = (heading) =>
    heading === ""
      ? ""
      : ' <span style="color:' + colour(heading) + ';font-size:0.85em">' + escapeHtml(heading) + "</span>";

  const open = mine.filter((t) => !t.checked);
  const available = [];
  lines.forEach((line, i) => {
    const m = line.match(TASK_RE);
    if (m === null || m[3] !== " ") return;
    const text = plain(m[4]);
    if (states[text] !== undefined) return;
    available.push({ i, text, heading: headingAt[i] === undefined ? "" : headingAt[i] });
  });

  // --- remove chips for tasks sitting in this note ---
  if (open.length > 0) {
    const label = document.createElement("div");
    label.textContent = "Remove from this day";
    label.style.cssText = "margin:2px 0 6px;font-size:0.8em;color:var(--text-muted)";
    host.appendChild(label);

    const row = document.createElement("div");
    row.style.cssText = "display:flex;flex-wrap:wrap;gap:6px;margin-bottom:12px";
    host.appendChild(row);

    for (const t of open) {
      const c = colour(textToHeading[t.text] || "");
      const b = document.createElement("button");
      b.type = "button";
      b.textContent = "× " + t.text;
      b.title = "Delete this line from the note";
      b.style.cssText =
        "text-align:left;font-size:0.8em;padding:2px 8px;background:transparent;" +
        "border:1px solid currentColor;color:" + c;
      b.addEventListener("mouseenter", () => (b.style.opacity = "0.6"));
      b.addEventListener("mouseleave", () => (b.style.opacity = "1"));
      b.addEventListener("click", async () => {
        await inNote((d, eol) => deleteTask(d, eol, t.text));
        await build();
      });
      row.appendChild(b);
    }
  }

  // --- buttons that copy a kanban task into this note ---
  const label = document.createElement("div");
  label.textContent = "Add to today";
  label.style.cssText = "margin:2px 0 6px;font-weight:600;color:var(--text-muted)";
  host.appendChild(label);

  if (available.length === 0) {
    host.appendChild(muted("No tasks available to add."));
    return;
  }

  for (const heading of headings) {
    const group = available.filter((t) => t.heading === heading);
    if (group.length === 0) continue;
    const c = colour(heading);

    const title = document.createElement("div");
    title.textContent = heading;
    title.style.cssText =
      "margin:8px 0 4px;font-size:0.85em;color:" + c + ";border-bottom:1px solid " + c + ";padding-bottom:2px";
    host.appendChild(title);

    const rowBox = document.createElement("div");
    rowBox.style.cssText = "display:flex;flex-wrap:wrap;gap:6px;margin-top:6px";
    host.appendChild(rowBox);

    for (const t of group) {
      const b = document.createElement("button");
      b.type = "button";
      b.textContent = t.text;
      b.style.cssText = "text-align:left;background:transparent;border:1px solid currentColor;color:" + c;
      b.addEventListener("mouseenter", () => (b.style.opacity = "0.7"));
      b.addEventListener("mouseleave", () => (b.style.opacity = "1"));
      b.addEventListener("click", async () => {
        await inNote((d, eol) => insertTask(d, eol, "- [ ] " + t.text + tagFor(t.heading), t.text));
        await build();
      });
      rowBox.appendChild(b);
    }
  }
};

await build();
```
