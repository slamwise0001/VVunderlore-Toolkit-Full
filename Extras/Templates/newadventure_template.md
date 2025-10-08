<%*  

let adventureName = await tp.system.prompt("Enter Adventure Name:");
if (!adventureName || adventureName.trim() === "") {
    new Notice("Adventure creation canceled.");
    return;
}
adventureName = adventureName.trim().replace(/[<>:"/\\|?*]/g, ''); 

const basePath = `Adventures/${adventureName}`;
const sessionNotesPath = `${basePath}/Session Notes`;
const newFilePath = `${basePath}/${adventureName} - Adventure Hub`;
let sessionTemplatePath = `Extras/Templates/(${adventureName})sessiontemplate`; 

const baseSessionTemplate = "[[Extras/Templates/base_sessiontemplate.md]]";

const fence = '```';

for (const f of [basePath, `${basePath}/Adventure`, `${basePath}/Materials`, sessionNotesPath]) {
    if (!(await app.vault.adapter.exists(f))) {
        await app.vault.createFolder(f);
    }
}
await new Promise(r => setTimeout(r, 500));  

let frontmatter = `---
players: []
adventure: "${adventureName}"
---`;

// adventure hub note
let noteContent = `${frontmatter}
# 📜 ${adventureName} - Adventure Hub

## 📚 Chapter Links
${fence}dataviewjs
const currentFolder = dv.current().file.folder;
const adventureFolder = currentFolder + '/Adventure';

const pages = dv.pages('"' + adventureFolder + '"').sort(p => p.file.name);
if (pages.length === 0) {
  dv.paragraph("You haven't written anything yet.");
} else {
  pages.forEach(p => dv.header(4, p.file.link));
}
${fence}

---
>[!success] Current Session:
>${fence}dataviewjs
>const pages = dv.pages('"Adventures/${adventureName}/Session Notes"')
>  .sort(p => p.file.ctime, 'desc')
>  .limit(1);
>if (pages.length) {
>  dv.header(4, pages[0].file.link);
>}
>${fence}

${fence}meta-bind-button
label: New Session
icon: ""
hidden: false
class: ""
tooltip: "Create a new session note for this adventure"
id: first-session
style: primary
actions:
  - type: templaterCreateNote
    templateFile: "Extras/Templates/(${adventureName})sessiontemplate.md"
    folderPath: "Adventures/${adventureName}/Session Notes"
    fileName: ""
    openNote: true
${fence}

---
## 👥 Player Characters
${fence}dataviewjs
const container = dv.container;
container.style.display = "flex";
container.style.flexDirection = "column";
container.style.gap = "10px";
container.style.marginBottom = "20px";

const buttonContainer = document.createElement("div");
buttonContainer.style.display = "flex";
buttonContainer.style.gap = "10px"; 
buttonContainer.style.alignItems = "center"; 
buttonContainer.style.justifyContent = "flex-start"; 

let playerCharacters = dv
  .pages('"World/People/Player Characters/Active"')
  .concat(dv.pages('"World/People/Player Characters/Inactive"'))
  .map(p => p.file.name)
  .sort();

const dropdownButton = document.createElement("button");
dropdownButton.innerHTML = "Select Player Characters";
dropdownButton.style.padding = "6px 12px";
dropdownButton.style.border = "1px solid #444";
dropdownButton.style.borderRadius = "4px";
dropdownButton.style.cursor = "pointer";
dropdownButton.style.backgroundColor = "#444";
dropdownButton.style.color = "#fff";
dropdownButton.style.minWidth = "180px"; 
dropdownButton.style.textAlign = "center";

const dropdownContent = document.createElement("div");
dropdownContent.style.display = "none";
dropdownContent.style.position = "absolute";
dropdownContent.style.backgroundColor = "#333";
dropdownContent.style.border = "1px solid #444";
dropdownContent.style.borderRadius = "4px";
dropdownContent.style.padding = "10px";
dropdownContent.style.width = "200px";
dropdownContent.style.zIndex = "9999";
dropdownContent.style.maxHeight = "200px";
dropdownContent.style.overflowY = "auto";

playerCharacters.forEach(character => {
  const label = document.createElement("label");
  label.style.color = "#fff";
  label.style.display = "flex";
  label.style.alignItems = "center";
  label.style.padding = "5px";
  label.style.cursor = "pointer";
  
  const checkbox = document.createElement("input");
  checkbox.type = "checkbox";
  checkbox.value = character;
  
  label.appendChild(checkbox);
  label.appendChild(document.createTextNode(" " + character));

  label.addEventListener("mouseover", () => (label.style.backgroundColor = "#555"));
  label.addEventListener("mouseout", () => (label.style.backgroundColor = "transparent"));

  dropdownContent.appendChild(label);
});

document.body.appendChild(dropdownContent);

dropdownButton.addEventListener("click", function () {
  if (dropdownContent.style.display === "none") {
    dropdownContent.style.display = "block";
    const rect = dropdownButton.getBoundingClientRect();
    dropdownContent.style.left = rect.left + "px";
    dropdownContent.style.top = rect.bottom + window.scrollY + "px";
  } else {
    dropdownContent.style.display = "none";
  }
});

document.addEventListener("click", function (event) {
  if (!dropdownButton.contains(event.target) && !dropdownContent.contains(event.target)) {
    dropdownContent.style.display = "none";
  }
});

const addCharacterButton = document.createElement("button");
addCharacterButton.innerHTML = "Add";
addCharacterButton.style.padding = "6px 12px";
addCharacterButton.style.border = "1px solid #444";
addCharacterButton.style.borderRadius = "4px";
addCharacterButton.style.cursor = "pointer";
addCharacterButton.style.backgroundColor = "#444";
addCharacterButton.style.color = "#fff";
addCharacterButton.style.minWidth = "80px";
addCharacterButton.style.textAlign = "center";

addCharacterButton.addEventListener("click", async () => {
  const selectedCharacters = Array.from(dropdownContent.querySelectorAll("input:checked"))
    .map(cb => "[[" + cb.value + "]]");

  if (selectedCharacters.length === 0) {
    new Notice("Please select at least one player character.");
    return;
  }

  const file = app.workspace.getActiveFile();
  if (!file) {
    new Notice("No active file detected.");
    return;
  }

  await app.fileManager.processFrontMatter(file, frontmatter => {
    if (!frontmatter.players) {
      frontmatter.players = [];
    }
    if (!Array.isArray(frontmatter.players)) {
      frontmatter.players = [frontmatter.players];
    }
    frontmatter.players = [...new Set([...frontmatter.players, ...selectedCharacters])];
  });

  new Notice("Updated players: " + selectedCharacters.join(", "));
});

buttonContainer.appendChild(dropdownButton);
buttonContainer.appendChild(addCharacterButton);
container.appendChild(buttonContainer);

const players = dv.current().players; 

if (!players || players.length === 0) {
    dv.paragraph("No players found.");
} else {
    const playerPages = players
        .map(player => dv.page(player)) 
        .filter(p => p);

    dv.table(
        ["Player", "Species", "Class", "Subclass", "Level", "Max HP", "AC", "STR", "DEX", "CON", "INT", "WIS", "CHA"],
        playerPages.map(p => [
            p.file.link, p.Species, p.Class, p.Subclass, p.level, p.hp, p.ac, 
            p.Strength, p.Dexterity, p.Constitution, p.Intelligence, 
            p.Wisdom, p.Charisma
        ])
    );
}

${fence}

---
## ⏳ Takeaways
${fence}dataviewjs
const folderPath = "${basePath}/Session Notes";

const extractTakeaways = (content, startHeader, endHeader) => {
const startPattern = new RegExp('^#+\\\\s*' + startHeader + '\\\\s*', 'im');
const endPattern   = new RegExp('^#+\\\\s*' + endHeader   + '\\\\s*', 'im');

  const startMatch = content.match(startPattern);
  const endMatch   = content.match(endPattern);
  if (!startMatch) return null;

  const startIndex = startMatch.index + startMatch[0].length;
  const endIndex   = endMatch ? endMatch.index : content.length;

  return content
    .slice(startIndex, endIndex)
    .replace(/^(#+.*|Notes last touched.*)$/gim, "")
    .trim() || null;
};

(async () => {
  const pages = dv.pages('"' + folderPath + '"');

  let tableData = await Promise.all(pages.map(async page => {
    const content  = await dv.io.load(page.file.path);
    const takeaway = content ? extractTakeaways(content, "Takeaways", "Next Session") : null;
    return takeaway ? [page.file.name, takeaway] : null;
  }));

tableData = tableData.filter(row => row !== null);

tableData.sort((a, b) => {
    const matchA = a[0].match(/Session (\\d+)/);
	const matchB = b[0].match(/Session (\\d+)/);
    const sessionNumberA = parseInt(matchA ? matchA[1] : 0, 10);
    const sessionNumberB = parseInt(matchB ? matchB[1] : 0, 10);
    return sessionNumberB - sessionNumberA;
});


  tableData = tableData.slice(0, 5);

  if (tableData.length > 0) {
    dv.table(["Note", "Takeaway"], tableData);
  } else {
    dv.paragraph("No matching pages or takeaways found.");
  }
})();

${fence}
`;

const baseTemplatePath = "Extras/Templates/base_sessiontemplate.md";
const baseFile = app.vault.getAbstractFileByPath(baseTemplatePath);
if (!baseFile) {
  new Notice("Base session template not found!");
  return;
}
let baseTemplateContent = await app.vault.read(baseFile);
let finalTemplate = baseTemplateContent
  .replace(/\$\{adventureName\}/g, adventureName)
  .replace(/ADVENTURE FOLDER/g, adventureName)
  .replace(/SESSION NOTES FOLDER/g, "Session Notes")
  .replace(/\(exampleadventure\)/g, `(${adventureName})`);
if (!sessionTemplatePath.endsWith(".md")) sessionTemplatePath += ".md";
await app.vault.create(sessionTemplatePath, finalTemplate);
new Notice(`Session template created!`);

let summaryPath = `Extras/SYS/${adventureName}_summary.md`;
await app.vault.create(summaryPath, "");
new Notice(`Summary file: ${summaryPath}`);

await tp.file.create_new(noteContent, newFilePath);
await app.workspace.openLinkText(newFilePath, "", true);
new Notice(`Adventure “${adventureName}” & session template done!`);
%>
