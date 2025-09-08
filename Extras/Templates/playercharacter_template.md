<%*
// ─── SETUP: Access APIs ───────────────────────────────────────────────
const modalForm = app.plugins.plugins.modalforms.api;
const dv = app.plugins.plugins.dataview ? app.plugins.plugins.dataview.api : null;
if (!dv) {
  new Notice("Dataview plugin is not enabled.");
  return;
}

// Cleanup helper for newly-created empty notes (when user cancels)
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

// Open the form + robust cancel detection

const seedName = (tp.file?.title || "").replace(/\.(md|markdown)$/i, "");
const res = await modalForm.openForm("new-player-character", { values: { Name : seedName } });

const cancelled =
  !res ||
  (typeof res === "object" && Object.keys(res).length === 0) ||
  res.cancelled === true || res.canceled === true;

if (cancelled) {
  new Notice("Form cancelled.");
  await cleanupIfBlankNewNote();
  return;
}

const result = res;
let className = String(result.get("pcclass") ?? "").trim();

// ─── Helpers ──────────────────────────────────────────────────────────
function asLink(v) {
  if (!v) return "";
  const raw =
    typeof v === "string" ? v :
    v?.name || (v?.basename && v?.extension ? `${v.basename}.${v.extension}` : v?.path || "");
  if (!raw) return "";
  const s = String(raw).trim();
  if (/^\[\[.*\]\]$/.test(s)) return s;
  const target = s.split("|")[0].split("#")[0];
  return `[[${target}]]`;
}
function scalar(v) {
  if (v == null) return "";

  // arrays → first item
  if (Array.isArray(v)) return scalar(v[0]);

  // objects from pickers
  if (typeof v === "object") {
    if (v.name) return String(v.name);
    if (v.basename && v.extension) return `${v.basename}.${v.extension}`;
    if (v.path) return String(v.path).split("/").pop();
    return "";
  }

  // string cleanup
  let s = String(v).trim();
  if (!s) return "";

  // already a wikilink → pull target
  const w = s.match(/\[\[\s*([^|\]]+?)\s*(?:\|[^\]]*)?\]\]/);
  if (w) s = w[1];

  // try JSON (handles '["Dragonborn"]' or '{"name":"file.png"}' or '"Dragonborn"')
  try { return scalar(JSON.parse(s)); } catch {}

  // strip surrounding quotes if present
  s = s.replace(/^\s*['"]+/, "").replace(/['"]+\s*$/, "");

  // final: drop pipes/anchors and trailing path
  s = s.split("|")[0].split("#")[0].trim();
  return s;
}

// wraps a target as a wikilink; set {basename:true} to force pathless
function wl(v, { basename = false } = {}) {
  let t = scalar(v);
  if (!t) return "";
  if (basename) t = t.split("/").pop();
  t = t.replace(/^['"]+|['"]+$/g, ""); // safety: no quotes left
  return `[[${t}]]`;
}

function yamlQuote(s) {
  return `"${String(s).replace(/\\/g, "\\\\").replace(/"/g, '\\"')}"`;
}

// wikilink → YAML-safe (uses your scalar/wl logic, then quotes)
function wlYaml(v, opts = {}) {
  const s = wl(v, opts);           // your existing wikilink wrapper
  return s ? yamlQuote(s) : '""';  // quoted so YAML doesn't turn it into arrays
}

function asLinkBasename(v) {
  const s = asLink(v);
  if (!s) return "";
  const inner = s.slice(2, -2);               // remove [[ ]]
  const base = inner.split("|")[0].split("#")[0].split("/").pop();
  return base ? `[[${base}]]` : "";
}

function removeBold(str) { return str.replace(/\*\*/g, "").trim(); }

function getSubclass() {
  const generic = result.get("subclass");
  if (generic && String(generic).trim() !== "") {
    return String(generic).trim();
  }
  const subclassFields = [
    "subclass-cleric","subclass-artificer","subclass-barbarian","subclass-Bard",
    "subclass-Druid","subclass-Fighter","subclass-Monk","subclass-Mystic",
    "subclass-paladin","subclass-ranger","subclass-ranger revised",
    "subclass-ranger spelless","subclass-rogue","subclass-sorcerer",
    "subclass-warlock","subclass-wizard"
  ];
  for (const key of subclassFields) {
    const val = result.get(key);
    if (val && String(val).trim() !== "") {
      return String(val).trim();
    }
  }
  return "";
}
const subclass = getSubclass();

function getProficiencyBonus(level) {
  const lvl = Math.max(1, parseInt(level || "1", 10));
  if (lvl <= 4)  return 2;
  if (lvl <= 8)  return 3;
  if (lvl <= 12) return 4;
  if (lvl <= 16) return 5;
  return 6;
}

function processList(value) {
  if (!value) return "[]";
  const items = String(value).split(",").map(s=>s.trim()).filter(Boolean);
  return JSON.stringify(items);
}

//────────── Yaml lists ───────────────────
function toYamlList(key, arr) {
  function needsQuote(s) {
    if (s === "" || /^\s|\s$/.test(s)) return true;
    if (s.startsWith("[[") || s.includes("]]")) return true; // wikilinks
    if (/[#:,[\]{}&*!?|>'"%@`]/.test(s)) return true;        // YAML specials
    if (/^(?:y|Y|yes|Yes|n|N|no|No|true|True|false|False|null|Null|~|on|On|off|Off)$/.test(s)) return true;
    if (/^-?(?:\d+|\d*\.\d+)$/.test(s)) return true;         // numbers
    return false;
  }
  function dq(s) { return `"${s.replace(/\\/g, "\\\\").replace(/"/g, '\\"')}"`; }

  const list = Array.isArray(arr) ? arr : [];
  if (!list.length) return `${key}: []`;

  const lines = list
    .map(v => {
      const s = String(v ?? "").trim();
      return `  - ${needsQuote(s) ? dq(s) : s}`;
    })
    .join("\n");

  return `${key}:\n${lines}`;
}

function normalizeMulti(value) {
  if (Array.isArray(value)) return value.map(v => String(v).trim()).filter(Boolean);
  if (value === undefined || value === null) return [];
  const s = String(value).trim();
  try {
    const j = JSON.parse(s);
    if (Array.isArray(j)) return j.map(v => String(v).trim()).filter(Boolean);
  } catch (_) {}
  return s ? s.split(",").map(v => v.trim()).filter(Boolean) : [];
}

async function readTraitsFromFrontmatter(pathCandidates = []) {
  for (const p of pathCandidates) {
    const f = app.vault.getAbstractFileByPath(p);
    if (!f) continue;
    try {
      const fm = app.metadataCache.getFileCache(f)?.frontmatter;
      if (fm && Array.isArray(fm.traits)) {
        const arr = fm.traits.map(t => String(t).trim()).filter(Boolean);
        return JSON.stringify(arr);
      }
    } catch (_) {}
  }
  return "[]";
}

// Merge multiple JSON-stringified arrays, de-dupe, return JSON string
function mergeTraitJsonArrays(...jsonArrays) {
  const out = [];
  for (const j of jsonArrays) {
    try {
      const a = JSON.parse(j || "[]");
      if (Array.isArray(a)) for (const s of a) if (s && !out.includes(s)) out.push(s);
    } catch (_) {}
  }
  return JSON.stringify(out);
}

// ─── Name, path, status ───────────────────────────────────────────────
let nameRaw = String(result.get("Name") ?? "").trim();
if (!nameRaw) { new Notice("PC creation cancelled."); await cleanupIfBlankNewNote(); return; }
let name = nameRaw.toLowerCase().endsWith(".md") ? nameRaw.slice(0, -3) : nameRaw;

let statusValue = (result.get("Status") || "").trim().toLowerCase();
const folder = (statusValue === "inactive") ? "Inactive" : "Active";
const filePath = `World/People/Player Characters/${folder}/${name}`;

// ─── Calculated stats & lookups ───────────────────────────────────────
const level = parseInt(result.get("level") || "1", 10);
const proficiencyBonus = getProficiencyBonus(level);

// ─── Initiative ───────────────────────────────────────────────
function calcMod(score) {
  return Math.floor((Number(score ?? 10) - 10) / 2);
}
function parseBonusNumber(x) {
  if (x === undefined || x === null) return 0;
  const m = String(x).trim().match(/-?\d+(\.\d+)?/);
  return m ? Number(m[0]) : 0;
}
function computeInitiative(dexScore, extra) {
  const bonuses = Array.isArray(extra)
    ? extra
    : (extra == null ? [] : [extra]);
  const extraSum = bonuses.reduce((sum, v) => sum + parseBonusNumber(v), 0);
  return calcMod(dexScore) + extraSum;
}
function signed(n) { return (n >= 0 ? "+" : "") + n; }

const initiativeFinal = computeInitiative(result.get("dexterity"), result.get("initiative"));

// ─── Speed ──────────────────────────────
let speciesSpeed = "";
const speciesName = String(result.get("pcspecies") ?? "").trim();
if (speciesName) {
  const candidates = [
    `Compendium/Player Build/Species/${speciesName}.md`,
    `Compendium/Player Build/Species/${speciesName}/${speciesName}.md`
  ];
  let file = null;
  for (const p of candidates) {
    const f = app.vault.getAbstractFileByPath(p);
    if (f) { file = f; break; }
  }
  if (file) {
    const fm = app.metadataCache.getCache(file.path)?.frontmatter;
    const s = String((fm?.speed ?? fm?.Speed) ?? "").trim();
    if (s) speciesSpeed = s;
  }
}


// ─── Traits lookups ───────────────────────────────────────────────────
// Names from the form
const speciesForm   = (result.get("pcspecies")   || "").toString().trim();
const raceForm      = (result.get("race")      || "").toString().trim();
const backgroundForm= (result.get("background")|| "").toString().trim();

// Candidate paths (adjust if your vault uses different folders)
const speciesTraitPaths    = speciesForm   ? [`Compendium/Player Build/Species/${speciesForm}.md`, `Compendium/Player Build/Species/${speciesForm}/${speciesForm}.md`] : [];
const raceTraitPaths = (speciesForm && raceForm)? [`Compendium/Player Build/Species/${speciesForm}/Races/${raceForm}.md`] : [];
const backgroundTraitPaths = backgroundForm? [`Compendium/Player Build/Backgrounds/${backgroundForm}.md`] : [];

// Read traits from each source
const speciesTraitsJson    = await readTraitsFromFrontmatter(speciesTraitPaths);   // JSON string
const raceTraitsJson   = await readTraitsFromFrontmatter(raceTraitPaths);      // JSON string
const backgroundTraitsJson = await readTraitsFromFrontmatter(backgroundTraitPaths);// JSON string
const speciesRaceTraitsJson = mergeTraitJsonArrays(speciesTraitsJson, raceTraitsJson);


// ─── Proficiency readers ──────────────────────────────────────────────
function arrify(v) {
  if (Array.isArray(v)) return v.map(x => String(x).trim()).filter(Boolean);
  if (v === undefined || v === null) return [];
  const s = String(v).trim();
  try { const j = JSON.parse(s); if (Array.isArray(j)) return j.map(x => String(x).trim()).filter(Boolean); } catch (_) {}
  return s ? s.split(",").map(x => x.trim()).filter(Boolean) : [];
}

function mergeUniqueArrays(...lists) {
  const out = [];
  for (const list of lists) {
    for (const s of (list || [])) {
      const t = String(s).trim();
      if (t && !out.includes(t)) out.push(t);
    }
  }
  return out;
}

async function firstExistingFile(paths) {
  for (const p of paths) {
    const f = app.vault.getAbstractFileByPath(p);
    if (f) return f;
  }
  return null;
}

async function readProfArrayFrom(paths, key) {
  const map = {
    "saving_throws": { fm: ["saving_throws","saving-throws","Saving Throws","savingThrows"], label: "Saving Throws" },
    "weapon-prof":   { fm: ["weapon-prof","weapons","weapon_proficiencies","weapon-proficiencies","Weapons"], label: "Weapons" },
    "armor-prof":    { fm: ["armor-prof","armor","armors","armor_proficiencies","armor-proficiencies","Armor"], label: "Armor" },
    "tool-prof":     { fm: ["tool-prof","tools","tool_proficiencies","tool-proficiencies","Tools"], label: "Tools" },
  };
  const conf = map[key];
  if (!conf) return [];

  const file = await firstExistingFile(paths);
  if (!file) return [];

  const fm = app.metadataCache.getCache(file.path)?.frontmatter || {};
  for (const k of conf.fm) {
    if (fm[k] !== undefined) return arrify(fm[k]);
  }

  const body = await app.vault.cachedRead(file);
  const lines = body.split("\n");
  const rx = new RegExp(`^\\*\\*${conf.label}\\*\\*\\s*:\\s*(.+)$`, "i");
  for (const raw of lines) {
    const t = raw.trim();
    const m = t.match(rx);
    if (m) return arrify(removeBold(m[1]));
  }
  return [];
}


let subclassPath = "";
if (subclass && className) {
  subclassPath = `Compendium/Player Build/Classes/${className}/Subclasses/${subclass}.md`;
}

const classTraitPathCandidates = className ? [
  `Compendium/Player Build/Classes/${className}/${className}.md`,
  `Compendium/Player Build/Classes/${className}.md`
] : [];

const classTraitsJson = await readTraitsFromFrontmatter(classTraitPathCandidates);

const subclassTraitPathCandidates = (subclass && className) ? [
  `Compendium/Player Build/Classes/${className}/Subclasses/${subclass}.md`
] : [];

const subclassTraitsJson = await readTraitsFromFrontmatter(subclassTraitPathCandidates);

const classTraitsCombinedJson = mergeTraitJsonArrays(classTraitsJson, subclassTraitsJson);

// ─── Proficiency sources ──────────────────────────────────────────────
const classPathCandidates = className ? [
  `Compendium/Player Build/Classes/${className}/${className}.md`,
  `Compendium/Player Build/Classes/${className}.md`
] : [];

const speciesPaths = (result.get("pcspecies") || speciesForm) ? [
  `Compendium/Player Build/Species/${speciesForm || String(result.get("pcspecies")).trim()}.md`,
  `Compendium/Player Build/Species/${speciesForm || String(result.get("pcspecies")).trim()}/${speciesForm || String(result.get("pcspecies")).trim()}.md`
] : [];

const racePaths = (speciesForm && raceForm) ? [
  `Compendium/Player Build/Species/${speciesForm}/Races/${raceForm}.md`
] : [];

const backgroundPaths = backgroundForm ? [
  `Compendium/Player Build/Backgrounds/${backgroundForm}.md`
] : [];

// ─── Read & merge ─────────────────────────────────────────────────────
const savingThrowsArr = mergeUniqueArrays(
  await readProfArrayFrom(classPathCandidates, "saving_throws"),
  await readProfArrayFrom(speciesPaths,        "saving_throws"),
  await readProfArrayFrom(racePaths,           "saving_throws"),
  await readProfArrayFrom(backgroundPaths,     "saving_throws")
);

const weaponProfArr = mergeUniqueArrays(
  await readProfArrayFrom(classPathCandidates, "weapon-prof"),
  await readProfArrayFrom(speciesPaths,        "weapon-prof"),
  await readProfArrayFrom(racePaths,           "weapon-prof"),
  await readProfArrayFrom(backgroundPaths,     "weapon-prof")
);

const armorProfArr = mergeUniqueArrays(
  await readProfArrayFrom(classPathCandidates, "armor-prof"),
  await readProfArrayFrom(speciesPaths,        "armor-prof"),
  await readProfArrayFrom(racePaths,           "armor-prof"),
  await readProfArrayFrom(backgroundPaths,     "armor-prof")
);

const toolProfArr = mergeUniqueArrays(
  await readProfArrayFrom(classPathCandidates, "tool-prof"),
  await readProfArrayFrom(speciesPaths,        "tool-prof"),
  await readProfArrayFrom(racePaths,           "tool-prof"),
  await readProfArrayFrom(backgroundPaths,     "tool-prof")
);


// Spells → YAML list (lowercase key)
const rawSpells = result.get("spells");
let spellsArray = [];
try { if (rawSpells) spellsArray = JSON.parse(rawSpells); } catch {}
const wikiLinkedSpells = spellsArray.map(spell => `[[${spell}]]`);
const spellsYaml = toYamlList("Spells", wikiLinkedSpells);

// Skills (convert CSV/inputs to array JSON)
const skillsJson = processList(result.get("Skill Proficiencies"));

// Multiselects as arrays for YAML lists
const skillsArr      = normalizeMulti(result.get("Skill Proficiencies"));
const languagesArr   = normalizeMulti(result.get("Languages"));
const keyItemsArr    = normalizeMulti(result.get("key_items"));
const keyItemsWl = keyItemsArr.map(v => wl(v));

const appearencesArr = normalizeMulti(result.get("appearances")); // optional

// ─── Build Frontmatter (NEW SCHEMA) ───────────────────────────────────
const frontmatter = `---
name: ${name}
aliases: ${result.get("aliases") || ""}
hp: ${result.get("hp") || ""}
current_hp: ${result.get("hp") || ""}
ac: ${result.get("ac") || ""}
level: ${result.get("level") || ""}
species: ${wlYaml(result.get("pcspecies"))}
race: ${wlYaml(result.get("race"))}  
class: ${wlYaml(result.get("pcclass"))}
subclass: ${wlYaml(result.get("subclass"))}
background: ${wlYaml(result.get("background"))}
species-traits: ${speciesRaceTraitsJson}
class-traits: ${classTraitsCombinedJson}
background-traits: ${backgroundTraitsJson}
Alignment: ${result.get("Alignment") || ""}
strength: ${result.get("strength") || ""}
dexterity: ${result.get("dexterity") || ""}
constitution: ${result.get("constitution") || ""}
intelligence: ${result.get("intelligence") || ""}
wisdom: ${result.get("wisdom") || ""}
charisma: ${result.get("charisma") || ""}
iniative: ${signed(initiativeFinal)}
speed: ${speciesSpeed}
Proficiency bonus: +${proficiencyBonus}
${toYamlList('skills', skillsArr)}
${toYamlList('saving_throws', savingThrowsArr)}
${toYamlList('weapon-prof',   weaponProfArr)}
${toYamlList('armor-prof',    armorProfArr)}
${toYamlList('tool-prof',     toolProfArr)}
${toYamlList('languages', languagesArr)}
${spellsYaml}
${toYamlList('appearences', appearencesArr)}
${toYamlList('key_items', keyItemsWl)}
vermun_credit: 0
image: ${wlYaml(result.get("pc_picture"), { basename: true })}

---`;

// ─── Assemble Note Content (NEW BODY) ─────────────────────────────────
const noteContent = `${frontmatter}
# ${name}
*${subclass || ""} ${result.get("pcclass") || ""} - ${result.get("race") || ""} ${result.get("pcspecies") || ""} - ${result.get("Alignment") || ""}*

### Stats:
![[pc_stats.base]]

### Abilities:
![[pc_abilities.base]]

#### Spells:
![[pc_spells.base]]

## Character Details:
---
### History:

### Personality:

### Bonds/Goals

### Open Plot Threads:

### Relationships:

### Fun Facts:
`;

await tp.file.create_new(noteContent, filePath);
await new Promise(resolve => setTimeout(resolve, 2000));
let newFile = tp.file.find_tfile(filePath) || app.vault.getAbstractFileByPath(filePath);
if (newFile) {
  app.workspace.openLinkText(newFile.basename, newFile.path, true);
} else {
  new Notice("File not found: " + filePath);
}

async function createSecondFileFromTemplate(playerName) {
  const secondTemplateTFile = tp.file.find_tfile("Extras/Templates/playerdash_template");
  if (!secondTemplateTFile) {
    new Notice("Second template not found: Templates/playerdash_template");
    return;
  }
  let secondTemplateContent = await app.vault.read(secondTemplateTFile);
  secondTemplateContent = secondTemplateContent.replace(/{{PlayerName}}/g, name);
  const secondFilePath = `Extras/PC Dashboards/dash-${name}`;
  await tp.file.create_new(secondTemplateContent, secondFilePath);
  new Notice(`Created extra file for ${name}`);
}
await createSecondFileFromTemplate(name);
%>
