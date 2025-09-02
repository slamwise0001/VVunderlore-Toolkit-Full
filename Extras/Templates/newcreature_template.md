<%*
const mf = app.plugins.plugins?.modalforms?.api;
if (!mf) { new Notice("Modal Forms plugin not found."); return; }

// Cleanup helper for newly-created empty notes (when user cancels)
async function cleanupIfBlankNewNote() {
  try {
    const cur = app.workspace.getActiveFile();
    if (cur) {
      const txt = await app.vault.read(cur);
      const ageMs = Date.now() - cur.stat.ctime; // creation age
      if (!txt.trim() && ageMs < 15000) await app.vault.delete(cur); // trash if enabled
    }
  } catch (_) {}
}

// Open the form + robust cancel detection
const res = await mf.openForm("new-creature");
const cancelled =
  !res ||
  (typeof res === "object" && Object.keys(res).length === 0) ||
  res.cancelled === true || res.canceled === true;

if (cancelled) {
  new Notice("Form cancelled.");
  await cleanupIfBlankNewNote();
  return;
}

const result = res; // alias

// ── helpers ───────────────────────────────────────────────────────────
function getFieldAsString(fieldName, def="") {
  const v = result[fieldName];
  if (v == null) return def;
  return String(v).trim();
}
function getFieldAsNumber(fieldName, def=0) {
  const v = result[fieldName];
  if (v == null) return def;
  const n = Number(v);
  return Number.isFinite(n) ? Math.floor(n) : def;
}
function getFieldAsBoolean(fieldName) {
  return !!result[fieldName];
}
function toYamlList(str) {
  if (!str) return "- None";
  const items = String(str).split(",").map(s => s.trim()).filter(Boolean);
  return items.length ? items.map(i => `- ${i}`).join("\n") : "- None";
}
function getProficiencyBonus(cr) {
  const map = {
    "0":2,"1/8":2,"1/4":2,"1/2":2,"1":2,"2":2,"3":2,"4":2,"5":3,"6":3,"7":3,"8":3,
    "9":4,"10":4,"11":4,"12":4,"13":5,"14":5,"15":5,"16":5,"17":6,"18":6,"19":6,"20":6,
    "21":7,"22":7,"23":7,"24":7,"25":8,"26":8,"27":8,"28":8,"29":9,"30":9
  };
  return map[cr] || 2;
}
// Extract a clean integer feet value from any input ("30", "30 ft.", "walk 30ft")
const feet = (v) => {
  const m = String(v ?? "").match(/-?\d+(\.\d+)?/);
  return m ? Math.round(parseFloat(m[0])) : 0;
};

// ── gather fields ─────────────────────────────────────────────────────
let name = getFieldAsString("name");
if (!name) { new Notice("Creature creation cancelled."); await cleanupIfBlankNewNote(); return; }

let classification = getFieldAsString("location"); // folder classification
let size = getFieldAsString("size", "Medium");
let type = getFieldAsString("type", "humanoid");
let alignment = getFieldAsString("alignment", "unaligned");

let ac = String(getFieldAsNumber("ac") || "10");
let naturalArmor = getFieldAsBoolean("armortype") ? "Natural Armor" : "None";
let hp = String(getFieldAsNumber("hp") || "1");
let hitDie = getFieldAsString("hitdie") || "1d6";

// speeds
let walkFt = feet(getFieldAsString("w-speed", "30"));
let swimFt = feet(getFieldAsString("s-speed", "0"));
let flyFt  = feet(getFieldAsString("f-speed", "0"));

// abilities
let str = String(getFieldAsNumber("cre-str") || "10");
let dex = String(getFieldAsNumber("cre-dex") || "10");
let con = String(getFieldAsNumber("cre-con") || "10");
let int = String(getFieldAsNumber("cre-int") || "10");
let wis = String(getFieldAsNumber("cre-wis") || "10");
let cha = String(getFieldAsNumber("cre-cha") || "10");

// misc
let challengeRating   = getFieldAsString("cr") || "0";
let proficiencyBonus  = String(getProficiencyBonus(challengeRating));

// FM lists (unchanged behavior)
let savingThrows        = toYamlList(getFieldAsString("savethrow"));
let skills              = toYamlList(getFieldAsString("skills"));
let vulnerabilities     = toYamlList(getFieldAsString("dmg_vul"));
let resistances         = toYamlList(getFieldAsString("dmg_res"));
let immunities          = toYamlList(getFieldAsString("dmg_imm"));
let conditionImmunities = toYamlList(getFieldAsString("cond_imm"));

// Senses: FM as array-of-strings when present; omit key when empty
const sensesInput = getFieldAsString("senses", "");
const sensesItems = sensesInput ? sensesInput.split(",").map(s => s.trim()).filter(Boolean) : [];
const fmSenses = sensesItems.length ? `[${sensesItems.map(s => `"${s}"`).join(", ")}]` : "";

// Frontmatter speed as array-of-strings
const speedItems = [];
if (walkFt > 0) speedItems.push(`Walk ${walkFt} ft.`);
if (flyFt  > 0) speedItems.push(`Fly ${flyFt} ft.`);
if (swimFt > 0) speedItems.push(`Swim ${swimFt} ft.`);
const fmSpeed = `[${speedItems.map(s => `"${s}"`).join(", ")}]`;

// free-text sections (RAW, no auto-format)
let traits           = getFieldAsString("traits");
let actions          = getFieldAsString("actions");
let reactions        = getFieldAsString("reactions");
let legendaryActions = getFieldAsString("legendary");
let lairActions      = getFieldAsString("lair-action");
let regionalEffects  = getFieldAsString("regional");
let description      = getFieldAsString("description");
let narrative        = getFieldAsString("narrative");

// ── FRONTMATTER ───────────────────────────────────────────────────────
let frontmatter = `---
name: "${name}"
size: "${size}"
type: "${type}"
alignment: "${alignment}"
armor_class: ${ac} (${naturalArmor})
hit_points: ${hp}
hit_die: "${hitDie}"
speed: ${fmSpeed}
${fmSenses ? `senses: ${fmSenses}\n` : ""}strength: ${str}
dexterity: ${dex}
constitution: ${con}
intelligence: ${int}
wisdom: ${wis}
charisma: ${cha}
proficiency_bonus: +${proficiencyBonus}
saving_throws:
${savingThrows}
skills:
${skills}
damage_vulnerabilities:
${vulnerabilities}
damage_resistances:
${resistances}
damage_immunities:
${immunities}
condition_immunities:
${conditionImmunities}

challenge_rating: "${challengeRating}"
---`;

// ── BODY (RAW user formatting preserved) ──────────────────────────────
const subtitle = `${size} ${type}, ${alignment}`.trim();
const descInline = description ? `\n${description}\n` : "";
const narrativeInline = narrative ? `\n_${narrative}_\n` : "";

// Body speed line (clean numbers; lowercase labels)
const speedBitsBody = [];
if (walkFt > 0) speedBitsBody.push(`walk ${walkFt} ft.`);
if (flyFt  > 0) speedBitsBody.push(`fly ${flyFt} ft.`);
if (swimFt > 0) speedBitsBody.push(`swim ${swimFt} ft.`);
const speedLine = speedBitsBody.length ? `**Speed**: ${speedBitsBody.join(", ")}` : "";

// Optional lines: show EXACTLY what the user typed (raw)
const savingThrowsRaw = getFieldAsString("savethrow");
const skillsRaw       = getFieldAsString("skills");
const sensesRaw       = sensesInput; // already a raw string
const languagesRaw    = getFieldAsString("languages");

// sections: raw (no transformation)
const traitsBlock      = traits && traits.trim() ? traits : "";
const actionsBlock     = actions && actions.trim() ? actions : "";
const reactionsBlock   = reactions && reactions.trim() ? reactions : "";
const legendariesBlock = legendaryActions && legendaryActions.trim() ? legendaryActions : "";
const lairBlock        = lairActions && lairActions.trim() ? lairActions : "";
const regionalBlock    = regionalEffects && regionalEffects.trim() ? regionalEffects : "";

// Assemble body
let statBlock = `# ${name}

${subtitle}
${narrativeInline}${descInline}
**Armor Class**: ${ac}
**Hit Points**: ${hp}${hitDie ? ` (${hitDie})` : ""}
${speedLine}
${savingThrowsRaw ? `**Saving Throws**: ${savingThrowsRaw}\n` : ""}${skillsRaw ? `**Skills**: ${skillsRaw}\n` : ""}${sensesRaw ? `**Senses**: ${sensesRaw}\n` : ""}${languagesRaw ? `**Languages**: ${languagesRaw}\n` : ""}**Challenge**: ${challengeRating}

| STR | DEX | CON | INT | WIS | CHA |
| --- | --- | --- | --- | --- | --- |
| ${str} | ${dex} | ${con} | ${int} | ${wis} | ${cha} |

${traitsBlock ? `### Traits\n${traitsBlock}\n` : ""}${actionsBlock ? `### Actions\n${actionsBlock}\n` : ""}${reactionsBlock ? `### Reactions\n${reactionsBlock}\n` : ""}${legendariesBlock ? `### Legendary Actions\n${legendariesBlock}\n` : ""}${lairBlock ? `### Lair Actions\n${lairBlock}\n` : ""}${regionalBlock ? `### Regional Effects\n${regionalBlock}\n` : ""}`.trim();

// ── file path & create ────────────────────────────────────────────────
let sanitizedClassification = classification.replace(/[^a-zA-Z0-9-_ ]/g, "").trim();
let fileName = name.replace(/[^a-zA-Z0-9-_ ]/g, "").trim();
let filePath = `Compendium/Bestiary/${sanitizedClassification}/${fileName}`;

await tp.file.create_new(frontmatter.trim() + "\n" + statBlock.trim(), filePath);

await new Promise(resolve => setTimeout(resolve, 2000));

// Open the created file
let newFile = tp.file.find_tfile(filePath) || app.vault.getAbstractFileByPath(filePath);
if (newFile) {
  app.workspace.openLinkText(newFile.basename, newFile.path, true);
} else {
  new Notice("File not found: " + filePath);
}
%>
