<%*
// ─── SETUP: Access APIs ───────────────────────────────────────────────
const modalForm = app.plugins.plugins.modalforms.api;
const dv = app.plugins.plugins.dataview ? app.plugins.plugins.dataview.api : null;
if (!dv) { new Notice("Dataview plugin is not enabled."); return; }

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

// Open the form (Map-aware cancel detection)
const res = await modalForm.openForm("new-npc-form");
const isMap = !!res && typeof res === "object" && typeof res.get === "function";
const explicitCanceled = isMap
  ? (res.get("canceled") === true || res.get("cancelled") === true)
  : (!!res && (res.canceled === true || res.cancelled === true));
const emptyPayload = isMap
  ? (res.size === 0)
  : (!!res && typeof res === "object" && Object.keys(res).length === 0);

if (!res || explicitCanceled || emptyPayload) {
  new Notice("Form cancelled.");
  await cleanupIfBlankNewNote();
  return;
}

const result = res;

// ─── Process NPC Basic Data ──────────────────────────────────────────
let npcName = String(result.get("Name") ?? "").trim();   // use Map.get
if (npcName.toLowerCase().endsWith(".md")) npcName = npcName.slice(0, -3);
if (!npcName) { new Notice("NPC creation cancelled."); await cleanupIfBlankNewNote(); return; }  // fix var
let notePath = `World/People/Non-Player Characters/${npcName}`;
let useStatBlock = String(result.get("Stats") || "").toLowerCase() === "true";

// ─── Retrieve NPC Level ───────────────────────────────────────────────
let npcLevel = parseInt(result.get("npc_level"), 10) || 1;

// ─── Ability Scores & Derived Stats ───────────────────────────────────
function rollAbilityScore() {
  let rolls = [
    Math.floor(Math.random() * 6) + 1,
    Math.floor(Math.random() * 6) + 1,
    Math.floor(Math.random() * 6) + 1,
    Math.floor(Math.random() * 6) + 1
  ];
  rolls.sort((a, b) => a - b);
  return rolls[1] + rolls[2] + rolls[3] + Math.floor(npcLevel / 2);  // Level-based boost
}

let npc_str = result.get("npc_STR") || rollAbilityScore();
let npc_dex = result.get("npc_DEX") || rollAbilityScore();
let npc_con = result.get("npc_CON") || rollAbilityScore();
let npc_int = result.get("npc_INT") || rollAbilityScore();
let npc_wis = result.get("npc_WIS") || rollAbilityScore();
let npc_cha = result.get("npc_CHA") || rollAbilityScore();

function abilityModifier(score) {
  return Math.floor((score - 10) / 2);
}

let bonusAC = Math.floor(npcLevel / 4);
let npc_ac = result.get("npc_ac") || (10 + abilityModifier(npc_dex) + bonusAC);

let totalHP = 0;
for (let i = 0; i < npcLevel; i++) {
  totalHP += (Math.floor(Math.random() * 8) + 1);
}
totalHP += npcLevel * abilityModifier(npc_con);
let npc_hp = result.get("npc_hp") || totalHP;

let npc_hit_dice = `${npcLevel}d8+${npcLevel * abilityModifier(npc_con)}`;

function asText(val) {
  if (!val) return "—";
  try {
    const arr = JSON.parse(val);
    if (Array.isArray(arr)) return arr.join(", ");
  } catch (_) {}
  return val;
}

function buildFrontmatterLinkList(raw) {
  let arr = [];
  if (raw) {
    try { arr = JSON.parse(raw); if (!Array.isArray(arr)) arr = raw.split(","); }
    catch { arr = raw.split(","); }
  }
  const items = Array.from(new Set(arr.map(v => String(v||"").trim().replace(/^"+|"+$/g,""))
    .filter(v=>v).map(v=>/^\[\[.*\]\]$/.test(v)?v:`[[${v}]]`)));
  return { items, yaml: items.map(v=>`  - "${v}"`).join("\n"), isEmpty: items.length===0 };
}

function parseArrayToPlainText(fieldValue) {
  if (!fieldValue) return "";
  try {
    let arr = JSON.parse(fieldValue);
    if (Array.isArray(arr)) {
      return arr.join(", ");
    } else {
      return fieldValue;
    }
  } catch (e) {
    return fieldValue;
  }
} 

const spells     = buildFrontmatterLinkList(result.get("Known Spells"));
const factions   = buildFrontmatterLinkList(result.get("Factions"));
const affiliates = buildFrontmatterLinkList(result.get("Affiliates"));
const statsOn = /^(true|yes|1|on)$/i.test(String(result.get("Stats") ?? "").trim());

const locRaw = String(result.get("Primary Location") || "").trim();
const locDisplay = (() => { const c = locRaw.replace(/^"+|"+$/g,"").replace(/^\[\[|\]\]$/g,""); return c ? `[[${c}]]` : "—"; })();

// ─── Build Frontmatter ─────────────────────────────────────────────────
let frontmatter = `---
Name: ${npcName}
Species: ${result.get("Species")}
Alignment: ${result.get("Alignment")}
${statsOn ? `Size: ${result.get("Size")}
Type: ${result.get("Type")}
hp: ${npc_hp}
ac: ${npc_ac}
Strength: ${npc_str}
Dexterity: ${npc_dex}
Constitution: ${npc_con}
Intelligence: ${npc_int}
Wisdom: ${npc_wis}
Charisma: ${npc_cha}
Skill Proficiencies: ${result.get("Proficiency")}
` : ""}Languages: 
${spells.isEmpty ? `Spells: []`       : `Spells:\n${spells.yaml}`}
Location: ${result.get("Primary Location")}
aliases: ${result.get("Aliases")}
${factions.isEmpty ? `Factions: []`   : `Factions:\n${factions.yaml}`}
${affiliates.isEmpty ? `Affiliates: []` : `Affiliates:\n${affiliates.yaml}`}
Appearances: ${parseArrayToPlainText(result.get("Appearances") ?? result.get("Appearences"))}

---`;

// ─── Build the Main Note Body ─────────────────────────────────────────
// Note that Appearances is just plain text here
let noteBody = `
# ${npcName}
_${result.get("Species")} ${result.get("Class")}, ${result.get("Alignment")}

**Appears In:** ${parseArrayToPlainText(result.get("Appearances"))}
**Primary Location:** ${locDisplay}
**Aliases:** ${asText(result.get("Aliases"))}
**Factions:** ${factions.items.join(", ") || "—"}
**Affiliates:** ${affiliates.items.join(", ") || "—"}

---
${result.get("bio") && result.get("bio").trim() !== "" ? `_${result.get("bio")}_` : ""}

${statsOn ? `
### Stat Block

**Size**: ${result.get("Size") || "—"}  
**Type**: ${result.get("Type") || "—"}  
**Armor Class**: ${npc_ac}  
**Hit Points**: ${npc_hp}  
**Skill Proficiencies**: ${asText(result.get("Proficiency"))}  
**Damage Vulnerabilities**: ${asText(result.get("Damage Vulnerability"))}  
**Damage Resistances**: ${asText(result.get("Damage Resistance"))}  
**Damage Immunities**: ${asText(result.get("Damage Immunity"))}  
**Condition Immunities**: ${asText(result.get("Condition Immunity"))}  

\`\`\`dataviewjs
const p = dv.current();
const nums = [p.Strength,p.Dexterity,p.Constitution,p.Intelligence,p.Wisdom,p.Charisma].map(Number);
if (nums.every(n => Number.isFinite(n))) {
  const mod = n => Math.floor((n-10)/2);
  dv.el("div", \`| STR | DEX | CON | INT | WIS | CHA |
|---|---|---|---|---|---|
| \${nums.map(mod).join(" | ")} |\`);
}
\`\`\`
` : ``}
`;

if (!spells.isEmpty) {
  noteBody += `
##### Spells
\`\`\`dataview
TABLE
  level AS "Lvl",
  school AS "School",
  damage_type AS "Damage",
  range AS "Range",
  casting_time AS "Casting Time"
FROM "Compendium/Spells"
WHERE contains(this.Spells, file.link)
SORT level ASC, file.name ASC
\`\`\`
`;
}


// ─── Combine All Content and Create the Note ─────────────────────────
let noteContent = `${frontmatter}

${noteBody}
`;

await tp.file.create_new(noteContent, notePath);
await new Promise(resolve => setTimeout(resolve, 2000));
let newFile = tp.file.find_tfile(notePath) || app.vault.getAbstractFileByPath(notePath);
if (newFile) {
  app.workspace.openLinkText(newFile.basename, newFile.path, true);
} else {
  new Notice("File not found: " + notePath);
}
%>
