---
Doc Lock: false
Adventure: ${adventureName}
Location: 
session:
session_name:
session_date: 
day: 
month: 
year:
---
# 🏔 `= this.Adventure + " - Session " + this.session`

**`= this.location`***
```dataviewjs 


const dayField = dv.current().day;
const monthField = dv.current().month;
const yearField = dv.current().year;

if (!dayField || !monthField || !yearField) {
  dv.el("em", "In-game date not set yet.");
} else {

  const days = Array.isArray(dayField) ? dayField : [dayField];
  const months = Array.isArray(monthField) ? monthField : [monthField];
  const years = Array.isArray(yearField) ? yearField : [yearField];


  const finalYear = years.length > 0 ? years[0] : "????";

  function getDaySuffix(d) {
    if (d % 100 >= 11 && d % 100 <= 13) return d + "th";
    switch (d % 10) {
      case 1: return d + "st";
      case 2: return d + "nd";
      case 3: return d + "rd";
      default: return d + "th";
    }
  }

  function formatOneDayMonth(dayNum, monthStr) {
    return `${getDaySuffix(dayNum)} day of ${monthStr}`;
  }

  function joinWithAnd(list) {
    if (list.length === 0) return "";
    if (list.length === 1) return list[0];
    if (list.length === 2) return list.join(" and ");
    // For 3 or more, do commas plus final "and"
    return list.slice(0, -1).join(", ") + ", and " + list[list.length - 1];
  }

  let daysAndMonthsStr = "";

  if (days.length === months.length && days.length > 1) {
    const pairs = [];
    for (let i = 0; i < days.length; i++) {
      pairs.push(formatOneDayMonth(days[i], months[i]));
    }
    daysAndMonthsStr = joinWithAnd(pairs);

  } else if (days.length > 1 && months.length === 1) {
    const dayStrings = days.map(d => getDaySuffix(d));
    const joinedDays = joinWithAnd(dayStrings);
    daysAndMonthsStr = `${joinedDays} day${days.length > 1 ? "s" : ""} of ${months[0]}`;

  } else if (days.length === 1 && months.length > 1) {
    const pieces = months.map(m => formatOneDayMonth(days[0], m));
    daysAndMonthsStr = joinWithAnd(pieces);

  } else if (days.length === 1 && months.length === 1) {
    daysAndMonthsStr = formatOneDayMonth(days[0], months[0]);

  } else {
    daysAndMonthsStr = "Mixed or mismatched days/months (update the code logic).";
  }

  const finalString = `${daysAndMonthsStr}, ${finalYear}`;

  const container = dv.el("p", "");
  container.innerHTML = `<em>${finalString}</em>`;
}

```
---
`BUTTON[previous,current,next,new]`
```meta-bind-button
label: NEW SESSION!
icon: ""
hidden: true
class: ""
tooltip: Create a new session note
id: new
style: primary
actions:
  - type: templaterCreateNote
    templateFile: "Extras/Templates/(${adventureName})sessiontemplate.md"
    folderPath: "Adventures/${adventureName}/Session Notes"
    fileName: ""
    openNote: true
```
---
```meta-bind-button
label: Previous Session
icon: ""
hidden: true
class: meta-bind-button
tooltip: "Go to the previous session note"
id: previous
style: default
actions:
  - type: inlineJS
    code: |
      const folderPath = `Adventures/${adventureName}/Session Notes`;
      const currentTitle = app.workspace.activeLeaf.view.file.basename;
      const filePattern = /Session (\d+)/;
      const match = currentTitle.match(filePattern);

      if (match && match[1]) {
        const sessionNumber = parseInt(match[1]) - 1;
        const previousSessionTitle = `Session ${sessionNumber}`;
        
        const files = app.vault.getAbstractFileByPath(folderPath)?.children || [];
        const previousNote = files.find(file => file.basename.startsWith(previousSessionTitle));

        if (previousNote) {
          app.workspace.activeLeaf.openFile(previousNote);
        } else {
          alert("Previous session does not exist.");
        }
      }
```
```meta-bind-button
label: Next Session
icon: ""
hidden: true
class: meta-bind-button
tooltip: "Go to the next session note"
id: next
style: default
actions:
  - type: inlineJS
    code: |
      const folderPath = `Adventures/${adventureName}/Session Notes`;
      const currentTitle = app.workspace.activeLeaf.view.file.basename;
      const filePattern = /Session (\d+)/;
      const match = currentTitle.match(filePattern);

      if (match && match[1]) {
        const sessionNumber = parseInt(match[1]) + 1;
        const nextSessionTitle = `Session ${sessionNumber}`;
        
        const files = app.vault.getAbstractFileByPath(folderPath)?.children || [];
        const nextNote = files.find(file => file.basename.startsWith(nextSessionTitle));

        if (nextNote) {
          app.workspace.activeLeaf.openFile(nextNote);
        } else {
          alert("Next session does not exist.");
        }
      }
```
```meta-bind-button
label: Current Session
icon: ""
hidden: true
class: meta-bind-button
tooltip: "Go to the most recent session note"
id: current
style: destructive
actions:
  - type: inlineJS
    code: |
      const folderPath = `Adventures/${adventureName}/Session Notes`;
      
      // Get all notes in the folder and filter for those matching "Session X" format
      const files = app.vault.getAbstractFileByPath(folderPath)?.children || [];
      const sessionFiles = files
        .filter(file => file.basename.match(/Session \d+/))
        .sort((a, b) => {
          // Sort by session number in descending order
          const aNum = parseInt(a.basename.match(/Session (\d+)/)[1]);
          const bNum = parseInt(b.basename.match(/Session (\d+)/)[1]);
          return bNum - aNum;
        });
      
      if (sessionFiles.length > 0) {
        // Open the most recent session
        app.workspace.activeLeaf.openFile(sessionFiles[0]);
      } else {
        alert("No sessions found.");
      }
```
#### [[${adventureName}_summary|Last time on...]]

>[!abstract]- Previously on...
>
## 🛠️ Prep
---


---
## 📝 Session Notes



---
### ☑️ Summary

##### Takeaways

##### Next Session

<%*
// ------------------------
// Session Template with Modal Form Date Entry
// ------------------------
const __createdFile = app.workspace.getActiveFile();
const __createdPath = __createdFile?.path || "";
const __createdCTime = __createdFile?.stat?.ctime || Date.now();

// Cleanup ONLY this just-created note, and ONLY if it still looks like an "Untitled" stub
async function __cleanupThisNewNoteOnly(maxAgeMs = 60 * 1000) {
  try {
    if (!__createdPath) return;
    const f = app.vault.getAbstractFileByPath(__createdPath);
    if (!f) return;

    const ageMs = Date.now() - __createdCTime;
    const base = f.name || "";
    const looksUntitled = /^Untitled(?: \d+)?\.md$/i.test(base);

    if (ageMs <= maxAgeMs && looksUntitled) {
      await app.vault.delete(f);
    }
  } catch (_) { }
}

async function cleanupIfNewSessionNote(targetFolderPath, maxAgeMs = 5 * 60 * 1000) {
  try {
    const cur = app.workspace.getActiveFile();
    if (!cur) return;
    const ageMs = Date.now() - cur.stat.ctime;
    if (ageMs <= maxAgeMs && cur.path.startsWith(`${targetFolderPath}/`)) {
      await app.vault.delete(cur);
    }
  } catch (_) { }
}

// Recursively find the first non-empty string for any of the given keys
function __pickStringDeep(obj, keys) {
  if (!obj || typeof obj !== "object") return "";
  for (const k of keys) {
    const v = obj[k];
    if (typeof v === "string" && v.trim()) return v.trim();
  }
  for (const v of Object.values(obj)) {
    if (v && typeof v === "object") {
      const found = __pickStringDeep(v, keys);
      if (found) return found;
    }
  }
  return "";
}

function __pickDateDeep(obj) {
  if (!obj || typeof obj !== "object") return null;
  if (obj instanceof Date) return obj;
  for (const [k, v] of Object.entries(obj)) {
    if (k === "date") return v;
  }
  for (const v of Object.values(obj)) {
    if (v && typeof v === "object") {
      const found = __pickDateDeep(v);
      if (found != null) return found;
    }
  }
  return null;
}

async function __getSessionTitlePreference() {
  const PID = "vvunderlore-toolkit-plugin";
  // Try live plugin instance
  try {
    const inst = app.plugins.getPlugin?.(PID) || app.plugins.plugins?.[PID];
    const v = inst?.settings?.sessionTitlePreference;
    if (v === "name" || v === "date") return v;
    // Legacy boolean fallback
    const legacy = inst?.settings?.preferNameWhenBoth;
    if (typeof legacy === "boolean") return legacy ? "name" : "date";
  } catch (_) {}

  // Fallback: read data.json on disk
  try {
    const txt = await app.vault.adapter.read(`.obsidian/plugins/${PID}/data.json`);
    const j = JSON.parse(txt);
    const v2 = j?.sessionTitlePreference ?? j?.settings?.sessionTitlePreference;
    if (v2 === "name" || v2 === "date") return v2;
    const legacy2 = j?.preferNameWhenBoth ?? j?.settings?.preferNameWhenBoth;
    if (typeof legacy2 === "boolean") return legacy2 ? "name" : "date";
  } catch (_) {}

  // Safe default
  return "name";
}

const folderPath = `Adventures/${adventureName}/Session Notes`;
const folder = app.vault.getAbstractFileByPath(folderPath);
const files = folder ? folder.children.filter(f => f.extension === "md") : [];

// Flexible parser for dates (supports M.D.YY / M.D.YYYY / ISO)
function __parseDateFlexible(s) {
  if (!s) return null;
  if (s instanceof Date) return s;
  s = String(s).trim();
  if (!s) return null;

  // ISO yyyy-mm-dd
  if (/^\d{4}-\d{2}-\d{2}$/.test(s)) {
    const [y, m, d] = s.split("-").map(n => parseInt(n, 10));
    return new Date(y, m - 1, d);
  }
  // M.D.YY or M.D.YYYY (also / or -)
  const m = s.match(/^(\d{1,2})[./-](\d{1,2})[./-](\d{2,4})$/);
  if (m) {
    let y = parseInt(m[3], 10);
    if (y < 100) y = 2000 + y;
    return new Date(y, parseInt(m[1], 10) - 1, parseInt(m[2], 10));
  }
  return null;
}

let maxSessionNumber = 0;
let latestSessionDate = null;

// Count ANY “Session N - …” title for numbering,
// and find the newest date using frontmatter (preferred) or filename fallback.
for (const file of files) {
  const base = file.basename; // no .md
  const mNum = base.match(/^Session\s+(\d+)\b/i);
  if (mNum) {
    const n = parseInt(mNum[1], 10);
    if (n > maxSessionNumber) maxSessionNumber = n;
  }

  // Prefer a saved date from frontmatter
  const fm = app.metadataCache.getFileCache(file)?.frontmatter || {};
  let dateStr = fm?.session_date || fm?.date || "";

  // Fallback: parse a trailing date in the filename
  if (!dateStr) {
    const mDate = base.match(/(\d{1,2})[./-](\d{1,2})[./-](\d{2,4})$/);
    if (mDate) dateStr = `${mDate[1]}.${mDate[2]}.${mDate[3]}`;
  }

  const d = __parseDateFlexible(dateStr);
  if (d && (!latestSessionDate || d > latestSessionDate)) latestSessionDate = d;
}

// Suggested date = latest non-empty session date + 7 days, otherwise today + 7
let suggestedDate = "";
if (latestSessionDate) {
  const nextDate = new Date(latestSessionDate);
  nextDate.setDate(nextDate.getDate() + 7);
  suggestedDate = `${nextDate.getMonth() + 1}.${nextDate.getDate()}.${String(nextDate.getFullYear()).slice(-2)}`;
} else {
  const today = new Date();
  today.setDate(today.getDate() + 7);
  suggestedDate = `${today.getMonth() + 1}.${today.getDate()}.${String(today.getFullYear()).slice(-2)}`;
}

// Next session number is now correct even if some titles use names
const nextSessionNumber = maxSessionNumber + 1;


// 4. Open the modal form for date entry (ID "next-session-date" with field "date")
const modalForm = app.plugins.plugins.modalforms.api;
const sessionResult = await modalForm.openForm("next-session-date");
const cancelled =
  !sessionResult ||
  (typeof sessionResult === "object" && Object.keys(sessionResult).length === 0) ||
  sessionResult.cancelled === true || sessionResult.canceled === true;

if (cancelled) {
  await __cleanupThisNewNoteOnly();
  new Notice("New session cancelled.");
  return;
}


// Accept either date or sessionName (or both). Only cancel if both are blank.
let sessionName = __pickStringDeep(sessionResult, [
  "sessionName", "session_name", "name", "title", "label", "session_label"
]);

// Normalize date from the form (accept Date, number epoch, or string; search nested)
let sessionDate = "";
let rawDate = ("date" in (sessionResult || {})) ? sessionResult.date : __pickDateDeep(sessionResult);

if (rawDate instanceof Date) {
  const y = rawDate.getFullYear(), m = rawDate.getMonth() + 1, d = rawDate.getDate();
  sessionDate = `${m}.${d}.${String(y).slice(-2)}`;
} else if (typeof rawDate === "number" && isFinite(rawDate)) {
  const d = new Date(rawDate);
  const y = d.getFullYear(), m = d.getMonth() + 1, dd = d.getDate();
  sessionDate = `${m}.${dd}.${String(y).slice(-2)}`;
} else if (typeof rawDate === "string") {
  const s = rawDate.trim();
  if (s) {
    if (/^\d{4}-\d{2}-\d{2}$/.test(s)) {
      const [yy, mm, dd] = s.split("-").map(n => parseInt(n, 10));
      const d = new Date(yy, mm - 1, dd);
      sessionDate = `${d.getMonth() + 1}.${d.getDate()}.${String(d.getFullYear()).slice(-2)}`;
    } else if (/^\d{1,2}[./-]\d{1,2}[./-]\d{2,4}$/.test(s)) {
      const parts = s.split(/[./-]/).map(n => parseInt(n, 10));
      let [m, d, y] = parts; if (y < 100) y = 2000 + y;
      sessionDate = `${m}.${d}.${String(y).slice(-2)}`;
    }
  }
}

const haveName = !!sessionName;
const haveDate = !!sessionDate;

if (!haveName && !haveDate) {
  await __cleanupThisNewNoteOnly();
  new Notice("No date or name provided—session creation cancelled.");
  return;
}

// Sanitize for filenames
if (haveName) sessionName = sessionName.replace(/[\\/:*?"<>|]/g, "").trim();

// Prefer Name or Date when both provided, based on plugin setting
const pref = await __getSessionTitlePreference();

let chosenLabel = "";
if (haveName && haveDate) {
  chosenLabel = (pref === "name") ? sessionName : sessionDate;
} else if (haveName) {
  chosenLabel = sessionName;
} else {
  chosenLabel = sessionDate;
}

if (!chosenLabel || !nextSessionNumber) {
  await __cleanupThisNewNoteOnly();
  return;
}



const newTitle = `Session ${nextSessionNumber} - ${chosenLabel}`;
await tp.file.rename(newTitle);

tp.hooks.on_all_templates_executed(async () => {
  const file = app.workspace.getActiveFile();
  if (!file) {
    new Notice("No active file found.");
    return;
  }

  await app.fileManager.processFrontMatter(file, (frontmatter) => {
    frontmatter["session"] = nextSessionNumber;
    frontmatter["session_date"] = sessionDate;
    frontmatter["session_name"] = sessionName;
  });

  // --- Begin [[Extraction]] and Insertion Code ---
  let previousNote = null;
  let previousNoteContent = "";

  if (nextSessionNumber > 1) {
    const previousSessionNumber = nextSessionNumber - 1;
    const previousNotePattern = new RegExp(`^Session ${previousSessionNumber} - `);
    previousNote = files.find(f => previousNotePattern.test(f.name));

    if (previousNote) {
      previousNoteContent = await app.vault.read(previousNote);

      // Get previous note's frontmatter (day, month, year) if available.
      const prevFrontmatter = app.metadataCache.getFileCache(previousNote)?.frontmatter || {};
      const prevDay = prevFrontmatter.day || "";
      const prevMonth = prevFrontmatter.month || "";
      const prevYear = prevFrontmatter.year || "";

      await app.fileManager.processFrontMatter(file, (frontmatter) => {
        frontmatter.day = prevDay;
        frontmatter.month = prevMonth;
        frontmatter.year = prevYear;
      });
    } else {
      new Notice("Previous session note not found.");
    }
  } else {
    console.log("No previous session exists.");
  }
  // Extract summary from the previous note and update the "Summary Temp" note.
  const summaryRegex = /### ☑️ Summary\s*\n([\s\S]*?)(?=\n##### Takeaways)/;
  const summaryMatch = previousNoteContent.match(summaryRegex);
  if (summaryMatch) {
    const summaryText = summaryMatch[1].trim();
    const newSummaryTempContent = summaryText;
    const summaryTempPath = "Extras/SYS/${adventureName}_summary.md";
    const summaryTempFile = app.vault.getAbstractFileByPath(summaryTempPath);
    if (summaryTempFile) {
      await app.vault.modify(summaryTempFile, newSummaryTempContent);
    } else {
      new Notice("Summary Temp note not found!");
    }
  } else {
    new Notice("Could not find summary text between '### ☑️ Summary' and '##### Takeaways' in the previous note.");
  }

  await new Promise(r => setTimeout(r, 250));
}); // <-- Ensure this closing bracket matches `tp.hooks.on_all_templates_executed`

%>
