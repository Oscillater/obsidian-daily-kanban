
```dataviewjs
// Keeps done:: in sync with this file's own checkboxes.
const TASK_RE = /^(\s*)([-*+])\s+\[([ xX])\]\s*(.*)$/;
const FIELD_RE = /\s*(planned|done)::\s*\d{4}-\d{2}-\d{2}/g;
const pad = (n) => (n < 10 ? "0" : "") + n;
const file = app.vault.getAbstractFileByPath("KanBan.md");
const store = window.__mydayKan || (window.__mydayKan = { state: null });

const plain = (text) => text.replace(FIELD_RE, "").trim();
const readStates = (data) => {
  const states = {};
  for (const line of data.split(/\r?\n/)) {
    const m = line.match(TASK_RE);
    if (m !== null) states[plain(m[4])] = m[3] !== " ";
  }
  return states;
};

const sync = async () => {
  if (file === null) return;
  const data = await app.vault.read(file);
  const states = readStates(data);
  const prev = store.state;
  store.state = states;
  if (prev === null) return;
  const flipped = Object.keys(states).filter((t) => prev[t] !== undefined && prev[t] !== states[t]);
  if (flipped.length === 0) return;

  const d = new Date();
  const stampDate = d.getFullYear() + "-" + pad(d.getMonth() + 1) + "-" + pad(d.getDate());
  const run = (raw) => {
    const eol = raw.includes("\r\n") ? "\r\n" : "\n";
    const ls = raw.split(/\r?\n/);
    let changed = false;
    for (let i = 0; i < ls.length; i++) {
      const m = ls[i].match(TASK_RE);
      if (m === null) continue;
      const text = plain(m[4]);
      if (flipped.indexOf(text) < 0) continue;
      let next = ls[i].replace(/\[([ xX])\]/, "[" + (states[text] ? "x" : " ") + "]");
      if (states[text]) {
        if (!/done::\s*\d{4}-\d{2}-\d{2}/.test(next)) next = next.replace(/\s+$/, "") + " done:: " + stampDate;
      } else {
        next = next.replace(/\s*done::\s*\d{4}-\d{2}-\d{2}/, "");
      }
      if (next !== ls[i]) {
        ls[i] = next;
        changed = true;
      }
    }
    return changed ? ls.join(eol) : raw;
  };

  if (typeof app.vault.process === "function") await app.vault.process(file, run);
  else {
    const out = run(data);
    if (out !== data) await app.vault.modify(file, out);
  }
};

if (dv.component.__mydayKanSync !== true) {
  dv.component.__mydayKanSync = true;
  dv.component.registerEvent(
    app.vault.on("modify", (f) => {
      if (file !== null && f.path === file.path) sync();
    })
  );
}
if (store.state === null) sync();
```

# Life

- [ ] Sort out the wardrobe
- [ ] Do the laundry
- [ ] Buy groceries for the week
- [x] Water the plants done:: 2026-10-12
- [x] Fix the bike bell done:: 2026-10-09

# Study

- [ ] Read chapter 4 and take notes
- [ ] Submit the weekly problem set
- [ ] Meet the advisor
- [x] Review the lecture slides done:: 2026-10-14

# Side Projects

- [ ] Write the README for the sync module
- [ ] Record a short demo video
- [ ] Refactor the index builder
- [x] Set up the CI pipeline done:: 2026-10-11

# Recurring

- [ ] Weekly review
- [x] Back up the vault done:: 2026-10-13
