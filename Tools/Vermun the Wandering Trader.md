
# Vermun the Wandering Trader

Vermun is a trader from another dimension. While she appears human, she is clearly something different. 

She has many items, but only ever 5 at a time. This is due to her stock being held in a pocket dimension. Whenever someone wants to trade, she pulls out her special box of holding and opens it up to the dimension with the items. She doesn't even know what will appear when she opens it.

- When the party speaks to Vermun, one player rolls a d4. The result determines how many items the entire group can closely inspect together before making a purchase. 
- Items are inspected one at a time. For each item, Vermun provides a brief description and the price (and a visual description is provided to the players). 
- If they would like, a player can roll a 15 arcana check to know the name and the details. This uses up one of the checks from the d4 roll at the start.
- After inspecting an item, the group must decide whether to buy it or pass. If they choose to pass, the item goes back into Vermun’s box and is no longer available for purchase during that session. If they use up all their inspections and have not purchased an item, they must either leave without buying or make a blind purchase from the remaining items.
- Each person can only possess one item bought from the store at a time.
- Must roll a 10 Arcana to know more about something you have already learned about

Any items previously purchased form Vermun can also be returned for the price they were bought, but only for store credit.

---
```meta-bind-button
label: NEW ITEM!!
icon: ""
style: primary
class: ""
cssStyle: ""
backgroundImage: ""
tooltip: ""
id: ""
hidden: false
actions:
  - type: runTemplaterFile
    templateFile: Extras/Templates/newitem_template.md

```
```dataviewjs
// Define the base folder and subfolder paths
const baseFolder = "Compendium/Items"; // The base folder
const homebrewSubfolder = "Homebrew Items"; // Name of the Homebrew subfolder

// Add a button to trigger the random selection
const button = document.createElement("button");
button.textContent = "VERMUN IT!";
button.style.marginTop = "20px";
button.style.marginBottom = "20px";
button.addEventListener("click", () => populateRandomTable());
dv.container.appendChild(button);

// Create a container for the table
const tableContainer = dv.container.appendChild(document.createElement("div"));

/**
 * Populates the table with 5 random notes from the folder,
 * prioritizing unseen notes and applying filtering rules for 'vermun' and subfolder conditions.
 */
function populateRandomTable() {
    // Clear the previous table
    tableContainer.innerHTML = "";

    // Fetch all notes from the base folder
    const notes = dv.pages(`"${baseFolder}"`)
        .filter((p) => shouldInclude(p)) // Apply filtering logic
        .array();

    if (notes.length === 0) {
        tableContainer.textContent = "No eligible notes available!";
        return;
    }

    // Separate unseen and seen notes
    const unseenNotes = notes.filter((note) => !note.seen);
    const seenNotes = notes.filter((note) => note.seen);

    // Select up to 5 notes, prioritizing unseen notes
    const prioritizedNotes = [
        ...getRandomItems(unseenNotes, Math.min(5, unseenNotes.length)),
        ...getRandomItems(seenNotes, Math.max(0, 5 - unseenNotes.length))
    ].slice(0, 5); // Ensure no more than 5 notes total

    // Create the table
    const table = document.createElement("table");
    table.style.width = "100%";
    table.style.borderCollapse = "collapse";
    table.style.border = "1px solid black";

    // Table header
    const header = table.insertRow();
    ["File Name", "Price", "Seen", "Owned"].forEach((headerText) => {
        const th = document.createElement("th");
        th.textContent = headerText;
        th.style.border = "1px solid black";
        th.style.padding = "5px";
        th.style.textAlign = "left";
        header.appendChild(th);
    });

    // Table rows
    prioritizedNotes.forEach((note) => {
        const row = table.insertRow();

        // File name (linked with tooltip preview)
        const fileCell = row.insertCell();
        fileCell.style.border = "1px solid black";
        fileCell.style.padding = "5px";

        const fileLink = document.createElement("a");
        fileLink.classList.add("internal-link"); // Required for Obsidian hover preview
        fileLink.setAttribute("data-href", note.file.path); // Tooltip target
        fileLink.textContent = note.file.name; // Use file name, not path
        fileCell.appendChild(fileLink);

        // Price
        const priceCell = row.insertCell();
        priceCell.style.border = "1px solid black";
        priceCell.style.padding = "5px";
        priceCell.textContent = note.price || "N/A";

        // Seen checkbox (editable)
        const seenCell = row.insertCell();
        seenCell.style.border = "1px solid black";
        seenCell.style.padding = "5px";

        const seenCheckbox = document.createElement("input");
        seenCheckbox.type = "checkbox";
        seenCheckbox.checked = note.seen || false;

        // Update the note when the "seen" checkbox is toggled
        seenCheckbox.addEventListener("change", async (e) => {
            const newValue = e.target.checked;
            await updateFrontmatter(note.file.path, "seen", newValue);
        });

        seenCell.appendChild(seenCheckbox);

        // Owned checkbox (editable)
        const ownedCell = row.insertCell();
        ownedCell.style.border = "1px solid black";
        ownedCell.style.padding = "5px";

        const ownedCheckbox = document.createElement("input");
        ownedCheckbox.type = "checkbox";
        ownedCheckbox.checked = note.owned || false;

        // Update the note when the "owned" checkbox is toggled
        ownedCheckbox.addEventListener("change", async (e) => {
            const newValue = e.target.checked;
            await updateFrontmatter(note.file.path, "owned", newValue);
        });

        ownedCell.appendChild(ownedCheckbox);
    });

    // Add the table to the container
    tableContainer.appendChild(table);
}

/**
 * Updates a specific frontmatter field in a note.
 * @param {string} filePath - Path to the file in Obsidian.
 * @param {string} field - Name of the field to update.
 * @param {string|boolean} value - New value for the field.
 */
async function updateFrontmatter(filePath, field, value) {
    const file = app.vault.getAbstractFileByPath(filePath);

    if (file) {
        const content = await app.vault.read(file);
        const updatedContent = content.replace(
            new RegExp(`(${field}:\\s*)(true|false)`), // Match the field and its value
            `$1${value}` // Replace with the new value
        );
        await app.vault.modify(file, updatedContent);
    }
}

/** * Determines if a note should be included based solely on the 'vermun' frontmatter. * @param {Object} note - The note to check. * @returns {boolean} - True if 'vermun' is true, false otherwise. */ function shouldInclude(note) { return note.vermun === true; }

/**
 * Returns a random selection of items from an array.
 * @param {Array} array - The array to select from.
 * @param {number} count - The number of items to select.
 * @returns {Array} - Randomly selected items.
 */
function getRandomItems(array, count) {
    const shuffled = array.sort(() => 0.5 - Math.random());
    return shuffled.slice(0, count);
}

```
```dataviewjs
// Get pages from both folders separately
let homebrew = dv.pages('"Compendium/Items/Homebrew Items"');
let magical = dv.pages('"Compendium/Items/Magical Items"');

// Combine the pages and filter for notes with vermun checked and not seen
let pages = homebrew.concat(magical).where(page => page.vermun === true && !page.seen);

// Output a bold line with the count
dv.el("strong", "Unseen Items: " + pages.length);


```

## Credit
```dataviewjs
// Define the folder to search for notes
const folder = "World/People/Player Characters/Active"; // Replace with your folder path

// Target the 'vcredit' field
const fieldName = "vcredit";

// Create a table
dv.table(["File Name", "VCredit"], 
    dv.pages(`"${folder}"`)
    .where(p => p[fieldName] !== undefined) // Ensure the 'vcredit' field exists
    .map(p => [
        p.file.link, 
        createEditableField(p.file.path, fieldName, p[fieldName])
    ])
);

/**
 * Creates an editable field for a specific note field.
 * @param {string} filePath - Path to the file in Obsidian.
 * @param {string} field - Name of the field to edit.
 * @param {string|number} value - Current value of the field.
 * @returns {HTMLElement} - HTML element for the editable field.
 */
function createEditableField(filePath, field, value) {
    const input = document.createElement("input");
    input.type = "number"; // Use "text" for non-numeric fields
    input.value = value;
    input.style.width = "100px";

    // Update the note on input change
    input.addEventListener("change", async (e) => {
        const newValue = e.target.value;
        await app.vault.modify(
            await app.vault.getAbstractFileByPath(filePath), 
            (await app.vault.read(await app.vault.getAbstractFileByPath(filePath))).replace(
                new RegExp(`(${field}:\\s*)(\\d+|\\d+\\.\\d+|".*")`), 
                `$1${newValue}`
            )
        );
    });

    return input;
}

```
## Owned Items
```dataviewjs
// Define the folder to search for items
const folder = "World/Compendium/Items"; // Replace with your items folder path

// Define the folder to search for player character notes
const charactersFolder = "People/Player Characters/Active"; // Replace with your player characters folder

// Fetch all player character notes
const characters = dv.pages(`"${charactersFolder}"`).map((p) => ({
    name: p.file.name,
    keyItems: (p.key_items || []).map((item) => extractFileName(item))
}));

// Create the table
dv.table(
    ["File Name", "Price", "Owned", "Owner"],
    dv.pages(`"${folder}"`)
        .where(p => p.owned === true && p.vermun === true) // Only include items with "owned: true" and "vermun: true"
        .map(p => {
            // Find the owner of the current item
            const owner = characters.find((char) =>
                char.keyItems.includes(p.file.name)
            )?.name || "Unassigned"; // Default to "Unassigned" if no owner found

            return [
                p.file.link, // File name as a link
                p.price || "N/A", // Price (default to "N/A" if not present)
                createEditableCheckbox(p.file.path, "owned", p.owned), // Editable "owned" checkbox
                owner // Owner name
            ];
        })
);

/**
 * Extracts the file name from a link in the format [[path/to/file|Displayed Name]].
 * @param {string} link - The link string to extract the displayed name from.
 * @returns {string} - The extracted displayed name (e.g., "ITEM").
 */
function extractFileName(link) {
    const match = /\[\[.*\|(.*?)\]\]/.exec(link); // Extract the text after the "|"
    return match ? match[1] : link; // Return the name or the original link if no match
}

/**
 * Creates an editable checkbox for a specific note field.
 * @param {string} filePath - Path to the file in Obsidian.
 * @param {string} field - Name of the field to edit.
 * @param {boolean} value - Current value of the checkbox.
 * @returns {HTMLElement} - HTML element for the editable checkbox.
 */
function createEditableCheckbox(filePath, field, value) {
    const checkbox = document.createElement("input");
    checkbox.type = "checkbox";
    checkbox.checked = value;

    // Update the note on checkbox change
    checkbox.addEventListener("change", async (e) => {
        const newValue = e.target.checked;
        const file = app.vault.getAbstractFileByPath(filePath);

        if (file) {
            const content = await app.vault.read(file);
            const updatedContent = content.replace(
                new RegExp(`(${field}:\\s*)(true|false)`), // Match the field and its value
                `$1${newValue}` // Replace with the new value
            );
            await app.vault.modify(file, updatedContent);
        }
    });

    return checkbox;
}

```

