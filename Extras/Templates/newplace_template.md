<%*
const mf = app.plugins.plugins?.modalforms?.api;
if (!mf) { new Notice("Modal Forms plugin not found."); return; }

// ── cleanup helper for “new + empty” notes ────────────────────────────
async function cleanupIfBlankNewNote() {
  try {
    const cur = app.workspace.getActiveFile();
    if (cur) {
      const txt = await app.vault.read(cur);
      const ageMs = Date.now() - cur.stat.ctime;
      if (!txt.trim() && ageMs < 15000) await app.vault.delete(cur);
    }
  } catch (_) {}
}

// ── open form ─────────────────────────────────────────────────────────
const seedName = (tp.file?.title || "").replace(/\.(md|markdown)$/i, "");
const res = await mf.openForm("new-place-form", { values: { name : seedName } });

const cancelled =
  !res ||
  (typeof res === "object" && Object.keys(res).length === 0) ||
  res.cancelled === true || res.canceled === true;

if (cancelled) { new Notice("Form cancelled."); await cleanupIfBlankNewNote(); return; }
const result = res;

// ── helpers ───────────────────────────────────────────────────────────
const getFieldAsString = (k, def="") => {
  const v = result[k];
  if (v == null) return def;
  if (Array.isArray(v)) return v.join(", ");
  return String(v).trim();
};
const arrayify = (val) => {
  if (!val) return [];
  if (Array.isArray(val)) return val.map(x => String(x).trim()).filter(Boolean);
  return String(val).split(",").map(s => s.trim()).filter(Boolean);
};
const yamlEscape = (s) => String(s ?? "").replace(/"/g, '\\"');

const toWikiLinkYamlList = (val) => {
  const items = arrayify(val);
  return items.length ? items.map(i => `  - "[[${yamlEscape(i)}]]"`).join("\n") : "  -";
};
const toPlainYamlList = (val) => {
  const items = arrayify(val);
  return items.length ? items.map(i => `  - "${yamlEscape(i)}"`).join("\n") : "  -";
};
const rawAppears  = result["appearances"] ?? result["appearences"] ?? [];
const appearsArr  = Array.isArray(rawAppears) ? rawAppears : arrayify(rawAppears);
const appearsLine = appearsArr.length ? `
**Appears In:** \`= this.appearances\`` : "";

// ── fields ────────────────────────────────────────────────────────────
let name = getFieldAsString("name");
if (!name) { await cleanupIfBlankNewNote(); return; }

let type          = getFieldAsString("type");
let narrativeDesc = getFieldAsString("narrative desc");
let desc          = getFieldAsString("desc");
let history       = getFieldAsString("history");
let location      = getFieldAsString("location");          // <- new single field
let popSize       = getFieldAsString("population size");
let climate       = getFieldAsString("climate");
let denizens      = getFieldAsString("denizens");
// support either spelling just in case the form still has the old key
let appearsRaw    = result["appearances"] ?? result["appearences"] ?? "";

// lists
let denizensYamlList  = toWikiLinkYamlList(denizens);
let appearsInYamlList = toPlainYamlList(appearsRaw);

// ── frontmatter ───────────────────────────────────────────────────────
let frontmatter = `---
name: "${yamlEscape(name)}"
location: "${yamlEscape(location)}"
size: "${yamlEscape(popSize)}"
type: "${yamlEscape(type)}"
climate: "${yamlEscape(climate)}"
denizens :
${denizensYamlList}
appearances :
${appearsInYamlList}
---`;

// ── body ──────────────────────────────────────────────────────────────
let locationLine = `***\`= this.type\`***`;
if (location) locationLine += ` in \`= this.location\``;

let noteBody = `# \`= this.name\`
${locationLine}
**Known Denizens:** \`= this.denizens\`

---
_${narrativeDesc}_

---
${desc}

${history}

${appearsLine}
`;

// ── write file ────────────────────────────────────────────────────────
let noteContent = frontmatter + "\n\n" + noteBody;

const folder = "World/Places";
async function ensureFolders(path) {
  const parts = path.split("/");
  let acc = "";
  for (const part of parts) {
    acc = acc ? `${acc}/${part}` : part;
    if (!(await app.vault.adapter.exists(acc))) {
      try { await app.vault.createFolder(acc); } catch (_) {}
    }
  }
}
await ensureFolders(folder);

let fileName = name.trim();
if (fileName.toLowerCase().endsWith(".md")) fileName = fileName.slice(0, -3);

let filePath = `${folder}/${fileName}`;
await tp.file.create_new(noteContent, filePath);

// open
await new Promise(r => setTimeout(r, 200));
let newFile = tp.file.find_tfile(filePath) || app.vault.getAbstractFileByPath(filePath);
if (newFile) app.workspace.openLinkText(newFile.basename, newFile.path, true);
else new Notice("File not found: " + filePath);
%>
