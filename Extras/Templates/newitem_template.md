<%*
const mf = app.plugins.plugins.modalforms?.api;
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
const seedName = (tp.file?.title || "").replace(/\.(md|markdown)$/i, "");
const res = await mf.openForm('new-item-form', { values: { itemName: seedName } });

const cancelled =
  !res ||
  (typeof res === "object" && Object.keys(res).length === 0) ||
  res.cancelled === true || res.canceled === true;

if (cancelled) {
  new Notice("Form cancelled.");
  await cleanupIfBlankNewNote();
  return;
}

const result = res; // alias for the rest

// Guard against blank item name (prevents "Untitled Item")
const safeItemName = (result.itemName ?? "").toString().trim();
if (!safeItemName) {
  new Notice("Item creation cancelled");
  await cleanupIfBlankNewNote();
  return;
}

let magicalToggle = String(result.magical).toLowerCase();
let magicalValue  = (magicalToggle === "yes" || magicalToggle === "true") ? "true" : "false";

let attunementToggle = String(result.attunement).toLowerCase();
let attunementLine   = (attunementToggle === "yes" || attunementToggle === "true") ? "**Requires attunement.**" : "";
let vermunToggle = String(result.vermun).toLowerCase();

const cap = (s) => (typeof s === "string" && s.length) ? s.charAt(0).toUpperCase() + s.slice(1) : "";

let finalItemType = "";
let subItem = "";
let displayType = "";

if (magicalValue === "true") {
  finalItemType = (result.magicalitemtype ?? "").toString().trim();

  if (finalItemType === "Armor") {
    subItem = cap((result.marmortype ?? "").toString().trim());        // light|medium|heavy
  } else if (finalItemType === "Weapon") {
    subItem = cap((result.mweapontype ?? "").toString().trim());       // simple|martial
  }

  displayType = finalItemType || "Magical Item";
} else {
  finalItemType = (result.itemtype ?? "").toString().trim();

  if (finalItemType === "Armor") {
    subItem = cap((result.armortype ?? "").toString().trim());
  } else if (finalItemType === "Weapon") {
    subItem = cap((result.weapontype ?? "").toString().trim());
  }

  displayType = finalItemType || "Item";
}

let folderPath = (magicalValue === "true")
  ? "Compendium/Items/Homebrew Items/Magical"
  : "Compendium/Items/Homebrew Items/Mundane";

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
await ensureFolders(folderPath);

// FRONTMATTER (unchanged)
let frontmatter = `---
name: ${result.itemName}
type: ${result.itemtype}
rarity: ${result.rarity}
cost: ${result.price}
magical: ${magicalValue}
attunement: ${(attunementToggle === "yes" || attunementToggle === "true") ? "true" : "false"}
vermun: ${(vermunToggle === "yes" || attunementToggle === "true") ? "true" : "false"}
itemType: "${finalItemType}"
---`;

let noteBody = `
# ${result.itemName}

*${displayType}, ${result.rarity}*. ${attunementLine}

*${result.narrativeDesc}*

${result.details}
`;

// Optional attribution footer (unchanged)
let attributionStr = result.attribution != null ? String(result.attribution).trim() : "";
let attributionBlock = attributionStr.length > 0 ? `

---
Source: *${attributionStr}*` : "";

let noteContent = `${frontmatter}

${noteBody}${attributionBlock}
`;

// File name + create + open
let fileName = safeItemName; // we already validated it's non-empty
if (fileName.toLowerCase().endsWith(".md")) fileName = fileName.slice(0, -3);

let notePath = `${folderPath}/${fileName}`;
await tp.file.create_new(noteContent, notePath);

await new Promise(r => setTimeout(r, 200));
const newFile = tp.file.find_tfile(notePath) || app.vault.getAbstractFileByPath(notePath);
if (newFile) {
  app.workspace.openLinkText(newFile.basename, newFile.path, true);
} else {
  new Notice("File not found: " + notePath);
}
%>
